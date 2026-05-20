# `blob-550.90.05.diff` 深度逆向分析

## 1. 总览

`patches/blob-550.90.05.diff` 不是 unified diff，而是仓库自定义的逐字节差分格式：

- 第一行：原始文件 sha256
- 中间各行：`偏移: 原字节 新字节`
- 最后一行：patch 后目标文件 sha256

本次分析对象对应的原始二进制已经在 IDA 中打开，目标为：

- 文件：`NVIDIA-Linux-x86_64-550.90.05-vgpu-kvm/kernel/nvidia/nv-kernel.o_binary`
- sha256：`0884124c5e623e8c71779b20cf196176ba9dccc19024936b9fc5e23517f52d14`

该 hash 与 `blob-550.90.05.diff` 第一行完全一致，因此可以确认当前 IDA 会话就是这份 patch 的原始 blob。

当前版本中，diff 偏移与 IDA EA 的换算关系已经通过多点核对确认：

- `IDA_EA = DIFF_OFFSET - 0x40`

已验证样例：

- `0x000D65A4 -> 0x000D6564`
- `0x0047CBB7 -> 0x0047CB77`
- `0x0047CC70 -> 0x0047CC30`
- `0x0051DDDB -> 0x0051DD9B`
- `0x02EADC58 -> 0x02EADC18`

`nv-kernel.o_binary` 在当前 IDA 中的段布局如下：

- `.text`: `0x0 - 0xBEC164`
- `.data`: `0xBEC180 - 0xC9FE10`
- `.rodata`: `0xC9FE20 - 0x307EB80`
- `.bss`: `0x307EB80 - 0x3139368`

因此，这份 patch 的 5 组改动中：

- 前 4 组命中 `.text`
- 最后 1 组命中 `.rodata`

本文证据来源包括：

- `patches/blob-550.90.05.diff`
- IDA 反编译、汇编、basic block、caller/callee、xref、字符串
- `NVIDIA-Linux-x86_64-stage_rel` 源码树

## 2. hunk 概览

| Hunk | diff 偏移 | IDA EA | 区域 | 当前函数/对象 | 一句话结论 |
| --- | --- | --- | --- | --- | --- |
| 1 | `0x000D65A4` | `0x000D6564` | `.text` | `_nv032674rm` | 把设备/特性支持判定函数直接改成恒返回成功 |
| 2 | `0x0047CBB7` | `0x0047CB77` | `.text` | `_nv046497rm` | 去掉“software runlist 正在运行时禁止修改 count max”的保护分支 |
| 3 | `0x0047CC70` | `0x0047CC30` | `.text` | `_nv046497rm` | 改写同一错误路径的参数准备逻辑，配合字符串改动增强日志信息 |
| 4 | `0x0051DDDB` | `0x0051DD9B` | `.text` | `_nv026411rm` | 强行把某个 capability 聚合输入置为真，影响后续状态字节写回 |
| 5 | `0x02EADC58` | `0x02EADC18` | `.rodata` | 日志字符串尾部 | 把错误日志尾部 `count` 改成两个 `%d` 参数位 |

---

## 3. Hunk 1：`0x000D65A4 -> 0x000D6564`

### A. 基本信息

- diff 偏移：`0x000D65A4`
- IDA EA：`0x000D6564`
- patch 前字节：`41 56 41 55 53 48`
- patch 后字节：`B8 01 00 00 00 C3`

这组 patch 命中函数入口附近，但不是从函数首字节开始，而是保留了 CET 的 `endbr64`，然后把后续函数体入口改成直接返回。

### B. IDA 定位

- 所在函数：`_nv032674rm`
- 函数范围：`0xD6560 - 0xD6CCA`
- 当前原始 blob 在 `0xD6564` 的字节：`41 56 41 55 53 48 83 ED`

当前原始函数入口可读成：

```asm
0x000D6560  endbr64
0x000D6564  push r14
0x000D6566  push r13
0x000D6568  push rbx
0x000D6569  sub  rbp, 0x20
```

而 patch 后，这一段会变成：

```asm
0x000D6560  endbr64
0x000D6564  mov eax, 1
0x000D6569  ret
```

也就是说，这个函数会被硬改成“保留 `endbr64`，随后恒返回 1”的 stub。

### C. 伪代码说明

IDA 对原始函数的反编译显示，它并不是一个简单的常量返回函数，而是包含大量白名单和特殊分支的布尔判定逻辑。其特征包括：

- 读取 `a1 + 2720`、`a1 + 2728` 一带的设备信息
- 把 `HIWORD(deviceId)` 与 `HIWORD(subdeviceId)` 拆开使用
- 对大量固定 device ID / subdevice ID 组合做白名单判定
- 多处直接 `return 1`
- 失败路径最终走到统一收尾逻辑，再返回布尔量

它还包含额外环境判断，例如：

- 某些非 silicon 或特殊标志位场景下直接返回成功
- 某些 fallback 路径还会再调用其他函数做补充判定

### D. 调用链与引用关系

已确认调用者之一：

- `_nv026411rm` 在 `0x51DE43` 调用 `_nv032674rm`

这说明 `_nv032674rm` 不是一个孤立 helper，而是会直接参与另一条“状态聚合/能力写回”链路的输入判定。

### E. 源码映射

**当前最可能的源码映射：**

- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:75-155`
- 函数：`isGridLicenseSupported(OBJGPU *pGpu)`
- 当前置信度：**高**

关键源码片段：

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
    {
        ...
        case NV_PCI_DEVID_DEVICE_PG171_SKU200_PG179_SKU220 :
        case NV_PCI_DEVID_DEVICE_PG133_SKU230:
        case 0x20BF:
            gridLicenseSupport = NV_TRUE;
            break;
        ...
        case NV_PCI_DEVID_DEVICE_P2405_SKU40:
            if ((subID != NV_PCI_SUBID_DEVICE_GM107_P2405_SKU40) &&
                (subID != NV_PCI_SUBID_DEVICE_GM107_P2405_SKUS40))
                gridLicenseSupport = NV_TRUE;
            break;
        ...
    }
    return gridLicenseSupport;
}
```

匹配依据：

1. 二者都明确读取 `PCIDeviceID` / `PCISubDeviceID`，并把高位 device ID、subdevice ID 拆开参与判断。
2. 二者主体都是“大量 PCI device ID / subdevice ID 白名单 + 少量特殊子型号分支 + 布尔返回”的函数形状。
3. 源码中的早退条件也能和反编译前半段对上，例如 `if (!(IS_SILICON(pGpu))) return NV_TRUE;`。
4. `_nv032674rm` 在 IDA 中被 `_nv026411rm` 调用；源码里的 `isGridLicenseSupported()` 也是一条典型的 GRID capability gate，会被后续 licensing / feature 查询链大量消费。

综合这些直接证据后，更稳妥的表述是：**它已经不是普通语义相似，而是当前版本上的高置信度源码对应候选；但仍不建议在报告里把它写成 100% 已锚定的事实。**

### F. patch 作用

这组 patch 的实际效果非常明确：

- 原本：执行一大段按设备、子设备、环境和附加条件组合而成的支持判定
- patch 后：进入函数就直接 `return 1`

也就是说，原本所有白名单、黑名单、子型号特判、额外环境检查，都会被整体短路掉。

### G. patch 前后业务影响

**patch 前：**

- 只有符合白名单/特定条件的 GPU、SKU、subdevice 组合才会被视为“支持”
- 下游功能链会据此决定是否允许继续走 licensing / capability 相关路径

**patch 后：**

- 下游调用方会把这个判定结果视为恒成功
- 原本不在支持列表里的 consumer / Quadro / 特定 SKU，也更容易被当成“可支持”继续往后走

**业务定性：**

这是一枚典型的“上游业务开关型 patch”。它本身不直接创建 vGPU，也不直接修改 profile，而是把一个原本基于 GPU 身份和平台状态的支持 gate 改成恒放行。

**风险：**

- 会把一些原本被明确排除的 SKU 也纳入后续链路
- 如果后续链路对 capability 的假设依赖这个 gate，可能带来“表面支持、深层初始化仍失败”的副作用

---

## 4. Hunk 2：`0x0047CBB7 -> 0x0047CB77`

### A. 基本信息

- diff 偏移：`0x0047CBB7`
- IDA EA：`0x0047CB77`
- 当前原始字节：`85 ED 0F 85 B1 00 00 00`
- patch 后字节：`31 ED 90 90 90 90 90 90`

需要注意，这组 patch 不是从一条完整指令边界开头开始写，而是从 `0x47CB77` 这个中间位置开始覆盖，因此它实际上把：

- 原始的 `45 85 ED`（`test r13d, r13d`）
- 连同紧随其后的 `jnz 0x47CC30`

整体重写成了：

- `45 31 ED`（`xor r13d, r13d`）
- 后面全部 NOP

### B. IDA 定位

- 所在函数：`_nv046497rm`
- 函数范围：`0x47CB60 - 0x47CC78`
- 命中 basic block：`0x47CB60 - 0x47CB7F`

关键原始汇编：

```asm
0x47CB76  test r13d, r13d
0x47CB79  jnz  0x47CC30
```

patch 后等价效果：

```asm
0x47CB76  xor  r13d, r13d
0x47CB79  nop
0x47CB7A  nop
...
```

### C. 伪代码说明

IDA 反编译出的原始逻辑非常清晰：

```c
if (*(_DWORD *)(a1 + 616))
{
    nv_printf(-1, "NVRM: Can't change software runlist max count.\n");
    return 64;
}
```

而这个函数整体又与下方的 timeslice 设置、`swrlCountMax` 写回在同一函数中，因此这是 software runlist scheduler 的真实运行时配置点，而不是调试残余代码。

### D. 调用链与引用关系

已确认：

- 调用者：`_nv046455rm`
- 调用点：`0x477756`

callee：

- `nv_printf`，调用点 `0x47CBFC` 与 `0x47CC44`
- `_nv039914rm`，调用点 `0x47CC6A`

### E. 源码映射

**高置信映射：**

- `drivers/resman/src/physical/gpu/fifo/objsched.c:3725-3817`
- 对应源码函数：`schedSwSwrlSetCountMax_IMPL`
- 置信度：**高**

关键源码：

```c
// Can't change count max while SWRL is running.
if (pSchedSw->swrlCount != 0)
{
    portDbgPrintf("NVRM: Can't change software runlist max count.\n");
    return NV_ERR_INVALID_STATE;
}
```

对应调用入口：

- `drivers/resman/src/physical/gpu/fifo/objschedmgr.c:935-968`

```c
status = schedSwSwrlSetCountMax(pSchedSw, pGpu, swrlCountMax);
```

### F. patch 作用

这组 patch 干了两件事：

1. 把“检测 `swrlCount != 0`”的逻辑从 `test` 改成 `xor r13d, r13d`，直接把寄存器清零。
2. 把跳往错误块的 `jnz` 整段 NOP 掉，确保后续不再进入“运行中禁止修改”的返回路径。

因此它不仅是“去掉一个分支”，而是把这条运行态保护规则彻底拔掉了。

### G. patch 前后业务影响

**patch 前：**

- 如果 software runlist scheduler 已经在运行，修改 `swrlCountMax` 会立即失败
- 返回 `NV_ERR_INVALID_STATE`
- 调用链到此终止

**patch 后：**

- 即使 `pSchedSw->swrlCount != 0`，也不会再进入失败路径
- 后续 timeslice 计算与 `swrlCountMax` 写回流程继续执行

**业务定性：**

这是整份 patch 中最明确的一处“运行时策略放宽”：把原本只允许在停止态修改的 software runlist 最大数量，改成了允许在运行态继续改。

**直接影响：**

- 调度器在运行中也能修改 `swrlCountMax`
- 关联的 timeslice 分配策略会随之重算
- 这会直接影响多 VM / 多 vGPU 软件调度时的 runlist 容量与时片行为

---

## 5. Hunk 3：`0x0047CC70 -> 0x0047CC30`

### A. 基本信息

- diff 偏移：`0x0047CC70`
- IDA EA：`0x0047CC30`
- 当前原始字节前缀：`41 BD 40 00 00 00 48 C7`
- patch 后前缀会变成：`8B 8F 64 02 00 00 48 C7`

这意味着原本的：

```asm
mov r13d, 0x40
```

会被改写成：

```asm
mov ecx, [rdi+0x264]
```

### B. IDA 定位

- 同样位于 `_nv046497rm`
- 命中 basic block：`0x47CC30 - 0x47CC52`

这个 block 在原始函数中是前面 `jnz 0x47CC30` 的错误路径落点，也就是“software runlist 已经在运行，禁止修改”的返回块。

关键原始汇编：

```asm
0x47CC30  mov  r13d, 0x40
0x47CC36  mov  rsi, offset "NVRM: Can't change software runlist max count.\n"
0x47CC44  call nv_printf
0x47CC49  mov  eax, r13d
0x47CC4C  ret
```

### C. 伪代码说明

在原始逻辑里，这一块就是简单的：

- 打印错误日志
- 返回 `64`（对应 `NV_ERR_INVALID_STATE`）

改写后，原本用于装载错误码 `64` 的指令，被替换成了一个从对象字段读取的 `mov ecx, [rdi+0x264]`。结合同函数其他高置信源码映射可知：

- `a1 + 0x268` 对应当前 `swrlCount`
- `a1 + 0x264` 对应当前 `swrlCountMax`

而在这个错误块里，`r13d` 原本在函数入口就保存的是 `swrlCount`。这意味着如果这个 block 在 patched blob 中仍被执行：

- `rcx` 会变成当前 `swrlCountMax`
- `rdx` 仍保留调用参数里的新 `swrlCountMax`
- 末尾 `mov eax, r13d` 返回的也不再是固定错误码 `64`，而更像是当前 `swrlCount`

### D. 调用链与引用关系

- 该 block 对应字符串 xref：`0x47CC36 -> 0x2EADBF0`
- 字符串内容：`"NVRM: Can't change software runlist max count.\n"`

### E. 源码映射

仍然是高置信映射到：

- `drivers/resman/src/physical/gpu/fifo/objsched.c:3737-3741`

但这组 patch 不直接对应源代码中的显式一行，它更像是对编译后二进制错误路径的“日志参数重排”。

### F. patch 作用

单看这组 patch，它至少同时改了两件事：

1. 原本固定写入 `r13d = 64` 的错误码准备被移除。
2. 新增 `mov ecx, [rdi+0x264]`，把当前对象中的 `swrlCountMax` 装入 `rcx`。

结合 Hunk 5 的字符串改动，这组 patch 的更完整解释是：

- 原始错误字符串尾部只有 `count`
- patch 后字符串尾部被改成两个 `%d`
- 在 System V x86_64 调用约定下：
  - `rsi` 是 format string
  - `rdx` 仍保留调用参数中的新 `swrlCountMax`
  - `rcx` 会变成当前对象里的旧 `swrlCountMax`
- 因而这条错误日志很可能被改造成“打印新值 + 当前值”的形式

按这个解释，error path 很可能会变成类似：

- 第一参数：尝试设置的新 `swrlCountMax`
- 第二参数：当前对象里的旧 `swrlCountMax`

同时，若这条错误路径仍被执行，它的返回值也不再是固定错误码 `64`，而会更接近函数入口时保存在 `r13d` 里的当前 `swrlCount`。不过由于 Hunk 2 已经去掉了本函数中通向该 block 的主分支，这个返回值变化在完整 patched blob 中更可能只是伴随性后果。

**其中“日志会打印新旧两个 count 值”的解释置信度为中等偏高；“返回值也会随之改变”的结论属于高置信汇编级事实。**

### G. patch 前后业务影响

这组 patch 单独看，更偏向于**可观测性变化**：

- patch 前：错误日志只说“不能改 count max”
- patch 后：错误日志很可能带上“新值/旧值”两个数字，方便排查

但在**完整 patch 组合**里，这个 block 的主要入口已经被 Hunk 2 去掉，因此这组改动在完整 patched blob 中的运行时影响很可能有限。

更准确的说法是：

- Hunk 2 负责让主线不再进入这个错误块
- Hunk 3 + Hunk 5 则把这个错误块本身也改造成了更详细的日志形式

因此它更像是一个配套的“错误路径改写/留痕增强”，而不是主业务变化的核心。

---

## 6. Hunk 4：`0x0051DDDB -> 0x0051DD9B`

### A. 基本信息

- diff 偏移：`0x0051DDDB`
- IDA EA：`0x0051DD9B`
- 当前原始字节：`85 C0 74 07 C7 45 0C 00`
- patch 后字节：`90 90 90 90 C7 45 0C 01`

也就是说，原始的：

```asm
test eax, eax
jz   0x51DDA6
mov  dword ptr [rbp+0Ch], 0
```

会被改成：

```asm
nop
nop
nop
nop
mov  dword ptr [rbp+0Ch], 1
```

### B. IDA 定位

- 所在函数：`_nv026411rm`
- 函数范围：`0x51DD40 - 0x51DF9D`
- 命中 basic block：`0x51DD82 - 0x51DD9F`

关键原始汇编上下文：

```asm
0x51DD8A  mov  rax, [r12+0x1B8]
0x51DD92  mov  dword ptr [rbp+0x0C], 0
0x51DD99  call __x86_indirect_thunk_rax
0x51DD9E  test eax, eax
0x51DDA0  jz   0x51DDA6
0x51DDA2  mov  dword ptr [rbp+0x0C], 0
```

而 patch 正好覆盖这一段的后半部分。

### C. 伪代码说明

IDA 反编译显示，这个函数的形状不是单一布尔开关，而是明显的“能力探测结果聚合函数”：

```c
if ((*(...))(a1, v8, 4278190106LL, v7 + 3))
    v7[3] = 0;
...
*(_BYTE *)(a1 + 18015) = v12;
*(_BYTE *)(a1 + 18012) = v13;
*(_BYTE *)(a1 + 18013) = v10;
*(_BYTE *)(a1 + 18014) = v11 != 0;
*(_BYTE *)(a1 + 18010) = nv026628rm(a1);
*(_BYTE *)(a1 + 18016) = (*( ... ))(a1);
*(_BYTE *)(a1 + 18017) = (*( ... ))(a1);
...
*(_BYTE *)(a1 + 18011) = v15;
```

同时，它还会在 `0x51DE43` 调用 `_nv032674rm`：

```c
LOBYTE(v13) = nv032674rm(a1);
```

这说明 `_nv026411rm` 的角色不是做单点判定，而是把多个子探测结果组合成一组连续状态字节，再供后续流程消费。

### D. 调用链与引用关系

已确认 callee：

- `_nv032674rm`，调用点 `0x51DE43`
- `_nv026628rm`，调用点 `0x51DDE0`
- `_nv039914rm`
- `_nv039090rm`
- 多个函数指针回调（经 `__x86_indirect_thunk_rax`）

这进一步证明它是一个“聚合多个能力来源”的中枢函数。

### E. 源码映射

**当前状态：不建议把它写成某个单独源码函数的精确一一对应。**

从当前可见源码出发，更稳妥的做法不是硬指认某一个函数，而是把它落到一组**源码逻辑簇**上：

- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1206-1283`
- `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1791-1804`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:267-291`
- `drivers/resman/src/kernel/virtualization/grid/grid_features.c:525-564`

当前置信度应分成两层：

- **语义置信度：高**
- **源码簇落点置信度：中等**
- **单函数精确映射置信度：低**

匹配依据：

1. `_nv026411rm` 在 IDA 中明确调用 `_nv032674rm`，而后者高置信对应 `isGridLicenseSupported()`；这说明 `_nv026411rm` 处在一条“支持判定 -> 状态聚合 -> 上层消费”的业务链上。
2. 反编译显示 `_nv026411rm` 会先做多个 capability probe / 回调查询，再把结果回填到 `a1 + 18010 ~ 18017` 一组连续状态字节。这种形状更像 attach/load 阶段的 capability state 初始化聚合，而不是公开源码里某个单独的 query helper。
3. `gpu_mgr.c:1206-1283` 这一段集中处理 `gridLicensedFeatures`、`GridGpupProfileType`、RM caps 注册以及后续初始化 helper 的调用，是当前最接近“把 GRID/vGPU 相关状态折叠进 OBJGPU”的初始化主区域。
4. `gpu_mgr.c:1791-1804` 则显示 state-load 路径会再次消费 `isGridLicenseSupported()` 的结果，并据此启用 GRID feature，说明这条 capability 链在 `gpu_mgr.c` 中确实存在后续消费点。
5. `grid_features.c:267-291` 与 `grid_features.c:525-564` 进一步提供了“支持判定 -> feature state 写回 / 查询消费”的源码侧业务对应，但它们同样更像关联逻辑块，而不是 `_nv026411rm` 的逐行直译。

相关源码片段一：`gpu_mgr.c:1206-1283`

```c
NvU32 gridLicensedFeatures = NV_REG_STR_RM_GRID_LICENSED_FEATURES_DISABLED;
NvU32 gridSupportHypervHost = NV_REG_STR_RM_GRID_SUPPORT_HYPERV_HOST_DISABLED;
NvU32 GridGpupProfileType = NV_REG_STR_RM_GRID_GPUP_PROFILE_TYPE_COMPUTE;

pGpu->gridLicensedFeatures = NV_REG_STR_RM_GRID_LICENSED_FEATURES_DISABLED;
osGetGridLicensedFeatures(&(pGpu->gridLicensedFeatures));
...
if (IS_VIRTUAL(pGpu))
{
    pGpu->gridLicensedFeatures |= FLD_SET_DRF(_REG_STR_RM,
                                              _GRID_LICENSED_FEATURES,
                                              _VGPU,
                                              _ENABLED,
                                              pGpu->gridLicensedFeatures);
}
```

相关源码片段二：`gpu_mgr.c:1791-1804`

```c
if (isGridLicenseSupported(pGpu))
{
    const GRID_FEATURE_STATE *pFeatureState = gridlicmgrGetGpuFeatureState(pGpu->gpuId);
    if ((pFeatureState != NULL) && (pFeatureState->featureCode != 0))
    {
        status = enableGridFeature(pGpu, pFeatureState->featureCode, pFeatureState->bLicenseState);
        ...
    }
}
```

相关源码片段三：`grid_features.c:267-291`

```c
s_enabledGridFeatureState[pGpu->gpuInstance].enabledFeature = 0;
s_enabledGridFeatureState[pGpu->gpuInstance].bIsLicensed = NV_FALSE;
s_enabledGridFeatureState[pGpu->gpuInstance].bIsUnlicensedTesla = NV_FALSE;

if (isGridLicenseSupported(pGpu) && !IS_VIRTUAL(pGpu))
{
    ...
    s_enabledGridFeatureState[pGpu->gpuInstance].bIsUnlicensedTesla = NV_TRUE;
}
```

以及：`grid_features.c:525-564`

```c
if (IS_GRID_LICENSED_FEATURES_DISABLED(pGpu))
{
    pParams->isLicenseSupported = NV_FALSE;
}
else
{
    pParams->isLicenseSupported = isGridLicenseSupported(pGpu);
}
```

更准确地说，`_nv026411rm` 当前最合理的描述应当是：

- **它更像一段横跨 `gpu_mgr.c` 与 `grid_features.c` 的 GRID/vGPU capability 初始化与消费链，在优化后的二进制里被收敛成一个聚合函数；**
- **而不是仓库公开源码中某个可以完全逐行对回的单独函数。**

### F. patch 作用

这组 patch 的直接效果是：

- 原始逻辑先把 `v7[3]` 初始化为 `0`
- 随后把 `v7 + 3` 作为输出参数传给一个回调，让回调在成功时写入结果
- 原始代码在回调返回非零时，会把这个输出参数重新清零
- patch 后则不再检查 `eax`，而是直接把 `v7[3]` 改写为 `1`

而这个 `v7[3]` 随后又会参与：

- `v11 = v7[3]`
- `*(_BYTE *)(a1 + 18014) = (v11 != 0)`
- 某些中间分支是否提前进入 `LABEL_6`

从当前汇编可以把这组状态字节写回关系写得更具体一些：

- `a1 + 18010`：来自 `nv026628rm(a1)`
- `a1 + 18011`：由 `18012`、`18015`、`18017` 的组合结果生成
- `a1 + 18012`：来自 `_nv032674rm(a1)` 所在链路的聚合结果
- `a1 + 18013`：来自另一条布尔子路径
- `a1 + 18014`：直接取决于 `v7[3] != 0`
- `a1 + 18015`：来自另一条中间判定结果
- `a1 + 18016`、`a1 + 18017`：来自两个额外函数指针回调

因此，patch 的本质是：

**把一项本来依赖回调输出参数和返回状态的 capability 输入，强制钉死为真。**

### G. patch 前后业务影响

**patch 前：**

- 某个底层回调失败时，`v7[3]` 会保持/回落为 0
- 后续状态聚合中，相应 capability byte 不会被置真

**patch 后：**

- 不论该回调返回什么、也不论它是否通过输出参数写入了别的值，这一路输入都会被强行写成真值
- 至少 `a1 + 18014` 对应的聚合状态会被稳定置为 1，而不是依赖回调结果

**业务定性：**

这不是简单的日志改写，也不是单函数内局部容错，而是**直接篡改 capability 聚合输入**。它会让上层逻辑看到一组更“乐观”的能力状态。

**与 Hunk 1 的关系：**

- Hunk 1：把某个上游“是否支持”的白名单判定改成恒成功
- Hunk 4：把另一条聚合输入也强行置真

两者组合后，更像是在多层 gate 上同时做放行，而不是只改一处入口。

---

## 7. Hunk 5：`0x02EADC58 -> 0x02EADC18`

### A. 基本信息

- diff 偏移：`0x02EADC58`
- IDA EA：`0x02EADC18`
- 当前原始字节前缀：`63 6F 75 6E 74 2E 0A 00`
- 当前字符串尾部可读为：`count.\n\0`
- patch 后这 5 个字节会改成：`25 64 20 25 64`
- 即：`%d %d`

因此，原始字符串尾部：

```text
count.
```

会被改成：

```text
%d %d.
```

### B. IDA 定位

- 命中 `.rodata`
- 位于 `0x2EADBF0` 那条错误字符串的尾部区域之后，紧邻 `0x2EADC20` 的 timeslice 日志字符串

当前已确认相关字符串：

- `0x2EADBF0`：`"NVRM: Can't change software runlist max count.\n"`
- `0x2EADC20`：`"NVRM: Software scheduler timeslice set to %uuS.\n"`

### C. 与代码 patch 的联动关系

这组 `.rodata` 改动不能孤立理解。它与 Hunk 3 明显联动：

- Hunk 5 把错误字符串从固定文本改成带两个 `%d` 的格式串
- Hunk 3 则把错误块里的寄存器准备从 `mov r13d, 0x40` 改成了 `mov ecx, [rdi+0x264]`

如果把两组 patch 合在一起看，最自然的解释就是：

- 原来的报错只告诉你“不能改 count max”
- 改完后，报错会把“请求值/当前值”两个数字一起打出来

### D. 源码映射

这组 `.rodata` patch 虽然命中的是字符串常量本体，但它仍然可以高置信映射回同一段源码：

- `drivers/resman/src/physical/gpu/fifo/objsched.c:3737-3741`
- 直接对应的源码行：`portDbgPrintf("NVRM: Can't change software runlist max count.\n");`
- 置信度：**高**

关键源码：

```c
// Can't change count max while SWRL is running.
if (pSchedSw->swrlCount != 0)
{
    portDbgPrintf("NVRM: Can't change software runlist max count.\n");
    return NV_ERR_INVALID_STATE;
}
```

映射依据是：

- IDA 中 `0x2EADBF0` 这条字符串被 `0x47CC36` 引用
- `0x47CC36` 所在错误块已经高置信对应到 `schedSwSwrlSetCountMax_IMPL`
- 本 hunk 改的正是这条字符串尾部 `count.\n` 对应的 `.rodata` 字节

### E. patch 前后业务影响

这组 patch 本身不改业务 gate，也不改状态位。

它的作用是：

- 把原错误路径从“固定语句”变成“带参数的诊断信息”
- 提高调试可观测性

不过，由于 Hunk 2 已经把进入这条错误路径的主分支去掉，所以在完整 patched blob 中，这组日志增强更像是一个**伴随修改**，而不是主行为变化的根因。

---

## 8. 整体总结

把 5 组 patch 合起来看，这份 blob patch 的目标不是单一的“修一个 bug”，而是同时对三类业务 gate 下手：

1. **上游支持判定 gate**
   - 由 Hunk 1 完成
   - 把一个设备/子设备白名单式的布尔判定函数改成恒成功

2. **运行态配置限制 gate**
   - 由 Hunk 2 完成
   - 把 software runlist scheduler“运行中禁止修改 `swrlCountMax`”的保护逻辑去掉

3. **能力聚合输入 gate**
   - 由 Hunk 4 完成
   - 把状态聚合函数内部一项原本依赖回调结果的 capability 输入强制置真

另外还有两组**伴随性可观测性改动**：

- Hunk 3
- Hunk 5

它们共同把 `_nv046497rm` 那条原本的错误路径改成更详细的日志格式，但由于主保护分支已经被去掉，因此它们在完整补丁中的主要意义更偏向：

- 保持错误块逻辑一致性
- 或保留更强诊断信息

### 对 vGPU 驱动行为的整体影响

从业务语义上，这份 patch 的综合效果更接近：

- 让原本需要 datacenter / GRID / 特定白名单设备身份才能通过的若干支持判定更容易通过
- 让某些 capability / feature 状态在内部缓存中更容易被置真
- 放宽 software runlist scheduler 的运行时配置限制

这与该仓库“在 consumer / 非官方支持 GPU 上解锁 vGPU 能力”的整体目标高度一致。

### 风险点

- 某些 gate 被放开后，只能保证“更容易往下走”，不保证底层所有硬件前提都真实满足
- capability 被强行置真后，后续链路可能出现“控制面认为支持，数据面仍失败”的场景
- 运行中修改 `swrlCountMax` 和 timeslice，可能影响调度公平性、时序稳定性，尤其是在多 VM / 多 vGPU 场景下

---

## 9. 后续版本快速定位与 patch 复用指南

这一节的重点不是“当前版本这些函数叫什么”，而是“到了后续版本，旧偏移失效后，怎样最快重新找到同一业务位置”。

### 9.1 先说结论：不要依赖自动命名函数名

像 `_nv026411rm`、`_nv032674rm`、`_nv046497rm` 这类名字，是 IDA 在当前二进制里自动生成的标签。

它们在后续版本里可能会：

- 整体换号
- 拆分/合并
- 因优化而消失
- 因重新识别而落到不同边界

因此：

- 它们可以作为**当前版本解释性标签**
- 不能作为**跨版本主定位方法**

同理，源码路径和源码函数名也只能作为“有源码时的加速器”，不能当成只有二进制时的前提条件。

### 9.2 多层定位锚点表

| patch 点 | 主要字符串锚点 | 主要控制流锚点 | 主要调用职责锚点 | 风险等级 |
| --- | --- | --- | --- | --- |
| `0x47CB77` | `Can't change software runlist max count` | 运行中禁止修改 `swrlCountMax` 的 early-return 分支 | runlist manager 下发到 software scheduler 配置函数 | 中 |
| `0x47CC30` | 同上 | 错误日志打印块 | 与上一个 patch 同函数同错误路径 | 低到中 |
| `0x2EADC18` | 同上字符串尾部 | `.rodata` 中紧邻 timeslice 日志字符串 | 与 `0x47CC30` 配套 | 低 |
| `0xD6564` | 无稳定字符串，优先靠函数形状 | 大量 device ID / subdevice ID 白名单分支，布尔返回 | 被状态聚合函数调用的支持判定函数 | 高 |
| `0x51DD9B` | 无稳定字符串 | 调用多个子探测后回填连续状态字节 | 状态聚合函数 -> capability 写回 | 高 |

### 9.3 推荐定位顺序

#### 路径 1：只有 IDA / 只有二进制时

1. **先看字符串窗口**
   - 搜：
     - `NVRM: Can't change software runlist max count.`
     - `Software scheduler timeslice set`
   - 再沿 xref 回到代码点

2. **若目标点自己没有稳定字符串，就立刻切换到间接定位法**
   - 先看调用者函数有没有稳定日志字符串、错误字符串、控制命令名
   - 再看被调 helper 有没有稳定字符串或错误码
   - 如果两边都没有，再看“谁消费它写回的状态位/返回值”

3. **再看控制流形状**
   - `_nv046497rm` 这组点的核心特征是：
     - 先判断某个 `swrlCount` 是否非零
     - 命中时走错误打印 + 错误码返回
     - 另一侧是 timeslice 计算和 `swrlCountMax` 写回

4. **再看调用职责链**
   - 找到“状态聚合函数”后，看它是否：
     - 调多个子判定/回调
     - 回填一组连续状态字节
     - 其中一个被调函数内部充满 device/subdevice 白名单分支

5. **最后才用字节特征兜底**
   - 例如当前版本可参考：
     - `0x47CB77`: `85 ED 0F 85 B1 00 00 00`
     - `0x47CC30`: `41 BD 40 00 00 00 48 C7`
     - `0x51DD9B`: `85 C0 74 07 C7 45 0C 00`
     - `0xD6564`: `41 56 41 55 53 48 83 ED`
   - 但跨版本绝不能只靠几字节硬搜定点，必须再做语义确认

#### 路径 2：同时有源码与 IDA 时

1. 先从源码中提取：
   - 日志字符串
   - 关键错误码
   - 关键条件判断
   - 关键职责链

2. 再回到 IDA 里通过：
   - 字符串
   - 控制流
   - caller/callee 关系
   - 状态字节写回模式
   - 必要时用调用者/被调者/状态消费者做二跳反推

3. 最后才把源码映射作为语义确认，而不是主定位手段。

### 9.4 在 IDA 中快速定位的明确方法

#### 方法 1：通过旧版本 diff 偏移直接落点

只适用于当前版本或极近版本。

步骤：

1. 确认当前打开对象是 `nv-kernel.o_binary`
2. 核对 hash 是否等于 `0884124c5e623e8c71779b20cf196176ba9dccc19024936b9fc5e23517f52d14`
3. 用公式：`IDA_EA = DIFF_OFFSET - 0x40`
4. 直接跳到 EA
5. 用字节比对确认当前位置仍是旧字节

已知样例：

- `0x000D65A4 -> 0x000D6564`
- `0x0047CBB7 -> 0x0047CB77`
- `0x0047CC70 -> 0x0047CC30`
- `0x0051DDDB -> 0x0051DD9B`
- `0x02EADC58 -> 0x02EADC18`

#### 方法 2：通过字符串 xref 快速定位 software runlist 这组 patch

这组点是后续版本里最好迁移的一组。

步骤：

1. 在 IDA 字符串窗口搜：
   - `NVRM: Can't change software runlist max count.`
   - `NVRM: Software scheduler timeslice set to`
2. 查看 xref
3. 沿 xref 回到引用点
4. 进入同一函数
5. 在函数内找：
   - 运行中禁止修改的保护分支
   - timeslice 设置后的日志打印路径

当前版本已验证 xref：

- `0x47CC36 -> "NVRM: Can't change software runlist max count.\n"`
- `0x47CBF1 -> "NVRM: Software scheduler timeslice set to %uuS.\n"`

如果后续版本字符串有轻微变化，优先搜关键词：

- `software runlist`
- `count max`
- `timeslice set`

#### 方法 3：通过“状态聚合函数 -> 支持判定函数”的职责链定位

适用于 `_nv026411rm` / `_nv032674rm` 这组点。

步骤：

1. 先找一个“状态聚合函数”，其特征是：
   - 会调多个子判定或 vtable 回调
   - 最终回填一组连续状态字节
   - 会把多个布尔量组合成最终 capability / feature state

2. 看它的 callees 中是否存在一个：
   - 大量 device ID / subdevice ID 条件分支
   - 布尔返回
   - 用于放行/拒绝特定 SKU

3. 先锁定这个判定函数，再回头确认聚合函数如何消费它的返回值并写回状态字节。

当前版本样例：

- 状态聚合函数会在 `0x51DE43` 调用一个设备支持判定函数
- 该聚合函数还会调用 `_nv026628rm` 和多个函数指针回调
- 之后回填 `a1 + 18010 ~ 18017` 一组连续状态字节

注意：这里要记住的是**职责模式**，不是 `_nv026411rm -> _nv032674rm` 这两个名字本身。

#### 方法 4：通过源码职责反推到 IDA（仅作加速器）

如果手头同时有源码和 IDA，最快的做法是：

1. 在源码里先找：
   - software runlist count/timeslice 配置逻辑
   - device support / license support 判定逻辑
   - feature state / capability 聚合逻辑
2. 把源码里的稳定锚点提取出来：
   - 日志字符串
   - 错误码
   - 关键 if 条件
   - 上下游调用关系
3. 再回到 IDA 里定位
4. 最后只把源码作为语义确认

没有源码时，也仍然可以按：

- 字符串 -> 控制流 -> 调用职责 -> 字节确认

这条路径完成定位。

### 9.5 basic block 和控制流的二次确认

找到疑似 patch 点后，不能只因“字节像”就收工，必须再确认：

1. 命中地址是否在预期 block 内
2. block 前驱/后继是否还是原来的业务流程
3. 改动的是：
   - 条件跳转
   - early-return
   - 日志参数准备
   - 状态位写回
   - 还是返回值构造

当前版本中：

- `0x47CB77` 的命中 block 是 `0x47CB60 - 0x47CB7F`
- `0x47CBB7 - 0x47CBD8` 是同函数后续独立 block，不是同一个命中 block
- `0x51DD9B` 的命中 block 是 `0x51DD82 - 0x51DD9F`

### 9.6 新版本复用时的语义一致性检查清单

在新版本里找到疑似位置后，至少要检查：

1. 这个函数的职责是否仍和旧版本一致
2. 关键日志是否仍一致或只是轻微改写
3. 关键分支条件是否仍表达同一业务规则
4. 关键返回值/错误码语义是否一致
5. 上下游调用者是否还是同一条业务链
6. patch 后改变的行为是否仍对应同一个业务目标

只有这几条都通过，才能认定“这是同一 patch 点”。

### 9.7 复用判定标准

#### 可直接复用

- 语义、控制流、调用链都基本一致
- 只是偏移变化

#### 需要重定位后改写

- 业务位置相同
- 但寄存器分配、分支布局或指令编码明显变化

#### 不建议直接复用

- 上游业务逻辑已经变化
- 即使位置相似，直接套旧 patch 也可能引入副作用

### 9.8 本次 patch 的具体复用建议

#### 对 software runlist 这组点

优先锚点：

1. `Can't change software runlist max count`
2. `Software scheduler timeslice set`
3. 运行中禁止修改 `swrlCountMax` 的 early-return 分支
4. manager 层调用 software scheduler 配置函数的职责关系

#### 对设备支持判定这组点

优先锚点：

1. 大量 device ID / subdevice ID 白名单分支
2. 布尔返回值
3. 被状态聚合函数调用
4. 和 licensing / feature support 查询路径的业务关系

#### 对状态聚合这组点

优先锚点：

1. 多个子判定/回调
2. 回填连续状态字节
3. 再由更上层 capability / feature 查询路径消费

---

## 10. 当前结论的置信度说明

### 高置信结论

- `0x47CB77` / `0x47CC30` / `0x2EADC18` 属于同一条 software runlist 配置逻辑链
- 这条链高置信对应 `drivers/resman/src/physical/gpu/fifo/objsched.c:3725-3817`
- `0x47CB77` 的核心作用是去掉“运行中禁止修改 `swrlCountMax`”的保护分支
- `0xD6564` 会把 `_nv032674rm` 改成恒返回 1
- `0x51DD9B` 会把 `_nv026411rm` 中的一项 capability 输入强制置真

### 中等置信结论

- `0x47CC30` + `0x2EADC18` 组合起来是在把错误日志改造成“输出两个 count 值”的形式
- `_nv026411rm` 更像一段横跨 `gpu_mgr.c` 与 `grid_features.c` 的 capability 初始化与消费逻辑簇，而不是公开源码里的单独函数直译

### 仍待继续收紧的部分

- `_nv026411rm` 在 `stage_rel` 中的源码级直接对应函数
- `0x47CC30` 这组错误路径在 patched blob 中究竟主要承担“日志增强”还是还保留了可达的返回值副作用

对这些点，正式报告应继续保持“高置信业务对应”与“非 100% 单函数锚定”之间的边界，不把尚未闭环的部分写成已完全证明的事实。
