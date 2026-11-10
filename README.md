# vGPU-Unlock-patcher
A solution to patch vGPU_Unlock into nvidia driver, including possibility to create a merged one.

**_Support:_** [Join VGPU-Unlock discord for Support](https://discord.com/invite/5rQsSV3Byq)

This repository contains a submodule, so please clone this project recursively, i.e. using `git clone --recursive` command.

[补充使用说明](#补充使用说明)

## Usage

1. download original vgpu kvm `.run` files as available in a grid package `.zip` release, use the version matching the name of chosen branch of this project (the latest one is recommended)
   - versions of vgpu releases are listed [here](https://docs.nvidia.com/grid/index.html)
   - downloads are available from nvidia site [here](https://nvid.nvidia.com/dashboard/), evaluation account may be obtained [here](https://www.nvidia.com/object/vgpu-evaluation.html)
   - if you need merged driver, download also standard linux desktop driver .run file, the version as set in GNRL in the beginning of the patch.sh script
   - 580 的输入版本与从 GRID 派生 general 的方式见[补充使用说明](#补充使用说明)。

2. optionally edit `patch.sh` file to add support for your gpu if it was not included yet
   - search for "vcfgclone" lines, like for example:
        ```shell
        vcfgclone ${TARGET}/vgpuConfig.xml 0x1E30 0x12BA 0x1E84 0x0000
        ```
     the first two hex numbers select officially supported gpu as listed in vgpuConfig.xml (which can be extracted from vgpu kvm .run file)
     
     the example above is for Quadro RTX 6000 listed in the xml file as following:
        ```xml
        <pgpu><devId deviceId="0x1E30" subsystemId="0x12BA"/></pgpu>
        ```
     (fields that are not interesting for this example have been omitted)
     
   - the "vcfgclone" line example above is adding support for RTX 2070 Super, which has 10de:1e84 pci device id (that you can find for your gpu via `lspci -nn` command), so we are using the device id part, the last number 0x0000 is subdevice id, which may be used to differentiate some specific models, usually not needed, so we can use zero number there

   - just try to match the gpu architecture when adding a vcfgclone line, i.e. clone an officially supported pascal gpu if your gpu is pascal based

   - another example would be adding support for GTX 1080 Ti by cloning Tesla P40:
     Tesla P40 has 10de:1b38 pci device id and 10de:11d9 subsystem device id, listed in the xml as
        ```shell
        <pgpu><devId deviceId="0x1B38" subsystemId="0x0"/></pgpu>
        ```
        
        while the 1080 Ti can have 10de:1b06 pci devid with 10de:120f subsystem id for example, so the new vcfgclone line would have the first two numbers from the xml, the third number pci dev id of the card to be added and the fourth number can be zero or subsystem id (0x120f):
        ```shell
        vcfgclone ${TARGET}/vgpuConfig.xml 0x1B38 0x0 0x1B06 0x0000
        ```
        
   - when adding new vcfgclone lines, always refer into the xml to get the first two parameters and be sure to copy them case sensitively as the script searches for them as they are provided

3. Run one of these commands, depending on what you need:
      ```shell
      # a driver merged from vgpu-kvm with consumer driver (cuda and opengl for host too)
      ./patch.sh general-merge
      # 合并时保留 .07 GRID 前端
      ./patch.sh grid-merge
      # 合并 general/KVM，输出名称采用 KVM 风格
      ./patch.sh vgpu-kvm-merge
      # display output on host not needed (proxmox) or you have secondary gpu
      ./patch.sh vgpu-kvm
      # driver for linux vm
      ./patch.sh grid
      # driver for linux vm functionally similar to grid one but using consumer .run as input
      ./patch.sh general
      # stuff for windows vm
      ./patch.sh wsys
      ```

4. Change directory into the output folder (ending with `-patched`) to install the patched driver:
      ```shell
      ./nvidia-installer --kernel-module-type=proprietary --dkms
      ```
   If you have installed the driver before, the command above might fail, with the error log mentioning some services are still running. In that case, manually stop then with `systemctl stop nvidia*.service` and try again.

5. Optionally, if you want to override existing profiles, you can edit `/usr/share/nvidia/vgpu/vgpuConfig.xml`. The change will take effect after reboot or driver reloading.

   [Formula](https://discord.com/channels/829786927829745685/1162008346551926824/1171897739576086650) for overriding `profileSize` for X GiB of VRAM (X >= 1):
   ```
   profileSize = X * 0x40000000
   fbReservation = 0x8000000 + (profileSize - 0x40000000) / 0x10
   framebuffer = profileSize - fbReservation
   ```

   The minimal `profileSize` was [reported to be 384 MiB](https://discord.com/channels/829786927829745685/1162008346551926824/1168010185181249597). 

## Changelog
Please see commits history [here](https://github.com/VGPU-Community-Drivers/vGPU-Unlock-patcher/commits/).

### Other Options 
- any options need to be put before the target name
- `--repack` option that can be used to create unlocked/patched `.run` file (usually not necessary as you can simply start nvidia-installer from the directory).

## 补充使用说明

### 特别提醒

记得要用 `git clone --recursive` 来克隆仓库。

直接点那个 ZIP 下载可能导致 `unlock` 这个 submodule 没拉下来，进而导致构建失败。

嫌麻烦可以直接用成品，大多数人的优先选择。

毕竟 GitHub 在大清真的难访问，而且都是洋文，大多数人也看不懂。

### 变更补充说明

新增了 zstd 压缩，以应对大清特有之国情。

比如说什么百度网盘限速 10kb 之类的。

`--zstd` 与 `--repack` 配合使用，并放在目标名称之前。

前置步骤做完了但没 mdev 可以尝试：

```shell
systemctl restart nvidia-vgpud.service nvidia-vgpu-mgr.service
```

跑完了还是不出来那就找群友问问，什么 QQ 群之类的。

至于你说 Discord，你都看得懂英文了还需要这笔记吗。

### 关于 CMP 系列

关于 CMP 矿卡系列的修正，不完全在这个 repo 里面。

主要是改起来麻烦，就不塞一起了。

就比如说 CMP 100 的支持，那个 `unlock` 里面还得改，但又不在这个 repo 里面。

然后还有 40HX 之类的，BAR 大小又不一致。

### 补丁驱动的自用案例

进行操作前请确保该下的该改的，还有 vGPU KVM 驱动准备好。

当前分支的 KVM 输入为 `580.196.09-vgpu-kvm`，GRID/general 输入为 `580.196.07`。\
Linux 原包统一先在当前工作目录查找，再尝试仓库父目录中的同名 `.run`。\
缺少桌面版 `NVIDIA-Linux-x86_64-580.196.07.run` 时，`general`、`general-merge` 和 `vgpu-kvm-merge` 可从 `NVIDIA-Linux-x86_64-580.196.07-grid.run` 派生 general。\
GRID 派生与 merged 清单处理需要 `python3`，用于 manifest 与 sandbox JSON 的处理。

KVM 和 merged 流程默认使用 gcc 编译并集成 `libvgpucompat.so`，通过 patchelf 接入两个 vGPU 守护进程，设备 ID 伪装默认开启。\
`--UnlockRS` 可切换到 Rust 替代库，`--spoof-devid` 可切回 C 库。\
同时指定时以最后一个选项为准，每次仅集成一种拦截库。\
C 库支持运行时通过 `SPOOF_DEVID_OVERRIDE=0` 禁用伪装。\
非零值沿用高 16 位指定设备 ID、低 16 位指定子系统 ID 的覆盖方式。\
高 16 位为零时保留自动映射的设备 ID。

#### 杂种驱动

需要提前准备 `.07` 普通驱动或 GRID 驱动，以及 `.09` vGPU KVM 驱动。\
然后再创建杂种驱动：

```shell
./patch.sh --spoof-devid --repack --zstd general-merge
```

需要保留 GRID 前端时，将目标换成 `grid-merge`。

#### 纯种驱动

以下为包含实验选项的自用示例，按需选取参数，并将选项放在 `vgpu-kvm` 之前：

```shell
./patch.sh --spoof-devid --repack --force-nvidia-gpl-I-know-it-is-wrong --enable-nvidia-gpl-for-experimenting --test-dmabuf-export --envy-probes --zstd vgpu-kvm
```

#### 580 merged 驱动与模块参数

准备 `.07` GRID 与 `.09` vGPU KVM 两个原包后，可以创建 merged 驱动。\
所有脚本选项仍放在目标名称之前。

```shell
./patch.sh --repack grid-merge
```

`grid-merge`、`general-merge`、`vgpu-kvm-merge` 三个入口均保留。\
merged 使用 `.07` 的显示 wrapper 和桌面用户态，配入 `.09` 的 RM、vGPU 组件及固件。\
`.07` 的 `NV_VERSION_STRING` 和固件安装路径版本保留，不将整个分发包统一改标为 `.09`。\
`general-merge` 和 `vgpu-kvm-merge` 缺少桌面 `.07` 原包时使用 GRID 派生的 general，后者使用 KVM 风格的输出名称。\
三个 merged 入口共用 `.07` NVML 兼容补丁，保留 NULL 指针与短缓冲区检查，并保持 SysV ABI 调用栈对齐。\
该补丁在生成阶段应用，不由 `vup_merged` 模块参数控制。

以下为 `nvidia` 模块参数，不是 `patch.sh` 的命令行选项，完整说明见 [doc/options.rst](doc/options.rst)。

- `vup_merged` 与 `cudahost` 的默认值均随编译宏 `VUP_MERGED_DRIVER`，merged 流程设置为 `1`，普通 KVM 为 `0`。\
  `vup_merged` 仅精确等于 `1` 时应用对应补丁，`cudahost` 为正数时写入 host CUDA 标志，`0` 不改写该标志。
- `vup_swrlwar` 默认 `0`，保留静态补丁的“活动仍设置”行为。\
  仅精确等于 `1` 时开启 workaround，先检查输入数量，活动计数非零时提前成功返回而不执行设置，非活动路径仍保留容量检查。
- KVM 的 `vgpukvm` 默认 `1`，大于 `0` 才通过带门控的 hook 和运行时补丁，`vup_qmode` 默认且仅精确等于 `0` 时应用关闭 qmode 的补丁。
- 非 KVM 的 `gridext` 默认 `1`，大于 `0` 才允许 `vup_general` 与 `vupdevid` 生效，`vup_general` 默认 `0` 且仅精确等于 `1` 时启用。\
  `klogtrace` 默认 `0`，不受 KVM 的 `vgpukvm` 或非 KVM 的 `gridext` 门控。

这些门控不回滚生成阶段已经应用的静态字节补丁。\
本节记录参数实现与离线验证范围，安装、模块加载及显示、CUDA、vGPU 真机验收另行进行。

### 安装成品驱动命令说明

NVIDIA 驱动包含 `kernel` 和 `kernel-open` 两套模块，本分支的补丁使用 `kernel` 闭源驱动。\
因此需要显式指定使用 `kernel` 驱动，安装参数 `-m kernel` 与 `--kernel-module-type=proprietary` 等价。

根据你使用的场景选取以下的命令进行安装，如果需要其他参数请自行追加。\
下面的命令带上了 `--dkms`，按需保留：

```shell
./nvidia-installer --kernel-module-type=proprietary --dkms
```

或者直接安装打包文件：

```shell
sh NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm-patched.run --kernel-module-type=proprietary --dkms
```

### Credits
- Thanks to the discord user @mbuchel for the experimental patches
- Thanks to the discord user @LIL'pingu for the extended 43 crash fix
- Special thanks to @DualCoder without his work (vGPU_Unlock) we would not be here
- and thanks to the discord user @snowman for creating this patcher
