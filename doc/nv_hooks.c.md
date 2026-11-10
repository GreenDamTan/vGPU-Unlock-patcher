# Linux 580：nv_hooks.c 的运行时修改与源码对应

## 1. 作用与适用版本

[`patches/nv_hooks.c`](../patches/nv_hooks.c) 在 `nvidia` 模块初始化时改写已加载的闭源 RM 代码，包含三个 CALL hook、KVM 分支的八组字节补丁，以及非 KVM 分支的 `general` 补丁。\
它不是用户态预加载库，也不是在磁盘上改写驱动包的工具。

本文对应当前实现的两种 RM 布局：

| 构建分支 | RM 输入 | `RM_IOCTL_OFFSET` | 适用入口 |
| --- | --- | --- | --- |
| 定义 `NV_VGPU_KVM_BUILD` | `580.196.09-vgpu-kvm` | `0xd67cf0` | 普通 KVM，以及采用 `.09` RM 的 merged |
| 未定义 `NV_VGPU_KVM_BUILD` | `580.196.07` GRID/general | `0xd67c80` | GRID、桌面 general 和从 GRID 派生的 general |

merged 的显示 wrapper、桌面用户态和版本头来自 `.07`，RM、vGPU 组件及固件来自 `.09`。\
所以版本字符串仍为 `.07` 并不意味着 runtime hook 应使用非 KVM 的 RM 基准。\
非 KVM 的判断也不能换成仅检查 `GRID_BUILD`，因为 GRID 派生 general 会移除 GRID 标志，但仍保留 `.07` RM 布局。

下文按模块、文件名和函数／宏列出源码对应，并摘录关键逻辑。\
目标 RM 中的私有字段使用寄存器与对象位移定位。

## 2. 接入、地址基准与写入机制

### 2.1 生成流程接入和调用顺序

[`patch.sh`](../patch.sh) 把本文件复制为输出驱动的 `kernel/unlock/nv_hooks.c`，并在闭源内核构建中加入：

```make
NVIDIA_SOURCES += unlock/nv_hooks.c
OBJECT_FILES_NON_STANDARD_nv_hooks.o := y
```

安装清单同时登记该内核模块源文件。\
merged 的构建额外定义 `VUP_MERGED_DRIVER=1`，未定义时本文件将其设为 `0`。\
这个宏只决定这里 `cudahost` 和 `vup_merged` 的默认值，不控制生成阶段的 NVML 二进制补丁。

[`setup-vup-hooks.patch`](../patches/setup-vup-hooks.patch) 在 `nv.c` 中声明并接入：

```c
extern void vup_hooks_init(void);
extern void vup_hooks_exit(void);
```

`nvidia_init_module()` 在 `nv_memdbg_init()` 前调用 `vup_hooks_init()`。\
`nvidia_exit_module()` 开头调用 `vup_hooks_exit()`，但后者当前为空，不恢复已写入的指令。

初始化的关键原文为：

```c
u8 *blob = (u8 *)rm_ioctl - RM_IOCTL_OFFSET;

#if defined(NV_VGPU_KVM_BUILD)
    vup_vgpukvm_opt = nv_vgpu_kvm_enable;
#else
    vup_vgpukvm_opt = vup_gridext;
#endif

if (vup_patching_start()) {
    vup_inject_hooks(blob);
    if (vup_vupdevid == 0)
        vup_vupdevid = 0x1e30;
    vup_apply_patches(blob - 0x40);
}
vup_patching_done();
```

顺序是先安装 hook，再把默认 `vupdevid=0` 转成 `0x1e30`，最后应用运行时字节补丁。\
这里只读出一次 KVM／GRID 总门控，随后各项使用 `vup_vgpukvm_opt` 判断。

### 2.2 两种偏移不能混用

`blob` 指向运行时 RM 的 `.text` 起点，不是 ELF 文件首地址。\
当前两种闭源 `nv-kernel.o_binary` 的 `.text` 文件起点都是 `0x40`。

| 数据 | 偏移单位 | 寻址方式 |
| --- | --- | --- |
| `VUP_HOOK` 的 `offset` | 相对 `.text` | `blob + offset` |
| `vup_diff_*` 的 `offset` | blob 文件偏移 | `(blob - 0x40) + offset` |
| 本文指令窗口中的 `.text` 地址 | 相对 `.text` | 文件偏移减 `0x40` |

例如 `vupdevid` hook 的 `.text` 偏移是 `0x601dfa`，对应文件偏移 `0x601e3a`。\
后文各字节补丁表统一使用文件偏移，不能将其直接代入 `VUP_HOOK`。

这套复制、接入和偏移对应闭源 `kernel/`。\
`kernel-open/` 的 RM 布局独立，本文不把相同偏移视为 open RM 的有效 hook。

### 2.3 hook：精确核对、NOP 和 CALL rel32

`struct vup_hook_info` 保存 wrapper 地址、模块参数、`.text` 偏移、是否受 vGPU 总门控，以及以 `-1` 终止的原字节序列。\
`vup_inject_hooks()` 对每项依次执行：

1. `ovgpu` 非零且总门控 `<=0` 时跳过。
2. 对应参数为负数时跳过。
3. 逐字节核对整个覆盖窗口，任一字节不匹配则记录该 hook 的 `failed`，不写该窗口，继续其他项。
4. 在窗口末尾五字节写 `E8 rel32`，前面剩余位置填 `90`。

若覆盖窗口长度为 `N`，CALL 位于 `offset + N - 5`，其返回地址是原窗口末尾。\
位移按 `wrapper - CALL下一条地址` 计算，代码写入低 32 位。\
CALL 要求目标位于有符号 rel32 可达范围内，该约束依赖模块内链接布局。

| hook | 分支 | `.text` 偏移 | 核对的完整原字节 | 注入形态 |
| --- | --- | --- | --- | --- |
| `cudahost` | KVM/merged | `0x004cef63` | `41 80 BE 68 09 00 00 00` | 三个 NOP，再 CALL |
| `vupdevid` | 两种布局 | `0x00601dfa` | `4C 89 E0 44 89 FB` | 一个 NOP，再 CALL |
| `klogtrace` | 两种布局 | `0x00019c55` | `48 81 ED 40 04 00 00` | 两个 NOP，再 CALL |

这不是通用指令搬迁器。\
三个 naked wrapper 分别手工重放被覆盖的指令，再通过 `ASM_RET` 返回原窗口末尾。

### 2.4 wrapper 的寄存器与栈约定

这里的 RM 用 `rbp` 指向独立软件栈，不能把它当成可随意占用的普通 host frame pointer。\
hook 增加的 CALL 使 wrapper 入口满足 `rsp % 16 == 8`，调用 C helper 前还需要恢复 SysV 要求的 16 字节对齐。

| wrapper | 保存／使用方式 | 返回前重放 |
| --- | --- | --- |
| `cudahost` | 保存 `rdi rsi rdx rcx r8 r9 rax r10 r11` 九个 caller-saved 寄存器 | `cmpb $0,0x968(%r14)` |
| `vupdevid` | 保存除 `rax` 外的八个 caller-saved，再 `sub rsp,8`，使用 helper 的 `eax` | `mov %r12,%rax`、`mov %r15d,%ebx` |
| `klogtrace` | 保存同 `cudahost` 的九个寄存器 | `sub $0x440,%rbp` |

`vupdevid` 必须保留 `r13d` 中的设备类别，因为后续能力表还会比较它。\
helper 调用没有通用的 flags 保存机制，依赖重放指令或后续比较产生后继需要的标志。\
`cudahost` 通过重放 CMP 重新产生条件码，`klogtrace` 重放原 SUB，`vupdevid` 的后继路径另有比较。

wrapper 使用 `naked`、`no_instrument_function`、`no_stack_protector`、`no_split_stack`、`noclone` 和 `function_return("keep")` 等属性，并登记 `STACK_FRAME_NON_STANDARD`。\
返回采用 `<asm/linkage.h>` 的 `ASM_RET`，以适配内核的 RETHUNK，而不是在裸汇编末尾固定写普通 RET。

### 2.5 写保护窗口与失败边界

`vup_patching_start()` 禁止抢占，若发现 CR4.CET 原先开启则先关闭，再清 CR0.WP，并读回确认 WP 已清。\
只有返回成功才安装 hook 和写字节补丁，随后无论成功与否都会调用 `vup_patching_done()`。

`vup_patching_done()` 将 CR0.WP 设为 1，若开始时 CET 开启则重新设置 CR4.CET，最后通过 `preempt_enable_no_resched()` 恢复抢占计数。\
准确说法是结束时重新设置 WP，而不是保存整个 CR0 原值再原样恢复。

写保护切换操作当前 CPU 的 CR0／CR4 和抢占状态。\
写入流程采用逐 hook、逐补丁组核对和修改，失败处理的具体粒度见第 2.3 节与第 2.6 节。

### 2.6 字节补丁的整组检查与重复应用

`struct vup_patch_item` 为每个文件偏移记录 `oldval` 和 `newval`。\
`struct vup_patch_info` 绑定参数、数组、数量、精确启用值和总门控。

`vup_apply_patches()` 的核心条件为：

```c
if (arg != pi->enabv)
    continue;
for (j = 0; j < pi->count; j++)
    if (base[pi->items[j].offset] != pi->items[j].oldval &&
        base[pi->items[j].offset] != pi->items[j].newval)
        break;
if (j != pi->count) {
    printk(KERN_ERR "nvidia: vup_apply_patches %s "
           "failed (%d)\n", name, j);
    continue;
}
for (j = 0; j < pi->count; j++)
    base[pi->items[j].offset] = pi->items[j].newval;
```

每组先核对全部项，再写全部项。\
每个字节可以是 old 或 new，因此接受组内两者混合，并收敛到 new。\
未知字节使该组跳过，但不阻止后续组，也不会撤销之前已经成功写入的组。

两种重复行为要区分：

- 字节补丁重复应用接受 new，属于成功幂等。
- hook 再安装时已经不再匹配原窗口，会记录 `failed`，虽然字节保持不变，却不是静默成功幂等。

参数禁用、总门控关闭和空的退出函数都不负责撤销既有修改。\
参数多数还是只读权限，因而也不能把模块加载后的参数显示理解为可逆切换接口。

## 3. 参数、默认值与门控总表

下表的 `M` 表示编译宏 `VUP_MERGED_DRIVER`，普通 KVM 为 `0`，脚本生成的 merged 为 `1`。

| 模块参数 | 存在分支 | 默认值 | 安装／应用条件 | 总门控 |
| --- | --- | --- | --- | --- |
| `cudahost` | KVM | `M` | `>=0` 安装，helper 仅在 `>0` 时写字节 | `vgpukvm>0` |
| `vupdevid` | 两种布局 | 初始 `0`，初始化转换为 `0x1e30` | `>=0` 安装，helper 返回非零才替换 ID | KVM 为 `vgpukvm>0`，非 KVM 为 `gridext>0` |
| `klogtrace` | 两种布局 | `0` | `>=0` 安装，`<1` 静默，`==1` 计数过滤，`>1` 不过滤 | 无 |
| `klogtracefc` | 两种布局 | `8` | 只影响 `klogtrace==1` 的命中项剩余次数 | 无 |
| `vup_vgpusig` | KVM | `1` | 精确 `==1` | `vgpukvm>0` |
| `vup_kunlock` | KVM | `1` | 精确 `==1` | `vgpukvm>0` |
| `vup_qmode` | KVM | `0` | 精确 `==0` | `vgpukvm>0` |
| `vup_merged` | KVM | `M` | 精确 `==1` | `vgpukvm>0` |
| `vup_swrlwar` | KVM | `0` | 精确 `==1` | `vgpukvm>0` |
| `vup_fbcon` | KVM | `1` | 精确 `==1` | `vgpukvm>0` |
| `vup_sunlock` | KVM | `0` | 精确 `==1` | `vgpukvm>0` |
| `vup_gspvgpu` | KVM | `0` | 精确 `==1` | `vgpukvm>0` |
| `gridext` | 非 KVM | `1` | `>0` 允许带门控的项 | 本身是总门控 |
| `vup_general` | 非 KVM | `0` | 精确 `==1` | `gridext>0` |

所有 `VUP_PATCH_DEF` 参数权限为 `0400`，`cudahost`、`vupdevid`、`gridext` 也是 `0400`。\
`klogtrace` 和 `klogtracefc` 为 `0600`，但如果加载时 `klogtrace<0` 导致未安装 hook，随后只改日志参数不会自动安装。

KVM 总门控来自 `nv_vgpu_kvm_enable`，即 `nvidia` 的 `vgpukvm` 参数，默认值由对应源码补丁设为 `1`。\
`nvidia-vgpu-vfio` 还有自己的 `vgpukvm`，不是本文件读取的同一个存储位置，切换工作模式时不能只考虑其中一个。

这里的 runtime 精确匹配尤其重要。\
例如 `vup_qmode=1` 或 `2` 都是不应用该组，而 `vup_vgpusig=2` 也不会应用签名比较补丁。\
这些值都不是主动写回相反指令，更不会撤销生成阶段的静态补丁。

## 4. 三个 hook 的具体行为

### 4.1 cudahost：写入一个字节，再重放原条件比较

该 hook 位于 KVM `.text 0x4cef63`，原窗口为：

```asm
cmp byte ptr [r14+0x968],0
```

wrapper 在 helper 前用 `lea 0x42c(%r13),%rdi` 传入另一个对象字节的地址。\
helper 的完整核心为：

```c
printk(KERN_INFO "nvidia: vup_hook cudahost=%d flag=%d\n",
       vup_cudahost, *flag);
if (vup_cudahost > 0)
    *flag = vup_cudahost;
```

- 负数不安装。
- `0` 仍安装，helper 记录原值但不改字节。
- 正数写到 `[r13+0x42c]`，按 `u8` 截断，不是把所有正数归一化成 `1`。

所以 `256` 虽然满足 `>0`，最终写入的字节仍是 `0`。\
普通 KVM 默认不写，merged 默认写 `1`，前提是总门控允许且原窗口核对成功。

后继路径会在 `[r14+0x968]` 非零时检查 `[r13+0x42c]`，并可能进一步写 `[r13+0x41e]=0`。

**相关阅读入口：RM GPU subdevice，`subdevice_ctrl_gpu_kernel.c`，`subdeviceCtrlCmdGpuSetComputeModeRules_IMPL()` 和 `subdeviceCtrlCmdGpuQueryComputeModeRules_IMPL()`。**\
该源码按 vGPU profile 的 CUDA 标志限制 compute mode，例如：

```c
VGPU_STATIC_INFO *pVSI = GPU_GET_STATIC_INFO(pGpu);
if ((pVSI != NULL) && !IS_GSP_CLIENT(pGpu) && (pVSI->vgpuConfig.cudaEnabled == 0))
{
    pQueryRulesParams->rules = NV2080_CTRL_GPU_COMPUTE_MODE_RULES_COMPUTE_PROHIBITED;
```

该节选用于理解 profile 标志与 compute mode 的关系，hook 写入的目标仍按 `[r13+0x42c]` 定位。

### 4.2 vupdevid：改变软件能力表匹配使用的设备 ID

两种布局的 hook 都在 `_nv030854rm` 的 `.text 0x601dfa`。\
窗口之前已经从 `[r14+0xcee]` 读取 16 位 device ID 到 `r15d`，把设备类别保存在 `r13d`，能力输出缓冲区由 `r12` 指向。

wrapper 的关键原文为：

```asm
mov    %r15d, %edi
mov   0xcf4(%r14), %esi
call  vup_hook_vupdevid
test   %eax, %eax
cmovne %eax, %r15d
```

helper 打印原始 device ID 和 subsystem DWORD，然后返回参数 `vup_vupdevid`。\
`[r14+0xcf4]` 的低 16 位为 subsystem vendor，高 16 位为 subsystem device，这里用于日志，不是本 hook 的第二个覆盖输出。

返回非零时，wrapper 将其写入 `r15d`，再重放：

```asm
mov    %r12, %rax
mov    %r15d, %ebx
```

它改变后续软件能力表匹配使用的 ID，不在这里写回 GPU 对象的原始 device ID，也不修改真实 PCI 配置空间。\
后续仍使用 `r13d` 的设备类别和原 subsystem 信息，因此不能把这个 hook 概括成整张 GPU 的全部 PCI 标识都已替换。

默认参数初始为 `0`，但初始化在安装 hook 后把它改为 `0x1e30`。\
所以默认不是关闭伪装，负数才跳过安装，正数可指定替代 ID。\
helper 的非零测试提供零值保留原 ID 的机器码分支，但正常默认初始化不会把参数停留在零。

**源码职责对应：RM physical GPU，`gpu_features.c`，`gpuInitSwFeatures_IMPL()`。**\
原始源码把 device ID、subsystem device ID、GPU 类别和软件特性输出组合起来：

```c
NvU32 gpuType = gpuGetSwFeaturesChipType(pGpu);
NvU16 devId = (NvU16)DRF_VAL(_PCI, _DEVID, _DEVICE, pGpu->idInfo.PCIDeviceID);
NvU16 subDevId = (NvU16)DRF_VAL(_PCI, _SUBID, _DEVICE, pGpu->idInfo.PCISubDeviceID);

// Get new feature array values filled in.
return _initSwFeatureForDevIdAndType(pGpu, devId, subDevId, gpuType, pGpu->swFeatureValues);
```

继续阅读 `gpuGetSwFeaturesChipType()`、`_initSwFeatureForDevIdAndType()` 和能力数据库输入 `nv_swfeaturedb.h`，可以理解类别判断和能力表项匹配。\
RM 的 `arch.nvmk` 在构建时从该输入生成 `g_swfeaturedata.h`，供 `gpu_features.c` 使用。\
这条职责对应比把 hook 直接叫作 `gpuDetectGridCapable()` 更准确，后者对应第 5.2 节的另一处能力初始化。

### 4.3 klogtrace：解码 trace 参数并做局部限次

两种布局的 hook 都在 `_nv047877rm` 的 `.text 0x19c55`，覆盖并重放：

```asm
sub rbp,0x440
```

helper 直接使用原 `rdi/rsi`，按以下方式拆分：

```c
id = rdi & 0xffffff;
pt = (rsi >> 16) & 0xffff;
a1 = (rdi >> 24) & 0xff;
a2 = rsi & 0xffff;
```

输出格式为：

```c
printk(KERN_DEBUG "NVTRACE %06x:%04x %04x%02x\n", id, pt, a2, a1);
```

默认 `klogtrace=0` 仍安装 wrapper，但 helper 立即返回。\
`klogtrace<0` 才不安装，该 hook 的 `ovgpu=0`，因此不受 `vgpukvm` 或 `gridext` 门控。

当前内置过滤项只有 `id=0xe3cee1`、`pt=0x0648`。

- `klogtrace==1`：命中该项时先看 `klogtracefc`，为零就不输出，否则递减一次再输出。
- 默认计数 `8`：命中项允许输出八次，此后抑制。
- 未命中项仍正常输出，不是只打印这一项。
- `klogtrace>1`：不执行内置计数过滤。
- 负的 `klogtracefc` 不会命中 `==0`，会继续递减，不能把负数说成关闭日志。

该 trace 入口按上述覆盖指令和参数位布局定位。

## 5. KVM 八组运行时字节补丁

八组按 `vgpusig → kunlock → qmode → merged → swrlwar → fbcon → sunlock → gspvgpu` 的顺序应用。\
下表及各小节中的偏移均为闭源 blob 的文件偏移，包含 `.text` 文件起点 `0x40`。

### 5.1 vgpusig：消除最后一次摘要比较的不相等分支

默认 `1`，精确 `vup_vgpusig==1` 且 `vgpukvm>0` 时应用。

| 文件偏移 | 字节变化 | 指令变化 |
| --- | --- | --- |
| `0x000c8677` | `85 → 31` | `test eax,eax → xor eax,eax` |

R580 `_nv060499rm` 在此前调用一个比较函数，比较长度为 `0x20`。\
原先根据返回值执行 `TEST/JNE`，修改后的 XOR 置 `eax=0`、ZF=1，使紧接着的“不相等”分支不再跳转。

它没有把整个函数入口改成成功返回。\
前面的 RSA 处理、解密失败和 digest 长度检查仍在，后续缓冲区释放和配置处理也仍执行。

**源码职责对应：RM vGPU，`vgpuconfigapi.c`，`verifyVgpuSignature()`、`computeDigest()` 和 `vgpuconfigapiCtrlCmdVgpuConfigSetInfo_IMPL()`。**\
验证函数的末段原文为：

```c
NV_CHECK_OR_RETURN(LEVEL_INFO, decryptLen == NV_SHA256_DIGEST_SIZE, NV_ERR_INVALID_DATA);

// Compute local digest for vGPU Info received
NV_ASSERT_OK_OR_RETURN(computeDigest(localDigest, vgpuInfo));

// Compare digests to verify signature
if (portMemCmp(digest, localDigest, NV_SHA256_DIGEST_SIZE) != 0)
{
    NV_PRINTF(LEVEL_ERROR, "Invalid vGPU information received. Digests do not match.\n");
    return NV_ERR_INVALID_ARGUMENT;
}
```

配置接口见 `ctrla081.finn` 的 `NVA081_CTRL_CMD_VGPU_CONFIG_SET_INFO`、`NVA081_CTRL_VGPU_CONFIG_INFO_PARAMS`，类方法声明见 `vgpuconfigapi.h`。

原始配置调用者遇到验证错误会记录错误并立即返回：

```c
if ((rmStatus = verifyVgpuSignature(&pParams->vgpuInfo)) != NV_OK)
{
    NV_PRINTF(LEVEL_ERROR, "Digital signature verification failed.\n");
    return rmStatus;
}
```

这里的 R580 单字节补丁只消除最后一次摘要比较的失败分支，前置验证错误仍按目标控制流返回。

### 5.2 kunlock：五处不同性质的能力／资格判断

默认 `1`，精确 `vup_kunlock==1` 且 `vgpukvm>0` 时应用。\
五项不是同一个函数内的五个检查，更不能统一解释成“所有返回值改为 1”。

| 文件偏移 | 字节变化 | R580 所属函数 | 直接行为 |
| --- | --- | --- | --- |
| `0x000ec778` | `75 → EB` | `_nv038624rm` | 条件 JNE 改为 JMP，进入内部返回真路径 |
| `0x005f0788` | `E0 → C8` | `_nv031118rm` | `and eax,1 → or eax,1` |
| `0x005f6325` | `95 → 93` | `_nv031130rm` | `setne al → setae al`，沿用 TEST 后的 CF=0 |
| `0x00602082` | `75 → EB` | `_nv030281rm` | 无条件进入写 `[GPU+0x6d74]=1` 的路径 |
| `0x0060fcef` | `F8 → C8` | `_nv030289rm` | `cmp eax,0x2f → or eax,0x2f`，改变随后的 SETA |

#### 资格返回路径

`_nv038624rm` 的 `.text 0xec738` 原为 `jne 0xec768`，目标写 `r13d=1` 并进入返回路径。\
改为无条件跳转后，内部这一分支不再依赖原条件。

**源码职责对应：RM GRID，`grid_features.c`，`isGridLicenseSupported()`。**\
原始入口先取设备／子系统 ID，并将支持结果初始化为假：

```c
NvBool
isGridLicenseSupported(OBJGPU *pGpu)
{
    NvU32 chipID = DRF_VAL(_PCI, _DEVID, _DEVICE, pGpu->idInfo.PCIDeviceID);
    NvU32 subID  = DRF_VAL(_PCI, _DEVID, _DEVICE, pGpu->idInfo.PCISubDeviceID);
    NvBool gridLicenseSupport = NV_FALSE;

    if (!(IS_SILICON(pGpu)))
        return NV_TRUE;

    switch (chipID)
```

非 silicon 环境提前返回真，实际设备按 SKU、子系统、VBIOS 和 VGX branding 等条件更新结果，尾部返回 `gridLicenseSupport`。

KVM 的静态 [`blob-580.196.09.diff`](../patches/blob-580.196.09.diff) 已在更早的函数入口写入 `mov eax,1; ret`。\
因此当前 KVM 产物通常从入口就返回，runtime 在 `0xec778` 写入的内部分支实际存在，却被这个静态提前返回遮蔽。\
非 KVM `general` 不应用这份 `.09` 静态入口补丁，其同址分支修改要按第 6 节单独理解。

#### 寄存器支持位

`_nv031118rm` 前面向寄存器读取 helper 传 `0x22400`，随后右移 16 位。\
`AND 1` 改成 `OR 1` 后，结果低位恒为 1，但高位不再由 AND 清零，所以不能说整个 EAX 严格恒为 1。

**源码职责对应：RM GPU Maxwell HAL，`gpu_gm107.c`，`gpuReadQuadroSupport_GM107()`。**

```c
NvBool gpuReadQuadroSupport_GM107(OBJGPU *pGpu)
{
    return FLD_TEST_DRF(_PTOP, _DEBUG_1, _OPENGL, _ON, GPU_REG_RD32(pGpu, NV_PTOP_DEBUG_1));
}
```

这是读取硬件支持位并转成布尔结果的对应，不代表该补丁重写了整个 GPU branding 流程。

#### 查询结果的 SETcc

`_nv031130rm` 经虚函数查询编号 `0xff000021`，把输出 DWORD 放在 RM 软件栈的 `[rbp+0xc]`，之后有 `test eax,eax; setne al`。\
修改为 `setae al` 后，TEST 已清 CF，因此 AL 恒为 1。\
该指令只写 AL，EAX 的高位保持原值。

该项以查询号 `0xff000021`、局部输出 `[rbp+0xc]` 和返回低字节 AL 定位。

#### GRID capability 初始化

`_nv030281rm` 先写 `[GPU+0x6d74]=0`，再根据 `[GPU+0xe08]`、`[GPU+0x90e]` 决定是否进入写 1 的路径。\
`0x602082` 的 JNE 改为 JMP 后，无条件进入这一写 1 路径，后面的 device/subsystem 表判断及可能写 `[GPU+0x6d75]` 的逻辑仍执行。

**源码职责对应：RM physical GPU，`gpu_features.c`，`gpuDetectGridCapable_IMPL()`。**

```c
if (IS_VIRTUAL(pGpu) || !(IS_SILICON(pGpu)))
    pGpu->bGridCapable = NV_TRUE;
```

源码随后通过设备表继续设置 `bGridCapable`，并根据它及其他资格设置 `bGridSdkQualified`。\
这对应 GRID capability 初始化及后续资格形成，目标对象位置为本节的 `+0x6d74`、`+0x6d75`。

#### device ID 区间测试

`_nv030289rm` 的局部指令为：

```asm
movzx eax,word ptr [rdi+0xcee]
and   eax,0x3f
cmp   eax,0x2f
seta  al
```

补丁把 CMP 改为 `or eax,0x2f`。\
OR 的结果非零且清 CF，使 SETA 写 AL=1，这里此前又已用 `AND 0x3f` 清掉高位，因而这条局部返回路径可以得到数值 1。

**源码职责对应：RM GPU Fermi/Kepler HAL，`gpu_gf119.c`，`gpuDetectQuadroDevid_GK104()`。**

```c
return (DRF_VAL(_XVE, _ID, _DEVICE_SOFT_STRAP, pGpu->idInfo.PCIDeviceID) >=
        NV_XVE_ID_DEVICE_SOFT_STRAP_QUADRO_START);
```

源码提供按 device ID soft strap 区间区分 NVS/Quadro 的职责。\
R580 被修改路径中的低六位局部区间判断结果改为真。

### 5.3 qmode：改变 ForceGeForce 属性的设置条件

默认 `0`，而且精确 `vup_qmode==0` 才应用。\
设置其他值只是保留输入指令，不是执行反向补丁。

| 文件偏移 | 字节变化 | 直接修改 |
| --- | --- | --- |
| `0x00609538` | `0D → 07` | 查询失败的短跳目标改为后面的条件跳转 |
| `0x00609541` | `84 → 85` | 比较 magic 后的近 JE 改为 JNE |

`_nv030845rm` 查询字符串 `feeb3241`，与 `0x01ab23cd` 比较。\
以下用 `registry_read` 表示配置查询 callee，地址为 `.text` 相对地址。\
补丁前的局部控制流为：

```asm
call registry_read
test eax,eax
jne  0x609506
cmp  dword ptr [rbp+8],0x01ab23cd
je   0x609c30
```

补丁后的控制流为：

```asm
call registry_read
test eax,eax
jne  0x609500
cmp  dword ptr [rbp+8],0x01ab23cd
jne  0x609c30
```

这里 `0x609500` 就是第二个 JNE，`0x609c30` 会写 `[r12+0x993]=1`，再回到公共后继。\
因此查询失败会跳过 CMP，直接使用 TEST 留下的 ZF=0 进入写 1 路径。

| 配置查询结果 | 原局部设置路径 | 补丁后局部设置路径 |
| --- | --- | --- |
| 成功，值等于 magic | 写 1 | 不执行该写入 |
| 成功，值不等于 magic | 不执行该写入 | 写 1 |
| 失败 | 不执行该写入 | 写 1 |

“不执行该写入”不等于清零，补丁本身也没有清除 `[r12+0x993]`。\
所以精确行为不是一句“关闭 qmode”，而是改变这个属性 setter 的触发条件，包括配置缺失／查询失败路径。

**源码职责对应：RM physical GPU，`gpu_registry_physical.c`，`gpuInitRegistryOverrides_IMPL()`。**

```c
if (NV_OK == osReadRegistryDword(pGpu, NV_REG_STR_FORCE_GEFORCE,
                                 &data32))
{
    if (data32 == NV_REG_STR_FORCE_GEFORCE_ENABLE)
    {
        pGpu->setProperty(pGpu, PDB_PROP_GPU_FORCE_QUADRO_AS_GEFORCE, NV_TRUE);
    }
}
```

公共配置头 `nvRmReg.h` 定义 `NV_REG_STR_FORCE_GEFORCE` 为 `"feeb3241"`，ENABLE 为 `0x01ab23cd`，DISABLE 为 `0`。\
`gpu_branding.c` 使用 `PDB_PROP_GPU_FORCE_QUADRO_AS_GEFORCE` 参与后续 branding 判断。

### 5.4 merged：取消 GRID capability 的快捷真值，保留配置查询

默认随 `VUP_MERGED_DRIVER`，精确 `vup_merged==1` 且 `vgpukvm>0` 时应用。

| 文件偏移 | 字节变化 | 所属函数 | 修改效果 |
| --- | --- | --- | --- |
| `0x0060464d` | `74 → EB` | `_nv038627rm` | JE 改为 JMP，强制跳到配置查询 |
| `0x005d6245` | `2A → 00` | `_nv030930rm` | `jne +0x2a → jne +0`，两种条件均落入配置查询 |

两处都涉及 GPU+`0x6d74` 导致的快捷真值，以及 `RmForceGridDisplayless` 查询。\
修改后不是直接返回真，而是取消该字段造成的提前成功，让相应路径继续读取配置。

以下判断仍然保留：

- 更早的 supported 标志。
- 显示存在性。
- 相关函数的输出指针检查。
- 配置查询是否成功，以及配置值是否等于 `1`。

因此配置为 `0` 或查询失败时仍可能得到假。\
不能把第一项的分支位移也改为零，否则会落到写真路径，与当前实现目的相反。

**源码职责对应：RM GPU，`gpu.c`，`gpuIsGridDisplaylessClassSupported_IMPL()`。**\
源码在检查显示互斥条件之后有：

```c
if (pGpu->bGridCapable)
{
    return NV_TRUE;
}
```

其后读取 `NV_REG_STR_FORCE_GRID_DISPLAYLESS`，仅查询成功且值为 `NV_REG_STR_FORCE_GRID_DISPLAYLESS_YES` 才返回真。

`_nv038627rm` 通过输出指针写入支持结果，并返回状态码。\
上述源码用于阅读 displayless 支持判断逻辑，目标 helper 的调用接口按输出参数和状态返回区分。

### 5.5 swrlwar：活动时提前成功，不再改软件 runlist 设置

默认 `0`，精确 `vup_swrlwar==1` 且 `vgpukvm>0` 才应用。\
其 old 字节以已应用静态 blob diff 的状态为基准。

| 文件偏移 | 字节变化 | 修改效果 |
| --- | --- | --- |
| `0x0053e5a0` | `31 → 85` | `xor r13d,r13d → test r13d,r13d` |
| `0x0053e5a2` | `EB → 74` | 无条件跳容量路径改回活动计数为零才跳 |
| `0x0053e5ae` | `0F → 90` | 将后面的近 JE 开头改成 NOP |
| `0x0053e5af` | `84 → E9` | 组合为 `NOP; JMP`，无条件跳公共成功返回 |

**源码职责对应：RM software scheduler，`objsched.c`，`schedSwSwrlSetCountMax_IMPL()`。**\
活动计数和请求数量的关键检查为：

```c
// Can't change count max while SWRL is running.
if (pSchedSw->swrlCount != 0)
{
    portDbgPrintf("NVRM: Can't change software runlist max count.\n");
    return NV_ERR_INVALID_STATE;
}

// Check for bounds.
if ((swrlCountMax > OBJSCHED_SW_RUNLIST_COUNT) ||
    (swrlCountMax == 0))
{
    NV_PRINTF(LEVEL_ERROR, "Invalid SWRL count max %d\n", swrlCountMax);
    return NV_ERR_INVALID_LIMIT;
}
```

R580 对应 `_nv055579rm`，实际检查顺序和对象布局不同。\
原始请求首先检查 `unsigned(request-1)<=31`，即只接受 `1..32`，非法返回 `0x2e`。\
随后若父对象+`0x54==1`，有效请求再加一，所以原始请求 `32` 可以变成有效数量 `33`。

软件调度对象的相关字段如下：

| R580 对象位移 | 用途 |
| --- | --- |
| `+0x90` | 父对象指针 |
| `+0xc4` | 当前配置数量 |
| `+0xc8` | 已分配容量 |
| `+0xcc` | 活动计数 |
| `+0xac` | 本次 timeslice |
| `+0xe8` | 条目数组指针，条目 stride 为 `0x68` |

静态 diff 把活动 TEST/JE 改成 XOR/JMP，所以默认无论活动与否都进入容量检查并继续设置。\
`vup_swrlwar=0` 不覆盖这些静态字节，保留这个“活动仍设置”的默认效果。

启用 runtime workaround 后：

| 条件 | 结果 |
| --- | --- |
| 原始请求不在 `1..32` | 仍先返回 `0x2e`，不被提前成功吞掉 |
| 请求合法，活动计数为零 | 保留容量检查，容量不足返回 `0x40`，否则执行设置 |
| 请求合法，活动计数非零 | 先清返回值，再直接返回成功 `0`，不写配置数量、timeslice 或条目 |

活动路径的旧数量与新数量比较指令仍在，但其后的条件跳转已经无条件化，比较结果不再决定返回。\
这个路径不检查容量，是因为根本不执行设置，并不是允许越界写入。

该组恢复活动状态区分，并将原本可能因新旧数量不同而报错的活动路径改为成功但不执行设置。

### 5.6 fbcon：不再因一次 hypervisor 判定跳过 console 处理

默认 `1`，精确 `vup_fbcon==1` 且 `vgpukvm>0` 时应用。

| 文件偏移 | 字节变化 | 指令变化 |
| --- | --- | --- |
| `0x00d70d9a` | `84 → 30` | `test al,al → xor al,al` |

R580 `_nv000789rm` 的 RmInitAdapter 路径先调用 `os_is_vgx_hyper()`，然后按 AL 执行 TEST/JNE。\
修改后 XOR 将 AL 清零且设置 ZF，使这一次 JNE 不再因为 hypervisor 查询结果而跳过后继 console 处理。

后续仍检查 GPU+`0xe08`、GPU 虚函数条件和 OS flags。\
获取 screen info 后，console size 是否为零、base address 是否匹配，以及 size 的 64 KiB 对齐处理也都保留。

相关 `_nv054163rm` 调用 `nv_get_screen_info()`，输出 console 的 base address 并返回 size。\
这条调用和后续内存保留处理支持以下职责对应。

**源码职责对应：Unix RM 初始化，`osinit.c`，`RmSetConsolePreservationParams()`。**\
该源码先排除虚拟 GPU，再通过 `rm_get_uefi_console_size()` 获取大小和地址，相关设置为：

```c
if (IS_VIRTUAL(pGpu))
    return;
```

```c
if ((fbConsoleSize > 0) && (fbBaseAddress != 0))
{
    KernelBus *pKernelBus = GPU_GET_KERNEL_BUS(pGpu);
    pKernelBus->bPreserveBar1ConsoleEnabled =
                          (fbBaseAddress == nv->fb->cpu_address);
}
```

上述源码用于阅读 console framebuffer 的保留逻辑，补丁直接修改的是本节的 `os_is_vgx_hyper()` 返回值分支。

### 5.7 sunlock：五项状态与报告修改，不是统一“返回真”

默认 `0`，精确 `vup_sunlock==1` 且 `vgpukvm>0` 时应用。\
它在 `kunlock` 之后，且共享一个字节位置。

| 文件偏移 | 字节变化 | 所属函数 | 直接行为 |
| --- | --- | --- | --- |
| `0x000ec7aa` | `01 → 00` | `_nv038624rm` | 内部路径的 `mov r13d,1 → mov r13d,0` |
| `0x000ec778` | `75 → EB` | `_nv038624rm` | 无条件进入上述内部返回路径，与 `kunlock` 共享 |
| `0x005d141a` | `00 → 03` | `_nv022215rm` | 一个输出 DWORD 的 fallback 值从 `0` 改为 `3` |
| `0x000cc8e4` | `4E → 00` | `_nv035624rm` | `je +0x4e → je +0`，不再跳过后续字段复制／状态更新 |
| `0x000cc251` | `10 → 00` | `_nv035598rm` | `or dword [r12+0x54e0],0x10 → or ...,0` |

#### 资格路径与 kunlock、静态补丁的关系

若只看 `_nv038624rm` 的内部路径，前两项组合会跳到返回 `0` 的位置，不是返回 `1`。\
因此不能从 `sunlock` 名称推断它必然把所有资格结果都改成真。

实际 KVM 产物中，静态入口已经提前 `mov eax,1; ret`，通常使这两个内部字节不可达。\
它们仍会被 runtime 数组校验和写入，只是从通常入口观察不到该内部返回值变化。

共享的 `0xec778` 已被 `kunlock` 写成 EB 时，`sunlock` 的 old/new 校验仍接受它。\
这解释了两组为什么可以顺序应用，不意味着存在全局冲突处理或原子回滚。

#### virtualization mode 的一个 fallback 分支

`_nv022215rm` 的局部指令从 `mov dword [rbx],0` 改为 `mov dword [rbx],3`，输出指针来自 RSI。\
只改变原先写 NONE 的这一个分支，函数的其他模式判断和返回路径仍保留。

**源码职责对应：RM Device 控制，`device_ctrl.c`，`deviceCtrlCmdGpuGetVirtualizationMode_IMPL()`。**\
接口定义在 `ctrl0080gpu.finn`，字段为 `NV0080_CTRL_GPU_GET_VIRTUALIZATION_MODE_PARAMS::virtualizationMode`。

| 接口常量后缀 | 数值 | 含义 |
| --- | --- | --- |
| `NONE` | `0` | 无虚拟化模式 |
| `NMOS` | `1` | NMOS |
| `VGX` | `2` | vGPU guest |
| `HOST`、`HOST_VGPU` | `3` | vGPU host，两者为别名 |
| `HOST_VSGA` | `4` | vSGA host |

原始服务端先识别 virtual 和 passthrough，host 分支再结合 hypervisor、`isGridLicenseSupported()` 和 host-vGPU／host-vSGA 属性选择模式，其余路径报告 `NONE`。\
R580 此项修改的是目标函数中一个 fallback 立即数，使该分支从 NONE 改报 HOST_VGPU。

#### 资格之后的字段复制／状态更新

`_nv035624rm` 先调用 `_nv038624rm` 并测试 AL。\
原本假结果会跳过一段处理，位移改为零后，无论真假都落入随后输入字段复制和状态更新路径。

直接可见的操作包括把输入 byte/DWORD 复制到对象+`0x2c`、`0x34`、`0x38`、`0x3c`。\
输入+`4` 为零时，后续还会调用 `_nv033877rm`，成功后向 `_nv000127rm` 传状态 `4`。

**相关阅读入口：RM GRID 的 `grid_features.c`，以及 RM vGPU 的 `vgpuctrl.c` 中 `vgpuSetLicenseInfo()`、`vgpuapiCtrlCmdVGpuSetLicenseInfo_IMPL()`。**\
这些源码用于阅读许可信息传递和状态更新，`_nv035624rm` 的具体写入位置为上述对象字段。

#### 不再置 bit4，不是清 bit4

`_nv035598rm` 原本对对象 DWORD 执行 `OR 0x10`。\
改为 `OR 0` 后不再新设置 bit4，但已经置位的 bit4 会保留，其他位也不改变。\
因此“清掉 bit4”是错误描述。

该函数包含对象查找、同 GPU 比较、16 字节 UUID 比较和资源创建，位操作目标为对象+`0x54e0`。\
补丁注释将此项标为来自 LIL'pingu 的 Xid 43 修正。

### 5.8 gspvgpu：使固件模式 helper 的主布尔输出为真

默认 `0`，精确 `vup_gspvgpu==1` 且 `vgpukvm>0` 时应用。

| 文件偏移 | 字节变化 | 修改效果 |
| --- | --- | --- |
| `0x00038c4b` | `1B → 00` | 第一次 JE 改为跳下一条，能力 helper 为假也继续 |
| `0x00038c57` | `74 → EB` | 第二次 JE 改为 JMP，跳过默认模式的后续布尔计算 |

R580 `_nv031435rm` 先将输入 ECX 的 bit4 转成一个副输出，再调用 `_nv031460rm`。\
局部关键路径原为：

```asm
test al,al
je   0x38c27
and  ebx,0xf
mov  eax,1
cmp  ebx,1
je   0x38c27
```

补丁后第一次 JE 即使成立也落入 `and ebx,0xf; mov eax,1`，第二次则无条件跳到 `0x38c27` 的输出写入。\
两项组合使该 helper 的主布尔输出写为 `1`，能力 helper 调用和前面的 bit4 副输出仍然存在。

调用关系提供了比参数名更直接的定位：未混淆的 `rm_set_rm_firmware_requested()` 调用该函数，并传入 `EnableGpuFirmware` 配置值。\
此时 R8 指向 nv 对象+`0x2c8` 的主输出，R9 指向 nv 对象+`0x2ca` 的策略位输出。\
caller 之后仍可能根据其他策略重设主输出，所以不能把 helper 恒真等同于驱动最终必定运行 GSP。

**源码职责对应：Unix RM 的 `osapi.c`，`rm_set_rm_firmware_requested()`，以及 RM GPU manager 的 `gpu_mgr.c`。**\
后者可读 `gpumgrIsDeviceRmFirmwareCapable()` 和 `gpumgrCheckRmFirmwarePolicy()`，理解能力、默认启用和策略分别在哪里处理。

学习源码中能力、模式和默认启用的组合为：

```c
nv->request_firmware =
    (bFirmwareCapable &&
     ((rmFirmwareMode == NV_FIRMWARE_MODE_ENABLED) ||
      (bEnableByDefault && (rmFirmwareMode != NV_FIRMWARE_MODE_DISABLED))));

nv->allow_fallback_to_monolithic_rm = bAllowFallback;
```

Unix 配置头 `nv-reg.h` 定义：

```c
#define NV_REG_ENABLE_GPU_FIRMWARE_MODE_MASK              0x0000000F
#define NV_REG_ENABLE_GPU_FIRMWARE_MODE_DISABLED          0x00000000
#define NV_REG_ENABLE_GPU_FIRMWARE_MODE_ENABLED           0x00000001
#define NV_REG_ENABLE_GPU_FIRMWARE_MODE_DEFAULT           0x00000002
#define NV_REG_ENABLE_GPU_FIRMWARE_POLICY_MASK            0x000000F0
#define NV_REG_ENABLE_GPU_FIRMWARE_POLICY_ALLOW_FALLBACK  0x00000010
#define NV_REG_ENABLE_GPU_FIRMWARE_DEFAULT_VALUE          0x00000012
```

低四位 mode 决定固件模式，bit4 决定 fallback 策略。\
原始 `rm_set_rm_firmware_requested()` 在计算上述请求公式前还有虚拟化门控：hypervisor 路径受 `VGPU_GSP_PLUGIN_OFFLOAD` 编译配置及 offload 注册表值控制，未进入启用路径时提前返回。\
目标 R580 helper 的主输出及后续策略按本节调用关系和指令描述。

本补丁修改固件请求判定，后续固件签名检查、启动处理和 caller 的策略判断继续保留。

## 6. 非 KVM general：资格、超时 fallback 和 guest 设备支持

非 KVM 编译分支只注册 `general` 字节补丁组，同时保留 `vupdevid` 和 `klogtrace` hook。\
`gridext` 默认 `1`，`vup_general` 默认 `0`，只有 `gridext>0` 且精确 `vup_general==1` 时才写这一组。

| 文件偏移 | 字节变化 | 所属函数 | 直接效果 |
| --- | --- | --- | --- |
| `0x000ec778` | `75 → EB` | `_nv038624rm` | 无条件进入内部返回真的资格路径 |
| `0x00abefa8` | `14 → A0` | `_nv033879rm` | 与下一项共同改变立即数低字节 |
| `0x00abefa9` | `00 → 05` | `_nv033879rm` | `mov edi,0x14 → mov edi,0x5a0` |
| `0x00d79ce3` | `75 → EB` | `_nv054166rm` | 读取并解除 MMIO 映射后，无条件跳成功返回 |

### 6.1 GRID 许可支持资格

第一项与 KVM `kunlock` 的同址修改相同，源码职责对应仍为 `grid_features.c::isGridLicenseSupported()`。\
区别在于这里没有应用 KVM `.09` 静态 diff 的入口提前返回，因此这条内部 JMP 本身直接改变通常入口可达的资格路径。

### 6.2 第一阶段超时：改 fallback，不扩大原输入比较上界

`_nv033879rm` 查询 `UnlicensedUnrestrictedStateTimeout` 和 `UnlicensedRestricted1StateTimeout`，分别处理两阶段超时。\
两字节修改只把一条 `mov edi,0x14` 的立即数由十进制 `20` 改成 `1440`。

前面的 `cmp edi,0x14` 没有变。\
所以准确效果是把该处无效／越界输入的 fallback 从 20 分钟改成 1440 分钟，不是把原本 `1..20` 的输入接受范围扩大到 `1..1440`。

后面的两阶段大小一致性检查仍在。\
第一阈值大于第二阈值时，后续仍可能把第一阈值重设为 20 分钟，不能把这份 Linux 两字节补丁与 Windows 的扩大上界补丁混为一谈。

**源码职责对应：RM GRID，`grid_license_manager.c`，`gridlicmgrUnlicensedStateMachineInit()`。**\
状态字段和默认常量的原文为：

```c
gridUnlicensedInfo.unrestrictedStateTimeout = GRID_UNLICENSED_UNRESTRICTED_STATE_TIMEOUT;
gridUnlicensedInfo.restricted1StateTimeout  = GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT;
```

`grid_license_manager.h` 定义两阶段的 20／1440 分钟常量。\
Unix `os-hypervisor.c::osGetGridUnlicensedStateTimeoutValues()` 负责读取超时配置，配置宏可在 `nvRmReg.h` 中按 `NV_REG_STR_RM_GRID_UNLICENSED_*_STATE_TIMEOUT` 查找。

原始参考源码在两个输入满足第一项小于等于第二项时，以 `GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT` 分别检查两项，因此第一项也接受到 1440，且条件中接受零值。\
这里 `.07` 目标保留 `CMP 0x14`，补丁只修改上述 fallback 立即数，具体接受范围按目标指令描述。

### 6.3 guest 设备支持：保留映射失败，绕过后续部分资格判定

`_nv054166rm` 根据设备类型映射 BAR 中的 `+0x8f000` 或 `+0xf000` 区域，读取映射内+`0x20`，解除映射后对读取值做 `AND 0x3000`。\
原来的 JNE 只有条件成立才跳成功返回，改为 JMP 后则无条件返回 `0`，不再进入后面的 `os_is_grid_supported()` 及相关设备表限制路径。

补丁位置已经在 unmap 之后，所以不是跳过解除映射。\
更早的映射失败仍保留，不能称为整个函数无条件成功。

**源码职责对应：Unix RM hypervisor，`os-hypervisor.c`，`rm_is_vgpu_supported_device()`。**\
源码包含相同 guest 判断、寄存器映射、`NV_VGPU_CONFIG` 检查、GRID 支持检测和设备表匹配，映射失败路径为：

```c
if (vgpu_reg_mapping == NULL)
{
    nv_printf(NV_DBG_ERRORS, "NVRM: failed to map vGPU register!\n");
    return NV_ERR_OPERATING_SYSTEM;
}
```


## 7. 与其他补丁层的关系

本文件必须放在三层机制中理解，不能把它当作全部 unlock 实现：

| 层次 | 本仓库入口 | 与本文的边界 |
| --- | --- | --- |
| legacy unlock | `unlock/`，由脚本复制 `vgpu_unlock_hooks.c` 和 `kern.ld` | 有自己的接入和链接布局，不是本文三个 wrapper 的别名 |
| 生成阶段静态修改 | [`blob-580.196.09.diff`](../patches/blob-580.196.09.diff) 等 | 写入输出副本，runtime 门控不会撤销 |
| runtime hooks/patches | [`nv_hooks.c`](../patches/nv_hooks.c) 和 [`setup-vup-hooks.patch`](../patches/setup-vup-hooks.patch) | 模块初始化时改已加载 RM，退出不恢复 |

最重要的交互有三处：

1. **资格函数入口**：静态补丁在 ENDBR64 后写 `mov eax,1; ret`，遮蔽 KVM `kunlock`／`sunlock` 对同一函数内部路径的通常执行效果。
2. **SWRL 默认**：静态补丁已经选择“活动仍设置”，`swrlwar` 的 old 字节按此定义，runtime `==1` 才进一步选择“活动时成功但不设置”。
3. **配置 fallback 和标志**：`_nv030823rm` 的回退分支与配置标志由静态 diff 覆盖，`kunlock` 当前没有重复列出这些字节。

所以 `vgpukvm<=0`、`vup_kunlock=0` 或 `vup_swrlwar=0` 不等于拿回原始 `.09` blob。\
同样，`vup_merged` 只是这里的 runtime 组，不能控制脚本在生成 merged 时已经执行的 NVML 等用户态修改。

静态 blob 的逐字节说明见 [`blob-580.196.09.diff.md`](blob-580.196.09.diff.md)。\
merged 的 NVML 修改另见 [`libnvidia-ml.so.580.196.diff.md`](libnvidia-ml.so.580.196.diff.md)，用户态 `ioctl` 结果改写另见 [`cvgpu.c.md`](cvgpu.c.md)，它们不与本文件共享参数或写入时机。

## 8. 源码阅读索引

| 主题 | 模块／文件 | 函数、结构或宏 | 阅读内容 |
| --- | --- | --- | --- |
| 软件特性 ID 匹配 | RM physical GPU，`gpu_features.c`、`nv_swfeaturedb.h`、`arch.nvmk` | `gpuInitSwFeatures_IMPL`、`gpuGetSwFeaturesChipType`、`_initSwFeatureForDevIdAndType` | `vupdevid` 的职责对应、数据库输入与生成规则 |
| GRID capability 初始化 | RM physical GPU，`gpu_features.c` | `gpuDetectGridCapable_IMPL`、`bGridCapable`、`bGridSdkQualified` | `kunlock` 初始化及状态消费对应 |
| GRID 许可资格 | RM GRID，`grid_features.c` | `isGridLicenseSupported` | 原始 SKU、子系统、VBIOS 与 branding 资格判断 |
| 硬件 Quadro 支持位 | RM GPU HAL，`gpu_gm107.c` | `gpuReadQuadroSupport_GM107`、`NV_PTOP_DEBUG_1` | `kunlock` 寄存器支持位职责对应 |
| device ID soft strap | RM GPU HAL，`gpu_gf119.c` | `gpuDetectQuadroDevid_GK104` | `kunlock` 低位区间测试职责对应 |
| ForceGeForce 设置和消费 | RM physical GPU，`gpu_registry_physical.c`、`gpu_branding.c`，公共 `nvRmReg.h` | `gpuInitRegistryOverrides_IMPL`、`PDB_PROP_GPU_FORCE_QUADRO_AS_GEFORCE`、`NV_REG_STR_FORCE_GEFORCE` | `qmode` 的键名、magic、setter 对应 |
| vGPU 配置签名 | RM vGPU，`vgpuconfigapi.c`、`vgpuconfigapi.h`，RMAPI `ctrla081.finn` | `verifyVgpuSignature`、`computeDigest`、`vgpuconfigapiCtrlCmdVgpuConfigSetInfo_IMPL`、`NVA081_CTRL_VGPU_CONFIG_INFO_PARAMS` | `vgpusig` 算法和接口对应 |
| displayless 支持 | RM GPU，`gpu.c`，公共 `nvRmReg.h` | `gpuIsGridDisplaylessClassSupported_IMPL`、`NV_REG_STR_FORCE_GRID_DISPLAYLESS` | `merged` 配置和控制流对应 |
| 软件 runlist 设置 | RM scheduler，`objsched.c` | `schedSwSwrlSetCountMax_IMPL`、`OBJSCHED_SW_RUNLIST_COUNT` | `swrlwar` 的活动状态、数量检查及设置逻辑 |
| console framebuffer 保留 | Unix RM，`osinit.c` | `RmSetConsolePreservationParams`、`rm_get_uefi_console_size` | `fbcon` 初始化职责对应 |
| 虚拟化模式报告 | RM Device，`device_ctrl.c`，RMAPI `ctrl0080gpu.finn` | `deviceCtrlCmdGpuGetVirtualizationMode_IMPL`、`NV0080_CTRL_GPU_GET_VIRTUALIZATION_MODE_PARAMS` | `sunlock` 的输出接口对应 |
| 许可信息和状态更新 | RM GRID，`grid_features.c`，RM vGPU，`vgpuctrl.c` | `subdeviceCtrlCmdGpuEnableGridLicense_IMPL`、`vgpuSetLicenseInfo`、`vgpuapiCtrlCmdVGpuSetLicenseInfo_IMPL` | `sunlock` 未命名复制路径的相关阅读入口 |
| CUDA profile 和 compute mode | RM subdevice，`subdevice_ctrl_gpu_kernel.c` | `subdeviceCtrlCmdGpuSetComputeModeRules_IMPL`、`subdeviceCtrlCmdGpuQueryComputeModeRules_IMPL`、`VGPU_STATIC_INFO` | `cudahost` 的相关阅读入口，未等同对象字段 |
| 固件请求和策略 | Unix RM，`osapi.c`、`nv-reg.h`，RM GPU manager，`gpu_mgr.c` | `rm_set_rm_firmware_requested`、`gpumgrIsDeviceRmFirmwareCapable`、`gpumgrCheckRmFirmwarePolicy`、`NV_REG_ENABLE_GPU_FIRMWARE_*` | R580 caller 直接命名，内部 helper 作职责对应 |
| 未许可状态超时 | RM GRID，`grid_license_manager.c`、`grid_license_manager.h` | `gridlicmgrUnlicensedStateMachineInit`、两阶段 timeout 常量 | 原始默认值、输入比较与 `general` 的 fallback 修改 |
| guest 设备支持和配置读取 | Unix RM，`os-hypervisor.c`，公共 `nvRmReg.h` | `rm_is_vgpu_supported_device`、`osGetGridUnlicensedStateTimeoutValues` | `general` 寄存器访问、字符串和调用对应 |
