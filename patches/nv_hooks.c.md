# `nv_hooks.c` 深度逆向分析

## 1. 总览

`patches/nv_hooks.c` 不是一份静态 `blob diff`，而是一个会在驱动运行时直接改写 `nv-kernel.o_binary` 代码页的**运行时补丁注入器**。

它的作用可以分成两层：

1. **运行时 hook 注入层**
   - 对 `nv-kernel.o_binary` 中若干稳定指令模板做原地重写。
   - 重写模板不是普通 `old -> new`，而是：
     - 校验原始字节模板；
     - 将末尾 5 字节改成 `CALL rel32`；
     - 将前导字节改成 `NOP`；
     - 在 `vup_hook_*_naked()` 里回放被覆盖原语义。

2. **直接字节 patch 层**
   - 对 `nv-kernel.o_binary` 中一组固定 offset 做 `oldval -> newval` 原地替换。
   - 这部分更接近 `blob-*.diff`，但仍然是在驱动加载后的内存映像上生效，而不是离线改文件。

当前 IDA 打开的目标二进制为：

- `NVIDIA-Linux-x86_64-550.90.05-vgpu-kvm/kernel/nvidia/nv-kernel.o_binary`
- sha256：`0884124c5e623e8c71779b20cf196176ba9dccc19024936b9fc5e23517f52d14`

这与 `mcp__ida-mcp__get_metadata` 返回值一致，因此本文默认当前 IDA 会话就是 `nv_hooks.c` 作用的目标 blob。

从编排流程看，`nv_hooks.c` 是和静态 `blob-*.diff` 并行叠加的，而不是谁替代谁：

- `patch.sh:675-689`

```sh
675 echo "integrating runtime nv blob hooks"
676 mkdir -p ${TARGET}/kernel/unlock
677 $CP "$BASEDIR/patches/nv_hooks.c" ${TARGET}/kernel/unlock
678 echo 'NVIDIA_SOURCES += unlock/nv_hooks.c' >> ${TARGET}/kernel/nvidia/nvidia-sources.Kbuild
679 echo 'OBJECT_FILES_NON_STANDARD_nv_hooks.o := y' >> ${TARGET}/kernel/nvidia/nvidia.Kbuild
...
683 if [ -e patches/blob-${VER_BLOB}.diff ]; then
684     blobpatch ${TARGET}/kernel/nvidia/nv-kernel.o_binary patches/blob-${VER_BLOB}.diff || exit 1
685 fi
689 applypatch ${TARGET} setup-vup-hooks.patch
```

因此，分析 `nv_hooks.c` 时必须同时回答三件事：

- 它**额外**注入了哪些运行时行为；
- 哪些业务链与 `blob-550.90.05.diff` 的静态 patch 重叠；
- 在重叠区域里，二者分别改了控制流的哪一层。

---

## 2. 证据基础与坐标系

### 2.1 两套核心数据结构

`nv_hooks.c` 内部天然分成两套机制：

- `patches/nv_hooks.c:33-42`

```c
struct vup_hook_info {
	void (*func)(void);
	const struct kernel_param *param;
	u32 offset;
	int ovgpu;
	s16 pbytes[14];
};
```

- `patches/nv_hooks.c:45-69`

```c
struct vup_patch_item {
	u32 offset;
	u8 oldval;
	u8 newval;
};

struct vup_patch_info {
	const struct kernel_param *param;
	struct vup_patch_item *items;
	int count;
	int enabv;
	int ovgpu;
};
```

前者描述 **hook 注入项**，后者描述 **直接 patch 项**。

### 2.2 目标 blob 基址

`nv_hooks.c` 并不是离线 patch 某个文件，而是通过 `rm_ioctl` 反推出运行时 blob 基址：

- `patches/nv_hooks.c:232-233`
- `patches/nv_hooks.c:455-470`

```c
#define RM_IOCTL_OFFSET 0xae2680

void vup_hooks_init(void)
{
	u8 *blob = (u8 *)rm_ioctl - RM_IOCTL_OFFSET;
	...
	vup_inject_hooks(blob);
	...
	vup_apply_patches(blob - 0x40);
}
```

这段代码直接给出两套坐标系：

1. `vup_hooks[]`
   - 基于 `blob`
   - 即：`hook offset` 直接相对 `nv-kernel.o_binary` 的 blob 基址

2. `vup_patches[]`
   - 基于 `blob - 0x40`
   - 即：`patch offset` 使用的是和 `blob-*.diff` 一致的 diff / 文件偏移风格

### 2.3 运行时写保护绕过

`nv_hooks.c` 在 patch 前后会显式关写保护、关 CET，再恢复：

- `patches/nv_hooks.c:417-453`

```c
static int vup_patching_start(void)
{
	...
	preempt_disable();
	...
	if (test_bit(X86_CR4_CET_BIT, &cr4)) {
		...
		vup_set_cr4(cr4);
	}
	cr0 = read_cr0();
	clear_bit(16, &cr0);
	vup_set_cr0(cr0);
	...
}

static void vup_patching_done(void)
{
	...
	set_bit(16, &cr0);
	vup_set_cr0(cr0);
	...
	preempt_enable_no_resched();
}
```

这说明 `nv_hooks.c` 的本质不是“配置开关”，而是**在模块初始化时直接改写内核地址空间中的 RM 代码页**。

---

## 3. 运行时 hook 注入机制

### 3.1 注入模板

`vup_inject_hooks()` 的逻辑非常清楚：

- `patches/nv_hooks.c:325-364`

```c
for (j = 0; hi->pbytes[j] >= 0; j++)
	if (blob[hi->offset + j] != hi->pbytes[j])
		break;
...
j -= 5;
blob[hi->offset + j] = 0xe8;
*(u32 *)(&blob[hi->offset + j + 1]) =
	(u8 *)hi->func - &blob[hi->offset + j + 5];
for (j--; j >= 0; j--)
	blob[hi->offset + j] = 0x90;
```

也就是：

1. 先校验 `pbytes[]` 原始模板。
2. 若模板匹配，设模板长度为 `n`。
3. 将 `offset + n - 5` 处改成 `E8 <rel32>`。
4. 将前面的 `n - 5` 字节全部 NOP。

因此 hook 的**字节 / 插桩层等效 patch**统一可写成：

- 原始模板：`pbytes[0..n-1]`
- 改写结果：`NOP*(n-5) + CALL rel32(vup_hook_*_naked)`

### 3.2 三个 hook 的共性

三处 `*_naked` hook 都遵循同一个结构：

1. 保存现场寄存器；
2. 调用普通 C helper；
3. 恢复寄存器；
4. 手工回放被覆盖原指令；
5. `ret` 回到原控制流后续。

这意味着：

- 它们不是简单的“跳走不回来”；
- 而是在原控制流中插入一个**最小侵入的旁路逻辑**。

---

## 4. `vup_hooks[]` 逐项分析

## 4.1 `vupdevid`

### A. 基本信息

- 定义：`patches/nv_hooks.c:115-155`
- 表项：`patches/nv_hooks.c:221-225`
- offset：`0x0051E3A7`
- 原始模板：`4C 89 E0 44 89 FB`
- 模板长度：6 字节

### B. IDA 落点与汇编

当前 IDA 落点：

- 函数：`_nv026445rm`
- 范围：`0x51E330 - 0x51E540`
- hook 点附近原始汇编：

```asm
0x0051E3A7  mov rax, r12
0x0051E3AA  mov ebx, r15d
```

`vup_hook_vupdevid_naked()` 会回放这两条指令：

- `patches/nv_hooks.c:137-150`

```asm
mov    %r15, %rdi
mov   0xaa8(%r14), %esi
call  vup_hook_vupdevid
...
mov    %r12, %rax
mov    %r15d, %ebx
ret
```

因此它的**字节 / 插桩层等效 patch**是：

```text
offset=0x0051E3A7
原始模板 = 4C 89 E0 44 89 FB
改写结果 = 90 E8 <rel32-to-vup_hook_vupdevid_naked>
```

也就是：

- 第 1 字节 NOP；
- 后 5 字节改成 `CALL rel32`；
- 被覆盖的 `mov rax,r12` 与 `mov ebx,r15d` 在 naked hook 末尾回放。

### C. helper 语义

- `patches/nv_hooks.c:118-124`

```c
static u32 vup_hook_vupdevid(u32 devid, u32 subdevid)
{
	printk(KERN_INFO "nvidia: vup_hook_vupdevid 10de:%04x %04x:%04x\n",
	       devid, subdevid & 0xffff, subdevid >> 16);
	return vup_vupdevid;
}
```

它的业务语义非常直接：

- 读取当前路径上的 `device id` 与 `subdevice id`；
- 若模块参数 `vupdevid` 非 0，则用该值覆写 `r15d`；
- 同时把 `r13d` 置 1，确保后续路径把“这个值可用”视为真。

### D. 源码映射

这一项当前**还不能高置信一一收敛到公开源码中的单函数**，但现在可以更精确地分成两层候选：

1. **更接近的设备名 / 设备信息查表簇**
   - `drivers/resman/src/physical/gpu/gpu_name.c`
   - `ChipInfoGetNameAscii`
   - `CustomChipInfoGetNameAscii`
2. **更接近的 pGPU identity / migration 编码簇**
   - `drivers/resman/src/kernel/virtualization/vgpu_mgr.c:1929-1978`
     - `vgpuMgrGetPgpuDevIdEncoding`
     - `vgpuMgrGetPgpuSubdevIdEncoding`
   - `drivers/resman/src/kernel/virtualization/kernel_vgpu_mgr.c:1784-1833`
     - `kvgpumgrGetPgpuDevIdEncoding`
     - `kvgpumgrGetPgpuSubdevIdEncoding`

关键源码片段一：

```c
NvU32 vgpuMgrGetPgpuDevIdEncoding(OBJGPU *pGpu, NvU8 *pgpuString, NvU32 strSize)
{
    NvU32 chipID = DRF_VAL(_PCI, _DEVID, _DEVICE, pGpu->idInfo.PCIDeviceID);
    NvU32 subID  = DRF_VAL(_PCI, _DEVID, _DEVICE, pGpu->idInfo.PCISubDeviceID);
    ...
}
```

关键源码片段二：

```c
static NV_STATUS ChipInfoGetNameAscii(OBJGPU *pGpu, NvU16 devId, NvU16 gpuId, ...)
static NV_STATUS CustomChipInfoGetNameAscii(OBJGPU *pGpu, NvU16 devId, NvU16 gpuId,
                                            NvU16 subSysId, NvU16 subSysVendorId, ...)
```

匹配依据：

- `_nv026445rm` 明确读取 `a1+2722 / a1+2730`，也就是设备 / 子设备标识；
- 它在二进制里扫描 `nv039110rm / byte_C231A2` 这类静态表；
- 这类“按 `devId/subSysId` 查表并构造一个短结果块”的形状，既像设备名 / 设备信息查表，也像 pGPU identity 编码前的身份归一化；
- 它的 caller `_nv026708rm` 明显处于设备初始化链中，因此当前最稳妥的结论仍然是“设备身份处理链”，而不是字符串格式化链本身。

当前置信度：**中等**。

更稳妥的表述是：

- 这条 hook 高置信落在一条“设备身份查表 / pGPU identity 处理”链上；
- 若按业务职责看，更接近 `vgpu_mgr.c / kernel_vgpu_mgr.c` 的 pGPU identity / migration 编码语义；
- 若按二进制“查静态表、输出短结果块”的形状看，则又与 `gpu_name.c` 的 `ChipInfoGetNameAscii / CustomChipInfoGetNameAscii` 这类 helper 有相似性；
- 因此当前还不建议把 `_nv026445rm` 直接写成某一个公开函数的 100% 逐行直译。

### E. 双层等效 patch

**业务层等效 patch：**

- 将原本从 RM 内部对象状态读取出来的 device ID / subdevice ID，改造成“可由模块参数强制覆写的设备身份”。
- 这会影响后续所有依赖 pGPU 身份做判断、编码、兼容分组或 profile 选择的路径。

**字节 / 插桩层等效 patch：**

```text
0x0051E3A7: 4C 89 E0 44 89 FB
=> 90 E8 <rel32-to-vup_hook_vupdevid_naked>
```

### F. patch 前后业务影响

**patch 前：**

- 该路径使用真实的设备 / 子设备标识继续后续逻辑；
- 设备身份是否属于受支持组合，由下游静态表或 capability 路径决定。

**patch 后：**

- 可以直接把某张卡伪装成另一张卡的 `device id`；
- 这会改变后续 profile / 迁移兼容 / 能力白名单等基于身份的行为。

---

## 4.2 `klogtrace`

### A. 基本信息

- 定义：`patches/nv_hooks.c:157-215`
- 表项：`patches/nv_hooks.c:222-226`
- offset：`0x00016185`
- 原始模板：`48 81 ED 40 04 00 00`
- 模板长度：7 字节

### B. IDA 落点与汇编

当前 IDA 落点：

- 函数：`_nv039916rm`
- 范围：`0x16170 - 0x16561`
- hook 点附近原始汇编：

```asm
0x00016180  push rbx
0x00016185  sub  rbp, 0x440
```

`vup_hook_klogtrace_naked()` 末尾回放：

- `patches/nv_hooks.c:196-211`

```asm
call vup_hook_klogtrace
...
sub    $0x440, %rbp
ret
```

因此它的**字节 / 插桩层等效 patch**是：

```text
offset=0x00016185
原始模板 = 48 81 ED 40 04 00 00
改写结果 = 90 90 E8 <rel32-to-vup_hook_klogtrace_naked>
```

### C. helper 语义

- `patches/nv_hooks.c:168-190`

```c
id = rdi & 0xffffff;
pt = (rsi >> 16) & 0xffff;
a1 = (rdi >> 24) & 0xff;
a2 = rsi & 0xffff;
...
printk(KERN_DEBUG "NVTRACE %06x:%04x %04x%02x\n", id, pt, a2, a1);
```

它并不直接解锁任何功能，而是把 RM 内部一条 packet / trace 风格路径的关键字段打印出来。

### D. 源码映射

这一项现在已经可以高置信收敛到公开源码函数：

- `drivers/resman/src/kernel/diagnostics/nvlog.c:1328-1415`
- 函数：`nvlogPrint_vprintf`

关键源码片段：

```c
static NV_STATUS
nvlogPrint_vprintf
(
    NvU32   dbgLevel,
    NvU32   file,
    NvU32   line,
    va_list arguments
)
{
    ...
    buffers     = (file >> 24) & 0xFF;
    file        =  file        & 0xFFFFFF;
    actualLine  = (line >> 16) & 0xFFFF;
    ...
    argList[0] = (line >> 12) & 0xF;
    argList[1] = (line >> 8)  & 0xF;
    argList[2] = (line >> 4)  & 0xF;
    argList[3] =  line        & 0xF;
    ...
    if (NvLogPrintLogger.runtimeSizes[argList[argIndex]] <= sizeof(NvU32))
    {
        data[cursor++] = va_arg(arguments, NvU32);
    }
    else if (NvLogPrintLogger.runtimeSizes[argList[argIndex]] <= sizeof(NvU64))
    {
        ...
    }
}
```

匹配依据：

- `_nv039916rm` 的二进制逻辑同样会把输入字段拆成多个 4-bit nibble；
- 它会把 payload 解包到局部缓冲，再按 `runtimeSizes` 语义分发；
- 其 caller 正好就是 `_nv039914rm / _nv039913rm / _nv039912rm / _nv039915rm` 这一组 `printf` 包装簇；
- 这与 `nvlog.c` 里 `nvlogPrint_vprintf()` 的职责和输入编码方式高度一致。

当前置信度：**高**。

### E. 双层等效 patch

**业务层等效 patch：**

- 给 RM 内部一条事件 / trace 解包路径注入内核日志，便于观察运行时 packet / opcode 行为。

**字节 / 插桩层等效 patch：**

```text
0x00016185: 48 81 ED 40 04 00 00
=> 90 90 E8 <rel32-to-vup_hook_klogtrace_naked>
```

### F. patch 前后业务影响

**patch 前：**

- 内部 trace 信息只在 RM 私有路径中流动；
- 内核侧看不到拆包后的简化事件流。

**patch 后：**

- 可以通过 `dmesg` / `kern.log` 观察到 `NVTRACE %06x:%04x %04x%02x` 形式的运行时事件。

---

## 4.3 `cudahost`

### A. 基本信息

- 定义：`patches/nv_hooks.c:74-111`
- 表项：`patches/nv_hooks.c:220`
- offset：`0x00416B9C`
- 原始模板：`41 80 BD 24 08 00 00 00`
- 模板长度：8 字节

### B. IDA 落点与汇编

当前 IDA 落点：

- 函数：`_nv036968rm`
- 范围：`0x416AF0 - 0x416CDD`
- hook 点附近原始汇编：

```asm
0x00416B98  mov byte ptr [rbx+0x4FA], 0
0x00416B9C  cmp byte ptr [r13+0x824], 0
0x00416BA4  jz  0x416BBA
```

`vup_hook_cudahost_naked()` 会在原比较前插入：

- `patches/nv_hooks.c:97-107`

```asm
lea   0x50c(%rbx), %rdi
call  vup_hook_cudahost
...
cmpb  $0, 0x824(%r13)
ret
```

因此它的**字节 / 插桩层等效 patch**是：

```text
offset=0x00416B9C
原始模板 = 41 80 BD 24 08 00 00 00
改写结果 = 90 90 90 E8 <rel32-to-vup_hook_cudahost_naked>
```

### C. helper 语义

- `patches/nv_hooks.c:78-85`

```c
static void vup_hook_cudahost(u8 *flag)
{
	printk(KERN_INFO "nvidia: vup_hook cudahost=%d flag=%d\n",
	       vup_cudahost, *flag);
	if (vup_cudahost > 0)
		*flag = vup_cudahost;
}
```

也就是说，它会在原始分支判断前，先把某个对象内的 flag 改成模块参数指定值，再让原逻辑继续执行。

### D. 源码映射

这一项当前仍**不建议写成单个公开源码函数的逐行直译**，但已经可以收窄到更具体的 CUDA limit 相关源码函数簇：

- `drivers/resman/src/kernel/gpu/perf/kern_cuda_limit.c:34-58`
  - `deviceKPerfCudaLimitCliDisable`
- `drivers/resman/src/physical/gpu/perf/cuda_limit.c:169-219`
  - `perfCudaLimitEvaluateLimit_IMPL`
- `drivers/resman/src/physical/gpu/perf/cuda_limit.c:292-332`
  - `subdeviceCtrlCmdInternalPerfCudaLimitDisable_IMPL`

关键源码片段一：

```c
NV_STATUS
deviceKPerfCudaLimitCliDisable
(
    Device  *pDevice,
    OBJGPU  *pGpu
)
{
    ...
    if (pDevice->nCudaLimitRefCnt > 0)
    {
        status = pRmApi->Control(... NV0080_CTRL_CMD_INTERNAL_PERF_CUDA_LIMIT_DISABLE ...);
        ...
        pDevice->nCudaLimitRefCnt = 0;
    }
}
```

关键源码片段二：

```c
NV_STATUS
perfCudaLimitEvaluateLimit_IMPL
(
   POBJGPU      pGpu,
   Perf        *pPerf,
   PCUDA_LIMIT  pCudaLimit,
   NvBool       bTrigger
)
{
    ...
    if (pDisp &&
        !pDisp->getProperty(pDisp, PDB_PROP_DISP_DISABLE) &&
        pDisp->getProperty(pDisp, PDB_PROP_DISP_BUG_1822079_GLITCH_PWR_CUDA_MAX_WAR))
    {
        ...
    }
}
```

匹配依据：

- `_nv036968rm` 同样是一条多入口调用的 CUDA host / CUDA limit 控制链；
- 二进制里既有多个 early-return，也有多条 error/log path，并持续改写 `a2 + 1274 / 1278 / 1292` 一类状态位；
- 这和 `kern_cuda_limit.c` / `cuda_limit.c` 中“启停 / 评估 / 回写 CUDA limit 状态”的职责最接近。

当前置信度：**中等**。

### E. 双层等效 patch

**业务层等效 patch：**

- 在原逻辑读取某个 host / capability flag 之前，先允许模块参数强制覆写它。
- 这样后续分支会自然把当前环境视为“满足某种 host CUDA 条件”。

**字节 / 插桩层等效 patch：**

```text
0x00416B9C: 41 80 BD 24 08 00 00 00
=> 90 90 90 E8 <rel32-to-vup_hook_cudahost_naked>
```

### F. patch 前后业务影响

**patch 前：**

- 原始分支直接读取对象状态中的 flag；
- 后续行为完全受真实初始化状态支配。

**patch 后：**

- 可以在分支判断前先改写该 flag；
- 从而把原本由运行时状态决定的分支，变成可由模块参数影响的行为。

---

## 5. `vup_patches[]`：KVM 主线逐组分析

## 5.1 `vgpusig`

### A. 基本信息

- 定义：`patches/nv_hooks.c:234-239`
- 项数：1
- 条件：`NV_VGPU_KVM_BUILD`
- 默认启用值：`1`

```c
static struct vup_patch_item vup_diff_vgpusig[] = {
	{ 0x000BBEF0, 0x85, 0x31 },
};
```

### B. IDA 落点

- diff offset：`0x000BBEF0`
- IDA EA：`0x000BBEB0`
- 命中函数：`_nv050770rm`
- 调用者：`_nv049279rm`

当前 `_nv049279rm` 的反编译形状说明，这条链在批量处理 host-vGPU 设备/类型配置条目：

- 先检查条目数；
- 再循环调用 `_nv050770rm`；
- 每个条目大小约为 `5088` 字节。

### C. 源码映射

当前最窄源码函数簇：

- `drivers/resman/src/kernel/virtualization/vgpu_mgr.c`
- `drivers/resman/src/kernel/virtualization/kernel_vgpu_mgr.c`
- 并与 `drivers/resman/src/kernel/virtualization/grid/grid_features.c:525-650` 的 licensable feature / signature 语义相邻

当前置信度：**中等**。

更稳妥的表述是：

- `vgpusig` 高置信落在一条 host-vGPU 配置 / type / identity 处理链上；
- 它与 `grid_features.c` 中 `subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL` 所处的“licenseEdition / licensedProductName / signature”业务域相邻；
- 但当前还没有足够证据把 `_nv050770rm` 压到 `vgpu_mgr.c` / `kernel_vgpu_mgr.c` 中某一个公开函数。

### D. 双层等效 patch

**业务层等效 patch：**

- 放宽一条 vGPU 配置 / 条目校验逻辑，使某个布尔判定更容易通过。

**字节层等效 patch：**

```text
0x000BBEF0: 85 -> 31
```

### E. patch 前后业务影响

**patch 前：**

- 该路径依据原始校验结果决定条目是否有效。

**patch 后：**

- 这项判断被改写为更宽松的形式，降低了配置签名 / 条目合法性对后续流程的阻断力度。

---

## 5.2 `kunlock`

### A. 基本信息

- 定义：`patches/nv_hooks.c:240-250`
- 项数：7
- 默认启用值：`1`

其中最关键的两项与此前 `blob-550.90.05.diff` 的核心业务链直接重叠：

```c
{ 0x0051DDDE, 0x07, 0x00 },
{ 0x0051DDE2, 0x00, 0x01 },
```

### B. 全部命中二进制函数

`kunlock` 这 7 个 item 当前已确认命中以下二进制函数：

- `0x000D65C8 -> 0x0D6588`：`_nv032674rm`
- `0x00512208 -> 0x5121C8`：`_nv026636rm`
- `0x005148F5 -> 0x5148B5`：`_nv026647rm`
- `0x0051DDDE -> 0x51DD9E`：`_nv026411rm`
- `0x0051DDE2 -> 0x51DDA2`：`_nv026411rm`
- `0x0051E600 -> 0x51E5C0`：`_nv025999rm`
- `0x00527E1F -> 0x527DDF`：`_nv026005rm`

其中：

- `_nv026636rm` 是一个单 bit capability helper；
- `_nv026647rm` 是一个基于 callback 输出参数的布尔 helper；
- `_nv025999rm` 是一条更大的 device/subdevice 白名单与状态位写回链；
- `_nv026005rm` 是一个很小的设备编号区间 helper。

这些 helper 虽然形状不同，但都挂在同一条“设备身份 / GRID 支持 / capability 聚合”故事线上。

### C. 关键落点 1：`_nv032674rm`

- `0x000D65C8 -> 0x0D6588`
- 命中：`_nv032674rm`
- 高置信源码函数：
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`

这条路径与 `blob-550.90.05.diff` 中“把支持判定改成恒成功”的静态 patch 业务完全同向：

- 原始逻辑：白名单 / 设备身份 / 子设备身份 / 品牌条件综合判定；
- `kunlock`：在同一支持链附近继续放宽门槛。

### C. 关键落点 2：`_nv026411rm`

- `0x0051DDDE -> 0x51DD9E`
- `0x0051DDE2 -> 0x51DDA2`
- 命中：`_nv026411rm`

其反编译显示它是一条 capability 聚合函数：

```c
if ((*(...))(a1, v8, 4278190106LL, v7 + 3))
    v7[3] = 0;
...
*(_BYTE *)(a1 + 18015) = v12;
*(_BYTE *)(a1 + 18012) = v13;
*(_BYTE *)(a1 + 18013) = v10;
*(_BYTE *)(a1 + 18014) = v11 != 0;
```

当前最窄源码函数簇：

- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1206-1283`
- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1772-1804`
- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1342-1372`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:gpuEnableGridFeature`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL`

对应源码侧业务链示例：

- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1206-1263`

```c
pGpu->gridLicensedFeatures = NV_REG_STR_RM_GRID_LICENSED_FEATURES_DISABLED;
osGetGridLicensedFeatures(&(pGpu->gridLicensedFeatures));
...
if (IS_VIRTUAL(pGpu))
{
    pGpu->gridLicensedFeatures |= FLD_SET_DRF(... _VGPU, _ENABLED, ...);
}
```

- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1791-1798`

```c
if (isGridLicenseSupported(pGpu))
{
    const GRID_FEATURE_STATE *pFeatureState = gridlicmgrGetGpuFeatureState(pGpu->gpuId);
    ...
    status = enableGridFeature(pGpu, pFeatureState->featureCode, pFeatureState->bLicenseState);
}
```

### D. 双层等效 patch

**业务层等效 patch：**

- 同时改两层 gate：
  1. 上游支持判定 gate；
  2. 下游 capability 聚合输入 gate。
- 结果是：不仅更容易被视为“支持 GRID/vGPU licensing”，而且 capability 状态写回也更偏向真值。

**字节层等效 patch：**

关键项示例：

```text
0x000D65C8: 75 -> EB
0x0051DDDE: 07 -> 00
0x0051DDE2: 00 -> 01
0x0051E600: 75 -> EB
```

### E. patch 前后业务影响

**patch 前：**

- 设备白名单不通过时，会被卡在支持判定链；
- 某些 capability 聚合输入失败时，会导致最终状态字节保持 0。

**patch 后：**

- 设备身份白名单更容易通过；
- capability 聚合中的部分输入被强制推向真值；
- 上层 licensing / feature / displayless / migration 等依赖状态会更“乐观”。

---

## 5.3 `qmode`

### A. 基本信息

- 定义：`patches/nv_hooks.c:251-256`
- 项数：2

```c
{ 0x00523E86, 0x0D, 0x07 },
{ 0x00523E8F, 0x84, 0x85 },
```

### B. IDA 落点

- IDA 命中函数：`_nv026437rm @ 0x523C50`

这条函数的反编译非常有辨识度：

- 读取 `OverrideGpuInit`
- 读取 `RMDisableFeatureDisablement`
- 读取 `RMGpuCacheOnly`
- 读取 `RMEnableReplayable`
- 以及大量其他 registry key
- 最后回写一整批 `OBJGPU` 成员字段

### C. 源码映射

这组映射已经可以高置信落到公开源码函数：

- `drivers/resman/src/physical/gpu/gpu_registry_physical.c:35-288`
- 函数：`gpuInitRegistryOverrides_IMPL`
- 其上游 helper：
  - `drivers/resman/src/physical/gpu/gpu_registry_physical.c:645-652`
  - `gpuInitOverridesFromRegistry`
- 相关对象字段定义：
  - `drivers/resman/inc/kernel/gpu/gpu.h:4107-4116`
  - `initOverrideTable`

关键源码片段：

```c
NV_STATUS
gpuInitRegistryOverrides_IMPL
(
    OBJGPU *pGpu
)
{
    ...
    gpuInitOverridesFromRegistry(pGpu);
    ...
    if (osReadRegistryDword(pGpu,
                            NV_REG_STR_RM_DISABLE_FEATURE_DISABLEMENT, &data32) == NV_OK)
    {
        ...
        pGpu->bSkipFeatureDisablement = !!data32;
    }
    ...
    if ((osReadRegistryDword(pGpu, NV_REG_STR_RM_GPU_CACHE_ONLY,
               &data32) == NV_OK) && (data32))
    {
        pGpu->bCacheOnlyMode = NV_TRUE;
    }
    ...
    if ((osReadRegistryDword(pGpu, NV_REG_STR_RM_ENABLE_REPLAYABLE,
               &data32) == NV_OK) && (data32))
    {
        pGpu->bReplayableTraceEnabled = NV_TRUE;
    }
}
```

这是当前所有支线里**最好锚定**的一条之一，因为二进制里出现的 registry key 与源码函数的 key 集高度一致。

当前置信度：**高**。

### D. 双层等效 patch

**业务层等效 patch：**

- 修改 GPU registry override 初始化路径中的某个条件判断，让某个“模式 / 能力 / 初始化选项”更容易被视为打开或可用。

**字节层等效 patch：**

```text
0x00523E86: 0D -> 07
0x00523E8F: 84 -> 85
```

### E. patch 前后业务影响

**patch 前：**

- GPU 初始化时严格按 registry 解析结果回写状态。

**patch 后：**

- 某个 override 分支的条件阈值被放宽，导致后续对象状态更偏向开启态。

---

## 5.4 `merged`

### A. 基本信息

- 定义：`patches/nv_hooks.c:257-262`
- 项数：2

```c
{ 0x000B3AD9, 0x97, 0x00 },
{ 0x004FA095, 0x2A, 0x00 },
```

### B. IDA 落点

- `0x000B3AD9 -> 0x0B3A99`：`_nv032676rm`
- `0x004FA095 -> 0x4FA055`：`_nv026510rm`

其中 `_nv026510rm` 的反编译直接出现：

```c
else if (!(unsigned int)nv041665rm(a1, "RmForceGridDisplayless", v2 + 12))
{
    LOBYTE(v3) = *(_DWORD *)(v2 + 12) == 1;
    return v3;
}
```

### C. 源码映射

这一组里，两处落点的收敛程度不同。

#### 1) `_nv026510rm @ 0x4FA010`

这一处现在已经可以高置信对应到：

- `drivers/resman/src/kernel/gpu/gpu.c:2225-2270`
- 函数：`gpuIsGridDisplaylessClassSupported_IMPL`

关键源码：

```c
NvBool
gpuIsGridDisplaylessClassSupported_IMPL
(
    OBJGPU *pGpu
)
{
    ...
    if (pGpu->bGridswOnQuadroSupported)
    {
        return NV_TRUE;
    }
    ...
    if (pGpu->bGridCapable)
    {
        return NV_TRUE;
    }
    ...
    if ((osReadRegistryDword(pGpu,
                            NV_REG_STR_FORCE_GRID_DISPLAYLESS, &data) == NV_OK) &&
        (data == NV_REG_STR_FORCE_GRID_DISPLAYLESS_YES))
    {
        return NV_TRUE;
    }

    return NV_FALSE;
}
```

匹配依据：

- `_nv026510rm` 的反编译直接检查 `RmForceGridDisplayless`；
- 它同时还检查一组与 display 是否存在、GRID capability 是否已置位相关的对象状态；
- 这些条件与 `gpuIsGridDisplaylessClassSupported_IMPL()` 的控制流几乎逐项对齐。

这一项当前置信度：**高**。

#### 2) `_nv032676rm @ 0x0B3A00`

这一处当前仍更适合落到源码函数簇，但已经可以收窄到同一条 `GRID_DISPLAYLESS` 支持判定链：

- 直接核心 helper：
  - `drivers/resman/src/kernel/gpu/gpu.c:2225-2270`
  - `gpuIsGridDisplaylessClassSupported_IMPL`
- 直接下游消费者：
  - `_nv015776rm`
  - `_nv015782rm`
- 业务侧配套函数簇：
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:170-242`
    - `enableGridFeature`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:256-313`
    - `disableAllGridLicensedFeatures`
  - `drivers/resman/src/physical/gpu/gpu_branding.c:212-235`
    - `gpuDetectVgxBranding_IMPL`

匹配依据：

- `_nv032676rm` 的反编译与 `_nv026510rm` 一样直接检查 `RmForceGridDisplayless`；
- `_nv032676rm` 会根据该判定把输出字节置成 0/1；
- 它的调用者 `_nv015776rm` / `_nv015782rm` 继续把这个布尔结果送入后续显示相关 helper，说明它不是 branding 主链，而是 displayless 支持状态的中间 helper。

关键源码片段：

- `drivers/resman/src/kernel/gpu/gpu.c:2225-2270`

```c
NvBool
gpuIsGridDisplaylessClassSupported_IMPL
(
    OBJGPU *pGpu
)
{
    ...
    if (pGpu->bGridCapable)
    {
        return NV_TRUE;
    }
    ...
    if ((osReadRegistryDword(pGpu,
                            NV_REG_STR_FORCE_GRID_DISPLAYLESS, &data) == NV_OK) &&
        (data == NV_REG_STR_FORCE_GRID_DISPLAYLESS_YES))
    {
        return NV_TRUE;
    }

    return NV_FALSE;
}
```

- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:214-223`

```c
if (status == NV_OK)
{
    OBJGRIDDISPLAYLESS *pGridDisplayless = GPU_GET_GRIDDISPLAYLESS(pGpu);
    ...
    griddisplaylessSetLicensedNumHeads_HAL(pGpu, pGridDisplayless);
    griddisplaylessSetLicensedMaxResolution_HAL(pGpu, pGridDisplayless);
    griddisplaylessSetLicensedMaxPixels_HAL(pGpu, pGridDisplayless);
}
```

这一项当前置信度：**中等**。

### D. 双层等效 patch

**业务层等效 patch：**

- 放宽 GRID / displayless / branding 相关判定，使 merged driver 更容易把当前 GPU 视为可走 GRID displayless / licensed feature 路径。

**字节层等效 patch：**

```text
0x000B3AD9: 97 -> 00
0x004FA095: 2A -> 00
```

### E. patch 前后业务影响

**patch 前：**

- displayless / branding / GRID capability 依赖真实品牌和 licensed feature 状态。

**patch 后：**

- merged driver 更容易在同一安装包里同时保留 host / GRID 两边需要的 capability 状态。

---

## 5.5 `swrlwar`

### A. 基本信息

- 定义：`patches/nv_hooks.c:263-270`
- 项数：4

```c
{ 0x0047CBBB, 0xB1, 0xCA },
{ 0x0047CC89, 0x44, 0x90 },
{ 0x0047CC8A, 0x89, 0x31 },
{ 0x0047CC8B, 0xE8, 0xC0 },
```

### B. IDA 落点与源码函数

- 命中函数：`_nv046497rm @ 0x47CB60`
- 命中源码函数：
  - `drivers/resman/src/physical/gpu/fifo/objsched.c:schedSwSwrlSetCountMax_IMPL`
- 关键源码：`drivers/resman/src/physical/gpu/fifo/objsched.c:3737-3817`

```c
if (pSchedSw->swrlCount != 0)
{
    portDbgPrintf("NVRM: Can't change software runlist max count.\n");
    return NV_ERR_INVALID_STATE;
}
...
portDbgPrintf("NVRM: Software scheduler timeslice set to %duS.\n",
              (pSchedSw->timeSlice / (1000)));
```

### C. 关键汇编

- `mcp__ida-mcp__get_basic_blocks` 已确认：
  - `0x47CB60-0x47CB7F` 是早期检查块
  - `0x47CC30-0x47CC52` 是错误日志块

错误字符串 xref：

- `0x47CC36 -> 0x2EADBF0`
- 字符串：`"NVRM: Can't change software runlist max count.\n"`

### D. 双层等效 patch

**业务层等效 patch：**

- 去掉“software runlist 运行中禁止修改 `swrlCountMax`”的保护规则；
- 并把同一错误路径上的返回值准备改写成更宽松 / 更偏诊断的形式。

**字节层等效 patch：**

关键项示例：

```text
0x0047CBBB: B1 -> CA
0x0047CC89: 44 -> 90
0x0047CC8A: 89 -> 31
0x0047CC8B: E8 -> C0
```

其中 `44 89 E8 -> 90 31 C0` 的效果，就是把：

```asm
mov eax, r13d
```

改成：

```asm
nop
xor eax, eax
```

### E. patch 前后业务影响

**patch 前：**

- software runlist 正在运行时，不允许修改 `swrlCountMax`；
- 会直接报错并返回 `NV_ERR_INVALID_STATE`。

**patch 后：**

- 运行中也可继续改 `swrlCountMax`；
- timeslice / ARR 相关后续流程继续执行。

---

## 5.6 `fbcon`

### A. 基本信息

- 定义：`patches/nv_hooks.c:271-275`

```c
{ 0x00AEA8F3, 0x84, 0x30 },
```

### B. IDA 落点与源码函数

- 命中函数：`_nv000720rm @ 0xAE9D20`
- 高置信源码函数：
  - `drivers/resman/arch/nvalloc/unix/src/osinit.c:RmInitAdapter`
- 同一初始化主链内部直接覆盖到：
  - `drivers/resman/arch/nvalloc/unix/src/osinit.c:RmSetupRegisters`

关键源码片段：

- `drivers/resman/arch/nvalloc/unix/src/osinit.c:1680-1740`

```c
static void
RmSetupRegisters(
    nv_state_t *nv,
    UNIX_STATUS *status
)
{
    ...
    if (nv->regs->map == NULL)
    {
        NV_DEV_PRINTF(NV_DBG_ERRORS, nv, "Failed to map regs registers!!\n");
        ...
    }
    ...
    ret = RmSetupHdacodecRegisters(nv, status);
}
```

- `drivers/resman/arch/nvalloc/unix/src/osinit.c:2168-2699`

```c
NvBool RmInitAdapter(
    nv_state_t *nv
)
{
    ...
    RmSetupRegisters(nv, &status);
    ...
#if RMCFG_FEATURE_ENABLED(GRID_LICENSE_MANAGER)
    disableAllGridLicensedFeatures(pGpu);
#endif
    ...
    NV_DEV_PRINTF(NV_DBG_SETUP, nv, "RmInitAdapter succeeded!\n");
}
```

### C. 双层等效 patch

**业务层等效 patch：**

- 改写一条初始化主线中的布尔条件，使 `fbcon / display / displayless` 相关初始状态更容易落到目标配置。

**字节层等效 patch：**

```text
0x00AEA8F3: 84 -> 30
```

### D. patch 前后业务影响

**patch 前：**

- 初始化主链严格按原 display / console 相关状态推进。

**patch 后：**

- 某个条件分支被改写后，初始化阶段对 `fbcon` / displayless 的处理更接近 vGPU unlock 目标。

---

## 5.7 `sunlock`

### A. 基本信息

- 定义：`patches/nv_hooks.c:276-286`
- 项数：5

### B. 全部命中二进制函数

`sunlock` 这 5 个 item 当前已确认命中以下二进制函数：

- `0x000D65FA -> 0x0D65BA`：`_nv032674rm`
- `0x000D65C8 -> 0x0D6588`：`_nv032674rm`
- `0x004F6D2A -> 0x4F6CEA`：`_nv019510rm`
- `0x000BECB1 -> 0x0BEC71`：`_nv030355rm`
- `0x000BE72C -> 0x0BE6EC`：`_nv030331rm`

其中 `_nv019510rm` 的反编译显示，它会综合：

- `v2[2988]`
- `v2[16634]`
- `nv030882rm()`
- `nv032674rm(v2)`
- `v2[2084] / v2[2085]`

再返回 `0/1/2/3/4` 这类状态码。

### C. 源码映射

当前最窄源码函数簇：

- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:gpuEnableGridFeature`
- `drivers/resman/src/physical/gpu/gpu_branding.c:gpuDetectVgxBranding_IMPL`
- `drivers/resman/src/physical/gpu/gpu_branding.c:deviceCtrlCmdGpuGetBrandCaps_IMPL`

当前置信度：**中等偏低**。

更稳妥的结论是：

- `sunlock` 不是单点 patch，而是一组围绕“是否支持 GRID / 是否 displayless / 是否许可路径可用”的状态消费补丁；
- 它与 `kunlock` 的上游支持判定链相互配合。

### D. 双层等效 patch

**业务层等效 patch：**

- 放宽 displayless / GRID capability / branding 消费链上的若干状态门槛，让上层更容易进入“支持 / 已启用 / 可继续”分支。

**字节层等效 patch：**

关键项示例：

```text
0x000D65FA: 01 -> 00
0x000D65C8: 75 -> EB
0x004F6D2A: 00 -> 03
0x000BECB1: 42 -> 00
0x000BE72C: 10 -> 00
```

### E. patch 前后业务影响

**patch 前：**

- 多条中下游状态消费链仍可能因为 branding / displayless / capability 组合不满足而退回。

**patch 后：**

- 即使上游不是标准 datacenter / GRID 受支持 SKU，下游也更可能继续推进到可用态。

---

## 5.8 `gspvgpu`

### A. 基本信息

- 定义：`patches/nv_hooks.c:287-293`
- 项数：2（另有 1 条注释掉的候选项）

```c
{ 0x00033EE5, 0x1B, 0x00 },
{ 0x00033EF1, 0x74, 0xEB },
```

### B. IDA 落点

- 命中函数：`_nv026896rm @ 0x33E70`

反编译显示它是一个很小的布尔 helper：

```c
*a6 = (a4 & 0x10) != 0;
result = nv026921rm(a1, a2, a3, v9 + 15);
...
*a5 = result;
```

### C. 源码映射

当前最窄源码函数簇：

- `drivers/resman/src/physical/gpu/gsp/gsp.c:405-449`
- 以及与 displayless / GSP capability 相关的小型 helper 链

关键源码片段：

- `drivers/resman/src/physical/gpu/gsp/gsp.c:405-413`

```c
// GSP-Falcon ucode currently only supports HDCP, and it halts itself on displayless chip...
if (UPROC_ENG_ARCH_FALCON(pFlcn) &&
    ((pDisp == NULL) || pDisp->getProperty(pDisp, PDB_PROP_DISP_DISABLE)))
{
    NV_PRINTF(LEVEL_ERROR, "GSP-Falcon ucode is disabled on displayless chip.\n");
    pGsp->setProperty(pGsp, PDB_PROP_GSP_SKIP_LOAD, NV_TRUE);
}
```

当前置信度：**中等偏低**。

### D. 双层等效 patch

**业务层等效 patch：**

- 放宽一条与 GSP / vGPU / displayless 相关的 capability helper，使上游更容易把当前环境视为“允许继续”。

**字节层等效 patch：**

```text
0x00033EE5: 1B -> 00
0x00033EF1: 74 -> EB
```

### E. patch 前后业务影响

**patch 前：**

- 某条 GSP / displayless / vGPU capability helper 会在不满足条件时返回否定结果。

**patch 后：**

- 否定分支被削弱，后续更容易走向继续初始化或继续启用的路径。

---

## 6. `NV_GRID_BUILD`：`general` 组

## 6.1 基本信息

- 定义：`patches/nv_hooks.c:307-321`
- 项数：4

```c
{ 0x000D65C8, 0x75, 0xEB },
{ 0x008DC797, 0x14, 0xA0 },
{ 0x008DC798, 0x00, 0x05 },
{ 0x00AF2C31, 0x75, 0xEB },
```

## 6.2 已确认的函数级映射

### A. 共享支持判定落点

- `0x000D65C8 -> _nv032674rm`
- 高置信源码函数：
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`

这说明 `general` 组和 `kunlock` 共享同一条最核心的“设备 / 子设备白名单支持链”。

### B. vGPU 支持检测落点

- `0x00AF2C31 -> 0xAF2BF1`
- 命中函数：`_nv045327rm`
- 高置信源码函数：
  - `drivers/resman/arch/nvalloc/unix/src/os-hypervisor.c:rm_is_vgpu_supported_device`

关键源码：

```c
if (vgpu_reg_mapping == NULL)
{
    nv_printf(NV_DBG_ERRORS, "NVRM: failed to map vGPU register!\n");
    return NV_ERR_OPERATING_SYSTEM;
}
...
if (os_is_grid_supported())
{
    for (i = 0; i < NV_ARRAY_ELEMENTS(sVgpuUsmTypes); i++)
    {
        if (pOsGpuInfo->pci_info.device_id == sVgpuUsmTypes[i].ulDevID && ...)
        {
            rmStatus = NV_OK;
            break;
        }
    }
}
```

### C. displayless / licensed capability 落点

- `0x008DC797 -> 0x8DC757`
- `0x008DC798 -> 0x8DC758`
- 命中函数：`_nv028908rm`

当前最窄源码函数簇：

- 上游直接 helper：
  - `drivers/resman/src/kernel/gpu/gpu.c:2225-2270`
  - `gpuIsGridDisplaylessClassSupported_IMPL`
- 下游业务函数簇：
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:enableGridFeature`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`

其反编译显示它会操作：

- `a1 + 18096` / `a1 + 18128` 一带的状态块；
- `64 / 72` 处的 `num heads` / `max resolution` 风格值；
- 并在成功后调用后续 helper 更新显示相关状态。

从控制流关系看，更稳妥的理解是：

- `general` 这两字节并不是直接改 `enableGridFeature()` 的核心判断；
- 它更像是在 `gpuIsGridDisplaylessClassSupported_IMPL()` 这一条 displayless 支持判定链的下游消费者里，放宽某个 capability / resolution / heads 状态初始化结果。

## 6.3 双层等效 patch

**业务层等效 patch：**

- `general` 组是 GRID 侧的“总开关补丁包”：
  - 一方面复用 `kunlock` 的支持判定放宽；
  - 另一方面复用 `merged` 的 displayless / vGPU 支持放宽。

**字节层等效 patch：**

```text
0x000D65C8: 75 -> EB
0x008DC797: 14 -> A0
0x008DC798: 00 -> 05
0x00AF2C31: 75 -> EB
```

---

## 7. 与 `blob-550.90.05.diff` 的关系

`nv_hooks.c` 与 `blob-550.90.05.diff` 至少有两类重叠：

1. **同一业务链，不同 patch 手段**
   - `kunlock` 与 `blob` 中 `_nv032674rm` / `_nv026411rm` 那条 licensing / capability 链重叠；
   - `swrlwar` 与 `blob` 中 `_nv046497rm` 的 software runlist 限制链重叠。

2. **静态 patch 与动态 hook 并存**
   - `blob-*.diff` 改的是离线 blob 字节；
   - `nv_hooks.c` 改的是运行时内存映像；
   - 两者最终叠加在同一条业务链的不同层级上。

因此更准确的理解不是“哪个补丁更核心”，而是：

- `blob-*.diff` 负责拔掉最硬的静态 gate；
- `nv_hooks.c` 再提供参数化、运行时、可插桩的细粒度控制。

---

## 8. 后续版本快速定位与 patch 复用指南

## 8.1 双坐标系

### A. hook 坐标系

- `vup_hooks_init()` 调的是 `vup_inject_hooks(blob)`
- 所以 `vup_hook_info.offset` 直接相对 `blob`

### B. patch 坐标系

- `vup_hooks_init()` 调的是 `vup_apply_patches(blob - 0x40)`
- 所以 `vup_patch_item.offset` 采用的是 diff / 文件偏移风格
- 与 `blob-550.90.05.diff` 一致：
  - `IDA_EA = PATCH_OFFSET - 0x40`

## 8.2 直接定位法

### hook 项

- `vupdevid`：`0x0051E3A7`
  - 模板：`4C 89 E0 44 89 FB`
- `klogtrace`：`0x00016185`
  - 模板：`48 81 ED 40 04 00 00`
- `cudahost`：`0x00416B9C`
  - 模板：`41 80 BD 24 08 00 00 00`

### patch 项

- `kunlock`：看 `0x0051DDDE / 0x0051DDE2 / 0x000D65C8`
- `swrlwar`：看 `0x0047CBBB / 0x0047CC89 / 0x0047CC8A / 0x0047CC8B`
- `general`：看 `0x00AF2C31` 与 `0x000D65C8`

## 8.3 间接定位法

如果后续版本里函数名全变、偏移也漂了，优先用下面这些锚点：

1. **日志字符串锚点**
   - `NVRM: Can't change software runlist max count.`
   - `NVRM: Software scheduler timeslice set to`
   - `NVRM: failed to map vGPU register!`
   - `RmInitAdapter succeeded!`
   - `RmInitAdapter failed!`

2. **registry key 锚点**
   - `OverrideGpuInit`
   - `RMDisableFeatureDisablement`
   - `RMGpuCacheOnly`
   - `RMEnableReplayable`
   - `RmForceGridDisplayless`

3. **业务职责锚点**
   - `PCIDeviceID / PCISubDeviceID` 白名单支持判定
   - `gridLicensedFeatures` 初始化与消费
   - displayless / licensed num heads / licensed max resolution 调整
   - host-vGPU identity / config / type 处理链

## 8.4 复用判定标准

### 可直接复用

- 日志字符串、条件职责、调用链都一致；
- 只是 offset 改了。

### 需要重定位后改写

- 业务位置一致；
- 但寄存器分配、指令模板长度或 basic block 形状变了。

### 不建议直接复用

- 上游业务链已经重构；
- 旧 patch 仍然能落字节，但语义已经不是同一条控制流。

---

## 9. 结论

把 `nv_hooks.c` 整体看完后，它并不是“又一份 blob diff”，而是一个分层很清楚的运行时补丁框架：

1. **hook 层**
   - 用最小侵入的 `NOP + CALL rel32` 插桩，把少数关键路径参数化；
   - 典型例子是：
     - `vupdevid`：改设备身份；
     - `cudahost`：改 host / CUDA 相关 flag；
     - `klogtrace`：增强调试可观测性。

2. **direct patch 层**
   - 直接放宽 licensing、displayless、software runlist、初始化 / branding / GSP / vGPU 支持等多条门槛链。

3. **与静态 blob patch 的关系**
   - `blob-*.diff` 更像“离线拔掉硬 gate”；
   - `nv_hooks.c` 更像“运行时可调的细粒度控制层”。

从仓库目标看，这正好解释了为什么它要同时存在：

- 静态 patch 负责把最难绕过的固定检查拔掉；
- 运行时 hook/patch 则负责把设备身份、displayless、licensed feature、host-vGPU config 这些更易漂移、也更需要参数化的控制点放到模块里动态接管。

## 10. 当前置信度说明

### 高置信

- `swrlwar -> _nv046497rm -> objsched.c:schedSwSwrlSetCountMax_IMPL`
- `kunlock/general -> _nv032674rm -> grid_features.c:isGridLicenseSupported`
- `qmode -> _nv026437rm -> gpu_registry_physical.c:gpuInitRegistryOverrides_IMPL`
- `fbcon -> _nv000720rm -> osinit.c:RmInitAdapter`
- `general -> _nv045327rm -> os-hypervisor.c:rm_is_vgpu_supported_device`
- `klogtrace -> _nv039916rm -> diagnostics/nvlog.c:nvlogPrint_vprintf`
- `merged` 中 `_nv026510rm -> kernel/gpu/gpu.c:gpuIsGridDisplaylessClassSupported_IMPL`

### 中等置信

- `vupdevid` 落在设备身份查表 / pGPU identity 处理链，最接近 `vgpu_mgr.c / kernel_vgpu_mgr.c` 与 `gpu_name.c` 的候选函数簇
- `cudahost` 落在 CUDA limit / host-side control 链，最接近 `kern_cuda_limit.c` 与 `perf/cuda_limit.c` 的函数簇
- `merged` 中 `_nv032676rm` 落在 `gpuIsGridDisplaylessClassSupported_IMPL()` 的下游 displayless 状态消费链
- `kunlock` 的 `_nv026411rm` 落在 `gpu_mgr.c + grid_features.c` 的 capability 聚合 / 消费链，而不是单个公开函数直译
- `vgpusig` 落在 `vgpu_mgr.c / kernel_vgpu_mgr.c` 的 host-vGPU 配置链

### 低到中等置信

- `sunlock` 的三处辅助函数
- `gspvgpu`

这些点当前都已经可靠落到了正确的**业务子系统**，但并非每一项都能仅靠当前公开源码 100% 压到单个公开函数。对这些点，本文已经按“最窄源码函数簇 / 调用链”方式保守表达。