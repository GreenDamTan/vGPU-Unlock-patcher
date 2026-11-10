# libnvidia-ml.so.580.196.diff：merged NVML 补丁与源码对应说明

## 1. 目标与接入位置

[`patches/libnvidia-ml.so.580.196.diff`](../patches/libnvidia-ml.so.580.196.diff) 修改 Linux x86-64 的 `libnvidia-ml.so.580.196.07`，供 merged 驱动使用。\
它包含两项行为：版本查询直接返回库内的 `580.196.07`，进程查询的 ID 类型固定进入 CLASS 选择路径。

[`patch.sh`](../patch.sh) 的 `grid-merge`、`general-merge`、`vgpu-kvm-merge` 都设置 `DO_MRGD=true`。\
merged 组合使用 `.07` 的显示 wrapper、桌面用户态与版本头，合入 `.09` 的 RM、vGPU 组件及固件，NVML 保留 `.07` 用户态版本。

脚本在 `DO_MRGD` 分支中的原文节选：

```bash
sed -e '/^NVIDIA_CFLAGS += .*BIT_MACROS$/aNVIDIA_CFLAGS += -DVUP_MERGED_DRIVER=1' -i ${TARGET}/kernel{,-open}/nvidia/nvidia.Kbuild
blobpatch ${TARGET}/libnvidia-ml.so.${VER_TARGET%.*}.* "$BASEDIR/patches/libnvidia-ml.so.${VER_TARGET%.*}.diff" || exit 1
```

这里 `VER_TARGET=580.196.07`，补丁文件名按 `580.196` 选择，目标库由完整输入摘要进一步限定。\
NVML 修改发生在驱动生成阶段，内核参数 `vup_merged` 则控制 [`nv_hooks.c`](nv_hooks.c.md) 中另一组运行时补丁，两者分别执行。

## 2. 地址、字节范围与补丁格式

下文偏移均为目标 NVML 的文件偏移。\
在该 ELF 的这些位置，文件偏移与 ELF 虚拟地址数值相同，反汇编中的分支目标可以直接对照表格。

| 修改区域 | 实际变化字节数 | 行为 |
| --- | --- | --- |
| `0x4A570–0x4A594` | 36 | 用带参数校验的短函数替换版本查询入口 |
| `0xAB7DF` | 1 | `JE` 改为 `JMP`，固定进入 CLASS 路径 |

版本查询的替换窗口为 37 字节，其中 `0x4A591` 的 `RET` 恰与原字节相同，因此 diff 只记录该窗口内的 36 个变化字节。\
全文件合计改变 37 字节。

`blobpatch()` 读取首尾 SHA-256 与中间的 `文件偏移: 原字节 新字节`，逐项检查旧值、写入新值，再用末尾摘要核对结果。\
写入方式是逐字节原地修改，重新生成时应使用原始库的副本。

## 3. 版本查询：直接复制库内版本

### 3.1 原行为与替换后指令

原入口位于 `0x4A570`，包含寄存器保存、局部栈分配、状态检查及后续获取路径。\
补丁在同一入口放入以下完整短函数，尾部再填三个 `NOP`：

```asm
; file 0x4A570
mov eax, 2
test rdi, rdi
je 0x4A591
mov al, 7
cmp esi, 11
jb 0x4A591
push rbp
lea rsi, [rip + 0x154018]  ; 内嵌字符串，地址 0x19E5A1
call 0x1B130              ; 原 strcpy@plt
pop rbp
xor eax, eax
; file 0x4A591
ret
nop
nop
nop
```

`rdi` 是输出指针，`esi` 是无符号缓冲区长度。\
`mov eax, 2` 先将整个返回寄存器置为 `2`，之后的 `mov al, 7` 可直接把它变为 `7`。

| 输入条件 | 返回值 | 对输出缓冲区的操作 |
| --- | --- | --- |
| 输出指针为 NULL，任意长度 | `NVML_ERROR_INVALID_ARGUMENT = 2` | 提前返回 |
| 输出指针非空，长度小于 11 | `NVML_ERROR_INSUFFICIENT_SIZE = 7` | 提前返回 |
| 输出指针非空，长度至少 11 | `NVML_SUCCESS = 0` | 复制 `580.196.07` 和结尾 NUL |

`580.196.07` 共 10 个字符，连同 NUL 需要 11 字节。\
长度判断使用无符号分支 `JB`，成功路径调用原库的 `strcpy@plt`，字符串来源为该库内嵌的版本常量。

### 3.2 源码对应

**源码定位：NVML API 层，`api.c`，`tsapiSystemGetDriverVersion()`。**\
以下为 Linux 分支的原文节选，各段按执行顺序摘录：

```c
if(!version)
    return NVML_ERROR_INVALID_ARGUMENT;
```

```c
result = NVML_GET_CACHEABLE_FIELD(globals.driverVersion,
                                  systemGetDriverVersion(globals.driverVersion,
                                                         sizeof(globals.driverVersion)));
```

```c
if (NVML_SUCCESS != result)
    return result;
```

```c
if (length < strlen(globals.driverVersion) + 1)
    return NVML_ERROR_INSUFFICIENT_SIZE;

// Returned the cached version information
strcpy(version, globals.driverVersion);

return NVML_SUCCESS;
```

该源码展示的原职责是：先校验输出指针，通过 `systemGetDriverVersion()` 获取并缓存版本，检查获取状态，然后检查长度并复制缓存内容。\
补丁保留指针和长度两类输入约束，将成功路径改为直接复制库内版本，提前返回使原有获取、初始化及缓存查询路径退出本次调用链。

因此该接口在 merged 下稳定报告 `.07` 用户态版本。\
`.09` RM 的静态修改见 [`blob-580.196.09.diff.md`](blob-580.196.09.diff.md)，两份文档分别描述各自输入版本和修改对象。

### 3.3 SysV ABI 栈对齐

x86-64 SysV ABI 下，函数入口 `rsp % 16 == 8`。\
补丁在调用 `strcpy` 前执行 `push rbp`，使调用前 `rsp % 16 == 0`，被调用函数入口又恢复为 `8`，返回后通过 `pop rbp` 恢复寄存器与栈位置。

`strcpy@plt` 地址为 `0x1B130`，其 GOT 项为 `0x22E098`。\
这保持了原来的动态符号解析路径，成功返回值由 `xor eax, eax` 显式设为零。

补丁保留替换窗口外的 ELF 展开表。

## 4. 进程查询：固定进入 CLASS 选择路径

### 4.1 修改前后的控制流

第二项只改一个 opcode：

```text
000AB7DF: 74 EB
```

其上下文如下：

```asm
cmp qword ptr [r12 + 0x1ABE0], 0
; 原始
je 0xAB808
; 修改后
jmp 0xAB808
```

原来的非空路径写入：

```asm
mov dword ptr [rbx], 1
mov dword ptr [r13], 0xA082
```

而跳转目标 `0xAB808` 开始执行：

```asm
mov dword ptr [rbx], 0
cmp r14d, 2
```

其中 `rbx` 对应 `idType` 输出指针，`r13` 对应 `id` 输出指针，`r14d` 对应进程查询 mode。\
改为无条件跳转后，选择 CLASS 类型，再按原有 mode、芯片类型、架构和 class 支持情况计算具体 ID。

### 4.2 源码对应

**源码定位：NVML RM 后端，`rm_pids.c`，`rmConvertNvmlProcessInfoModeToRmId()`。**\
关键原文节选：

```c
nvmlResult = NVML_GET_CACHEABLE_FIELD(device->chipInfo,
                        deviceGetChipInformation(device, &device->chipInfo));
if (nvmlResult != NVML_SUCCESS)
{
    return nvmlResult;
}

if (device->vgpu != NULL)
{
    *idType = NV2080_CTRL_GPU_GET_PIDS_ID_TYPE_VGPU_GUEST;
    *id = NVA082_HOST_VGPU_DEVICE;
    return NVML_SUCCESS;
}

*idType = NV2080_CTRL_GPU_GET_PIDS_ID_TYPE_CLASS;
if (mode == NVML_PROCESS_MODE_MPS)
{
    *id = MPS_COMPUTE;
    return NVML_SUCCESS;
}
```

`device->vgpu` 分支的语义与 R580 的 `1 / 0xA082` 写入对应。\
这次修改强制采用该分支之后的 CLASS 路径，前置芯片信息获取及其错误返回继续执行。

**源码定位：SDK RM 控制接口头 `ctrl2080gpu.h`。**\
类型常量原文为：

```c
#define NV2080_CTRL_GPU_GET_PIDS_ID_TYPE_CLASS      (0x00000000)
#define NV2080_CTRL_GPU_GET_PIDS_ID_TYPE_VGPU_GUEST (0x00000001)
```

`NV2080_CTRL_GPU_GET_PIDS_PARAMS` 携带 `idType`、`id`、`pidTblCount` 和 PID 数组。\
`rm_pids.c` 的调用链将转换后的 `id`、`idType` 传给 `rmGetRunningProcesses()`，供 RM 进程查询使用。

CLASS 表示查询选择器的类型，具体 class ID 仍有多个。\
例如学习源码的 MPS 路径选择 `MPS_COMPUTE`，其他路径继续区分 compute／graphics、GPU 架构及支持的 class。

## 5. 源码阅读索引

| 功能 | 本仓库入口 | 源码阅读入口 |
| --- | --- | --- |
| merged 的版本来源和生成条件 | `patch.sh` 的 `DO_MRGD` 分支 | NVML 与 RM 分别属于用户态库和资源管理器 |
| 版本接口的参数与复制逻辑 | diff 的 `0x4A570–0x4A594` | NVML `api.c` 的 `tsapiSystemGetDriverVersion()` |
| 进程查询的类型与 class 选择 | diff 的 `0xAB7DF` | NVML RM 后端 `rm_pids.c` 的 `rmConvertNvmlProcessInfoModeToRmId()` |
| RM 接收的 PID 查询结构 | CLASS 分支的输出 | SDK `ctrl2080gpu.h` 的 `NV2080_CTRL_GPU_GET_PIDS_PARAMS` |
