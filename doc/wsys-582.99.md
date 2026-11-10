# Windows GRID 582.99 适配说明

## 1. 修改内容

本补丁面向 Windows GRID 582.99，修改 `nvlddmkm.sys` 的六处指令，并处理八个用户态 DLL 中的 GPU 方法和进程内签名检查。

本仓库的修改集中在三个文件：

| 文件 | 修改内容 |
| --- | --- |
| [`patches/wsys-582.99.diff`](../patches/wsys-582.99.diff) | 新增 582.99 SYS 补丁，六个指令窗口合计改变 27 个字节 |
| [`patch.sh`](../patch.sh) | Windows 默认版本改为 582.99，完善提取、选中文件复制、去签名、工具编译及失败退出流程 |
| [`nsigpatch.c`](../nsigpatch.c) | 补充函数入口匹配，修正 x86 定位、输出写回及返回清栈方式 |

下文通过模块、文件名和函数／宏列出源码对应，并摘录关键逻辑。

## 2. SYS：六项修改

以下地址均为 `nvlddmkm.sys` 的**文件偏移**。\
补丁应用于 `osslsigncode remove-signature` 生成的 SYS，输入和输出摘要由 diff 首尾记录，并由 `blobpatch()` 核对。

| 功能 | 582.99 文件偏移 | 修改内容 |
| --- | --- | --- |
| 运行期 envelope 校验一 | `0xb08af` | `JE → JMP`，跳过不匹配时的计数递增 |
| 运行期 envelope 校验二 | `0x13cb07` | `JE → JMP`，跳过不匹配时的计数递增 |
| GRID 许可支持资格判断 | `0x271c63` | `0F 85 → 90 E9`，跳入返回真的路径 |
| 第一阶段超时上界 | `0x38c6f8` | 等长替换 24 字节片段，将可接受上界由 20 扩为 1440 分钟 |
| INIT envelope 校验一 | `0x69557bf` | `JE → JMP`，跳过不匹配时的计数递增 |
| INIT envelope 校验二 | `0x69558ef` | `JE → JMP`，跳过不匹配时的计数递增 |

### 2.1 四处 envelope 完整性检查

**源码定位：公共安全封装 `nvSecurity.h`。**\
关键定义为 `ENVELOPEDATA`、`END_MARKER`、`isEnvelopeHacked()`、`isEnvelopeHackedNoDTO()` 和 `DecodeEnvelope()`。

`ENVELOPEDATA` 保存目标代码地址、期望累加和、分段长度／重定位表、正常与 fallback 目标，以及 `bHacked` 状态。\
`isEnvelopeHacked()` 的关键逻辑为：

```c
cp = (unsigned char *)pEnvelope->dwTargetAddr;
pNBytes = pEnvelope->wRelocations;
dwCRC = 0;

while (pNBytes[0] != END_MARKER)
{
    for (i = 0; i < pNBytes[0]; i++)
    {
        dwCRC += cp[i];
    }
    cp = cp + i + 4;
    pNBytes++;
}
if (dwCRC != pEnvelope->dwCRC)
{
    pEnvelope->bHacked++;
}
return (pEnvelope->bHacked);
```

其中 `END_MARKER` 为 `0xffff`。\
`isEnvelopeHackedNoDTO()` 使用相同分段求和算法，源码将其说明为不同代码布局的实现。

582.99 中的校验同样使用数据指针、期望和、以 `0xffff` 终止的 16 位长度表，每段后跳过四字节，不匹配时递增字节状态。\
相应对象访问位置为 `+0`、`+0xc`、`+0x20` 和 `+0x430`，求和实现使用 SIMD 与标量尾部。

**运行期调用定位：RM GPU manager 的 `gpu_mgr_sli.c`，函数 `_gpumgrDecodeSliEnvelopes()`。**\
该函数对 `gpumgrAddSLIConfigEnvelope` 调用一次 `isEnvelopeHacked()`，对以下三个对象调用 `isEnvelopeHackedNoDTO()`：

- `gpumgrCreateDeviceEnvelope`。
- `gpumgrUpdateSLIConfigEnvelope`。
- `gpumgrGetSLIConfigsEnvelope`。

582.99 对两个校验例程的调用数量同样为一次和三次。\
据此，表中的运行期校验一对应 `isEnvelopeHacked()`，运行期校验二对应 `isEnvelopeHackedNoDTO()`。\
源码中，这条初始化路径由 `gpumgrConstruct_SLI()` 调用 `_gpumgrDecodeSliEnvelopes()` 进入。

**INIT 调用定位：Windows 显示 miniport 的 `nvdm.cpp`，函数 `DriverEntryHelper()`。**\
其中在初始化成功、非 SoC 且非 emulator 的条件下，依次检查 `setCoprocApprovedEnvelope` 和 `linkDeviceSLIProtectedEnvelope`，通过后调用 `DecodeEnvelope()`。\
582.99 的两处 INIT 补丁作用于同类内联检查，可结合这组 Hybrid envelope 初始化逻辑阅读。

四处补丁均保留原跳转目标，只把条件跳转改为无条件跳转，精确跳过此次累加和不匹配时的计数递增。\
已有 `bHacked` 值及其后续消费逻辑继续保留，`DecodeEnvelope()` 仍依据该状态决定是否设置正常目标。

### 2.2 GRID 许可支持资格判断

**源码定位：RM GRID 的 `grid_features.c`，函数 `isGridLicenseSupported(OBJGPU *pGpu)`。**

原始函数先读取设备／子系统 ID，默认不支持，仅对非 silicon 环境提前返回真，入口节选为：

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

后续按 SKU 白名单、`gpuIsGridswOnQuadroSupported()`、M10 子系统排除项，以及 VBIOS 的 `COMPUTE_ONLY` 标记和 VGX branding 决定 `gridLicenseSupport`，函数尾返回该值。\
因此原始实现是有条件的 vGPU Software Licensing 支持资格判断。

582.99 在文件偏移 `0x271c63` 将：

```text
0F 85 10 01 00 00
```

改为：

```text
90 E9 10 01 00 00
```

修改后无条件转到文件偏移 `0x271d79` 的 `mov al, 1` 返回路径，使到达该判定点的路径返回支持。\
这里的源码对应为函数职责和判断结构，R582 对象中的具体字段仍按二进制访问位置描述。

### 2.3 未授权状态第一超时上界：20 → 1440 分钟

**源码定位：RM GRID 的 `grid_license_manager.c`，函数 `gridlicmgrUnlicensedStateMachineInit()`。**\
它通过 Windows OS 层读取两个注册表参数，再初始化许可状态机的两个时间阈值。

| 参数 | 类型／单位 | 582.99 原始接受范围 | 补丁后接受范围 | 无效值回退 |
| --- | --- | --- | --- | --- |
| `UnlicensedUnrestrictedStateTimeout` | DWORD／分钟 | `1..20` | `1..1440` | 20 分钟 |
| `UnlicensedRestricted1StateTimeout` | DWORD／分钟 | `1..1440` | `1..1440` | 1440 分钟 |

原始两个 DWORD 输入还需满足第一项小于等于第二项，否则两项均使用默认值。\
582.99 补丁保留这一无符号大小关系判断，同时保留零值及超界值的回退处理。

补丁后行为可用以下伪代码概括，其中 `first`、`second` 是读取到的原始无符号 DWORD：

```c
firstTimeout = (first >= 1 && first <= 1440) ? first : 20;
secondTimeout = (second >= 1 && second <= 1440) ? second : 1440;
if (first > second)
{
    firstTimeout = 20;
    secondTimeout = 1440;
}
```

24 字节替换片段重排了第一项的范围检查和两项存储，保持原片段长度，并保留后续第二项检查及原始输入的比较。

源码中的默认值定义于 `grid_license_manager.h`：

```c
#define GRID_UNLICENSED_UNRESTRICTED_STATE_TIMEOUT  (20ULL)
#define GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT  (24ULL * 60ULL)
```

原始源码先设置上述默认值，再按两个输入的大小关系和各自上界覆盖，关键节选为：

```c
if (regUnrestrictedStateTimeout <= regRestricted1StateTimeout)
{
    // User configured unrestricted state timeout value has to be less than
    // restricted 1 default timeout.
    if (regUnrestrictedStateTimeout <= GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT)
    {
        gridUnlicensedInfo.unrestrictedStateTimeout = regUnrestrictedStateTimeout;
```

同一分支随后用 `GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT` 检查并赋值第二项。\
这份原始参考源码的第一项上界已经是 1440，条件中也接受零值，582.99 目标的原始范围则是上表中的 `1..20`，补丁后为 `1..1440`，零值仍回退到 20 分钟。

**参数名定位：RM 接口头 `nvRmReg.h`。**\
对应宏为 `NV_REG_STR_RM_GRID_UNLICENSED_UNRESTRICTED_STATE_TIMEOUT` 和 `NV_REG_STR_RM_GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT`，注释明确类型为 DWORD、单位为分钟。

**Windows 读取定位：RM Windows OS 适配层的 `os.c`，函数 `osGetGridUnlicensedStateTimeoutValues()`。**\
该函数以 `DCHUVen == 0x10de` 选择 DCH 注册表位置，读取失败时将对应输出设为 `0xffffffff`，交给上层范围处理。

| 源码选择 | 注册表位置 |
| --- | --- |
| DCH | `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\nvlddmkm\Global\GridLicensing` |
| 其他安装形式 | `HKEY_LOCAL_MACHINE\SOFTWARE\NVIDIA Corporation\Global\GridLicensing` |

**阈值消费定位：`grid_license_manager.c` 的 `_gridlicmgrUnlicensedStateMachineTransitionCallback()` 和 `gridlicmgrTransitionToState()`。**\
回调计算 `curTime - gridUnlicensedInfo.stateMachineStartTime`，状态转换函数再将两个分钟阈值分别乘以 `MINUTES_TO_NANOSECONDS` 比较。\
因此两个阈值均相对同一个状态机启动时刻，分别控制进入 `UNLICENSED_RESTRICTED_1` 和最终 `UNLICENSED` 状态的时间。\
补丁扩大第一项的可配置范围，状态机及默认 20／1440 分钟阈值继续保留。

## 3. DLL：GPU 方法替换与进程内签名检查

八个 DLL 均先经过 `libspatch()`，再经过 `nsigpatch`：

| 组件 | x86 | x64 |
| --- | --- | --- |
| D3D 用户态驱动 | `nvd3dum.dll` | `nvd3dumx.dll` |
| 用户态加载组件 | `nvldumd.dll` | `nvldumdx.dll` |
| OpenGL 用户态驱动 | `nvoglv32.dll` | `nvoglv64.dll` |
| WGF2 用户态驱动 | `nvwgf2um.dll` | `nvwgf2umx.dll` |

### 3.1 `libspatch()`：`TEST_FOR_QUADRO → NO_OPERATION`

六条全局字节替换继续沿用原规则。\
其共同语义是把匹配位置发送的 GPU `TEST_FOR_QUADRO` 方法改为 `NO_OPERATION`，涵盖 pushbuffer 方法头、构造方法头的 CPU 指令，以及存放在 DLL 数据节中的 MME 宏代码。

**源码定位：GPU class 头 `cl9097.h`。**

```c
#define NV9097_TEST_FOR_QUADRO  0x1a24
#define NV9097_NO_OPERATION    0x0100
```

方法头编码可在 `nvnMethodEnc.h` 的 `NV_METHOD()` 中对照，方法地址以右移两位后的值编码：

```text
TEST_FOR_QUADRO: 0x1a24 >> 2 = 0x689
NO_OPERATION:   0x0100 >> 2 = 0x040
```

按 `libspatch()` 中的顺序，具体替换为：

| 规则 | 原字节 → 新字节 | 对应含义 |
| --- | --- | --- |
| L1 | `89 06 01 20 → 40 00 01 20` | 小端方法头 `0x20010689 → 0x20010040`，保留 incrementing 类型、count 1 和 subchannel 0 |
| L2 | `B8 89 06 01 00 → B8 40 00 01 00` | CPU 指令 `mov eax, 0x10689 → mov eax, 0x10040`，后续设置 bit 29 形成方法头 |
| L3 | `21 40 A2 01 41 → 21 00 10 00 41` | Fermi MME 指令字 `0x01a24021 → 0x00100021`，立即数 `0x689 → 0x40` |
| L4 | `A1 40 A2 01 41 → A1 00 10 00 41` | Fermi MME 指令字 `0x01a240a1 → 0x001000a1`，保留 `ENDNEXT` |
| L5 | `A2 01 C0 18 00 03 8C 71 → 10 00 C0 18 00 03 8C 31` | Turing MME 96-bit 指令的后两个 DWORD，共同将跨字立即数 `0x689 → 0x40` |
| L6 | `A2 01 C0 18 00 03 0C 70 → 10 00 C0 18 00 03 0C 30` | Turing MME 的另一种方法输出形式，同样改变跨字立即数 |

L3／L4 末尾的 `41` 属于下一条 MME 指令，用于字节匹配上下文。\
L5／L6 中 `0x18c001a2` 变为 `0x18c00010`，同时后一 DWORD 分别从 `0x718c0300`、`0x700c0300` 变为 `0x318c0300`、`0x300c0300`，两部分共同编码该立即数。

**MME 源码定位：OpenGL 生成宏 `setBackPolygonMode.nmm`、生成代码表 `g_gf100mmecode.h` 与 `g_tu100mmecode.h`。**\
前者在特定 `QUADRO_TUNE` 条件下发送：

```text
EMITONE MME_METHOD(NV9097_TEST_FOR_QUADRO), 0x7353;
```

源码注释将其解释为短暂等待。\
Fermi 生成表的 `mmeCodeOGL_9097_SET_FRONT_POLYGON_MODE` 等数组包含 `0x01a24021` 及立即数 `1673` 的反汇编注释。\
Turing 生成表包含 `0x18c001a2, 0x718c0300` 及 `0x18c001a2, 0x700c0300` 的相邻机器字，注释明确方法立即数为 `0x0689`。

**CPU 侧用途定位：OpenGL 的 `gf100util.c`，函数 `__glGF100DoQuadroSlowdown()`。**\
该函数通过 `NV_3D_CMD_1U_CHECKED(NV9097_TEST_FOR_QUADRO, ticks)` 发送延迟方法。

因此，这组补丁将匹配位置的延迟方法发送重定向为 no-op 方法，同时保留相应的数据发送。\
`libspatch()` 使用全文件字节替换，具体命中同时涉及 CPU 代码和 GPU 宏数据，应结合编码与上下文解释。

582.99 原始 DLL 上的 L1–L6 命中数如下，每行代表一对 x86／x64 DLL，二者计数相同：

| DLL 组 | L1 | L2 | L3 | L4 | L5 | L6 |
| --- | --- | --- | --- | --- | --- | --- |
| `nvd3dum`／`nvd3dumx` | 0 | 0 | 2 | 0 | 2 | 0 |
| `nvldumd`／`nvldumdx` | 0 | 0 | 0 | 0 | 0 | 0 |
| `nvoglv32`／`nvoglv64` | 1 | 9 | 8 | 8 | 20 | 16 |
| `nvwgf2um`／`nvwgf2umx` | 0 | 0 | 3 | 0 | 6 | 0 |

`nvldumd`／`nvldumdx` 的修改来自随后执行的 `nsigpatch`。

### 3.2 `nsigpatch`：签名验证 stub 与 ABI 适配

工具通过 `crypt32.dll` 和以下函数的字符串引用定位动态加载代码，再沿调用关系查找目标验证函数：

```text
CryptQueryObject
CryptMsgClose
CertCloseStore
CertFreeCertificateContext
```

**源码定位：公共安全加载组件 `nvSecureLoadLibrary.c` 的 `nvVerifyFileSignatureW()`，接口见 `nvSecureLoadLibrary.h`。**\
该源码解析相同的 Crypt32 函数，读取嵌入式 PKCS#7 签名、查找签名者证书、检查 NVIDIA 身份和时间戳，并进入 Windows trust 验证。\
WGF2 的 `nvSecureLoadLibrary_link.cpp` 直接包含这一共享实现。

源码接口为：

```c
BOOL nvVerifyFileSignatureW( LPCWSTR filePath, BOOL * pSignedByNvidia );
```

它返回验证结果，并可通过输出参数返回是否由 NVIDIA 签名，可用于阅读补丁所替换的进程内签名检查逻辑。\
582.99 的若干 x86 调用带有额外参数，stub 按下文各 DLL 的输出位置和返回方式生成。

补丁写入的 stub 执行以下操作：

- 返回 `EAX = 0xffffffff`。
- 输出指针非 NULL 时，写入一个 DWORD `0xffffffff`。
- 保留对应 ABI 的返回方式，并保持非易失寄存器与栈平衡。

按源码的 BOOL 语义，`0xffffffff` 为非零真值，stub 使这项进程内文件签名及 NVIDIA 签名身份检查返回真。\
生成文件的系统加载继续由 Windows 代码签名策略处理，脚本另有去签名和测试签名阶段。

本次 `nsigpatch.c` 的适配包括：

- 增加新版 x64 序言及 x86 短栈帧序言模板。
- x86 按距离选择最近的匹配入口，区分 Crypt32 lazy loader 与目标 caller，避免写入邻近包装函数。
- 在目标栈帧函数的 epilogue 提取原有 `ret`／`ret N`，搜索范围止于下一栈帧入口。
- 从寄存器或入口栈位置取输出指针，并修正为写入 `0xffffffff`。
- 入口或返回 ABI 定位失败时，先返回失败，再由脚本终止该生成流程。

582.99 的 x86 返回方式如下：

| DLL | 输出指针位置 | 原返回方式 |
| --- | --- | --- |
| `nvd3dum.dll` | `EDX` | `ret 4` |
| `nvldumd.dll` | 入口 `ESP+8` | `ret 12` |
| `nvoglv32.dll` | 入口 `ESP+8` | `ret`，caller 清理 12 字节 |
| `nvwgf2um.dll` | `EDX` | `ret 4` |

全部 x64 DLL 使用 `RDX` 输出指针及裸 `ret`。\
工具从目标函数提取 x86 返回形式，保留各 DLL 的调用约定。

## 4. `patch.sh` 的 Windows 流程变化

Windows 默认输入目录更新为 `NVIDIA-Windows-x86_64-582.99`。\
`wsys` 的处理顺序为：

```text
选择 SYS 或 SYS + 八个 DLL
→ 提取原安装包，或复用已提取源文件
→ 复制选中文件到 patched 目录
→ 去除原签名，生成 -unsigned 副本
→ SYS 应用 wsys-582.99.diff
→ DLL 依次执行 libspatch 与 nsigpatch
→ 测试签名，或按 --no-testsign 复制最终文件
```

具体调整如下：

- 主 SYS diff 为必需输入，缺失时立即报错。
- 用 `WSYS_FILES` 统一管理提取、检查、复制、去签名和输出文件集合，`--no-libs-patch` 仅选择 SYS。
- 提取同时处理完整 `.sys`／`.dll` 与 `.sy_`／`.dl_` 压缩形式，后者通过 `msexpand` 展开。
- DLL 处理需要的 `nsigpatch` 缺失或源码更新时，使用 `gcc -fshort-wchar` 编译。
- 提取、源文件检查、复制、去签名、DLL 补丁及签名失败均上抛非零退出。

默认测试签名接口继续保留。\
离线生成使用以下命令，输出 SYS／DLL 目录及对应 `-unsigned` 中间文件：

```bash
bash patch.sh --no-testsign wsys
bash patch.sh --no-testsign --no-libs-patch wsys
```

输出目录为 `NVIDIA-Windows-x86_64-582.99-patched`，每次生成前会删除并重建同名输出目录，应从独立工作目录执行并保留原始输入。

## 5. 源码阅读索引

以下按模块、文件名与符号列出源码阅读索引：

| 模块 | 文件名 | 关键符号 | 对应内容 |
| --- | --- | --- | --- |
| 公共安全封装 | `nvSecurity.h` | `ENVELOPEDATA`、`isEnvelopeHacked`、`isEnvelopeHackedNoDTO`、`DecodeEnvelope` | 分段累加校验、状态递增与 envelope 解码 |
| RM GPU manager | `gpu_mgr_sli.c` | `_gpumgrDecodeSliEnvelopes`、`gpumgrConstruct_SLI` | 两种校验函数的一次／三次调用关系 |
| Windows 显示 miniport | `nvdm.cpp` | `DriverEntryHelper` | 两个 Hybrid envelope 的初始化检查 |
| RM GRID | `grid_features.c` | `isGridLicenseSupported` | 许可支持资格判断 |
| RM GRID | `grid_license_manager.c` | `gridlicmgrUnlicensedStateMachineInit`、`_gridlicmgrUnlicensedStateMachineTransitionCallback`、`gridlicmgrTransitionToState` | 阈值初始化和状态转换 |
| RM GRID 头文件 | `grid_license_manager.h` | `GRID_UNLICENSED_UNRESTRICTED_STATE_TIMEOUT`、`GRID_UNLICENSED_RESTRICTED_1_STATE_TIMEOUT` | 20／1440 分钟默认值 |
| RM Windows OS 适配 | `os.c` | `osGetGridUnlicensedStateTimeoutValues` | Windows 注册表读取 |
| RM 注册表接口 | `nvRmReg.h` | `NV_REG_STR_RM_GRID_UNLICENSED_*_TIMEOUT` | 参数名、DWORD 类型和分钟单位 |
| GPU class 定义 | `cl9097.h` | `NV9097_TEST_FOR_QUADRO`、`NV9097_NO_OPERATION` | GPU 方法地址 |
| 方法头编码 | `nvnMethodEnc.h` | `NV_METHOD` | 方法地址右移两位后的编码 |
| OpenGL MME 宏 | `setBackPolygonMode.nmm` | `NV9097_TEST_FOR_QUADRO` | 延迟方法的宏汇编发送位置 |
| OpenGL MME 生成表 | `g_gf100mmecode.h`、`g_tu100mmecode.h` | `mmeCodeOGL_9097_SET_FRONT_POLYGON_MODE`、`mmeCodeOGL_C597_TOKEN_TINY_DRAW_ARRAYS_GEFORCE` | Fermi／Turing 指令字及反汇编注释 |
| OpenGL CPU 侧发送 | `gf100util.c` | `__glGF100DoQuadroSlowdown` | 通过 pushbuffer 发送延迟方法 |
| 公共安全加载 | `nvSecureLoadLibrary.c`、`nvSecureLoadLibrary.h` | `nvVerifyFileSignatureW` | 进程内文件签名和 NVIDIA 签名身份检查 |
| WGF2 安全接入 | `nvSecureLoadLibrary_link.cpp` | 共享实现包含点 | WGF2 对公共安全加载代码的复用 |
