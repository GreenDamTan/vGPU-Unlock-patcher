# blob-580.196.09.diff：RM 静态补丁与源码对应说明

## 1. 修改对象与执行阶段

[`patches/blob-580.196.09.diff`](../patches/blob-580.196.09.diff) 修改 `580.196.09-vgpu-kvm` 的闭源 RM 文件 `kernel/nvidia/nv-kernel.o_binary`，共改变 11 个字节。\
这些修改分为四组：资格判断函数固定返回真、软件 runlist 的活动状态分支调整、GPU 配置查询的回退分支强制执行，以及该回退值从零改为一。

[`patch.sh`](../patch.sh) 的 `vgpu-kvm` 和三个 merged 入口都使用 `.09` RM。\
脚本先复制输入到输出目录，再通过以下调用写入静态补丁：

```bash
if $DO_VGPU || [ -e "$BASEDIR/patches/blob-${VER_BLOB}.diff" ]; then
    blobpatch ${TARGET}/kernel/nvidia/nv-kernel.o_binary "$BASEDIR/patches/blob-${VER_BLOB}.diff" || exit 1
fi
```

`blobpatch()` 按 diff 的首尾 SHA-256 核对输入与输出，中间每项按文件偏移检查原字节并原地写入新字节。\
它逐项写入，失败后的输出副本应从原始输入重新生成。

这些修改发生在驱动生成阶段。\
模块初始化时还会应用 [`nv_hooks.c`](nv_hooks.c.md) 的运行时补丁，其中 `swrlwar` 与本 diff 的两处字节有意重叠，下文单独说明。

## 2. 地址口径与全部字节改动

目标 RM 是可重定位 ELF，`.text` 的文件起点为 `0x40`。\
本 diff 使用文件偏移，反汇编使用 `.text` 节内地址，两者关系为：

```text
.text 节内地址 = 文件偏移 - 0x40
```

这里的节内地址用于对应 ET_REL 对象，运行时实际地址由模块装载决定。

| 文件偏移 | 原字节 | 新字节 | 所在行为组 |
| --- | --- | --- | --- |
| `0x000EC754` | `41` | `B8` | 资格函数固定返回真 |
| `0x000EC755` | `56` | `01` | 同上 |
| `0x000EC756` | `41` | `00` | 同上 |
| `0x000EC757` | `55` | `00` | 同上 |
| `0x000EC758` | `53` | `00` | 同上 |
| `0x000EC759` | `48` | `C3` | 同上 |
| `0x0053E5A0` | `85` | `31` | SWRL 活动状态分支 |
| `0x0053E5A2` | `74` | `EB` | 同上 |
| `0x006017FD` | `0F` | `90` | 强制进入查询回退块 |
| `0x006017FE` | `85` | `E9` | 同上 |
| `0x006018A3` | `00` | `01` | 查询回退值改为真 |

## 3. 资格函数：入口直接返回真

### 3.1 原始函数与入口替换

R580 混淆符号 `_nv038624rm` 起于 `.text+0xEC710`。\
原入口为：

```asm
endbr64
push r14
push r13
push rbx
sub rbp, 0x20
```

六个字节的修改从文件 `0xEC754`，即 `.text+0xEC714` 开始，结果为：

```asm
endbr64
mov eax, 1
ret
```

`endbr64` 保留，函数在修改后的入口直接返回 `1`。\
后续原函数体仍在文件中，正常入口调用在执行 `RET` 时结束。

### 3.2 原函数的设备资格判断

原函数从 GPU 对象读取两个 DWORD：

```asm
mov ebx, dword ptr [rdi + 0xCEC]
mov r14d, dword ptr [rdi + 0xCF4]
```

随后分别右移 16 位，按 device ID、subsystem ID 及多个对象标志进行判断。\
可见比较包含 `0x13BD / 0x1160`、`0x2233 / 0x165A`、`0x26B3 / 0x1934` 等组合，并存在返回真和返回假两类路径。

因此，入口补丁的直接效果是将该资格布尔判断统一为真，原来的设备组合筛选和辅助判断被入口短路。\
本节使用 R580 原符号与指令定位，GPU 字段仍按实际位移表示。

### 3.3 源码对应

**源码定位：RM GRID 功能检测，`grid_features.c`，`isGridLicenseSupported()`。**\
该函数负责判断 GPU 是否支持 vGPU Software licensing，原始入口读取设备／子系统 ID，并把支持结果初始化为假：

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

非 silicon 环境提前返回真，实际设备继续按 SKU 白名单、VBIOS 和 branding 判断，尾部返回 `gridLicenseSupport`。\
其中 M10 所属分支原文为：

```c
case NV_PCI_DEVID_DEVICE_P2405_SKU40   :
    // check for the 4-part IDs of GRID M40 and M40-SP
    // so that they don't get listed as gridLicenseSupported
    if ( (subID != NV_PCI_SUBID_DEVICE_GM107_P2405_SKU40) &&
         (subID != NV_PCI_SUBID_DEVICE_GM107_P2405_SKUS40) )
        gridLicenseSupport = NV_TRUE;    // Tesla M10 pGPU and all its vGPU configs
    break;
```

**常量定位：RM PCI 设备定义头 `nvdevid.h`。**\
用于核对这一分支的原文宏为：

```c
#define NV_PCI_DEVID_DEVICE_P2405_SKU40        0x13BD
#define NV_PCI_SUBID_DEVICE_GM107_P2405_SKU70  0x1160
#define NV_PCI_SUBID_DEVICE_GM107_P2405_SKU40  0x110A
#define NV_PCI_SUBID_DEVICE_GM107_P2405_SKUS40 0x1145
```

原始源码的该分支排除 M40／M40-SP 子系统后接受其他值，当前目标函数则有 `0x13BD / 0x1160` 的直接真路径，其余子系统继续进入辅助判断。\
两者均结合设备／子系统 ID、VBIOS 标志和 VGX branding 进行资格判断。

同文件的 `subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL()` 将该函数结果填入 `pParams->isLicenseSupported`。\
这个值表达可许可功能的支持资格，后续许可状态由独立状态与功能链管理。

**相关能力入口：RM GPU 能力检测，`gpu_features.c`，`gpuDetectGridCapable_IMPL()`。**\
该函数同样读取 `pGpu->idInfo.PCIDeviceID`，设置 `bGridCapable`，再结合 Tesla、Edison 和 NVENC／Capture SDK 白名单形成 `bGridSdkQualified`，可配合阅读资格标志的消费关系。

## 4. SWRL：活动状态下继续设置数量

### 4.1 原控制流与静态修改

该组位于 `_nv055579rm`，函数起点为 `.text+0x53E520`。\
原关键指令为：

```asm
; .text+0x53E54F
mov r13d, dword ptr [rbx + 0xCC]
; .text+0x53E55F
test r13d, r13d
je 0x53E5A0
```

`rbx` 指向软件调度器对象，`+0xCC` 是活动计数。\
原计数为零时进入 `.text+0x53E5A0` 的容量检查和设置路径，非零时先检查新旧配置数量，再决定原有返回路径。

静态补丁只改两个 opcode，结果为：

```asm
; .text+0x53E55F
xor r13d, r13d
jmp 0x53E5A0
```

`r13d` 被清零，并无条件进入容量检查和后续设置。\
该变化绕开活动计数分支，成功路径继续更新配置数量和相应 timeslice。

R580 在此处使用的字段为：

| 对象位移 | 用途 |
| --- | --- |
| `+0xC4` | 当前配置数量 |
| `+0xC8` | 已分配容量 |
| `+0xCC` | 活动计数 |

函数前部仍按无符号比较 `request - 1 <= 31` 接受原始请求 `1–32`。\
父对象类型为 `1` 时，后续有效数量会加一，容量检查使用这个有效数量，因此原始请求 `32` 可能对应有效数量 `33`。

容量不足仍走原错误返回路径。\
这些检查及后续数组、timeslice 更新属于原函数的剩余控制流，静态两字节修改只改变活动分支的选择。

### 4.2 源码对应

**源码定位：RM FIFO 软件调度器，`objsched.c`，`schedSwSwrlSetCountMax_IMPL()`。**\
函数的活动状态与数量校验原文为：

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

函数随后根据配置设置 timeslice，并在尾部更新数量，关键原文为：

```c
pSchedSw->swrlCountMax = swrlCountMax;
return NV_OK;
```

该函数负责 SWRL 最大数量及调度参数的设置。\
目标 RM 先检查请求数量，再处理活动状态和容量，具体顺序与对象位移见上节。

### 4.3 与 `vup_swrlwar` 的关系

`nv_hooks.c` 中 `vup_swrlwar` 默认值为 `0`，只有精确等于 `1` 且 KVM 门控开启时，才应用四字节运行时补丁。\
其中两字节将本静态修改恢复为 `TEST / JE`，另两字节把活动路径的条件跳转改成直接跳往成功返回。

| 模式 | 活动计数非零时的行为 |
| --- | --- |
| 原始 `.09` | 检查新旧数量，按原分支返回 |
| 静态 diff，`vup_swrlwar=0` | 继续容量检查及实际设置 |
| 静态 diff，再应用 `vup_swrlwar=1` | 合法请求提前成功返回，保留已有配置和条目 |

三种模式均先经过原始请求数量检查。\
运行时 workaround 的完整字节、门控和默认值见 [`nv_hooks.c.md`](nv_hooks.c.md)。

## 5. GPU 配置查询：强制回退并将值设为真

### 5.1 两处补丁属于同一函数

剩余三字节位于 `_nv030823rm`，函数起点为 `.text+0x601760`。\
它取得 GPU `+0x1E80` 指向的对象，通过该对象的函数指针查询 `0xFF000021`，输出存到 RM 软件栈局部 `[rbp+0xC]`。

调用后的原指令为：

```asm
; .text+0x6017BB
test eax, eax
jne 0x601860
```

文件 `0x6017FD–0x6017FE` 的修改将其改为：

```asm
nop
jmp 0x601860
```

原来的条件跳转占六字节，新 `NOP` 加五字节 `JMP` 仍占六字节，保留的 rel32 位移继续指向同一块。\
因此无论查询返回状态如何，控制流都会进入原先查询失败时执行的回退块。

### 5.2 回退值由零改为一

原回退块位于 `.text+0x601860`：

```asm
mov dword ptr [rbp + 0xC], 0
jmp 0x6017C3
```

文件 `0x6018A3` 修改其立即数低字节，结果为：

```asm
mov dword ptr [rbp + 0xC], 1
jmp 0x6017C3
```

后续路径读取局部结果并形成 GPU 布尔字段：

```asm
mov edx, dword ptr [rbp + 0xC]
...
test edx, edx
...
setne byte ptr [rbx + 0x542D]
```

这两项组合的直接效果是：保留原查询调用，将查询后的局部结果统一为 `1`，随后在到达该赋值路径时把 GPU `+0x542D` 设为真。\
同函数在 `.text+0x601873` 还调用本 diff 第一组修改的 `_nv038624rm`，其返回值影响后面的资格分支，因此应连同入口固定返回一起阅读。

### 5.3 源码对应

这一段包含属性查询、设备资格判断和多个 GPU 能力字节的初始化。\
相关源码位于 RM GPU branding 模块 `gpu_branding.c`，`gpuInitBranding_IMPL()` 汇总图形、compute 和 VGX 等标志，`gpuDetectBranding_IMPL()` 在 Quadro fuse 检测前调用 `isGridLicenseSupported()`。

该调用条件的原文节选为：

```c
if (IsMAXWELLorBetter(pGpu) &&
    !isGridLicenseSupported(pGpu) &&
    !RMCFG_IS_PLATFORM(MODS))
```

因此可从 `gpu_branding.c` 联读 `grid_features.c`，理解许可支持资格如何参与 branding 和能力形成。\
原始 `gpuDetectVgxBranding_IMPL()` 在 vGPU 上读取 profile 的 GR capability，在物理 GPU 上查询 VGX fuse，关键分支为：

```c
if (IS_VIRTUAL(pGpu))
{
    VGPU_STATIC_INFO *pVSI = GPU_GET_STATIC_INFO(pGpu);
    return (NV0080_CTRL_GR_GET_CAP(pVSI->grCapsBits, NV0080_CTRL_GR_CAPS_VGX)) ?
           BRANDING_TYPE_VGX : BRANDING_TYPE_NONE;
}

if ((pFuse != NULL) &&
    (NV_OK == fuseGetStateByOption_HAL(pGpu, pFuse, HAL_FUSE_OPT_VGX, &isVgx)) &&
     isVgx)
{
    return BRANDING_TYPE_VGX;
}
return BRANDING_TYPE_NONE;
```

这组补丁通过 `_nv030823rm` 查询属性 `0xFF000021`、写入局部 `[rbp+0xC]`，再形成 GPU `+0x542D` 的布尔值。\
上述源码用于理解 branding 和资格标志的关系，补丁的对象字段按第 5.1、5.2 节的位移描述。

## 6. 阅读索引

| 修改组 | R580 定位 | 学习源码阅读入口 |
| --- | --- | --- |
| 固定真返回 | `_nv038624rm`，`.text+0xEC714` | RM GRID `grid_features.c` 的 `isGridLicenseSupported()`，设备常量见 `nvdevid.h` |
| 活动状态下继续设置 SWRL | `_nv055579rm`，`.text+0x53E55F` | RM FIFO 软件调度器，`objsched.c`，`schedSwSwrlSetCountMax_IMPL()` |
| 强制查询回退并设真 | `_nv030823rm`，`.text+0x6017BD` 与 `.text+0x601860` | RM GPU `gpu_branding.c` 的 `gpuInitBranding_IMPL()`／`gpuDetectBranding_IMPL()`，相关职责对照 |
| 后续运行时调整 | `nv_hooks.c` 的 `swrlwar` 数组 | [`nv_hooks.c.md`](nv_hooks.c.md) |
