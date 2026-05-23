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

## 1.1 从业务角度看，`nv_hooks.c` 一共在动哪几条链

如果不先把业务链分开，`nv_hooks.c` 很容易被看成“到处改几字节”。但把所有点位放回 RM 的真实职责后，它主要是在同时改 8 条链：

1. **宿主卡身份链**
   - 代表点：`vupdevid`、`kunlock` 中 `_nv032674rm` 周边项。
   - 业务问题：RM 先判断“这张宿主卡到底是谁”，再决定它能不能被视为 GRID / vGPU 软件许可对象，以及它在 migration / profile / branding 上属于哪一类。

2. **vGPU profile 导入链**
   - 代表点：`vgpusig`。
   - 业务问题：即使前面放宽了“这卡支持不支持”，如果 host 侧根本没把某条 `vgpuType` 导入进 RM，可创建 profile 列表仍然是空的或不完整的。

3. **licensed feature 与 capability 聚合链**
   - 代表点：`kunlock`。
   - 业务问题：RM 不只是做一次白名单判断，还会把结果折叠成 `gridLicensedFeatures`、feature code、license state、displayless 派生状态等一组对象字段，供后续所有控制命令消费。

4. **启动期运行模式底座链**
   - 代表点：`qmode`。
   - 业务问题：在很多 unlock 场景里，真正卡住流程的不是某个单点 capability，而是 RM 初始化阶段仍然过于保守，导致一堆 override、debug、compatibility 模式没真正落到 `OBJGPU` 状态上。

5. **GRID displayless class 链**
   - 代表点：`merged`、`general`。
   - 业务问题：这条链决定当前板卡能不能被当成 `NVA083_GRID_DISPLAYLESS` 那种“无物理显示但仍需提供显示相关 vGPU 能力”的对象来对待。

6. **unlicensed state machine / 降级状态消费链**
   - 代表点：`sunlock`、`general`。
   - 业务问题：即使前面通过了 licensing / displayless 判定，后续状态机、模式码、输出结构写回仍可能把结果重新压回保守态。

7. **host CUDA / CUDA limit 链**
   - 代表点：`cudahost`。
   - 业务问题：merged 驱动不仅要保住 guest 侧 vGPU，还要尽量不丢宿主机侧 CUDA / compute 相关能力；这一类点位更像“在原控制流继续前，先把宿主状态修成目标值”。

8. **宿主调度与 GSP 兼容链**
   - 代表点：`swrlwar`、`gspvgpu`、`fbcon`、`klogtrace`。
   - 业务问题：前两类解决的是“能不能继续”，后两类解决的是“继续之后会不会因为调度、displayless、GSP、观测性不足而马上撞墙”。

从这个角度看，`nv_hooks.c` 并不是一个单目的补丁，而是：

- 先让 RM **承认这张卡、承认这组 profile、承认这条 licensed/displayless 路径**；
- 再让 RM **在真正初始化、运行、调度和 GSP 路径里别自己把这些结果又关掉**；
- 最后补上一点 **可观测性**，确保这些链条在出问题时至少还能看见内部状态。

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
- 但从 caller 关系看，它的位置已经很清楚：`_nv026708rm` 会在设备初始化中先跑一串 capability / mode 检查，再调用 `_nv026445rm`，然后继续执行覆盖表、override 与后续对象初始化 helper。这说明 `vupdevid` 改的是**初始化中段的身份传播点**，不是最终展示层或最终控制命令层。

### E. 双层等效 patch

**业务层等效 patch：**

- 将原本从 RM 内部对象状态读取出来的 device ID / subdevice ID，改造成“可由模块参数强制覆写的设备身份”。
- 它影响的不只是一个局部分支，而是整条 **宿主卡身份传播链**：
  1. RM 先决定“当前 pGPU 叫什么、属于哪一组设备身份”；
  2. 再决定这张卡是否能和某些 vGPU type / migration 兼容组对上；
  3. 再把这些身份结果继续传播到 profile 选择、兼容分组、licensed product 命名、甚至某些 branding / displayless 派生逻辑。
- 因此 `vupdevid` 的真正业务作用不是“把一个寄存器改一下”，而是把 **后续所有基于 pGPU 身份做出的业务决策** 都切换到另一张卡的视角下执行。

**字节 / 插桩层等效 patch：**

```text
0x0051E3A7: 4C 89 E0 44 89 FB
=> 90 E8 <rel32-to-vup_hook_vupdevid_naked>
```

### F. patch 前后业务影响

**patch 前：**

- 该路径使用真实的设备 / 子设备标识继续后续逻辑；
- 在业务流程里，它通常发生在 **设备初始化已经完成基本 attach、开始派生 pGPU identity / name / encoding** 的阶段；
- 这意味着后面接它的不是单一判断，而是一串“基于宿主卡身份生成可消费元数据”的路径：
  - 设备名字 / 品牌字符串；
  - pGPU 的 migration / equivalency 编码；
  - 某些 vGPU type 对宿主卡身份的兼容约束；
  - 再往后是 licensed product 命名与 profile 家族归属。
- 因此，原始逻辑下如果真实 `device id / subdevice id` 不在预期组合里，后果常常不是立刻报错，而是后续整条链都按“这不是目标宿主卡家族”的前提继续计算。

**patch 后：**

- 可以直接把某张卡伪装成另一张卡的 `device id`；
- 这会改变后续 profile / 迁移兼容 / 能力白名单等基于身份的行为。
- 更具体地说，后续链条里至少有三类结果会跟着变：
  1. **host 侧 profile 可接受性**
     - 某些原本只对特定 pGPU 身份开放的 `vgpuType`，更容易被当成“属于兼容对象”；
  2. **migration / equivalency 编码**
     - `vgpuMgrGetPgpuDevIdEncoding()` / `vgpuMgrGetPgpuSubdevIdEncoding()` 一类路径看到的是伪装后的身份；
  3. **branding / licensing 派生路径**
     - 后面的 `isGridLicenseSupported()`、displayless、licensed product 命名也可能连带改变。
- 所以 `vupdevid` 的可见结果，往往不是某个单独日志变化，而是 host 和 guest 两边对“这张宿主卡属于哪个 profile 家族”的认识一起变化。
- 从调用链位置看，这个改写发生在 caller `_nv026708rm` 的中前段，而 `_nv026708rm` 在后面还会继续做更多对象初始化、状态设置和 helper 调用。也就是说，`vupdevid` 改的不是一个最终展示字段，而是**在更后续的初始化和派生流程开始之前，先把宿主卡身份底稿改掉**。
- `_nv026708rm` 在调用 `_nv026445rm` 之后，还会继续执行额外的对象初始化、override 检查、feature helper 以及进一步的状态设置，因此这里改掉的身份会沿着后续整条初始化链继续传播，而不是只影响一个局部 helper。
- 这类改写最直接的外部效果不是“名字变了”，而是：
  - host 在后面做 `vgpuType` 兼容检查时，会按另一张卡的家族去判断；
  - migration / equivalency 编码也会按另一张卡的身份去生成；
  - guest 侧最终看到的可创建 profile 集合，本质上也会跟着这份被伪装过的宿主身份一起变化；
  - 某些随后才会发生的 feature 初始化或 override 应用，也会默认把这张卡当成伪装后的目标宿主卡去处理。
- 更具体地说，源码里的：
  - `vgpuMgrCreateRequestVgpu()` / `kvgpumgrCreateRequestVgpu()`
  - `vgpuMgrCheckVgpuTypeCreatable()` / `kvgpumgrCheckVgpuTypeCreatable()`
  - `vgpuMgrGetCreatableVgpuTypes()` / `kvgpumgrGetCreatableVgpuTypes()`
  - `vgpuconfigapiCtrlCmdVgpuConfigGetCreatableVgpuTypes_IMPL()`
  这类路径最终决定了 host 当前会把哪些 `vgpuTypeId` 视为可创建对象，以及把哪些 creatable type 列表真正返回给上层配置接口；而这些路径前面依赖的 pGPU identity / encoding 一旦被 `vupdevid` 改写，guest 侧最终能看到的 profile 集合就会跟着漂移。
- 也就是说，`vupdevid` 的最终业务效果很像：
  - 先改“这张宿主卡在 RM 眼里是谁”；
  - 再让后续所有“这张卡能不能承接哪些 profile” 的判断一起跟着改；
  - 最后把这组变化体现在 **mdev / vGPU config 查询接口实际返回的可创建 profile 列表** 上，而不只是停留在内部状态。
- 对外部使用者来说，这意味着：
  - host 侧管理面查询“这张卡现在有哪些 creatable vGPU types”时，返回值本身就可能改变；
  - 这类返回值会沿着 `vgpuconfigapiCtrlCmdVgpuConfigGetCreatableVgpuTypes_IMPL()` 这类控制接口真正暴露给上层管理面，而不是只停在 RM 内部；
  - `drivers/resman/kernel/inc/vgpuconfigapi.h:64-67` 还明确把 `NVA081_CTRL_CMD_VGPU_CONFIG_GET_CREATABLE_VGPU_TYPES` 定义成 `RMCTRL_EXPORT(... NON_PRIVILEGED)`，说明这不是内部自检数据，而是本来就打算给外部管理面消费的结果；
  - 在 KVM 场景里，`vgpuMgrCreateRequestVgpu()` / `kvgpumgrCreateRequestVgpu()` 处理的就是 `mdev_create` 路径，因此这类变化最终不只是“列表长得不一样”，而是会直接改变某个 mdev/profile 创建请求能不能被 host 接受；
  - guest 侧最终能选到的 profile family，也会因为这份伪装后的宿主身份而发生变化。
- 而且这种变化不只体现在“有哪些 `vgpuTypeId` 可见”，还会体现在“这些 type 对外声称自己具备什么属性”上。当前源码里：
  - `vgpuMgrCreateVgpuType()` / `kvgpumgrCreateVgpuType()` 会把 `maxPixels`、`frlConfig`、`cudaEnabled`、`vgpuName`、`vgpuClass`、`licensedProductName` 等字段挂进 `VGPU_TYPE`；
  - `vgpuconfigapiCtrlCmdVgpuConfigGetVgpuTypeInfo_IMPL()` 与 `hostvgpudeviceapiCtrlCmdGetVgpuTypeInfo_IMPL()` 又会把这些字段原样拷回公开控制返回值；
  - `apps/nvml/dmal/rm/rm_vgpu.c:368-399` 还会把同一份 `GET_VGPU_TYPE_INFO` 返回值继续拷进 NVML 的 `vgpuTypeStaticInfo` 缓存；
  - `apps/nvml/api.c:8976-9017, 9041, 9073-9074, 9167-9168, 9224-9227` 则会继续把这些缓存字段直接暴露成 NVML 的 `vgpuTypeClass`、`vgpuTypeName`、`gpuInstanceProfileId`、`deviceID/subsystemID`、最大分辨率和 frame-rate limit 查询结果；
  - `sdk/nvidia/inc/ctrl/ctrla081.h:83-118` 与 `sdk/nvidia/inc/ctrl/ctrla082.h:72-105` 也直接把这些字段定义进 `GET_VGPU_TYPE_INFO` 结构里。
- 这意味着 `vupdevid` 一旦让 RM 走到了另一组 `vgpuTypeInfo`，外部看到的不只是 creatable 列表变化，还包括：
  - `maxPixels` / `maxResolutionX/Y` 这种显示容量边界；
  - `frlConfig` / `frlEnable` 这种帧率限制相关配置；
  - `cudaEnabled` / `gpuDirectSupported` / `nvlinkP2PSupported` 这类功能位；
  - `vgpuName`、`vgpuClass`、`licensedProductName` 这类管理面和 UI 直接展示的 profile 身份信息；
  - `pdevId / vdevId / gpuInstanceProfileId` 这类会继续暴露给工具层的设备与实例标识；
  - 以及 NVML 最终返回给用户态工具的 `vgpu type name/class/frame rate limit/max resolution` 这些直接展示值。
- 这条外显链还不只停在 NVML。当前源码里 `drivers/gpgpu/cuda/src/cui/dmal/rm/rm_control_al.c:2364-2371` 会通过 `NVA080_CTRL_CMD_VGPU_GET_CONFIG` 读取 `cudaEnabled`，再把它折叠成 `vgpuCaps->isCudaCapable`；而 `cuictx.c:4134-4136` 又会在这个位为假时直接报 `CUDA is not supported on this vGPU profile`。这意味着一旦 `vupdevid` 把宿主身份切到另一组 profile 家族，后续 guest 侧看到的也不只是“名字和 FRL 变了”，连 CUDA capability 判断和最终报错行为都可能跟着切换。
- 换句话说，`vupdevid` 最终影响的不是单一控制面，而是 **可创建列表 → type 元数据 → NVML 静态属性 → CUDA capability/报错行为** 这一整条 profile 属性传播链。
- 再往下压一层源码，`kvgpumgrGetCreatableVgpuTypes()` 不是简单把一个缓存数组原样抄出去，而是会遍历 `pgpuInfo->vgpuTypes[]`，逐项调用 `kvgpumgrCheckVgpuTypeCreatable()` 过滤，只有检查通过的项才会写进 `vgpuTypes[*numVgpuTypes]` 并增加计数。
- 这意味着 `vupdevid` 改掉的“宿主是谁”并不会只影响 profile 导入阶段一次，而是会继续参与 **导入 → creatable 判定 → 控制接口返回** 这三层链：
  1. 先在 `vgpuMgrGetPgpuDevIdEncoding()` / `vgpuMgrGetPgpuSubdevIdEncoding()` 里生成伪装后的 identity / alias encoding；
  2. 再让 `vgpuMgrCheckVgpuTypeCreatable()` / `kvgpumgrCheckVgpuTypeCreatable()` 以这套身份去筛选哪些 type 还算“当前宿主可创建”；
  3. 最后由 `vgpuconfigapiCtrlCmdVgpuConfigGetCreatableVgpuTypes_IMPL()` 把筛过的 `numVgpuTypes` 与 `vgpuTypes[]` 原样回填给控制调用者。
- 因而从外部视角看，`vupdevid` 带来的不是单纯“内部 alias 变了”，而是 **host 对外公布的 creatable profile 清单本身被重算了一遍**。只要某个 `vgpuTypeId` 在这套伪装身份下能通过 `CheckVgpuTypeCreatable`，它就会真正进入返回数组；反过来，原本可见的 type 也可能因为这套新身份被过滤掉。
- 而且这条属性传播链不会停在 RM 私有控制面：
  - `hostvgpudeviceapiCtrlCmdGetVgpuTypeInfo_IMPL()` / `vgpuconfigapiCtrlCmdVgpuConfigGetVgpuTypeInfo_IMPL()` 会把 `vgpuName`、`vgpuClass`、`maxResolutionX/Y`、`maxPixels`、`frlConfig`、`cudaEnabled`、`licensedProductName` 等字段原样返回；
  - NVML 的 `apps/nvml/dmal/rm/rm_vgpu.c:368-399` 会把这些字段缓存进 `vgpuTypeStaticInfo`，而 `apps/nvml/api.c:8976-9017, 9167-9168, 9224-9227` 又会把它们继续暴露成用户态直接能看到的 `vgpuTypeName`、`vgpuTypeClass`、最大分辨率和 frame-rate limit；
  - CUDA 的 `drivers/gpgpu/cuda/src/cui/dmal/rm/rm_control_al.c:2364-2371` 还会通过 `NVA080_CTRL_CMD_VGPU_GET_CONFIG` 读取 `cudaEnabled`，再把它折叠成 `vgpuCaps->isCudaCapable`。
- 这意味着 `vupdevid` 真正改掉的，不只是“能不能创建某个 type”，而是 **创建后这个 profile 对外自报的名字、类别、分辨率/像素上限、FRL 配置以及 CUDA capability**。对 guest 和管理工具来说，这已经不是“内部身份漂移”，而是整套 profile 元数据包一起换了。

```c
gpuconfigapiCtrlCmdVgpuConfigGetCreatableVgpuTypes_IMPL(...)
{
    ...
    rmStatus = kvgpumgrGetCreatableVgpuTypes(pGpu, pVgpuMgr, pgpuIndex,
                                            &pParams->numVgpuTypes, pParams->vgpuTypes);
}
```

这段接口级代码很关键，因为它说明 `vupdevid` 最终不只是影响内部判断，而是会一路传到“当前 host 对外公布哪些 creatable profiles”的控制返回值上。
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
- 这条 hook 的真正价值，不在于直接改变 host/guest 功能，而在于它给前面这些复杂链条补了一个**运行时观察窗**：
  - 当 profile 导入、license state、displayless、GSP 或 runlist 相关路径出现异常时，可以更容易看到 RM 内部到底走到了哪类 packet / event；
  - 当 guest 侧只表现成“创建 profile 失败 / mdev 不出现 / 启动后能力异常”时，宿主侧终于有机会把失败点缩到具体 packet/trace 类型，而不是只能看最终报错。
- 因此它更像一条“让后续深挖和复用更可操作”的辅助业务链，而不是解锁本身的主动作点。

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
- 这对 merged driver 很关键，因为它意味着：
  - host 侧 CUDA / compute 相关限制不一定会因为同时引入 GRID/vGPU 语义而被动触发；
  - 某些本应只在“非 CUDA host”场景下走的保守路径，可以被挡在更前面；
  - 最终表现为：同一个 merged 包在保留 guest 能力的同时，更有机会继续保住宿主机侧 CUDA/compute 工作流。

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
- 它最接近的公开源码函数，其实已经可以进一步收窄到：
  - `drivers/resman/src/kernel/virtualization/vgpu_mgr.c:424-476` `vgpuMgrCreateVgpuType`
  - `drivers/resman/src/kernel/virtualization/vgpu_mgr.c:500-589` `vgpuMgrPgpuAddVgpuType`
  - 以及 `kernel_vgpu_mgr.c` 中的同职责版本；
- 这些函数会真正把 `vgpuType`、`maxInstance`、`numHeads`、`maxResolutionX/Y`、`maxPixels`、`frlConfig`、`cudaEnabled`、`license`、`licensedProductName` 等字段挂进 RM 的可用 type 列表；
- 因此 `_nv050770rm` 更像是“在 type 进入 `vgpuMgrCreateVgpuType()` 之前的单条 profile 合法性门”，而不是抽象的任意布尔校验。

### D. 双层等效 patch

**业务层等效 patch：**

- 放宽一条 vGPU 配置 / 条目校验逻辑，使某个布尔判定更容易通过。

**字节层等效 patch：**

```text
0x000BBEF0: 85 -> 31
```

### E. patch 前后业务影响

**patch 前：**

- 这条链本质上是 **host 侧导入 / 注册 vGPU type** 的入口之一：上游 `vgpud` / XML 解析器先把 `vgpuType`、`maxPixels`、`frlConfig`、`license`、`cudaEnabled` 等字段整理成 `NVA081_CTRL_VGPU_INFO` 风格记录，再由 RM 侧循环导入。
- `_nv049279rm` 负责批量吃这些记录，`_nv050770rm` 更像“校验并落单条 profile”的内部 helper。
- 如果这条校验链失败，后果不是简单的一个布尔失败，而是：
  - 某个 vGPU type 不会被加入 host 的可用 type 列表；
  - 后续 `vgpuMgrCreateVgpuType()` / `vgpuMgrPgpuAddVgpuType()` 不会看到这条 profile；
  - 再往后 guest 可创建的 profile、licenseEdition、licensedProductName、maxInstance、FRL 等能力都会缺失。

**patch 后：**

- 这项判断被改写为更宽松的形式，等价于降低“单条 vGPU type 记录必须完全满足某个内部校验”的严格度。
- 业务上，它更接近：
  - **让 host 更容易接受 vGPU profile 描述记录本身**；
  - 从而让本来会被拒掉的 `vgpuType` 仍能被挂进 `vgpuMgrCreateVgpuType()` / `vgpuMgrPgpuAddVgpuType()` 管理的 type 列表；
  - 进而影响 guest 侧最终能看到哪些 profile、license 名称、分辨率/像素上限、FRL、CUDA 能力以及可创建实例数。
- 这意味着 `vgpusig` 影响的是 unlock 链里非常靠前的一步：
  - **profile 有没有被 host 收下**；
  - 而不是 profile 收下之后怎么显示。
- 换句话说，`vgpusig` 不是直接“开功能”，而是先放宽 **profile 元数据导入** 这道门。没有这一步，很多后面的 displayless、licensed feature、migration、CUDA 等调整根本没有对象可作用。

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

- `kunlock` 命中的并不是一条单独的“是否支持”判断，而是一条从 **设备身份识别** 贯穿到 **licensed feature 状态聚合** 的长链：
  1. `_nv032674rm` 这类 helper 先判断当前 `PCIDeviceID / PCISubDeviceID` 是否属于 NVIDIA 官方认可的 GRID/vGPU 软件许可对象；
  2. `gpu_mgr.c` 的 attach/load 路径再把 `gridLicensedFeatures`、`featureCode`、`license state` 等状态挂进 `OBJGPU`；
  3. `grid_features.c` 再基于这些状态决定：
     - guest 能不能看到 vGPU / Quadro / Gaming / Compute licensable feature；
     - baremetal / NMOS 能不能进入 unlicensed state machine；
     - displayless 降级路径、licensed num heads、max resolution、max pixels 应该怎么设置。
- 因此，原始逻辑一旦在前面某个 helper 里给出“这张卡不支持”或“某个 capability bit 为 0”，后面的整条 licensing / displayless / feature 链都会变得保守：
  - `subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL()` 返回的 feature 列表更少；
  - `gpuEnableGridFeature()` / `disableAllGridLicensedFeatures()` 走向更保守的状态；
  - unlicensed state machine 可能根本不会启动，或者启动后状态更受限；
  - 某些 displayless / migration / branding 消费链也会因此继续拒绝当前板卡。

**patch 后：**

- `kunlock` 做的不是“把一个 if 改成 true”这么简单，而是同时把这条长链的 **上游身份门** 和 **中游 capability 聚合门** 一起抬高为更乐观的结果。
- 业务上的连锁效果是：
  - 本来不在官方 GRID/vGPU licensing 白名单里的 SKU，更容易被后续逻辑视为“许可链可继续”；
  - `OBJGPU` 上的 `gridLicensedFeatures` 及其周边布尔状态更容易呈现为“支持 / 已启用 / 可发布”；
  - `grid_features.c` 对 licensable features、unlicensed 模式降级、displayless 派生能力的后续计算都会建立在更乐观的输入之上。
- 对 vGPU unlock 来说，`kunlock` 的意义在于：
  - 它不是直接生成某个 guest profile；
  - 而是让 **“这张宿主卡可不可以被当成 GRID / vGPU 软件许可对象”** 这件事，更早、更广泛地通过，从而给后续所有 feature / displayless / migration 铺路。

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

- `qmode` 所在链路是典型的 **GPU 启动期 registry override 汇总器**：它不直接决定某个 vGPU feature 是否可见，而是决定 RM 初始化后，`OBJGPU` 身上有哪些“偏实验 / 偏调试 / 偏放宽”的运行模式会被打开。
- 这些 key 覆盖的业务面其实很广：
  - `OverrideGpuInit`：影响寄存器 / instmem 初始化覆盖表；
  - `RMDisableFeatureDisablement`：影响是否跳过某些 feature disablement；
  - `RMGpuCacheOnly`：影响 cache-only mode；
  - `RMEnableReplayable`：影响 replayable trace / fault 相关路径；
  - 再加上 power / bandwidth / clock / registry cache / display mux 等一整串初始化状态。
- 这条链的上游输入其实非常朴素：
  - OS / registry 提供一串 override key；
  - `gpuInitOverridesFromRegistry()` 和 `gpuInitRegistryOverrides_IMPL()` 把它们逐条折叠成 `OBJGPU` 成员字段。
- 但它的下游消费非常广：
  - 这些字段会在后面的 display、power、CUDA、GSP、fault、RC recovery、甚至 profile 兼容路径里被继续读取。
- 所以原始逻辑如果更保守，就意味着：
  - RM 启动后仍然更接近默认官方配置；
  - 某些为实验、调试、兼容 consumer/非标准板卡而准备的 override 不容易真正变成对象状态。

**patch 后：**

- `qmode` 的真正业务意义，是让这条“初始化 override 汇总器”对某类条件更宽容，从而更容易把某些 override 变成 `OBJGPU` 上的最终状态。
- 对 unlock 主线的价值不在于它单独开启了 vGPU，而在于它降低了 RM 启动阶段的保守性：
  - 让后续 licensing / displayless / CUDA / GSP / 调试路径更容易在一个“已经放宽约束”的 GPU 对象上继续运行；
  - 减少某些默认 feature disablement 或默认保守初始化对后续 patch 链的反作用。
- 从最终结果看，`qmode` 更像一个 **启动期模式底座补丁**：
  - host 侧启动出来的 RM 对象更接近“愿意配合 unlock 的形态”；
  - guest 侧后面能看到的 feature / profile / displayless 能力，也更少被前面的默认保守初始化提前压掉。

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

- `merged` 所在的不是单纯“是否支持”的抽象判断，而是 **是否允许创建 / 暴露 `NVA083_GRID_DISPLAYLESS` 这条类与能力路径**。
- 在官方逻辑里，这条路径要求几件事同时成立：
  - 当前 GPU 不能被系统当成已有正常 display engine 的板卡；
  - 它要么天然 `bGridCapable`，要么是 `bGridswOnQuadroSupported`，要么显式被 `RmForceGridDisplayless` 打开；
  - 后续还要把 licensed num heads / max resolution / max pixels 这些 displayless 派生能力写回对象状态。
- 这意味着对 merged driver 来说，原始逻辑的约束其实非常强：
  - 如果 host 侧把卡看成普通显示卡，就不会走 GRID_DISPLAYLESS；
  - 如果 licensing / branding 不满足，displayless 这条支线也会被关掉；
  - 最终 guest 侧与 host 侧需要的那组“无物理显示、但仍要提供 vGPU 显示相关 profile 能力”的状态就建立不起来。

**patch 后：**

- `merged` 的业务意义，是把这条 `GRID_DISPLAYLESS` 支持链从“只给官方 GRID / 少数受支持板卡使用”，改成“在 merged driver 场景下更容易被保留下来”。
- 更具体地说，它是在帮 merged driver 同时保住两边语义：
  - **host 侧**：仍然保留原本的 CUDA / OpenGL / console / 初始化链；
  - **guest / GRID 侧**：仍然能得到 displayless class、licensed heads / resolution / pixels 这组派生能力。
- 所以 `merged` 的真正业务作用不是“再多开一个功能”，而是让 merged 产物不要在 host 语义和 GRID displayless 语义之间二选一。

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
- 这对业务层的意义不是“日志不报错了”这么简单，而是会直接影响 **宿主机 software runlist scheduler 对多 VM / 多 vGPU 的承载策略**：
  - `PVMRL` 相关设置不再要求先停掉当前 software runlist；
  - 已经在跑的 VM/vGPU 组合，也能在运行中被重新分配 `swrlCountMax` 与 timeslice；
  - 对最终行为的影响会体现为：可支持的并发 VM 数、每个 VM 的轮转时间片、以及在高负载下 guest 侧感受到的调度公平性和响应性都会变化。
- 对 unlock 场景来说，它解决的是“前面的 profile 和 licensing 都放开了，但宿主调度器还不允许按目标方式承载这些实例”这个更靠后的瓶颈。
- 换成更直观的话说：`swrlwar` 影响的不是 guest 能不能看到 profile，而是 guest 真创建出来以后，宿主还能不能以目标方式把这些实例调度起来。

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
- 这条链发生得非常早：在 `RmInitAdapter()` 里，很多更靠后的 licensing / displayless / GSP / profile 逻辑都还没开始，它就已经决定了显示相关对象要按哪条路径建起来。

**patch 后：**

- 某个条件分支被改写后，初始化阶段对 `fbcon` / displayless` 的处理更接近 vGPU unlock 目标。
- 这类点在业务上很重要，因为它影响的是 **RM 启动早期“显示相关对象到底按哪条世界观初始化”**：
  - 是按“这是一张正常有显示控制器的宿主卡”继续走；
  - 还是按“这张卡需要让 displayless / GRID 派生路径继续保留”去走。
- 对 merged 驱动而言，这类初始化差异最终会反映到：
  - host 侧 console / display common 路径是否还能保住；
  - 同时 guest 侧需要的 displayless / GRID 派生能力是否不会在最早期就被剪掉。
- 所以 `fbcon` 这组虽然只有 1 字节，但它干预的是**初始化时的站位选择**，而不是后面某个小功能开关。
- 如果这个阶段站错位，后面即使 `kunlock` / `merged` / `general` 都放开了，也可能因为显示世界观一开始就错了，导致 host/guest 只保住一边。

---

## 5.7 `sunlock`

### A. 基本信息

- 定义：`patches/nv_hooks.c:276-286`
- 项数：5
- 仓库自带说明：`doc/options.rst:74-76` 直接把 `vup_sunlock` 描述成“based on patch from LIL'pingu fixing xid 43 crashes when running vgpu with consumer cards”，并注明“not needed when vup_kunlock is used”。这条说明很重要，因为它提示这组 patch 的初始动机并不只是“让结果更像 licensed”，还包括**避免 consumer 卡跑 vGPU 时在后续状态消费链里崩掉或掉回错误态**；同时也说明在维护者心里，`sunlock` 更像是一组补救型/兜底型状态消费修正，而不是最核心的解锁主线。

### B. 全部命中二进制函数

`sunlock` 这 5 个 item 当前已确认命中以下二进制函数：

- `0x000D65FA -> 0x0D65BA`：`_nv032674rm`
- `0x000D65C8 -> 0x0D6588`：`_nv032674rm`
- `0x004F6D2A -> 0x4F6CEA`：`_nv019510rm`
- `0x000BECB1 -> 0x0BEC71`：`_nv030355rm`
- `0x000BE72C -> 0x0BE6EC`：`_nv030331rm`

其中需要特别注意的是：两处命中 `_nv032674rm` 的 patch 点都落在函数很前面的 early-gate 区域，发生在后面那段大规模 `deviceId/subdeviceId` 白名单分支之前。也就是说，这两处更像是在改“白名单判断之前是否提前 short-circuit”的前置 gate，而不是直接去改整张官方支持表本身。

其中 `_nv019510rm` 的反编译显示，它会综合：

- `v2[2988]`
- `v2[16634]`
- `nv030882rm()`
- `nv032674rm(v2)`
- `v2[2084] / v2[2085]`

再返回 `0/1/2/3/4` 这类状态码。

这组返回码现在已经可以和公开控制头里的常量直接对上：

- `0 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_UNKNOWN`
- `1 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_UNINITIALIZED`
- `2 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_UNLICENSED_UNRESTRICTED`，也就是 **no capping**
- `3 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_UNLICENSED_RESTRICTED_1`，也就是 **partial capping**
- `4 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_UNLICENSED`，也就是 **full capping**
- `5 = NV2080_CTRL_GPU_GRID_LICENSE_STATE_LICENSED`，也就是 **licensed / no capping**

其中最关键的一点是：`LICENSED = 5` 并不是 `_nv019510rm` 直接产出的分支。源码里的 `subdeviceCtrlCmdGpuEnableGridLicense_IMPL()` 会在真正拿到 license 后，单独把状态推进到 `NV2080_CTRL_GPU_GRID_LICENSE_STATE_LICENSED`。这说明 `_nv019510rm` 更像“**未授权 state machine 当前应该落在哪个阶段**”的分级器，而不是整个 licensing 流程的最终裁决点。

而且这组状态不是静态标签。当前源码里能直接看到一条明确的时序链：

- `gridlicmgrUnlicensedStateMachineStart()` / `gridlicmgrUnlicensedStateMachineRestart()` 会把 vGPU 从 `UNKNOWN` 或 `UNINITIALIZED` 推到 `UNLICENSED_UNRESTRICTED`；
- 之后 1Hz 定时回调 `_gridlicmgrUnlicensedStateMachineTransitionCallback()` 会按超时把状态从 `UNLICENSED_UNRESTRICTED` 推到 `UNLICENSED_RESTRICTED_1`，再继续推到 `UNLICENSED`；
- 这些超时值还不是写死的，`gridlicmgrUnlicensedStateTimeoutValues` 会先取默认值，再允许 registry 覆盖。

所以 `_nv019510rm` 的业务意义，已经可以更具体地理解成：它不是单纯在吐一个“当前状态码”，而是在决定 **当前 guest/host 应该落在未授权状态机的哪一级降级台阶上**。

### C. 源码映射

当前最窄源码函数簇，已经可以比之前更具体地分成四层：

- **上游 gating / feature 背景**
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:gpuEnableGridFeature`
- **状态机与公开状态常量**
  - `drivers/resman/src/kernel/virtualization/grid/grid_license_manager.c:gridlicmgrGetCurrentState`
  - `drivers/resman/src/kernel/virtualization/grid/grid_license_manager.c:gridlicmgrGetFPSValue`
  - `drivers/resman/src/kernel/virtualization/grid/grid_license_manager.c:gridlicmgrGetCUDASleepIntervalValue`
  - `drivers/resman/src/kernel/virtualization/grid/grid_license_manager.c:gridlicmgrTransitionToState`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:subdeviceCtrlCmdGpuGetGridUnlicensedStateMachineInfo_IMPL`
  - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:subdeviceCtrlCmdGpuEnableGridLicense_IMPL`
  - `sdk/nvidia/inc/ctrl/ctrl2080/ctrl2080gpu.h:5160-5175`
- **host / guest license info 写回层**
  - `drivers/resman/src/kernel/virtualization/grid/grid_license_manager.c:gridlicmgrSetLicenseStateCapabilitiesToHost`
  - `drivers/resman/kernel/vgpu/nv/vgpuctrl.c:vgpuSetLicenseInfo`
  - `drivers/resman/kernel/vgpu/nv/hostvgpudeviceapi.c:hostvgpudeviceapiCtrlCmdSetVmLicenseInfo_IMPL`
  - `sdk/nvidia/inc/ctrl/ctrla080.h:709-715`
  - `sdk/nvidia/inc/ctrl/ctrla082.h:194-200`
- **下游品牌 / capability consumer**
  - `drivers/resman/src/physical/gpu/gpu_branding.c:gpuDetectVgxBranding_IMPL`
  - `drivers/resman/src/physical/gpu/gpu_branding.c:deviceCtrlCmdGpuGetBrandCaps_IMPL`

当前置信度可以拆开看：

- 对 `_nv019510rm` 的 `0/1/2/3/4` 返回码与公开 `GRID_LICENSE_STATE_*` 常量的一一对应：**高置信**；
- 对 `_nv030355rm -> hostvgpudeviceapiCtrlCmdSetVmLicenseInfo_IMPL()` 的对应：**高置信**，因为它同时满足三层对位：
  1. 先经 `nv032674rm(v2)` 做与 `isGridLicenseSupported(pGpu)` 对应的 gating；
  2. 再按 `licensed / licenseState / fpsValue / licenseExpiryTimestamp / licenseExpiryStatus` 的布局拷贝字段；
  3. 最后调用 `nv000119rm(v2, 4)`，而源码里的 `NVA081_NOTIFIERS_EVENT_VGPU_GUEST_LICENSE_STATE_CHANGED` 恰好就是 `4`；
- 对 `_nv030331rm` 的单函数源码落点：**中等置信**，当前更像 guest/vGPU 管理结构挂接与后续状态传播的一段承接逻辑；
- 对它们整体落在“unlicensed state machine + host 同步 + 对外 consumer”这条业务链上：**高置信**。

更稳妥的结论是：

- `sunlock` 不是单点 patch，而是一组围绕“是否支持 GRID / 是否 displayless / 是否许可路径可用”的状态消费补丁；
- 它和 `grid_features.c:subdeviceCtrlCmdGpuGetGridUnlicensedStateMachineInfo_IMPL`、`subdeviceCtrlCmdGpuEnableGridLicense_IMPL`、`gridlicmgrUpdateGpuLicenseState()`、`gridlicmgrSetLicenseStateOnHost_Wrapper()` 这一组控制命令 / 状态同步函数处在同一业务域；
- 同时，它最终还会体现在一组公开 getter / 查询面上，例如：
  - `gpuGetGridEnabledFeature()`
  - `gpuGetIsGridLicensed()`
  - `gpuGetIsGridUnlicensedTesla()`
- 换句话说，它更接近“license state / unlicensed state machine / host 可见状态”的消费与回写层，而不是最上游的支持白名单层。
- 更具体地说，源码里 `gridlicmgrSetLicenseStateCapabilitiesToHost()` 会把 `licenseState`、`fpsValue`、`licenseExpiryTimestamp`、`licenseExpiryStatus` 以及 `bvGPUDegradationDisable` 一并打包，再通过 `vgpuSetLicenseInfo()` 同步给 host；这正说明 `sunlock` 所影响的不是抽象布尔量，而是最终会被 host/guest 共同观察到的一组 license / degradation 状态。
- 再往 `_nv030355rm` 的反编译形状看，它现在已经不只是“像某个输出结构写回”，而是高度贴近 `drivers/resman/kernel/vgpu/nv/hostvgpudeviceapi.c:535-580` 这条 host vGPU device license info 写回路径：
  - `licensed -> guestVmInfo.licensed`
  - `licenseState -> guestVmInfo.licenseState`
  - `licenseExpiryTimestamp -> guestVmInfo.licenseExpiryTimestamp`
  - `licenseExpiryStatus -> guestVmInfo.licenseExpiryStatus`
  - `if (licenseState != UNKNOWN) guestVmInfo.frameRateLimit = fpsValue`
  - 最后再触发 `NVA081_NOTIFIERS_EVENT_VGPU_GUEST_LICENSE_STATE_CHANGED`
- 这说明 `_nv030355rm` 更像 **host 侧把 per-VM license / frame-rate 结果真正落库并广播事件** 的 consumer，而不只是泛泛的“某个输出结构”。

### D. 双层等效 patch

**业务层等效 patch：**

- 不单纯是“所有地方都更宽松”，而是把若干原本会落到 `UNKNOWN / unsupported / 早退失败` 的状态消费路径，重新导向 **仍可传播、仍会被 host/guest 继续消费的降级态**，从而避免 consumer 卡在后续 licensing / displayless / guest 挂接链里直接崩掉或只剩空结果。

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

- `sunlock` 这组 patch 所在的位置，已经不是最上游的“这卡支不支持”判断，而是 **更靠后的状态消费链**：
  - 有的地方在把 displayless / capability 组合折叠成 `0/1/2/3/4` 这类状态码；
  - 有的地方在把 enable/license 结果写回某个输出结构；
  - 有的地方在把这些结果继续挂接到 vGPU / GRID 管理结构里。
- 换句话说，到了 `sunlock` 这一步，系统往往已经知道：
  - 这张卡大概是什么；
  - 上游许可链是否倾向支持；
  - displayless / branding / feature state 当前是什么。
- 原始逻辑在这里依然可能“临门一脚收紧”：
  - 虽然前面已经部分放行，但只要消费链里某个状态码、某个模式位、某个输出结构字段仍不满意，下游控制命令、license state、displayless 派生状态还是会退回或置零。

**patch 后：**

- `sunlock` 的业务意义，不宜简单理解成“所有地方都更乐观”。更贴近当前证据的说法是：它会把若干 **中下游消费点** 重新布线，让结果尽量落到“仍可继续传播的降级态”，而不是直接掉回 `UNKNOWN / unsupported / 早退失败`。
- 这也是它和 `kunlock` 最不同的地方：
  - `kunlock` 更像是在前面把门打开；
  - `sunlock` 则更像是在后面决定：如果这扇门不能以“官方完全支持”的姿势通过，那至少要把结果导向一个 **受限但仍可消费** 的状态机档位，而不是直接崩掉或只剩空结果。
- 这和 `doc/options.rst:74-76` 把 `vup_sunlock` 直接描述成“fixing xid 43 crashes when running vgpu with consumer cards”是吻合的：它的目标不只是“看起来更像支持”，还包括**把 consumer 卡从危险的失败态导回可承受的降级态**。
- 更具体地说，它影响的是三类“最终对外可见”的结果：
  1. **unlicensed state machine 档位本身**
     - 这里已经不是抽象的 mode code，而是公开定义好的 license state：
       - `0 = UNKNOWN`
       - `1 = UNINITIALIZED`
       - `2 = UNLICENSED_UNRESTRICTED`
       - `3 = UNLICENSED_RESTRICTED_1`
       - `4 = UNLICENSED`
     - 这意味着 `_nv019510rm` 实际上是在决定 guest / host 当前会被视为“未知、未初始化、未授权但不限速、部分限速、完全限速”中的哪一档；
  2. **由 state 派生出来的性能/降级参数**
     - `gridlicmgrGetFPSValue()` 会把这些状态进一步翻译成具体 FRL：
       - `2` 走 licensed FPS 值，也就是“未授权但仍不降帧”；
       - `3` 会把 FPS 压到 `min(licensedFPS, 15)`；
       - `4` 会压到 `3 FPS`；
       - `0/1` 则返回 unbounded；
     - `gridlicmgrGetCUDASleepIntervalValue()` 还会把 `3/4` 这两档进一步翻译成 `20 ms` CUDA enforced delay，而其他状态为 `0`；
  3. **被同步/暴露出去的结构化状态**
     - `gridlicmgrSetLicenseStateCapabilitiesToHost()` 会把 `bvGPUDegradationDisable + licenseState + fpsValue + licenseExpiryTimestamp + licenseExpiryStatus` 一起同步给 host；
     - `_nv030355rm` 的字段拷贝形状又说明，这些值后面会继续被写进 host vGPU device 侧的 per-VM license info 对象；
     - `hostvgpudeviceapiCtrlCmdSetVmLicenseInfo_IMPL()` 还会把 `fpsValue` 落成 `guestVmInfo.frameRateLimit`，并触发 `NVA081_NOTIFIERS_EVENT_VGPU_GUEST_LICENSE_STATE_CHANGED`；
     - `vgpu_mgr.c:2549-2572` 与 `kernel_vgpu_mgr.c:2404-2427` 还会在 guest 迁移/复制路径里继续保留 `guestVmInfo.licenseState` 与 `guestVmInfo.frameRateLimit`，并在“unknown + unlicensed”场景下把 frame-rate 结果钳成 `3 FPS`；
     - `vgpuapiCtrlCmdVGpuGetConfig_IMPL()` 还显示出同一 licensing 域存在另一条 guest-facing framerate consumer：当 `isGridLicenseSupported(pGpu)` 为真但 `PDB_PROP_GPU_IS_GRID_VGPU_ENABLED` / `PDB_PROP_GPU_IS_GRID_LICENSED` 不满足时，它会直接把 `frameRateLimiter` 改成 `0xE0400003`；这说明 guest 最终读到的限速并不只存在于内部 state，而是会在配置返回值里落成具体 FRL 掩码；
     - `NV2080_CTRL_GPU_GRID_UNLICENSED_STATE_MACHINE_INFO_PARAMS` 本身就把 `currentState / fpsValue / cudaSleepInterval / licenseExpiryTimestamp / licenseExpiryStatus` 定义成公开控制返回值，这说明这组状态不是 RM 私货，而是原本就打算直接给 guest/userland 消费；
     - 这些位一旦归零或落到更保守的 state，前面虽然放行了，后面仍可能表现成“guest 看不见 / host 不发布 / state machine 继续降级”。
- 对最终业务行为的影响是：
  - guest / host 看到的已经不是模糊的“许可更乐观”，而是 **当前到底处于 unrestricted、15 FPS partial capping，还是 3 FPS full capping**；
  - `gridlicmgrSetLicenseStateOnHost_Wrapper()` 一类宿主同步路径更不容易把结果重新压回保守态；
  - 某些依赖这些状态码的控制命令和后续对象挂接路径，不会再因为官方 SKU / branding / displayless 组合不标准而回退。
- 从外部可见效果来看，`sunlock` 更接近“把已经放行的结果真正做实”：
  - host 侧不只是理论上支持，而是真的持有一组更乐观的 `licenseState / fpsValue / degradation` 字段；
  - guest 侧不只是 profile 存在，而是 profile 对应的 FRL、CUDA enforced delay、license state 也更不容易在后续查询里退回到保守值。
- 结合当前源码与反编译形状，更具体地说：
  - `_nv032674rm` 里的两处 `sunlock` patch 需要合在一起看：`0x000D65C8 -> 0x0D6588` 把一条 `jnz +0x2E` 改成了 `jmp +0x2E`，而 `0x000D65FA -> 0x0D65BA` 又把落点处 `mov r13d, 1` 的立即数改成了 `0`；两者组合后，等效于把这条早退路径从“直接 `return 1`”改成了“直接 `return 0`”。更关键的是，这两处都发生在 `_nv032674rm` 进入大段 `deviceId/subdeviceId` whitelist 之前，因此它们更像是在改 **前置 short-circuit gate**。单独看它会显得更保守，但和下面 `_nv019510rm` 的 `0 -> 3` 配合起来看，它更像是在把某类原本会落到 `unsupported/unknown` 的路径，重新导向“受限但仍可继续”的 unlicensed state machine 分支；
  - `_nv019510rm` 不只是“把一组上游 capability 组合折叠成 state code”，而且在字节级上已经能看到：`0x004F6D2A -> 0x4F6CEA` 命中的正是一条 `mov dword ptr [rbx], 0` 的立即数字段，patch 后等效于把这里改成 `mov dword ptr [rbx], 3`；也就是说，在原本会回落到 `UNKNOWN(0)` 的分支上，补丁直接把结果推成了 `UNLICENSED_RESTRICTED_1(3)`；
  - `_nv030355rm` 不只是“写回某个输出结构并触发一次后续通知/刷新”，而且 `0x000BECB1 -> 0x0BEC71` 命中的正是 `jz +0x42` 的位移字节；把它改成 `0x00` 之后，等效于让这条“license unsupported 就返回 `86`”的早退跳转失效，后续 `guestVmInfo` 写回与 `LICENSE_STATE_CHANGED` 事件得以继续发生；
  - `_nv030331rm` 不只是“把这组结果继续挂接到更上层资源或管理结构”，而且 `0x000BE72C -> 0x0BE6EC` 命中的正是 `or dword ptr [...], 0x10` 的立即数字段；把 `0x10` 改成 `0x00` 后，等效于阻止这条后续 guest/resource 挂接路径再额外打上一个保守 flag。
- 这也解释了为什么 `sunlock` 的整体效果不像“把一切都伪造成 licensed”。更贴近当前证据的说法是：它把若干原本会掉进 `UNKNOWN / unsupported / 直接早退` 的路径，重新分流到 **`UNLICENSED_RESTRICTED_1` / `UNLICENSED` 这种仍可继续传播、仍会被上层消费的降级档位**，从而让 host/guest 后面的 consumer 有状态可读，而不是直接只剩一个失败或空结果。
- 再往源码侧对照，`gridlicmgrSetLicenseStateCapabilitiesToHost()` / `gridlicmgrSetLicenseStateOnHost_Wrapper()` 会把 `licenseState`、`fpsValue`、`licenseExpiryTimestamp`、`licenseExpiryStatus` 这类字段同步回 host；`gpuGetGridEnabledFeature()`、`gpuGetIsGridLicensed()`、`gpuGetIsGridUnlicensedTesla()` 则会把这些状态继续暴露给后续查询者。
- 这些 getter 和同步状态并不是“摆在那里不用”的。当前源码已经能看到至少七类直接 consumer：
  - `subdevice_ctrl_gpu_kernel.c` 会据此决定 `NV2080_CTRL_GPU_INFO_INDEX_TESLA_ENABLE` 一类对外暴露状态；
  - `kernel_graphics.c` 会据此裁剪或保留 Quadro / VGX / GeForce 相关 graphics caps；
  - `gpu_branding.c` 会据此修正最终对外暴露的品牌位；
  - `apps/nvml/dmal/rm/rm_nvml.c:268-299` 与 `apps/nvml/dmal/wddm/wddm_nvml.c:278-303` 会通过 `NV2080_CTRL_GPU_INFO_INDEX_TESLA_ENABLE` 再把这些结果折叠成 NVML 看到的产品品牌；
  - `apps/nvml/dmal/rm/rm_mappings.c:229-238` 与 `rm_nvml.c:1453-1458` / `wddm_nvml.c:1603-1608` 还会把 RM 的 `GRID_LICENSE_STATE_*` 一一映射成 `NVML_GRID_LICENSE_STATE_*`，所以 `sunlock` 不只影响“品牌像什么”，还会直接影响 NVML 查询到的 license state；
  - `hostvgpudeviceapiCtrlCmdSetVmLicenseInfo_IMPL()` 触发的 `NVA081_NOTIFIERS_EVENT_VGPU_GUEST_LICENSE_STATE_CHANGED` 又会经 `apps/nvml/dmal/rm/rm_vgpu_event.c:214-215` 映射成 `nvmlVgpuEventTypeGuestLicenseStateChanged`，说明这组变化不仅能被轮询查到，还能被 NVML 的 vGPU 事件流直接观察到；
  - `drivers/gpgpu/cuda/src/cui/dmal/rm/rm_control_al.c:2329-2336` 与 `cuiinit.c:4109-4116` 会把 `currentState + cudaSleepInterval` 直接翻成 `CUDA_ERROR_DEVICE_NOT_LICENSED` 与实际 `cuosSleep()` 延迟；
  - `drivers/xfree86/4.0/nvidia/disp/virtual/nv_virtual.c:1094-1112` 会按 `fpsValue` 更新 guest 显示侧 frame-rate limiter，而 `nv_virtual.c:1170-1178` / `1364-1375` 还会在 license-state change 事件后重查显示参数与 maxPixels。
  - 这意味着 `sunlock` 最终不只影响 RM 自己的内部世界观，还会影响用户态工具、CUDA 调度，以及 guest 显示栈看到的限速和配置刷新。

```c
// subdevice_ctrl_gpu_kernel.c
if (gpuGetIsGridUnlicensedTesla(pGpu->gpuInstance) ||
    (gpuGetGridEnabledFeature(pGpu->gpuInstance) == NV_GRID_LICENSE_FEATURE_CODE_COMPUTE))
{
    data = 1;
}

// kernel_graphics.c
if (gpuGetIsGridUnlicensedTesla(pGpu->gpuInstance))
{
    RMCTRL_CLEAR_CAP(... _VGX);
    RMCTRL_CLEAR_CAP(... _GEFORCE_SMB);
}
```

这两段 consumer 说明，`sunlock` 改出来的不是“内部状态更乐观”这么抽象的结果，而是会直接改写：

- 外部查询到的 `TESLA_ENABLE`；
- 最终图形 capability 表里还保留哪些品牌/产品位。

而且 `TESLA_ENABLE` 在这里也不是一个孤立标志位。`subdevice_ctrl_gpu_kernel.c` 会先取物理品牌 `gpuIsTeslaBranded(pGpu)`，再用 `gpuGetIsGridUnlicensedTesla()` 与 `gpuGetGridEnabledFeature()` 覆盖它：

- `COMPUTE` 或 `unlicensedTesla` 会把结果强行推到 `1`；
- `QUADRO` / `GAMING` 会把结果压回 `0`。

这意味着 `_nv019510rm` / `_nv030355rm` / `_nv030331rm` 这组 `sunlock` patch 所守住的，并不只是“许可状态本身”，而是 **许可状态如何被重新解释成对外品牌/产品类别结论**。换句话说，license state machine 产出的中间态，到了这一步会再次被折叠成一个更粗粒度、但对外更常被消费的产品判断位。

- 这意味着 `sunlock` 最终影响的不只是内部布尔量，而是 guest/host 在控制查询里真正能看到的结果：
  - 某些查询会把卡看成 Tesla / Compute 倾向，还是 Quadro / Gaming 倾向；
  - graphics caps 里哪些品牌/能力位最终对外可见；
  - host 侧同步出去的 license / degradation 信息是否还保持保守值。
- 这些结果并不只存在于 RM 私有接口里。当前源码已经能看到几类直接 consumer：
  - `NV2080_CTRL_GPU_INFO_INDEX_TESLA_ENABLE` 会经 `subdevice_ctrl_gpu_kernel.c` 暴露出去；
  - NVML 的 `rm_nvml.c:276-299` / `wddm_nvml.c:286-305` 会再根据这个结果，把非 vGPU 场景下的产品品牌折叠成 `NVML_BRAND_TESLA` 或 `NVML_BRAND_NVIDIA`；
  - `rm_mappings.c:229-238` 与 `rm_nvml.c:1453-1458` / `wddm_nvml.c:1603-1608` 还会把 RM 的 `GRID_LICENSE_STATE_*` 一一映射成 `NVML_GRID_LICENSE_STATE_*`，所以 `sunlock` 不只影响“品牌像什么”，还会直接影响 NVML 查询到的 license state；
  - `hostvgpudeviceapiCtrlCmdSetVmLicenseInfo_IMPL()` 触发的 `NVA081_NOTIFIERS_EVENT_VGPU_GUEST_LICENSE_STATE_CHANGED` 又会经 `apps/nvml/dmal/rm/rm_vgpu_event.c:214-215` 映射成 `nvmlVgpuEventTypeGuestLicenseStateChanged`，说明这组变化不仅能被轮询看到，还能被事件流直接观察到；
  - `drivers/gpgpu/cuda/src/cui/dmal/rm/rm_control_al.c:2329-2336` 与 `cuiinit.c:4109-4116` 会把 `currentState + cudaSleepInterval` 直接翻成 `CUDA_ERROR_DEVICE_NOT_LICENSED` 与实际 `cuosSleep()` 延迟；
  - `drivers/xfree86/4.0/nvidia/disp/virtual/nv_virtual.c:1094-1112` 会按 `fpsValue` 更新 guest 显示侧 frame-rate limiter，而 `nv_virtual.c:1170-1178` / `1364-1375` 还会在 license-state change 事件后重查显示参数与 maxPixels；
  - `kernel_graphics.c` 会裁剪 Quadro / VGX / GeForce 相关图形 capability；
  - `gpu_branding.c` 会据此修正最终对外暴露的品牌位。
- 于是从完整链条看，`sunlock` 影响的是：
  1. `_nv019510rm` 一类内部 helper 先把上游 capability / license 组合压成 state code；
  2. `_nv030355rm` / `_nv030331rm` 再把这组结果写回并挂接到更高层状态；
  3. `gridlicmgrSetLicenseStateOnHost_Wrapper()` 把结果同步回 host，并经 host vGPU device API 落成 `guestVmInfo.licenseState / frameRateLimit` 与 `LICENSE_STATE_CHANGED` 事件；
  4. `subdeviceCtrlCmdGpuGetGridUnlicensedStateMachineInfo_IMPL()` 再把这组状态暴露成公开的 `currentState / fpsValue / cudaSleepInterval` 控制返回值；
  5. NVML、CUDA 和 guest 显示栈分别把它消费成 `NVML_GRID_LICENSE_STATE_*`、`CUDA_ERROR_DEVICE_NOT_LICENSED` / `cuosSleep()` 延迟，以及 frame-rate limiter / maxPixels 刷新；
  6. `NV2080_CTRL_GPU_INFO_INDEX_TESLA_ENABLE`、NVML brand、graphics caps、brand caps 再把同一组中间态折叠成用户真正能看到的“这张卡算哪一类产品”。
- 所以 `sunlock` 作用的不是单个业务点，而是**状态码生成 → 输出结构写回 → host 同步 → 公开控制返回 → NVML/CUDA/guest 显示栈消费 → 品牌与 capability 折叠** 这一整段尾部承接链。

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
- 已确认 caller：`rm_set_rm_firmware_requested() @ 0xAE5FE0`

反编译显示它是一个很小的布尔 helper：

```c
*a6 = (a4 & 0x10) != 0;
result = nv026921rm(a1, a2, a3, v9 + 15);
...
*a5 = result;
```

这条 helper 现在已经可以更精确地放回 caller `rm_set_rm_firmware_requested()` 里理解：

- `*a5` 更接近后续是否请求 firmware 的候选结果；
- `*a6` 保留了来自 `a4 & 0x10` 的另一位策略输出；
- 调用点就发生在 `rm_set_rm_firmware_requested()` 里正式写入 `nv->request_firmware` / `nv->allow_fallback_to_monolithic_rm` 之前。

也就是说，它不是泛泛的 capability 检查，而是 **直接参与“要不要请求 firmware、允不允许后续回退”这组宿主启动路线参数的生成**。

更关键的是，`nv_hooks.c` 里的 patch 偏移是按 `blob - 0x40` 坐标系写的，所以：

```asm
0x33EA2  test al, al
0x33EA4  jz   short loc_33EC1
0x33EA9  mov  eax, 1
0x33EAE  cmp  ebx, 1
0x33EB1  jz   short loc_33EC1
...
0x33EC1  mov  [r12], al
```

- `0x00033EE5` 实际命中的是 IDA 里的 `0x33EA5`，也就是第一条 `jz short loc_33EC1` 的位移字节；把 `0x1B` 改成 `0x00` 后，这条“helper 失败就直接跳到尾部”的短跳等效于被中和掉；
- `0x00033EF1` 实际命中的是 IDA 里的 `0x33EB1`，也就是第二条 `jz short loc_33EC1` 的 opcode；把 `0x74` 改成 `0xEB` 后，这条条件跳转被改成了无条件跳转。

两者组合起来，效果已经不只是抽象的“更容易继续”，而是 **在 helper 里直接去掉一条失败短路，再把后一条成功分支改成无条件成功分支**。从现有反汇编路径看，只要执行到这组指令，`*a5` 在 helper 内部就会被强制写成成功值 `1`：前面的 `test al, al` 失败不再能把路径短路掉，后面的 `cmp ebx, 1` 也不再能把结果拉回条件分支。换句话说，这组 patch 改的不是“返回 1 的概率更高”这么弱的效果，而是 **把这条前置 helper 的 request-firmware 候选结果几乎硬推成了真**；只是 caller 在 helper 之后仍然可能基于更高层的 platform/policy 条件把最终结果重新压回保守值。

### C. 源码映射

当前最窄源码函数簇：

- `drivers/resman/src/physical/gpu/gsp/gsp.c:405-449`
- `drivers/resman/arch/nvalloc/unix/src/osapi.c:4766-4876` `rm_set_rm_firmware_requested`
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

当前置信度：**中等**。更准确地说：对 caller `rm_set_rm_firmware_requested()` 的对应是高置信；对它在源码里更接近 `gsp.c` 哪个 helper 语义、以及 `nv026921rm()` 具体对应哪一段小 helper 链，仍是中等。

### D. 双层等效 patch

**业务层等效 patch：**

- 不只是泛泛地“放宽一个 helper”，而是把 `rm_set_rm_firmware_requested()` 前面的一个 firmware-request 候选生成 helper 改成：
  - 去掉一条失败短路；
  - 再把后一条条件成功分支改成无条件成功分支；
- 从业务效果看，它更接近 **强行保住“继续考虑 firmware / GSP 路线”的候选资格**，而不是在最前面的 capability helper 阶段就被否掉。
- 但这仍不是最终裁决：后面的 `bFirmwareCapable`、registry/policy、firmware fetch、firmware validation、`kgspInitRm()` 以及 fallback 逻辑依旧会继续决定宿主最终是成功进入 GSP、回退到 monolithic RM，还是直接初始化失败。

**字节层等效 patch：**

```text
0x00033EE5: 1B -> 00   ; neutralize the first short-jump after `test al, al`
0x00033EF1: 74 -> EB   ; turn the second `jz` after `cmp ebx, 1` into unconditional jump
```

把这两项放回 `_nv026896rm` 的局部控制流里看，它们分别对应：

- 不再因为 `nv026921rm(...)` 给出的失败结果而立刻短路到尾部；
- 不再要求 `ebx == 1` 这个附加条件成立才进入成功落点。

### E. patch 前后业务影响

**patch 前：**

- `gspvgpu` 所在链条更像 **GSP 固件请求 / GSP capability 裁决** 的前置 helper，而不是最后的固件装载函数本身。
- 它的上游输入通常来自两类信息：
  - 当前是否处在 vGPU / virtualization 场景；
  - 当前 display / GSP / firmware 相关 capability bit 是否满足。
- 在现代 RM 里，GSP 是否请求、是否允许、是否因为 displayless / inst_in_sys / virtualization 环境而被跳过，会直接影响后面整条初始化路径：
  - 是否走 firmware client RM；
  - 是否允许某些 display / HDCP / capability 路径继续；
  - 在 vGPU / displayless 组合场景下，是进入“继续初始化”，还是更早被判成“不允许”。
- 原始逻辑一旦在这里给出否定结果，后面的 GSP 相关初始化就会更保守，甚至根本不走。

**patch 后：**

- `gspvgpu` 的业务意义，是削弱这一前置 helper 的否定分支，让系统在 vGPU / displayless 组合下更容易继续走 GSP 相关初始化链。
- 更精确地说，它不是直接替宿主把 `request_firmware` 最终写成真，而是先在 helper 层把“请求 firmware 的候选结果”硬推向成功；后面的固件请求、skip-load、displayless 特判以及 fallback 逻辑仍然会继续生效。
- 因此它的价值更偏向：
  - **减少 GSP 前置裁决过早否决 unlock 场景的概率**；
  - 而不是直接替代 `gsp.c` 里的真正固件装载与 skip-load 逻辑。
- 从最终结果看，它影响的是：
  - host 侧初始化是否有机会真正进入后续 GSP 相关分支；
  - 某些 guest 所依赖的 display / capability 路径，是否在最前面的 capability helper 阶段就被提前砍掉。
- 结合 caller `rm_set_rm_firmware_requested()` 的源码，这条链最终会直接影响两个宿主侧决策：
  1. `nv->request_firmware` 会不会被置成真，也就是后面是否尝试进入 firmware client RM / GSP 路径；
  2. `nv->allow_fallback_to_monolithic_rm` 的策略是否还有机会生效，也就是失败后还能不能回退到 monolithic RM。
- 再具体一点说，这意味着它会影响宿主启动时的路线选择：
  - 是完全停留在 monolithic RM；
  - 还是至少尝试走一次 firmware client RM / GSP 路线，再由后面的 displayless / skip-load / fallback 逻辑决定能否真正落地。
- 这条链在 `osinit.c` 里还会继续体现成两个紧接着的分叉：
  - `nv->request_firmware` 为真时，先决定是否取 `gsp.bin` 并把 `nv->request_fw_client_rm` 置位；
  - 之后再由 `kgspInitRm()` 真正尝试进入 GSP client RM，失败时根据 `nv->allow_fallback_to_monolithic_rm` 决定是 hard fail 还是退回 monolithic RM。
- 因而 `gspvgpu` 的最终业务意义，不只是“某个 helper 更容易返回 true”，而是它会改变 **宿主驱动到底有没有资格进入 GSP 请求路径**，以及失败时是“直接终止”还是“有资格回退”。
- 从 host/guest 的最终可见结果看：
  - host 侧会更早决定是否把当前 GPU 归入“尝试 firmware client RM / GSP”的路线；
  - 如果这条路线根本不尝试，guest 后面依赖的一些 capability / display 相关路径连进入机会都没有；
  - 如果这条路线被放行，后面即使仍可能因为 displayless 特判或 skip-load 回退，至少已经从“前置 helper 直接否决”升级成“真正进入 GSP 路线再决定成败”；
  - 宿主机上连 `/proc` 暴露面的 `GPU Firmware` 字段也会受这条路线影响：`rm_get_firmware_version()` 只有在 `request_firmware` 路径成立时才会返回固件版本或 `N/A`，否则该字段直接为空，不会被 `nv-procfs` 打印出来；
  - 换句话说，`gspvgpu` 不只是影响内部初始化岔路，连宿主机最终给用户看的“这张卡有没有走到 firmware/GSP 路线”都能留下直接痕迹。
- 再压到更具体的宿主可见失败结果，`osinit.c` 已经把几类 GSP 失败路径拆得很清楚：
  - 取不到 `gsp.bin` 且禁止回退时，会落成 `RM_INIT_FIRMWARE_FETCH_FAILED`；
  - `gsp.bin` 哈希 / 版本校验不过时，会落成 `RM_INIT_FIRMWARE_VALIDATION_FAILED`；
  - `kgspInitRm()` 真正起 GSP client RM 失败时，会打印 `Cannot initialize GSP firmware RM`，并落成 `RM_INIT_FIRMWARE_INIT_FAILED`；
  - 已经请求了 firmware client RM、但中途没能真正启用，而且又禁止回退时，会落成 `RM_INIT_FIRMWARE_POLICY_FAILED`；
  - 只有在允许回退时，宿主日志里才会明确出现 `Falling back to monolithic RM`。
- 因此从宿主机运维视角看，`gspvgpu` 带来的不是单一“成功/失败”两态，而更像三态：
  1. **根本没走 firmware 路线**：`/proc` 里没有 `GPU Firmware:` 字段；
  2. **尝试走了 firmware 路线，但没真正成功**：`/proc` 里字段存在但可能是 `N/A`，同时日志或 init status 会落到 fetch/validation/init/policy 失败之一；
  3. **真正走通了 GSP 路线**：`/proc` 里能看到实际 firmware version。
- 更关键的是，同样是“GSP 路线失败”，在允许回退和禁止回退两种策略下，宿主机最终行为完全不同：
  - **允许回退**时，失败会表现成日志里的 `Falling back to monolithic RM`，驱动仍可能继续起来；
  - **禁止回退**时，同类失败会直接落成 `RM_INIT_FIRMWARE_FETCH_FAILED / VALIDATION_FAILED / INIT_FAILED / POLICY_FAILED` 之一，驱动初始化在 `shutdown` 路径终止。
- 甚至在“主流程没挂，但 GSP 附带资源不完整”的情况下，宿主机也会留下直接痕迹：`gsp_log.bin` 缺失不会让初始化失败，但日志里会明确打印 `Failed to load gsp_log.bin, no GSP-RM logs will be printed (non-fatal)`。这说明 `gspvgpu` 不只影响“能不能进 GSP”，还会影响宿主机后续有没有足够的 GSP 诊断可见性。
- 再往实处说，`gspvgpu` 影响的不是抽象“内部路线”，而是宿主机真正能观察到的一组外部信号：
  - init status 是否落到 `RM_INIT_FIRMWARE_FETCH_FAILED / VALIDATION_FAILED / INIT_FAILED / POLICY_FAILED`；
  - 内核日志里是否出现 `Cannot initialize GSP firmware RM` 或 `Falling back to monolithic RM`；
  - `/proc/driver/nvidia/gpus/*/information` 一类信息面里，`GPU Firmware:` 字段是不存在、是 `N/A`，还是一个真实版本串。
- 因此从运维与排障视角看，`gspvgpu` 的价值不只是“让某段 capability helper 更容易放行”，而是它会直接改变宿主机在 **启动是否成功、日志长什么样、/proc 暴露什么固件状态** 这三类信号上的最终表现。

```c
// osapi.c
nv->request_firmware =
    (bFirmwareCapable &&
     ((rmFirmwareMode == NV_FIRMWARE_MODE_ENABLED) ||
      (bEnableByDefault && (rmFirmwareMode != NV_FIRMWARE_MODE_DISABLED))));

// nv-procfs.c
rm_get_firmware_version(sp, nv, firmware_version, sizeof(firmware_version));
if (firmware_version[0] != '\0')
{
    seq_printf(s, "GPU Firmware:\t %s\n", firmware_version);
}
```

这说明 `gspvgpu` 的影响不只体现在“内部是否去调 `kgspInitRm()`”，还会直接体现在宿主机对外可见的 firmware 路线、失败码、日志以及 firmware 信息暴露上。
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

- `general` 组不是简单把 KVM 主线 copy 过去，而是把 GRID build 真正在意的两条业务链一起放宽：
  1. **支持判定链**：先借用 `kunlock` 同款的 `isGridLicenseSupported()` 放宽，让当前板卡先别在入口被踢掉；
  2. **运行态承接链**：再在 `rm_is_vgpu_supported_device()` 与 displayless / licensed capability 消费链上放宽，让这张卡在 GRID/general 包里真正进入“可承接 profile / displayless class / unlicensed state machine”的运行态。
- 这条链的业务位置其实很靠前：`rm_is_vgpu_supported_device()` 处于宿主驱动判断“当前板卡是否属于可接受的 vGPU / GRID host 设备”的早期路径，如果这里仍然拒绝，后面根本不会有 profile 发布、displayless class 建立或 license state machine 启动的机会。
- 这里的“承接链”不是抽象概念，而是会真实落到：
  - `gridlicmgrUnlicensedStateMachineInit()`：vGPU 场景里一旦 feature type 已知，会直接启动 unlicensed state machine；
  - `gridlicmgrUnlicensedStateMachineStart()`：把状态从 `UNKNOWN / UNINITIALIZED` 推到 `UNLICENSED_UNRESTRICTED`；
  - `gridlicmgrSetLicenseStateOnHost_Wrapper()`：把这组结果同步回 host；
  - displayless capability 消费链：继续把 licensed heads / resolution / pixels 这组约束挂到对象上。
- 这意味着 `general` 的业务目标，比 `kunlock` 更偏向“让 GRID build 真能跑起来”：
  - 不是只回答“这卡理论上支不支持”；
  - 而是继续回答“既然前面说支持了，后面能不能把 vGPU register、displayless class、license state machine 和 profile 约束一起接起来”。
- 对最终业务结果来说，`general` 更接近“把 host 侧可运行状态补全”的补丁包：
  - 让 host 不只是接受这张卡，还能继续导出 vGPU register 与 profile 元数据；
  - 让 displayless class 与 licensed heads / resolution / pixels 这组 guest 侧会直接看到的约束真正落地；
  - 让 unlicensed state machine 在需要的时候真的启动并把状态同步回 host，而不是停在一个“理论支持但运行态没承接起来”的半开状态。

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
- `vgpusig` 落在 `vgpu_mgr.c / kernel_vgpu_mgr.c` 的 host-vGPU profile 导入链，最接近 `vgpuMgrCreateVgpuType / vgpuMgrPgpuAddVgpuType` 这类函数
- `general` 的 `_nv028908rm` 落在 `grid_license_manager.c` 与 displayless capability 消费链的交界处，更偏向 state machine / capability 承接逻辑，而不是最上游支持判定

### 低到中等置信

- `sunlock` 的三处辅助函数
- `gspvgpu`

这些点当前都已经可靠落到了正确的**业务子系统**，但并非每一项都能仅靠当前公开源码 100% 压到单个公开函数。对这些点，本文已经按“最窄源码函数簇 / 调用链”方式保守表达。