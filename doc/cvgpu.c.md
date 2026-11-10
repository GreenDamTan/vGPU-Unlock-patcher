# cvgpu.c：用户态 RM 查询兼容与源码对应说明

## 1. 用途、范围与加载方式

[`patches/cvgpu.c`](../patches/cvgpu.c) 是用户态 `ioctl()` interposer，编译为 `libvgpucompat.so` 后，先转调真实 `ioctl()`，再修改调用者收到的少量 RM Control 查询结果。\
默认生效的修改是 PCI device／subsystem ID 映射、将虚拟化模式查询结果设为 host-vGPU，以及一项独立的 InfoROM 错误状态转换。

这层修改作用于守护进程看到的查询结果，供其后续选择已有配置和执行初始化。\
XML 扩展与 profile 注册是另外的链路，`cvgpu.c` 自身不生成 profile，也不写 ROM 或 PCI 配置空间。

下文按模块、文件名和函数／宏列出源码对应，并摘录关键逻辑。

### 1.1 `patch.sh` 的默认接入

[`patch.sh`](../patch.sh) 初始设置 `SPOOF_DEVID=true`、`UnlockRS=false`、`FRANKENSTEIN=false`。\
`vgpu-kvm`、`grid-merge`、`general-merge`、`vgpu-kvm-merge` 都设置 `DO_VGPU=true`，进入以下 C 库编译与注入分支。

以下为脚本原文节选，省略外围条件和相邻补丁步骤：

```bash
gcc -o ${TARGET}/libvgpucompat.so -shared -fPIC -O2 -s -Wall "$BASEDIR/patches/cvgpu.c" || die "failed to build libvgpucompat.so"
echo "libvgpucompat.so 0755 VGX_LIB NATIVE MODULE:vgpu" >> ${TARGET}/.manifest
for s in nvidia-vgpu-mgr nvidia-vgpud
do
    patchelf --add-needed libvgpucompat.so ${TARGET}/${s} || die "failed to inject libvgpucompat.so into ${s}"
done
```

共享库作为 ELF `DT_NEEDED` 依赖随两个守护进程加载，导出的同名 `ioctl` 接管其动态符号调用。\
当前默认分支不是通过设置 `LD_PRELOAD` 接入，也没有向 `nvidia-smi` 添加这个依赖。

`--UnlockRS` 将 `UnlockRS` 设为真并将 `SPOOF_DEVID` 设为假，改为复制和注入 Rust 库。\
`--spoof-devid` 作相反选择，两项按参数出现顺序处理，最后指定的一项生效，默认路径只集成一种库。\
普通 `grid`、`general` 不设置 `DO_VGPU`，因此不经过此处的兼容库接入。

### 1.2 转调真实 `ioctl`

`cvgpu.c` 导出可变参数形式的 `int ioctl(int fd, unsigned long request, ...)`，取出第三个参数后，通过 `dlsym(RTLD_NEXT, "ioctl")` 保存下一层实现。\
默认编译下先执行 `real_ioctl(fd, request, data)`，后处理符合条件的返回数据，最终保留真实调用的整数返回值 `ret`。

当前实现按 request 和 RM command 选择修改，不额外检查 fd 对应的设备路径。\
`nvidia-vgpud`／`nvidia-vgpu-mgr` 的进程名判断只出现在调试转储函数中，常规结果修改的作用范围由共享库的实际加载范围决定。

## 2. RM Control 请求头与后处理门控

### 2.1 `0xC020462A` 与 `NVOS54_PARAMETERS`

`REQ_QUERY_GPU` 定义为 `0xC020462A`。\
在本项目的 Linux x86-64 ioctl 编码中，它对应读写方向、参数头长度 `0x20`、magic `'F'` 和 Unix escape `0x2A`，也就是 `_IOWR('F', NV_ESC_RM_CONTROL, NVOS54_PARAMETERS)`。

**源码定位：RM Unix 接口头 `nv_escape.h`、`nv-ioctl-numbers.h` 与 SDK 接口头 `nvos.h`。**\
以下为各文件的原文节选，分别列出相关宏和完整的参数结构：

```c
/* nv_escape.h */
#define NV_ESC_RM_CONTROL                           0x2A

/* nv-ioctl-numbers.h */
#define NV_IOCTL_MAGIC      'F'

/* nvos.h */
typedef struct
{
    NvHandle hClient;
    NvHandle hObject;
    NvV32    cmd;
    NvU32    flags;
    NvP64    params NV_ALIGN_BYTES(8);
    NvU32    paramsSize;
    NvV32    status;
} NVOS54_PARAMETERS;
```

`cvgpu.c` 的 `iodata_t` 使用自定义成员名，但其布局与这个 32 字节 RM Control 头对应：

| 字节偏移 | `cvgpu.c` 成员 | `NVOS54_PARAMETERS` 成员 | 含义 |
| --- | --- | --- | --- |
| `+0x00` | `unknown_1` | `hClient` | 客户端句柄 |
| `+0x04` | `unknown_2` | `hObject` | 被控制的对象句柄 |
| `+0x08` | `op_type` | `cmd` | RM Control 命令号 |
| `+0x0c` | `padding_1` | `flags` | 控制标志，不应只按自定义名字理解为填充 |
| `+0x10` | `result` | `params` | 命令参数／结果缓冲区指针，八字节 |
| `+0x18` | `op_size` | `paramsSize` | 命令参数长度，四字节 |
| `+0x1c` | `status` | `status` | RM 操作状态，四字节 |

C 实现对头大小和 `op_type`、`result`、`op_size`、`status` 偏移均有 `_Static_assert`。\
头中的 `status` 与系统调用的 `ret` 是两个返回通道，前者描述 RM 命令结果，后者描述 ioctl 调用结果。

**源码定位：RM Unix 用户态库 `rmapi.c`，`MANGLE_FUNC_NAME(NvRmControl)`。**\
该函数以 `NVOS54_PARAMETERS Parms` 组装请求，关键原文节选如下：

```c
memset(&Parms, 0, sizeof(Parms));
Parms.hClient         = hClient;
Parms.hObject         = hObject;
Parms.cmd             = cmd;
Parms.params          = NV_PTR_TO_NvP64(pParams);
Parms.paramsSize      = paramsSize;

if (!RmDoDeprecatedControl(&Parms, &nvosStatus))
    nvosStatus = DO_API_ESCAPE(ctl_handle, NV_ESC_RM_CONTROL, NVOS54_PARAMETERS, Parms);

if (nvosStatus != NV_OK)
    Parms.status = nvosStatus;
```

由此可将调用者的 `NvRmControl(..., cmd, pParams, paramsSize)` 与 interposer 读取的 `op_type`、`result`、`op_size` 逐项对应。

### 2.2 默认后处理顺序

以下是 `cvgpu.c` 中真实调用和公共提前返回条件的原文节选，中间调试与条件兼容代码省略：

```c
ret = real_ioctl(fd, request, data);

// Not a call we care about.
if (request != REQ_QUERY_GPU || data == NULL || ret < 0) return ret;
iodata = (iodata_t *)data;
```

后续先保留重试状态，再分别判断两种成功查询。\
以下同为原文节选：

```c
//Driver will try again
if (iodata -> status == STATUS_TRY_AGAIN) return ret;

if (spoof_devid_enabled > 0 && iodata->status == STATUS_OK &&
    iodata->result != NULL && iodata->op_size == 0x10 &&
    iodata->op_type == OP_READ_PCI_ID) {
```

`STATUS_OK=0`，`STATUS_TRY_AGAIN=3`。\
SDK 的 `nvstatuscodes.h` 将状态 `3` 命名为 `NV_ERR_BUSY_RETRY`，因此此处直接返回，让调用者保留原有重试处理，库本身不等待或重发。

| 后处理 | 公共条件 | 额外条件 | 修改对象 |
| --- | --- | --- | --- |
| PCI ID 查询 | request 精确匹配、`data != NULL`、`ret >= 0` | spoof 开启、RM 状态为零、`result != NULL`、长度精确为 `0x10`、命令为 `0x20801801` | 结果中的两个 16 位半字 |
| 虚拟化模式查询 | 同上 | spoof 开启、RM 状态为零、`result != NULL`、长度精确为 `8`、命令为 `0x800289` | 结果开头的 32 位模式值 |
| InfoROM 状态转换 | 同上 | 命令为 `0x2080014b`、RM 状态为 `0x56` | RM 头中的 `status` |

两个 spoof 分支以“精确长度”而非“至少这么长”判断。\
InfoROM 分支独立于 spoof 开关，不检查结果指针或参数长度，详见第 6 节。

## 3. PCI 查询、自动映射与 override

### 3.1 `0x20801801` 的十六字节结果

**源码定位：SDK 子设备 bus 控制头 `ctrl2080bus.h`。**\
相关定义原文节选如下，命令宏省略行尾的生成器注释：

```c
#define NV2080_CTRL_CMD_BUS_GET_PCI_INFO (0x20801801)

typedef struct NV2080_CTRL_BUS_GET_PCI_INFO_PARAMS {
    NvU32 pciDeviceId;
    NvU32 pciSubSystemId;
    NvU32 pciRevisionId;
    NvU32 pciExtDeviceId;
} NV2080_CTRL_BUS_GET_PCI_INFO_PARAMS;
```

在这里的小端布局中，`pciDeviceId` 低 16 位是 vendor ID，高 16 位是 device ID。\
`pciSubSystemId` 同样由 subsystem vendor ID 和 subsystem ID 组成，高 16 位才是本补丁修改的 subsystem ID。

| 结果位置 | 所属字段 | `cvgpu.c` 的操作 |
| --- | --- | --- |
| `+0x00..+0x01` | `pciDeviceId` 低半字 | 保留 vendor ID |
| `+0x02..+0x03` | `pciDeviceId` 高半字 | 读取／写入 device ID |
| `+0x04..+0x05` | `pciSubSystemId` 低半字 | 保留 subsystem vendor ID |
| `+0x06..+0x07` | `pciSubSystemId` 高半字 | 读取／写入 subsystem ID |
| `+0x08..+0x0b` | `pciRevisionId` | 保留 |
| `+0x0c..+0x0f` | `pciExtDeviceId` | 保留 |

补丁中的真实取址与初始化为：

```c
uint16_t * devid_ptr = (uint16_t * ) iodata -> result + 1;
uint16_t * subsysid_ptr = (uint16_t * ) iodata -> result + 3;
uint16_t actual_devid = * devid_ptr;
uint16_t spoofed_devid = actual_devid;
uint16_t actual_subsysid = * subsysid_ptr;
uint16_t spoofed_subsysid = actual_subsysid;
```

这里 `uint16_t *` 的 `+1`、`+3` 分别对应字节偏移 `+2`、`+6`。\
自动映射未赋新值的字段保留真实 ioctl 已返回的值，`actual_devid` 也是该返回值，不是重新读取 PCI 硬件得到的 ID。

### 3.2 完整自动映射表

下表按 C 中的执行顺序列出全部七个独立 `if`。\
所有区间均为**闭区间**，分组名和目标型号名沿用代码注释，精确行为以数值条件为准。\
这些宽区间和跨系列目标用于描述现有映射规则，不能将注释中的“同芯片”意图当作每个输入都满足的硬件保证。

| 顺序／代码分组 | 对 `actual_devid` 的精确条件 | 输出 device ID | 输出 subsystem ID | 目标注释 |
| --- | --- | --- | --- | --- |
| 1，Maxwell | `[0x1340, 0x137f]` 或 `[0x1380, 0x13bf]` | `0x13bd` | `0x1160` | Tesla M10 |
| 2，Maxwell 2.0 | `[0x17c0, 0x17ff]` 或 `[0x13c0, 0x13ff]` 或 `[0x1400, 0x143f]` | `0x13bd` | `0x1160` | Tesla M10 |
| 3，Pascal | `[0x15c0, 0x15ff]` 或 `[0x1b00, 0x1b3f]` 或 `[0x1b80, 0x1bbf]` 或 `[0x1c00, 0x1c3f]` 或 `[0x1c80, 0x1cbf]` 或 `[0x1d00, 0x1d3f]` | `0x2233` | `0x165a` | A5500 |
| 4，GV100 Volta | `[0x1d80, 0x1dbf]` 或 `0x1df4` | `0x1db6` | 保留输入值 | Tesla V100 32GB PCIE |
| 5，Turing | `actual_devid != 0x1eb8`，且位于（`[0x1e02, 0x1ff9]` 或 `[0x2182, 0x21d1]`） | `0x1e30` | `0x12ba` | Quadro RTX 6000 |
| 6，Ampere | `[0x2080, 0x20bf]` 或 `0x20c2` 或 `[0x2200, 0x2600]` | `0x2230` | 保留输入值 | RTX A6000 |
| 7，GH | `[0x2300, 0x233f]` 或 `[0x2600, 0x263f]` | `0x26b3` | `0x1934` | NVIDIA RTX 5880 Ada Generation |

所有条件均比较同一个 `actual_devid`，不是上一步得到的 `spoofed_devid`，所以映射结果不会再次作为输入级联匹配。\
但它们也不是 `else if`，后命中的赋值会覆盖前面的结果，GH 与 Ampere 的交集为 `[0x2300, 0x233f]` 和单点 `0x2600`，最终都取 GH 的 `0x26b3／0x1934`。

T4 的 `0x1eb8` 被明确排除于 Turing 分支，在没有 override 时保留原有 PCI 标识。\
未命中任何分支的输入也保留原 device／subsystem ID，Volta 和仅命中 Ampere 的输入则只改变 device ID。\
这些 PCI 映射是否命中，不影响另一个虚拟化模式查询分支的独立判断。

### 3.3 `SPOOF_DEVID_OVERRIDE` 的实际语义

解析发生在首次初始化真实 `ioctl` 函数指针时，而不是每次 PCI 查询时。\
以下为初始化代码原文节选：

```c
if (!real_ioctl) {
  char *str = getenv("SPOOF_DEVID_OVERRIDE");
  if (str != NULL) {
      unsigned val = strtoul(str, NULL, 0);
      spoof_devid_override = val;
      spoof_devid_enabled = val ? 1 : 0;
  }
  real_ioctl = dlsym(RTLD_NEXT, "ioctl");
}
if (spoof_devid_enabled < 0)
  spoof_devid_enabled = spoof_devid_opt[strlen((char *)spoof_devid_opt) - 1] == '1' ? 1 : 0;
```

未设置环境变量时，回退到内嵌字符串 `enable_spoof_devid=1` 的最后一个字符，当前默认为开启。\
设置变量时使用 `strtoul(..., NULL, 0)`，base 0 允许十进制、`0x` 十六进制和前导零八进制，结果保存到 `unsigned`，代码未另行检查解析终点或错误状态。

| 解析并保存后的值 | 运行行为 |
| --- | --- |
| `0` | 关闭 PCI ID 和虚拟化模式两项 spoof，InfoROM 状态转换仍保留 |
| 非零且高 16 位非零 | 开启 spoof，在自动映射后以高 16 位覆盖 device ID，以低 16 位覆盖 subsystem ID |
| 非零且高 16 位为零 | 开启 spoof，device ID 保留自动映射结果或原值，低 16 位仍覆盖 subsystem ID |

override 在全部自动映射之后执行，随后才统一写回两个半字。\
对应原文如下：

```c
if (spoof_devid_override) {
    if ((spoof_devid_override >> 16) != 0)
        spoofed_devid = (spoof_devid_override >> 16);
    spoofed_subsysid = spoof_devid_override & 0xffff;
}
* devid_ptr = spoofed_devid;
* subsysid_ptr = spoofed_subsysid;
```

例如 `0x2230165a` 强制输出 device `0x2230`、subsystem `0x165a`。\
`0x0000165a` 只覆盖 subsystem，`0x22300000` 则覆盖 device 并明确将 subsystem 写为零。\
`0x00000000` 的意义是关闭 spoof，而不是“保持开启并把 subsystem 改为零”，进程内后续环境变量变化也不会重新触发这段初始化。

### 3.4 RM 服务端的原始 PCI 信息填充

**源码定位：RM kernel bus 的 `kern_bus_ctrl.c`，`subdeviceCtrlCmdBusGetPciInfo_IMPL()`。**\
原始函数先检查 bus family，支持 PCI 时从 GPU 对象取出 PCI 信息，以下为填充四个结果字段的原文节选：

```c
pPciInfoParams->pciDeviceId = pGpu->idInfo.PCIDeviceID;
pPciInfoParams->pciSubSystemId = pGpu->idInfo.PCISubDeviceID;
pPciInfoParams->pciRevisionId = pGpu->idInfo.PCIRevisionID;

//
// Return device ID field.  We no longer support probing past the BR02 bridge.
//
pPciInfoParams->pciExtDeviceId = REF_VAL(NV_CONFIG_PCI_NV_0_DEVICE_ID, pPciInfoParams->pciDeviceId);
```

这些字段填充完成后，原始函数返回 `NV_OK`，不支持 PCI bus family 时返回 `NV_ERR_NOT_SUPPORTED`。\
`cvgpu.c` 的处理发生在 ioctl 返回之后，只写 device／subsystem 的两个半字，`pciExtDeviceId` 保留服务端已经返回的值。

## 4. 虚拟化模式：`0x800289` 与接口布局

**源码定位：SDK device GPU 控制头 `ctrl0080gpu.h`。**\
学习源码中的命令、模式枚举和参数结构如下：

```c
#define NV0080_CTRL_CMD_GPU_GET_VIRTUALIZATION_MODE   (0x800289)
#define NV0080_CTRL_GPU_VIRTUALIZATION_MODE_HOST      (0x00000003)
#define NV0080_CTRL_GPU_VIRTUALIZATION_MODE_HOST_VGPU NV0080_CTRL_GPU_VIRTUALIZATION_MODE_HOST

typedef struct NV0080_CTRL_GPU_GET_VIRTUALIZATION_MODE_PARAMS {
    NvU32 virtualizationMode;
} NV0080_CTRL_GPU_GET_VIRTUALIZATION_MODE_PARAMS;
```

这里的 `3` 是 `HOST`／`HOST_VGPU` 模式枚举，C 中的 `DEV_TYPE_VGPU_CAPABLE` 是对此值的本地命名。\
对应命令是 GET 查询，修改的是此次返回值，而不是发送 `SET_VIRTUALIZATION_MODE` 命令。

| 接口来源 | 参数布局 | 总长 | 当前 C 的处理 |
| --- | --- | --- | --- |
| 学习源码 SDK 头 | 一个 `NvU32 virtualizationMode` | `4` 字节 | 用于解释命令和枚举，四字节结果不满足当前 C 的长度门控 |
| 当前 R580 兼容结构 | `uint32_t virtualization_mode`、`uint8_t is_grid_build`、尾部对齐填充 | `8` 字节 | 精确匹配后只将前四字节写为 `3` |

以下为 `cvgpu.c` 的结构和静态断言原文：

```c
typedef struct {
    uint32_t virtualization_mode;
    uint8_t is_grid_build;
} gpu_virtualization_mode_t;

_Static_assert(sizeof(gpu_virtualization_mode_t) == 8, "virtualization mode parameter size");
_Static_assert(offsetof(gpu_virtualization_mode_t, is_grid_build) == 4, "isGridBuild offset");
```

实际赋值在成功、非空和八字节门控之后执行，原文节选如下：

```c
gpu_virtualization_mode_t *params = iodata->result;
params->virtualization_mode = DEV_TYPE_VGPU_CAPABLE;
```

因此 `is_grid_build` 和尾部三个填充字节均保留真实返回内容。\
PCI override 的具体 device／subsystem 数值也不参与模式赋值，只共享 spoof 是否开启这一开关。

**服务端定位：RM device 的 `device_ctrl.c`，`deviceCtrlCmdGpuGetVirtualizationMode_IMPL()`。**\
原始函数先根据 virtual 和 passthrough 选择 guest 模式，host 分支要求 hypervisor 及 `isGridLicenseSupported()` 成立，再按 host-vGPU／host-vSGA 属性选择对应模式，其余路径报告 `NONE`。\
该函数用于理解模式从何而来，R580 C 库则在真实查询完成后统一改写符合门控的返回值。

## 5. 查询结果在 vgpud 中的消费位置

### 5.1 PCI ID 成为配置查找键

**源码定位：vGPU 守护进程的 `vgpuConfig.c`，`vgpuConfigOpenPhysicalGpu()`。**\
它建立 device／subdevice 句柄后查询 PCI 信息，再把两个组合字段的高半字转为配置查找键，以下为原文节选：

```c
memset(&pci_info, 0, sizeof(NV2080_CTRL_BUS_GET_PCI_INFO_PARAMS));
ret = NvRmControl(vgpuConfigClient,
                  pGpu->subdevice,
                  NV2080_CTRL_CMD_BUS_GET_PCI_INFO,
                  &pci_info,
                  sizeof(NV2080_CTRL_BUS_GET_PCI_INFO_PARAMS));
```

查询成功后的赋值原文为：

```c
pGpu->devId.vendorId = PCI_VENDOR_ID_NVIDIA;
pGpu->devId.deviceId = (VGPU_CONFIG_U16) (pci_info.pciDeviceId >> 16);
pGpu->devId.subsystemVendorId = NV_PCI_SUBID_VENDOR_NVIDIA;
pGpu->devId.subsystemId = (VGPU_CONFIG_U16) (pci_info.pciSubSystemId >> 16);
```

这正是 C 库选择字节偏移 `+2`、`+6` 的消费依据。\
同一源码的 `vgpuConfigSetConfigInfo()` 取得 pGPU 列表后查询虚拟化模式，调用失败走错误处理，模式为 `VGX` 或 `NMOS` 时以 guest 不支持配置为由退出，其余路径继续读取配置。

### 5.2 模式检查后查找 pGPU 条目并注册配置

配置存储在函数开头经 `vgpuConfigOpen()`／`xmlFileLoad()` 加载，模式检查之后进行的是按设备键查找 pGPU 配置条目，再逐项读取和提交类型。\
`vgpuConfigReadPhysGpu()` 将该键交给 `xmlReadPhysGpu()`，因此查询得到的 device／subsystem ID 会进入 XML 条目匹配。

原始 `vgpuConfigSetConfigInfo()` 逐个使用 `&pGpus[index].devId` 调用 `vgpuConfigReadPhysGpu()`，查找失败时记录日志并跳过该 GPU。\
查找成功后，后续字段继续使用 `pGpus[index].devId`。

**XML 匹配定位：vGPU 守护进程的 `vgpuConfigXmlParser.c`，`xmlReadPhysGpu()`。**\
它按四段 PCI 标识匹配，XML 条目的 subsystem ID 为零时先将查询键的同一字段设为零，命中后把 XML 的四段 ID 写入输出 pGPU 结构，关键原文为：

```c
if (devId.subsystemId == 0x0)
    pDevId->subsystemId = 0x0;

if(!devIdCmp(&devId, pDevId))
{
    // Matched devId
    memcpy(&ppGpu->devId, &devId, sizeof(VGPU_CONFIG_VGPU_4_PART_DEVID));
```

因此查询返回值是配置匹配的输入键，匹配后的 `pGpus[index].devId` 还可能由 XML 条目更新。

随后独立读取支持的 vGPU 类型，填充 `NVA081_CTRL_VGPU_CONFIG_INFO_PARAMS` 中的 `NVA081_CTRL_VGPU_INFO`，经 `NVA081_CTRL_CMD_VGPU_CONFIG_SET_INFO` 提交给 RM。\
以下是同一函数中配置填充和提交的原文节选，不相邻部分以注释标出：

```c
configParams.vgpuInfo.vgpuType           = vgpuType.vgpuTypeId;
configParams.vgpuInfo.maxInstance        = supportedVgpu.maxVgpus;
configParams.vgpuInfo.vdevId             = ((NvU32)vgpuType.virtualDevId.deviceId << 16)
                                           | vgpuType.virtualDevId.subsystemId;
configParams.vgpuInfo.pdevId             = pGpus[index].devId.deviceId;
configParams.vgpuInfo.fbLength           = vgpuType.framebufferSize;
/* 省略其他配置字段赋值。 */
convertSignatureStringToBinary(supportedVgpu.signature,
                               configParams.vgpuInfo.vgpuSignature);

// Send configuration data for this vGPU type to RM
ret = NvRmControl(vgpuConfigClient,
                  vgpuConfigHandle,
                  NVA081_CTRL_CMD_VGPU_CONFIG_SET_INFO,
                  &configParams,
                  sizeof(configParams));
```

SDK 的 `ctrla081.h` 定义该命令为 `0xa0810101`，并定义配置结构和签名字段。\
这条输入配置注册链决定具体 profile 的显存、显示头、CUDA 等字段，`cvgpu.c` 对前述两种查询的修改只作用于它之前的设备识别与模式结果。

**原始验签逻辑定位：RM vGPU 配置 API 的 `vgpuconfigapi.c`，`vgpuconfigapiCtrlCmdVgpuConfigSetInfo_IMPL()`。**\
配置接口调用 `verifyVgpuSignature()`，验证失败时记录错误并立即返回：

```c
if ((rmStatus = verifyVgpuSignature(&pParams->vgpuInfo)) != NV_OK)
{
    NV_PRINTF(LEVEL_ERROR, "Digital signature verification failed.\n");
    return rmStatus;
}
```

C 库改写的是前面的查询结果，配置签名验证在 RM 的配置注册路径中执行。

## 6. 独立的 InfoROM 状态转换

`cvgpu.c` 末尾保留以下原文逻辑：

```c
if (iodata->op_type == 0x2080014b && iodata->status == 0x56) {
  iodata->status = 0x57;
}
```

**接口定位：SDK 子设备 GPU 控制头 `ctrl2080gpu.h`，`NV2080_CTRL_CMD_GPU_GET_INFOROM_OBJECT_VERSION`。**\
命令值为 `0x2080014b`，参数 `NV2080_CTRL_GPU_GET_INFOROM_OBJECT_VERSION_PARAMS` 由三字节 `objectType`、一字节 `version`、一字节 `subversion` 组成，共五字节。\
该接口参数长五字节，状态转换只检查命令号和状态码，不依赖参数长度或结果缓冲区内容。

`nvstatuscodes.h` 中的原文定义为：

```c
NV_STATUS_CODE(NV_ERR_BUSY_RETRY,                      0x00000003, "System is busy, retry later")
NV_STATUS_CODE(NV_ERR_NOT_SUPPORTED,                   0x00000056, "Call not supported")
NV_STATUS_CODE(NV_ERR_OBJECT_NOT_FOUND,                0x00000057, "Requested object not found")
```

此处将“不支持此调用”转为“请求对象不存在”，仍保留错误状态而不是改成 `NV_OK`。\
代码注释说明其用途是兼容某些不支持 InfoROM 读取的 Maxwell 卡，条件本身没有再按 GPU 型号或 objectType 限定。\
该处理仍受 request、头指针和 `ret >= 0` 的公共条件约束，但不受 `spoof_devid_enabled`、结果指针和精确长度门控，系统调用返回值 `ret` 原样保留。

**服务端定位：RM physical subdevice 的 `subdevice_ctrl_gpu.c`，`subdeviceCtrlCmdGpuGetInforomObjectVersion_IMPL()`。**\
原始函数在缺少 InfoROM 对象时返回 `NV_ERR_NOT_SUPPORTED`：

```c
if (!pInforom)
{
    return NV_ERR_NOT_SUPPORTED;
}
```

对象存在时，函数检查锁并调用 `inforomGetObjectVersionInfo()`。\
它可定位状态来源与接口职责，C 库的转换则发生在服务端已经返回之后。

`patch.sh` 的 `DO_VGPU` 流程还会应用 [`workaround-for-cards-with-inforom-error.patch`](../patches/workaround-for-cards-with-inforom-error.patch)。\
该补丁在内核 wrapper 的 `nvidia_ioctl()` 中、`rm_ioctl()` 返回后，对 RM Control 头执行同名状态转换，因而是另一个层面的配套处理。\
若内核侧已经返回 `0x57`，C 库末尾的 `0x56` 条件自然不会再次命中。

## 7. 当前默认行为与保留的条件兼容代码

### 7.1 默认编译状态

当前 C 文件为 `//#define DO_FIXUPS`，`patch.sh` 的上述 gcc 命令也未传入 `-DDO_FIXUPS`。\
`dbg_trace_level` 初值为 `0`，因此默认有效路径是两种成功查询的 spoof 加独立 InfoROM 状态转换，结构转换和请求转储默认关闭。

`ioctl_fixups` 的扫描代码虽然在函数中保留，但未定义 `DO_FIXUPS` 时数组只有 `{ 0, 0, 0 }` 终止项，不会选择替换缓冲区。\
`dbg_trace_level` 为源码内静态变量，当前实现没有为它读取环境变量。

### 7.2 `DO_FIXUPS` 条件分支

`DO_FIXUPS` 包含以下结构转换和状态处理，当前默认编译不启用这些分支。

| 条件代码 | 保留的处理方式 | 对应源码接口位置 |
| --- | --- | --- |
| NUMA ioctl | 把不含 `use_auto_online` 的结构转换为含该字节的 `_535` 结构，设该字段为零，转调后复制共同字段并返回 | RM Unix 的 `nv-ioctl-numa.h`，`NV_ESC_NUMA_INFO` |
| RM allocation | 对 `NVOS21_PARAMETERS_535`／`NVOS64_PARAMETERS_535` 和列出的 Maxwell、Pascal、Volta、Turing、Ampere、Hopper GPFIFO class，扩大 channel 参数缓冲区，复制前 `0x130` 字节并在调用后恢复原指针 | SDK `nvos.h` 的 RM 分配头，扩容字节数见 C 实现 |
| RM Control 尺寸表 | 命令与旧长度都匹配时分配新尺寸缓冲区，扩展部分清零，调用后恢复旧指针／长度并复制共同部分 | 对应命令的 SDK control 参数头 |
| guest MSI 布局 | 在尺寸替换中，将 `vmIdType`／`guestVmId` 转为 `_535` 的 `vgpuUuid`／`domainId`，返回时保留旧请求结构内容 | SDK `ctrl2080event.h`，`NV2080_CTRL_CMD_EVENT_SET_GUEST_MSI` |
| 部分失败状态 | 将 `0xa082012c` 和 guest MSI 命令 `0x20800305` 的失败 RM 状态改为零，保留系统调用返回值 | `0xa082012c` 按 C 中的命令数值定位，guest MSI 见 `ctrl2080event.h` |

表中的 `_535` 结构由 `cvgpu.c` 自行定义，用于描述转调目标布局。\
原始 SDK 提供旧请求结构和相关命令的阅读入口，转换字段及尺寸以 C 实现为准。

条件尺寸表的全部有效条目如下，数值按 C 原样记录：

| RM command | 输入长度 | 转调长度 |
| --- | --- | --- |
| `0x20800102` | `0x001f4` | `0x00204` |
| `0x20801228` | `0x001b0` | `0x001c0` |
| `0x2080a0a4` | `0x1073c` | `0x10740` |
| `0x2080182b` | `0x00014` | `0x0001c` |
| `0xa0840101` | `0x00104` | `0x00100` |
| `0x20800a2a` | `0x00cc0` | `0x00d40` |
| `0x20800188` | `0x00004` | `0x00008` |
| `0x20800305` | `0x00028` | `0x00030` |

表中的 `0x20800188` 在原始 SDK 中是 `NV2080_CTRL_CMD_GPU_GET_MAX_SUPPORTED_PAGE_SIZE`，与当前虚拟化模式命令 `0x800289` 分别处理。\
源文件还在 `DO_FIXUPS` 内保留一个嵌套 `#if 0` 分支，针对 `0x137` 输出并重写四个版本字，分别写入 `0x070001`、`0x110001`、`0x070001`、`0x110001`，即使定义 `DO_FIXUPS`，该分支也仍被排除。

### 7.3 调试代码

提高源码中的 `dbg_trace_level` 后，级别 1 开始记录失败命令，级别 2 增加失败状态与尺寸修复日志，级别 3 增加常规请求／返回日志，级别 4 才调用 `dbgdump()` 转储请求和结果。\
转储函数只为两个守护进程使用临时目录下的 `vgpu-traces` 子目录，其中 `nvidia-vgpud` 的序号从 `0` 开始，`nvidia-vgpu-mgr` 从 `0x100` 开始。\
当前默认级别为零，这些目录和文件不会因默认路径的转储调用而生成。

## 8. 源码阅读索引

以下按模块、文件名与符号列出源码阅读索引：

| 模块 | 文件名 | 关键符号 | 阅读目的 |
| --- | --- | --- | --- |
| SDK RM 操作接口 | `nvos.h` | `NVOS54_PARAMETERS`、`NVOS21_PARAMETERS`、`NVOS64_PARAMETERS` | RM Control 头与 RM 分配头 |
| RM Unix ioctl 接口 | `nv_escape.h`、`nv-ioctl-numbers.h` | `NV_ESC_RM_CONTROL`、`NV_IOCTL_MAGIC` | ioctl 编码的 escape 与 magic |
| RM Unix 用户态库 | `rmapi.c` | `MANGLE_FUNC_NAME(NvRmControl)`、`DO_API_ESCAPE` 调用点 | 命令、指针、尺寸和状态的封装 |
| SDK subdevice bus 控制 | `ctrl2080bus.h` | `NV2080_CTRL_CMD_BUS_GET_PCI_INFO`、`NV2080_CTRL_BUS_GET_PCI_INFO_PARAMS` | 十六字节 PCI 查询布局 |
| RM kernel bus | `kern_bus_ctrl.c` | `subdeviceCtrlCmdBusGetPciInfo_IMPL` | PCI 查询的服务端填充 |
| SDK device GPU 控制 | `ctrl0080gpu.h` | `NV0080_CTRL_CMD_GPU_GET_VIRTUALIZATION_MODE`、`NV0080_CTRL_GPU_GET_VIRTUALIZATION_MODE_PARAMS`、`NV0080_CTRL_GPU_VIRTUALIZATION_MODE_HOST_VGPU` | 学习源码的四字节结构和模式 `3` |
| RM device 控制 | `device_ctrl.c` | `deviceCtrlCmdGpuGetVirtualizationMode_IMPL` | 原始 guest／host 模式选择和 NONE 回退 |
| vGPU 守护进程 | `vgpuConfig.c` | `vgpuConfigOpenPhysicalGpu`、`vgpuConfigSetConfigInfo` | 高半字消费、模式检查、配置查找与提交 |
| vGPU XML 解析 | `vgpuConfigXmlParser.c` | `xmlReadPhysGpu`、`devIdCmp` | 四段键比较、零 subsystem 处理和匹配键写回 |
| SDK vGPU 配置接口 | `ctrla081.h` | `NVA081_CTRL_VGPU_INFO`、`NVA081_CTRL_VGPU_CONFIG_INFO_PARAMS`、`NVA081_CTRL_CMD_VGPU_CONFIG_SET_INFO` | 独立 profile 注册参数 |
| RM vGPU 配置 API | `vgpuconfigapi.c` | `vgpuconfigapiCtrlCmdVgpuConfigSetInfo_IMPL`、`verifyVgpuSignature` | 原始配置验签及错误返回 |
| SDK subdevice GPU 控制 | `ctrl2080gpu.h` | `NV2080_CTRL_CMD_GPU_GET_INFOROM_OBJECT_VERSION`、`NV2080_CTRL_GPU_GET_INFOROM_OBJECT_VERSION_PARAMS` | InfoROM 查询命令和五字节参数 |
| RM physical subdevice | `subdevice_ctrl_gpu.c` | `subdeviceCtrlCmdGpuGetInforomObjectVersion_IMPL` | 原始 InfoROM 对象检查和状态来源 |
| SDK 状态码 | `nvstatuscodes.h` | `NV_ERR_BUSY_RETRY`、`NV_ERR_NOT_SUPPORTED`、`NV_ERR_OBJECT_NOT_FOUND` | `3`、`0x56`、`0x57` 的准确名称 |
