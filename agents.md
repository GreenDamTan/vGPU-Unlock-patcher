# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目与版本范围

本项目是 NVIDIA 驱动补丁器，在原始驱动分发包上集成 unlock、源码补丁、二进制补丁和 vGPU profile，输出可安装目录或重新打包的 `.run`。\
驱动源码及编译所需的 Makefile 来自 NVIDIA 分发包。

当前分支的 KVM 输入为 `580.196.09-vgpu-kvm`，GRID/general 输入为 `580.196.07`，主要目标为 `kernel/` 中的闭源模块。\
`patches/nv_hooks.c` 同时保留 KVM 与非 KVM 分支，非 KVM 使用 `.07` 的 RM 偏移及 `gridext` 门控。\
merged 使用 `.07` 的显示 wrapper 和桌面用户态，配入 `.09` 的 RM、vGPU 组件及固件，同时保留 `.07` 的 `NV_VERSION_STRING` 与固件安装路径版本。\
`grid-merge`、`general-merge`、`vgpu-kvm-merge` 三入口均保留，缺少桌面 `.07` 原包时可从 `.07` GRID 派生 general。\
版本兼容性通过分支管理，Windows 入口及旧版本资源仍有各自的版本配置。\
修改前先核对脚本开头的 `VGPU`、`GNRL`、`GRID`、`WSYS`。

`README.md` 包含使用说明与补充使用说明。\
`doc/options.rst` 介绍模块参数的设计背景，当前参数、默认值和启用条件以 `patches/nv_hooks.c` 及对应源码补丁为准。

## 常用命令

以下命令从仓库根目录运行。\
Linux 补丁流程使用 Bash、GNU patch/coreutils，GRID 派生与 merged 的 manifest/sandbox JSON 处理还需要 `python3`。\
KVM 和 merged 默认集成 C 兼容库，需要 gcc 和 patchelf。\
内核构建还需要 make、与目标内核匹配的编译器及 headers。

### 初始化与轻量检查

```bash
git submodule update --init --recursive
bash -n patch.sh
gcc -fshort-wchar -fsyntax-only nsigpatch.c
git -c core.whitespace=cr-at-eol diff --check
```

克隆时使用 `git clone --recursive`。`unlock/` 是固定 gitlink 的子模块，需连同仓库一起准备。`nsigpatch.c` 属于 Windows PE 补丁路径，`-fshort-wchar` 与脚本内的实际编译参数保持一致。

### 生成与重打包

KVM 准备 `NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm.run`，GRID/general 准备 `NVIDIA-Linux-x86_64-580.196.07-grid.run` 或桌面 `NVIDIA-Linux-x86_64-580.196.07.run`。\
Linux `.run` 输入统一先在当前工作目录查找，再尝试仓库父目录中的同名包。\
`general`、`general-merge` 和 `vgpu-kvm-merge` 找不到桌面 `.07` 原包或当前目录中的已解包目录时，可从 `.07` GRID 派生 general。

```bash
./patch.sh vgpu-kvm
./patch.sh grid
./patch.sh general
# merged 需要同时准备 .07 GRID/桌面版与 .09 vGPU KVM 输入
./patch.sh grid-merge
./patch.sh general-merge
./patch.sh vgpu-kvm-merge
```

KVM 默认输出目录为 `NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm-patched`。\
`grid-merge` 输出 `NVIDIA-Linux-x86_64-580.196.07-grid-vgpu-kvm-patched`，`general-merge` 输出 `NVIDIA-Linux-x86_64-580.196.07-merged-vgpu-kvm-patched`，`vgpu-kvm-merge` 输出 `NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm-merged-patched`。\
脚本会复用已有原始解包目录，并删除、重建同名 patched 输出目录，merged 还会重建对应合并源目录。\
运行前确认其中没有需要保留的修改，保留原始 `.run`，补丁失败后使用干净输入重新生成。

需要 `.run` 安装包时：

```bash
./patch.sh --repack vgpu-kvm
```

使用仓库内 `tools/zstd` 压缩并嵌入解压器：

```bash
./patch.sh --repack --zstd vgpu-kvm
```

选项放在目标名称之前；`--zstd` 和 `--strip_kernel_open` 都依赖 `--repack`。重打包会覆盖同名 `.run`，并为 `tools/zstd` 增加执行权限，即使未选择 `--zstd`。重打包直接归档输出目录，应使用干净 payload；模块编译在副本中进行，避免将 `.o`、`.ko`、`.cmd` 和构建日志带入安装包。

### 编译与检查闭源模块

先创建独立副本，再在副本中编译：

```bash
build_dir=$(mktemp -d "$PWD/NVIDIA-Linux-x86_64-580.196.09-build.XXXXXX")
target="$build_dir/patched"
cp -a NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm-patched "$target"
make -C "$target/kernel" -j"$(nproc)" \
  KERNEL_UNAME="$(uname -r)" \
  NV_KERNEL_MODULES="nvidia nvidia-vgpu-vfio" modules
modinfo -F version "$target/kernel/nvidia.ko"
modinfo -F vermagic "$target/kernel/nvidia.ko"
modinfo -F version "$target/kernel/nvidia-vgpu-vfio.ko"
modinfo -F vermagic "$target/kernel/nvidia-vgpu-vfio.ko"
```

生成目录的 `kernel/Makefile` 支持用 `KERNEL_UNAME` 选择其他已安装 headers 的内核，也支持 `SYSSRC`、`SYSOUT` 显式指定源码和输出树。`module` 是 `modules` 的兼容目标。以上命令只编译和查询模块信息。

### 定向验证

仓库的验证入口是语法检查、补丁应用、生成内容检查和内核模块构建；`test-*.patch` 是功能补丁。验证单个源码补丁时，对尚未应用该补丁的驱动副本使用 dry-run，例如：

```bash
target=NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm
patch --batch --fuzz=0 --dry-run -d "$target" -p1 < patches/setup-vup-hooks.patch
```

共享源码补丁在 `kernel-open/` 中对应 `-p2`，与脚本的 `applypatchx()` 一致；完整验证仍按脚本中的顺序执行。

检查重打包产物：

```bash
package=NVIDIA-Linux-x86_64-580.196.09-vgpu-kvm-patched.run
sh "$package" --check
sh "$package" --info
sh "$package" --script
sha256sum "$package"
```

当前 `patch.sh` 的重打包入口使用 `./nvidia-installer -m kernel`；README 中手动安装使用等价的显式选择 `--kernel-module-type=proprietary`。归档检查与模块构建之外，安装、加载、服务操作和真机验证应由用户明确指定。

## 跨文件架构

### 补丁流水线

`patch.sh` 是入口和编排层。Linux `vgpu-kvm` 的主链路为：解包或复用输入 → 复制到 patched 目录 → 集成 legacy unlock 与 runtime hook 源文件 → 应用静态 blob/vgpud diff → 应用源码补丁 → 扩展 XML → 可选重打包。

- `extract()` 处理 NVIDIA 自解压包，Linux `.run` 在当前工作目录找不到时统一回退到仓库父目录。\
  解包目录和输出仍按当前工作目录定位，`patches/`、`unlock/` 等资源按脚本的 `BASEDIR` 定位。\
  首次提取还会给 `nvidia.Kbuild` 追加换行，并为文件增加 owner-write 权限。
- `applypatch()` 使用 `-p1 --fuzz=0`，失败终止，`applypatchx()` 将同一补丁先应用到闭源树，再以 `-p2` 检查和应用到 open 树。\
  修改共享补丁时需核对两棵源码树。
- `.manifest` 是 NVIDIA 安装器的文件清单。\
  新增 hook、链接脚本或用户态库时，同时维护文件复制、Kbuild 接入和 manifest 记录，GRID 派生及 merge 流程还使用 `python3` 同步处理 manifest/sandbox JSON。\
  merged 的显示 wrapper、桌面用户态和版本头来自 `.07`，RM、vGPU 组件及固件来自 `.09`，不可把所有版本字符串机械替换为 `.09`。\
  保留 `.07` 的固件安装路径版本，以便 `.07` wrapper 请求的路径对应合入的 `.09` 固件内容，同时维护两份清单中所需文件。
- 三个 merged 入口通过同一个 `DO_MRGD` 分支应用 `patches/libnvidia-ml.so.580.196.diff`，目标是 `.07` 的 NVML。\
  该补丁保留版本查询的 NULL 指针与短缓冲区检查，并保持调用原 `strcpy@plt` 时的 SysV ABI 栈对齐。\
  该补丁属于生成阶段的用户态二进制补丁，不由 `vup_merged` 控制。
- KVM 和 merged 默认编译 `patches/cvgpu.c` 为启用设备 ID 伪装的 `libvgpucompat.so`，并通过 patchelf 添加到 `nvidia-vgpud`、`nvidia-vgpu-mgr` 的依赖。\
  `--UnlockRS` 选择 Rust 替代库，`--spoof-devid` 选择 C 库，最后指定的选项生效且仅集成一种。\
  C 库的 `SPOOF_DEVID_OVERRIDE=0` 可在运行时禁用伪装，非零值保留设备和子系统 ID 覆盖能力。

### 三层内核补丁机制

1. **legacy unlock**：KVM 和 merged 从 `unlock/` 复制 `vgpu_unlock_hooks.c` 和 `kern.ld`，在 `os-interface.c` 中 include hooks，并链接定制脚本。\
   `kern.ld` 将 blob 的 rodata 放入可写 data，配合 `vgpu_unlock_hooks-510.patch` 提供接入，GRID/general 不集成这一层。
2. **静态字节补丁**：KVM 和 merged 使用 `patches/blob-580.196.09.diff` 修改 `kernel/nvidia/nv-kernel.o_binary`，使用 `patches/vgpud-580.196.09.diff` 修改用户态守护进程。\
   GRID/general 仅在存在对应 `.07` diff 时应用，不应误用 `.09` diff。\
   `blobpatch()` 读取的格式为首尾 SHA-256，加上 `十六进制文件偏移: 原字节 新字节`，支持 `#` 注释，写入前核对旧字节，首尾摘要分别核对输入和输出。\
   它逐项原地写入，因此失败后应重新生成输出副本。
3. **runtime hooks**：`patches/nv_hooks.c` 复制为 `kernel/unlock/nv_hooks.c`，通过 Kbuild 编译，再由 `setup-vup-hooks.patch` 接入模块初始化和退出。\
   `vup_hooks_init()` 从 `rm_ioctl - RM_IOCTL_OFFSET` 求出 blob 基址，KVM/merged 使用 `.09` 偏移，非 KVM 使用 `.07` 偏移，先注入 hook，再应用运行时字节补丁。

`VUP_HOOK` 偏移相对 `.text`，字节 diff 使用文件偏移，当前两种 RM blob 的 `.text` 文件起点为 `0x40`，所以运行时调用为 `vup_apply_patches(blob - 0x40)`。\
迁移版本时必须一起核对地址、原始指令、寄存器和字段、被覆盖指令的重放、调用栈对齐，以及静态与 runtime 修改的重叠。

裸汇编跳板使用 `<asm/linkage.h>` 的 `ASM_RET` 适配 RETHUNK。\
`VUP_PATCH_DEF` 的 `enabval` 为精确匹配值，不是一般的非零即启用。\
当前参数及门控如下，完整说明见 `doc/options.rst`。

- KVM 的 `vgpukvm` 默认 `1`，大于 `0` 开启 vGPU 模式，`<= 0` 跳过带门控的 hook 和运行时补丁。\
  `nvidia` 与 `nvidia-vgpu-vfio` 各有独立的 `vgpukvm` 参数，需要切换模式时应保持两者一致。
- `patch.sh` 为 merged 的 Kbuild 设置 `VUP_MERGED_DRIVER=1`，普通 KVM 默认 `0`。\
  `vup_merged` 和 `cudahost` 默认均随该宏，前者精确等于 `1` 才应用补丁，后者为正数时才写入 CUDA 标志。
- `vup_swrlwar` 默认 `0`，保留静态补丁的“活动仍设置”行为。\
  仅精确等于 `1` 才启用 workaround，先检查输入数量，活动计数非零时提前成功返回而不执行设置，非活动路径仍保留容量检查。
- `vup_vgpusig`、`vup_kunlock`、`vup_fbcon` 默认 `1`，`vup_sunlock`、`vup_gspvgpu` 默认 `0`，这些运行时补丁均仅精确等于 `1` 时启用。\
  `vup_qmode` 默认 `0`，仅精确等于 `0` 时应用关闭 qmode 的补丁。
- 非 KVM 的 `gridext` 默认 `1`，大于 `0` 才允许 `vup_general` 与 `vupdevid` 生效。\
  `vup_general` 默认 `0`，仅精确等于 `1` 时启用。
- `vupdevid` 初始默认 `0`，初始化时转换为自动使用 `0x1e30`，负值跳过 hook 安装。\
  `klogtrace` 默认 `0`，值为 `1` 时按内置过滤项输出，值大于 `1` 时不应用该过滤，负值跳过 hook 安装。\
  `klogtracefc` 默认 `8`，控制过滤项剩余输出次数，`klogtrace` 不受 `vgpukvm` 或 `gridext` 门控。

这些门控只控制对应源码路径和运行时 hook/补丁，不回滚生成阶段已经写入的静态字节补丁。\
本节记录参数实现与离线验证范围，离线验证涵盖参数和路径核对、补丁应用与模块编译。\
安装、模块加载及显示、CUDA、vGPU 真机验收另行进行。

### XML profile 与设备映射

vGPU 配置按 `vcfg-v16mpp.patch` → `vcfg-v15vcs.patch` → `vcfg-testing.patch` 的顺序扩展。`vcfgclone()`、`vcfgpatch()` 和 `remap_P40_to_V100D()` 基于 XML 文本布局处理设备映射；`vcfg` 子命令可单独克隆配置。

新增卡型号时按 README 的方法选择同架构来源卡，源 device/subsystem ID 从原 XML 按大小写复制。核对原有 profile 保留、profile ID 和完整 PCI 标识唯一、每个 pgpu 引用去重、所有引用均可解析；同时保留已有跨设备映射的设计。

## 文档书写规范

- 中文长段落按句号 `。` 断句，段内下一句需要另起一行时使用 Markdown 显式硬换行，在行末添加反斜杠 `\`。\
  段落末或列表项末由块边界换行，不添加尾部反斜杠，避免渲染为可见字符。\
  普通源码换行会被渲染为段内空白，不能替代显示换行。
- 不同段落之间保留空行，代码块、表格和列表按各自语法排版。
- 中文文案优先使用逗号、句号和分段，尽量避免分号 `;` 或 `；`。
- 修改后检查 Markdown 渲染结果，确认换行、标题链接、列表和代码块符合预期。

## 提交说明规范

commit 采用“标题 + 整体说明 + 逐文件明细”的结构，正文使用中文。

- 标题简洁概括提交主题，与正文之间空一行。
- 正文先概述本次改动的目标、具体行为变化及影响；涉及多个主题时分段说明。
- 整体说明后逐文件列出改动，每个文件使用仓库相对路径单独占一行，下面用列表说明该文件的具体变化。不同文件之间空一行。
- 明细采用“具体功能点 + 新增／修改／修复／删除 + 具体内容”的结构，功能点概括该条改动所属的功能，按实际变更选择类型。同一文件有多项独立改动时分别列出。
- “具体功能点”是格式占位说明，实际提交时填写对应功能名称，用于区分同一提交中的多个相关功能，不逐条重复“改动点”等占位词。
- 说明与实际 diff 对应，写清改变了什么以及改变后的行为，避免仅写“优化”“调整”等笼统描述。

## 工作产物边界

`.gitignore` 忽略 `NVIDIA-Linux-x86_64-*`、Windows 驱动样本和生成的 `nsigpatch`。原始驱动、解包副本、验证目录和安装包可能并存；版本适配修改的入口仍是 `patch.sh`、`patches/` 和固定的子模块内容。生成目录中的修正应回写到对应补丁，再从冻结输入完整重跑。
