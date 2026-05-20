# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库概览

这是一个围绕 `patch.sh` 组织的驱动补丁仓库，目标不是从源码“编译整个项目”，而是把 NVIDIA 官方驱动包解包、合并、打补丁、注入 vGPU unlock 相关钩子，然后产出新的 `*-patched` 目录或可选的 `.run` 安装包。

仓库依赖 `unlock/` 子模块，克隆时必须使用递归方式；缺少该子模块时，`vgpu_unlock_hooks.c`、`kern.ld` 等关键文件不会存在。

当前主流程依赖 `patch.sh` 顶部硬编码的版本名：
- `GNRL="NVIDIA-Linux-x86_64-550.90.07"`
- `VGPU="NVIDIA-Linux-x86_64-550.90.05-vgpu-kvm"`
- `GRID="NVIDIA-Linux-x86_64-550.90.07-grid"`
- `WSYS="NVIDIA-Windows-x86_64-552.55"`

如果要升级驱动版本，不只是改这几个变量，还要同时检查 `patches/` 里的版本化补丁和差分文件是否仍然匹配，例如 `blob-*.diff`、`vgpud-*.diff`、`libnvidia-ml.so.*.diff`、`wsys-*.diff`。

## 常用命令

### 获取仓库

```bash
git clone --recursive <repo-url>
```

如果已经不是递归克隆，补拉子模块：

```bash
git submodule update --init --recursive
```

### 语法检查与快速自检

仓库没有现成的 lint、单元测试或统一测试入口。改动 `patch.sh` 后，最基础的检查是：

```bash
bash -n patch.sh
```

查看脚本支持的目标和参数：

```bash
./patch.sh
```

### 生成补丁驱动

常用目标都由 `patch.sh` 触发：

```bash
# 生成合并驱动：同时保留 vgpu-kvm 能力和宿主机 CUDA / OpenGL 能力
./patch.sh general-merge

# 生成纯 vgpu-kvm 主机驱动
./patch.sh vgpu-kvm

# 生成 Linux VM 用 GRID 驱动
./patch.sh grid

# 生成 Linux VM 用 general 驱动（必要时可从 grid 反推生成）
./patch.sh general

# 生成 Windows VM 相关产物
./patch.sh wsys
```

README.cn.note.md 中维护者常用的参数组合是：

```bash
./patch.sh --spoof-devid --repack --force-nvidia-gpl-I-know-it-is-wrong --enable-nvidia-gpl-for-experimenting --test-dmabuf-export --envy-probes --zstd general-merge
```

如果只需要纯 vgpu-kvm 变体，把最后的目标改成 `vgpu-kvm`。

### 重新打包 `.run`

```bash
./patch.sh --repack general-merge
```

如需 `zstd` 压缩：

```bash
./patch.sh --repack --zstd general-merge
```

### 只验证局部功能

仓库没有“单个测试用例”概念；如果只想验证小范围逻辑，最接近单项验证的入口是这两个子命令：

```bash
# 只验证 vgpuConfig.xml 克隆逻辑
./patch.sh vcfg <xml文件> <源deviceId> <源subsystemId> <目标deviceId> <目标subsystemId>

# 只验证 P40 -> V100D profile remap 逻辑
./patch.sh remap-p2v <xml文件>
```

### 安装生成结果

进入生成的 `*-patched` 目录后安装：

```bash
./nvidia-installer --dkms -m kernel
```

`README.cn.note.md` 特别指出，550 系列在部分卡上会默认走 `kernel-open`，因此安装时显式加 `-m kernel` 很重要。

如果补丁和安装都完成但没有出现 mdev，可尝试：

```bash
systemctl restart nvidia-vgpud.service nvidia-vgpu-mgr.service
```

## 高层架构

### 1. `patch.sh` 是唯一的编排入口

几乎所有开发工作最终都会回到 `patch.sh`：
- 识别目标模式（`vgpu-kvm`、`grid`、`general`、`general-merge`、`wsys` 等）
- 解包官方 `.run` 或 Windows 安装器
- 按目标生成 `SOURCE` 和 `TARGET`
- 复制 `unlock/` 子模块里的钩子文件
- 应用 `patches/` 下的源码补丁和二进制差分
- 可选地重新打包为新的 `.run`

如果想改流程行为，优先检查 `patch.sh`，不要先从零散目录里找入口。

### 2. 仓库不是“源码树”，而是“厂商目录 + 补丁资产 + 编排脚本”

重要目录分工如下：

- `patch.sh`：核心编排脚本，决定版本、目标模式、补丁应用顺序、重打包行为。
- `patches/`：仓库真正的改动资产中心，既有 `patch` 文本补丁，也有 `blob-*.diff`、`vgpud-*.diff`、`libnvidia-ml.so.*.diff` 这类二进制差分，还包含辅助源文件如 `nv_hooks.c`、`cvgpu.c`。
- `unlock/`：Git 子模块，来自 `DualCoder/vgpu_unlock`，这里提供 `vgpu_unlock_hooks.c`、`kern.ld` 和原始说明文档。`patch.sh` 会把其中的文件复制进目标驱动树。
- `NVIDIA-Linux-x86_64-*`、`NVIDIA-Windows-x86_64-*`：官方驱动包解包后的工作目录，也是脚本的输入基础。它们不是普通“源码目录”，很多内容是厂商原始文件。
- `tools/`：附加运行时资产，当前主要是 `libvgpu_unlock_rs.so` 和 repack 时使用的 `zstd`。
- `doc/options.rst`：记录内核模块参数含义，尤其是运行时补丁暴露出来的开关。
- `nsigpatch.c`：仅 Windows `wsys` 流程会用到的辅助工具源码。

### 3. 补丁流程分成“解包/合并”、“注入钩子”、“再应用补丁”三层

主线流程大致如下：

1. `extract()` 解包 `.run` 文件到与版本同名的目录。
2. 按目标模式决定 `SOURCE`：
   - 直接使用 `VGPU` / `GRID` / `GNRL`
   - 或创建 merged 目录，把 vgpu-kvm 与 GRID/general 叠加成新的源树
3. 把 `SOURCE` 复制成 `TARGET`，实际修改基本都发生在 `*-patched` 目录。
4. 把 `unlock/` 里的 `kern.ld` 和 `vgpu_unlock_hooks.c` 注入到内核源码树，并修改 `nvidia.Kbuild`、`os-interface.c`、`.manifest`。
5. 再叠加 `patches/` 下的文本补丁、二进制差分和可选 helper library。
6. 如果启用了 `--repack`，用目标目录内自带的 `makeself.sh` 重新打包成新的 `.run`。

理解这一层次很重要：很多修改不是直接改某个厂商文件，而是通过 `patch.sh` 在生成期注入进去。

### 4. `applypatchx()` 很关键：很多补丁必须同时作用于 `kernel` 和 `kernel-open`

`applypatch()` 只对主目录应用补丁；`applypatchx()` 会先对主目录打补丁，再检查并把同一份补丁应用到 `kernel-open/`。如果你修改的是内核相关补丁，必须判断它是否也应该同步到 `kernel-open`，否则脚本会在这类双路径场景里产生不一致。

### 5. merged 驱动不是简单拼接，而是有选择地覆盖与回填

`general-merge` / `grid-merge` / `vgpu-kvm-merge` 这些模式会：
- 先复制一个基础树
- 再覆盖另一个驱动树的内容
- 然后回填特定文件（例如 `.manifest`、`conftest.sh`、`nvidia-sources.Kbuild`、firmware、`nv-kernel.o_binary` 等）
- 最后再打额外的兼容与功能补丁

因此如果某个 merged 结果异常，先看 `patch.sh` 中 `DO_MRGD` 分支，不要只盯着某个最终文件。

### 6. 这个仓库大量依赖 `.manifest` 与 Kbuild 注入

除了常规 `patch -p1` 外，`patch.sh` 还会直接修改：
- `.manifest`
- `kernel/nvidia/nvidia.Kbuild`
- `kernel-open/nvidia/nvidia.Kbuild`
- `kernel/nvidia/nvidia-sources.Kbuild`
- `kernel-open/nvidia/nvidia-sources.Kbuild`
- `kernel/nvidia/os-interface.c`
- `kernel/conftest.sh`
- `kernel-open/conftest.sh`

如果你发现某个文件“明明在源码里存在但打包后没带上”，优先检查 `.manifest` 是否同步更新。

## 开发时的仓库约束

### 不要直接在 `*-patched` 目录里做长期修改

`patch.sh` 在生成目标目录时会执行：
- 重新创建 `SOURCE`
- `rm -rf ${TARGET}`
- 再从 `SOURCE` 复制出新的 `TARGET`

所以直接手改 `*-patched` 目录很容易在下一次运行脚本后丢失。长期要保留的改动应落在：
- `patch.sh`
- `patches/`
- `unlock/` 子模块内容
- 或上游厂商目录中那些会被复制进 `SOURCE` 的文件

### 修改输入包或版本时，注意脚本会跳过已存在的解包目录

`extract()` 如果发现目标目录已经存在，会直接跳过解包。这意味着更换 `.run` 文件后，如果老的解包目录还在，脚本可能继续使用旧内容。遇到“明明换了输入包但结果没变”的情况，先检查同名目录是不是还在。

### `general` 目标可能从 `grid` 反推生成

当脚本找不到 `${GNRL}.run` 但找得到 `${GRID}.run` 时，会走 `SWITCH_GRID_TO_GNRL=true` 的兼容路径：先解包 grid，再反向应用 `vgpu-kvm-merge-grid-scripts.patch` 去构造 general 源树。涉及 `general` 行为的问题时，要确认自己走的是原生 general 还是这个 fallback 路径。

### Windows `wsys` 路径是单独的一套流程

`wsys` 不走 Linux `.run` 的逻辑，它会：
- 从 Windows 驱动安装器里抽取 `.sys` / `.dll`
- 去签名
- 打二进制差分
- 重新签名或保留 unsigned 结果

如果你不是在修 Windows 支持，不要把 `wsys` 的工具依赖（`7z`、`msexpand`、`osslsigncode`、`makecert`）和 Linux 主线混在一起处理。

## 与 README / 现有文档一致的重要事实

- 本仓库必须递归克隆，因为 `unlock/` 是子模块。
- 添加新 GPU 支持时，README 建议通过 `vcfgclone` 克隆同架构、官方已支持的条目，而不是随意改 XML。
- `doc/options.rst` 说明了运行时补丁暴露的模块参数；如果修改运行时 hook 或 blob patch，别忘了同步检查这份文档是否仍准确。
- `README.cn.note.md` 提醒 550 系列安装时显式使用 `-m kernel`，并给出了维护者常用的构建参数组合；在复现本仓库现有行为时应优先参考这些参数。
