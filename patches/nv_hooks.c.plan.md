# `nv_hooks.c` 深度逆向分析计划

## 1. 目标

产出一份放在 `patches/` 同目录下的中文 Markdown 深度分析文档，系统说明 `patches/nv_hooks.c` 的作用、注入机制、目标二进制落点、源码映射关系、业务影响，以及它对 `nv-kernel.o_binary` 的**等效 patch**到底是什么样子。

与 `blob-550.90.05.diff` 不同，`nv_hooks.c` 不是一个静态的“字节差分清单”，而是一个**运行时补丁注入器**。因此最终分析链路不能只写成：

`old byte -> new byte`

而应拆成两条主线：

1. **直接补丁主线**
   - `vup_patch_item old/new -> IDA 地址 -> 源码映射 -> patch 前后业务影响`
2. **运行时插桩主线**
   - `hook offset + 原始字节模板 -> 运行时重写规则 (NOP + CALL) -> vup_hook_*_naked / vup_hook_* 语义 -> 被覆盖指令如何回放 -> 业务影响`

最终正式文档建议文件名：

- `patches/nv_hooks.c.md`

当前文件是执行前的计划留档：

- `patches/nv_hooks.c.plan.md`

---

## 2. 已明确的需求边界

### 2.1 交付深度

本次交付默认采用**深度版**。

这意味着最终文档除了基础说明外，还需要尽量覆盖：

- `nv_hooks.c` 在整个补丁流程中的角色
- `vup_hooks[]` 与 `vup_patches[]` 的结构化拆解
- `RM_IOCTL_OFFSET`、`blob` 基址、`blob - 0x40` 这类关键地址换算关系
- 每个 hook / patch 项的 IDA 落点
- 被覆盖原始指令、替换后指令模板、回放逻辑
- 源码相对路径、行号、代码节选
- patch 前后业务影响
- 与现有 `blob-*.diff` 的重叠或互补关系
- 后续版本如何快速定位与复用

### 2.2 文档组织方式

最终正式文档不建议完全照搬 `blob-*.diff` 的“按 hunk 展开”，因为 `nv_hooks.c` 的内部结构天然分成两类：

1. `vup_hooks[]`：运行时 hook 注入项
2. `vup_patches[]`：直接字节 patch 项

因此最终正式文档建议采用**按 patch family / hook family 展开**的结构：

- 先讲 `nv_hooks.c` 的整体机制
- 再按 `NV_VGPU_KVM_BUILD` 主线展开：
  - 先 `vup_hooks[]`
  - 再 `vup_patches[]`
- 再补 `NV_GRID_BUILD` 分支的 `general` 组与差异说明
- 最后汇总成整体业务影响与复用指南

如果后续发现某个 patch 组内部本身需要再细分为多个小 hunk，可在对应小节内部继续拆分。

### 2.3 分析对象

本次默认围绕以下对象展开：

- `patches/nv_hooks.c`
- 该文件所作用的目标 blob，当前高概率仍是 `nv-kernel.o_binary`
- `NVIDIA-Linux-x86_64-stage_rel` 源码树
- `patch.sh` 中对 `nv_hooks.c` 的集成方式

默认不把以下对象扩展成主线：

- `patches/vgpud-*.diff`
- `patches/libnvidia-ml.so.*.diff`
- `patches/cvgpu.c`
- 其他非 `nv-kernel.o_binary` 目标

如果分析过程中发现某个 hook/patch 项必须借助其他补丁才能解释，会在正式文档中以“关联说明”的方式提及，但不展开为独立主线。

### 2.4 已确认的用户决策：等效 patch 采用“双层表达”

你已经明确选择：`双层（推荐）`。

因此最终计划与正式文档都必须把“等效 patch”写成两层：

#### A. 业务 / 控制流层等效 patch

说明某个 hook / patch 组在业务语义上等效改了什么，例如：

- 放宽某个能力判定
- 改写某个设备 ID 来源
- 运行时插入 trace 逻辑
- 动态绕过某个检查
- 修改某个默认模式或行为开关

#### B. 字节 / 插桩层等效 patch

说明它在机器码层面实际等效成什么，例如：

- 对 `vup_patch_item`：可直接写成类似 `blob-*.diff` 的 offset/old/new 形式
- 对 `vup_hook`：写成“校验原始字节模板 -> 把末尾 5 字节改成 `call rel32` -> 把前导字节 NOP 掉 -> 在 `vup_hook_*_naked` 内回放被覆盖指令”的运行时 patch 模板

也就是说，`nv_hooks.c` 的“等效 patch”不只是一份静态 diff，而是：

- 一部分是**静态可表达的直接 patch**
- 另一部分是**动态注入型 patch recipe**

### 2.5 已确认的用户决策：分析顺序采用“先 KVM 后 GRID”

你已经明确选择：`先 KVM 后 GRID`。

因此这份计划与后续正式文档都应按以下优先级组织：

1. **第一优先级：`NV_VGPU_KVM_BUILD` 主线**
   - 作为正文主线展开
   - 优先覆盖：
     - `cudahost`
     - `vupdevid`
     - `klogtrace`
     - `vgpusig`
     - `kunlock`
     - `qmode`
     - `merged`
     - `swrlwar`
     - `fbcon`
     - `sunlock`
     - `gspvgpu`
2. **第二优先级：`NV_GRID_BUILD` 补完线**
   - 作为第二阶段或正文后半段展开
   - 重点覆盖：
     - `general`
   - 主要回答：
     - 它与 KVM 主线共用哪些 patch 思路
     - 哪些 offset / 业务锚点可以直接复用
     - 哪些行为是 GRID/general 特有的

这意味着：

- 计划中的第一轮 IDA 定位、源码映射、业务解释，应优先围绕 `NV_VGPU_KVM_BUILD` 分支进行
- `NV_GRID_BUILD` 不忽略，但默认作为对照线、补完线或附录线来组织
- 如果后续正式分析过程中发现某些关键落点两边完全一致，可以在正文里合并解释；否则应保持“先 KVM、后 GRID”的叙事顺序

### 2.6 当前已确认的技术事实（基于代码阅读与首轮 IDA 采样）

以下内容是当前已经从 `nv_hooks.c`、`patch.sh`、IDA 与 `stage_rel` 源码中确认、可作为正式分析起点的事实。

### 2.7 已提升为硬性要求：每一处被 hook 或 patch 的目标二进制函数，都必须落到 `stage_rel` 的源码文件与源码函数

你这次新增的要求，需要在计划里明确提升成正式交付约束：

- 不仅要定位每个 hook / patch item 命中的 **IDA 函数**
- 还必须继续把这个 **IDA 函数** 对应到 `NVIDIA-Linux-x86_64-stage_rel` 里的：
  - 源码相对路径
  - 源码函数名
  - 必要时的行号范围

这条要求适用于两类对象：

1. **被 hook 的函数**
   - 例如 `vupdevid`、`klogtrace`、`cudahost` 命中的二进制函数
2. **被 patch 的函数**
   - 例如 `kunlock`、`swrlwar`、`merged`、`general` 等各组命中的二进制函数

因此后续正式分析时，不能只写：

- `offset -> _nvXXXXXrm`
- `offset -> 某个源码逻辑簇`

而必须优先收敛成：

- `offset -> IDA 函数 -> stage_rel/xxx.c:源码函数`

只有在公开源码里确实无法精确一一对应时，才允许退一步写成：

- `offset -> IDA 函数 -> 最窄源码函数簇 / 最窄源码调用链`

并且要在文档里明确说明：

- 为什么无法继续收窄到单函数
- 当前最接近的源码函数候选是谁
- 哪些证据支持这一收敛

### 2.8 当前已收敛的源码映射状态（第一版）

下面这张表记录的是：截至本轮 IDA + `stage_rel` 联合阅读后，`nv_hooks.c` 里各 hook / patch 目标函数已经收敛到什么程度。

#### A. 已经可以高置信写成“二进制函数 -> 源码函数”的项

1. `swrlwar` 命中的 `_nv046497rm`
   - 二进制函数：`_nv046497rm @ 0x47CB60`
   - 高置信源码函数：
     - `drivers/resman/src/physical/gpu/fifo/objsched.c:schedSwSwrlSetCountMax_IMPL`
   - 主要证据：
     - 直接命中 `Can't change software runlist max count` 与 `Software scheduler timeslice set to` 两条稳定日志
     - 控制流与 `swrlCount` / `swrlCountMax` / timeslice 写回完全对齐

2. `kunlock` / `general` 共享落点 `_nv032674rm`
   - 二进制函数：`_nv032674rm @ 0x0D6560`
   - 高置信源码函数：
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`
   - 主要证据：
     - 明确读取 `PCIDeviceID` / `PCISubDeviceID`
     - 大量 device/subdevice 白名单分支
     - 返回布尔支持态
     - 被 GRID / licensing 能力链反复消费

3. `general` 中的 `_nv045327rm`
   - 二进制函数：`_nv045327rm @ 0xAF2B60`
   - 高置信源码函数：
     - `drivers/resman/arch/nvalloc/unix/src/os-hypervisor.c:rm_is_vgpu_supported_device`
   - 主要证据：
     - 直接出现 `NVRM: failed to map vGPU register!`
     - 逻辑中直接检查 `os_is_grid_supported()`
     - 调用者就是 `rm_is_supported_device`

4. `fbcon` 命中的初始化主线函数
   - 二进制函数：`_nv000720rm @ 0xAE9D20`
   - 高置信源码函数：
     - `drivers/resman/arch/nvalloc/unix/src/osinit.c:RmInitAdapter`
     - 同函数内部直接覆盖 `RmSetupRegisters` 相关日志与错误路径
   - 主要证据：
     - 二进制中完整出现：
       - `RmSetupRegisters`
       - `Failed to map regs registers!!`
       - `Failed to map hdacodec registers!!`
       - `RmInitAdapter succeeded!`
       - `RmInitAdapter failed!`
     - 与 `osinit.c:1680-1740`、`2168-2699` 的初始化流程高度一致

#### B. 已经可以高置信写成“二进制函数 -> 源码函数簇 / 源码调用链”的项

1. `kunlock` 的核心聚合函数 `_nv026411rm`
   - 二进制函数：`_nv026411rm @ 0x51DD40`
   - 当前最窄源码函数簇：
     - `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1206-1283`
     - `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1772-1804`（`gpumgrStateLoadGpu`）
     - `drivers/resman/src/kernel/gpu_mgr/gpu_mgr.c:1342-1372`（`gpumgrDetachGpu`）
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported`
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:gpuEnableGridFeature`
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL`
   - 当前结论：
     - 它已经明确落在一条“`isGridLicenseSupported()` -> capability 聚合 -> GRID feature state 消费”的调用链上
     - 但当前还不建议把它误写成 `stage_rel` 中某一个单独公开函数的逐行直译

2. `merged` 的 displayless 判定支线
   - 二进制函数：
     - `_nv032676rm @ 0x0B3A00`
     - `_nv026510rm @ 0x4FA010`
   - 当前最窄源码函数簇：
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:disableAllGridLicensedFeatures`
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c:gpuEnableGridFeature`
     - `drivers/resman/src/physical/gpu/gpu_branding.c:gpuDetectVgxBranding_IMPL`
     - `drivers/resman/src/physical/gpu/gpu_branding.c:deviceCtrlCmdGpuGetBrandCaps_IMPL`
     - 以及 `sdk/nvidia/inc/ctrl/ctrla080.h` / `ctrla083.h` 所对应的 GRID displayless 控制语义
   - 当前结论：
     - 这两处二进制函数已经明确挂在 `GRID / displayless / branding` 这条业务线上
     - 其中 `_nv026510rm` 对 `RmForceGridDisplayless` 的直接检查，使它与 displayless 判定 helper 的关系非常强

3. `vgpusig` 的 vGPU host/device 配置链
   - 二进制函数：
     - `_nv050770rm @ 0x0BBA40`
     - 调用者 `_nv049279rm @ 0x5278F0`
   - 当前最窄源码函数簇：
     - `drivers/resman/src/kernel/virtualization/vgpu_mgr.c`
     - `drivers/resman/src/kernel/virtualization/kernel_vgpu_mgr.c`
     - 并与 `grid_features.c:subdeviceCtrlCmdGpuGetLicensableFeatures_IMPL` 中的 `licenseEdition / licensedProductName / signature` 语义相邻
   - 当前结论：
     - 这条链已经明确属于 `host vGPU device / type / config` 处理路径
     - 但还需要继续把它收窄到 `vgpu_mgr.c` 或 `kernel_vgpu_mgr.c` 的具体单函数

4. `general` 中的 `_nv028908rm`
   - 二进制函数：`_nv028908rm @ 0x8DC670`
   - 当前最窄源码函数簇：
     - `drivers/resman/src/kernel/virtualization/grid/grid_features.c` 中与 `displayless`、`licensed max resolution`、`licensed num heads` 相关的控制路径
   - 当前结论：
     - 其字段语义已经明显靠近 GRID displayless capability / resolution / head-count 控制
     - 但还需要继续收窄到单个控制函数

#### C. 当前已经定位到二进制函数，但还要继续收窄到源码函数的项

以下函数已经有稳定 IDA 落点，但在正式报告前，还要继续把它们压到更窄的 `stage_rel` 源码函数：

1. `vupdevid`
   - 二进制函数：`_nv026445rm @ 0x51E330`
   - 当前线索：
     - 被 `_nv026708rm` 调用
     - 明确使用 device/subdevice 信息与静态表
     - 与 `vgpu_mgr.c` / `kernel_vgpu_mgr.c` 中的 pGPU identity / migration encoding 语义相邻

2. `klogtrace`
   - 二进制函数：`_nv039916rm @ 0x16170`
   - 当前线索：
     - 明显属于一条 packet / trace / decode / dispatch 风格链
     - 但暂时还没有在 `stage_rel` 中找到足够稳定的单函数源码锚点

3. `cudahost`
   - 二进制函数：`_nv036968rm @ 0x416AF0`
   - 当前线索：
     - 属于一条对象状态修改后再继续主逻辑的 helper 链
     - 会与多个函数指针回调、错误打印和状态字节更新联动
     - 但当前还需要继续收窄到具体源码函数

4. `qmode`
   - 二进制函数：`_nv026437rm @ 0x523C50`
   - 当前线索：
     - 明显是一条大量读取 registry key 并回填 `OBJGPU` 字段的初始化链
     - 但 `stage_rel` 里公开源码暂未直接暴露这些同名 key 的文本形式，仍需继续收窄

5. `sunlock`
   - 二进制函数：
     - `_nv019510rm @ 0x4F6C30`
     - `_nv030355rm @ 0x0BEC40`
     - `_nv030331rm @ 0x0BE480`
   - 当前线索：
     - 与 `isGridLicenseSupported()`、displayless / branding / capability state 消费链相邻
     - 但三者各自的公开源码单函数还要继续细分

6. `gspvgpu`
   - 二进制函数：`_nv026896rm @ 0x33E70`
   - 当前线索：
     - 与一条布尔 capability helper 相关
     - 且仓库源码里存在 `gsp.c` 对 displayless chip 的明确处理语义
     - 但目前还不能直接写成某个已证实的单函数

#### A. `nv_hooks.c` 是运行时 blob hook 组件，而不是单纯辅助源码

- `patch.sh:675-681` 明确把它描述为：
  - `integrating runtime nv blob hooks`
- `patch.sh:677-681` 会把：
  - `patches/nv_hooks.c`
  - 复制到 `${TARGET}/kernel/unlock/nv_hooks.c`
  - 并把它加入 `nvidia-sources.Kbuild` / `nvidia.Kbuild` / `.manifest`

这说明它不是文档、样例或离线工具，而是会真正编进目标驱动里的**运行时补丁模块源码**。

#### B. `nv_hooks.c` 与 `blob-*.diff` 是并行叠加关系，不是谁替代谁

- `patch.sh:683-685` 在集成 `nv_hooks.c` 后，仍会继续执行：
  - `blobpatch ${TARGET}/kernel/nvidia/nv-kernel.o_binary patches/blob-${VER_BLOB}.diff`

这说明：

- `nv_hooks.c` 不是对 `blob-*.diff` 的简单源码翻译
- 它更像一层额外的**运行时补丁 / 运行时插桩框架**
- 分析时必须区分：
  - 哪些行为来自静态 blob diff
  - 哪些行为来自 `nv_hooks.c` 运行时改写
  - 哪些位置两者可能重叠或互补

#### C. `nv_hooks.c` 内部有两套机制

代码层面已经可以明确分成两套：

1. **hook 注入机制**
   - 结构体：`struct vup_hook_info`（`nv_hooks.c:34-40`）
   - 宏：`VUP_HOOK`（`nv_hooks.c:42-43`）
   - 入口：`vup_inject_hooks()`（`nv_hooks.c:326-364`）
   - 配套函数：`vup_hook_*` / `vup_hook_*_naked`

2. **直接 patch 机制**
   - 结构体：`struct vup_patch_item`（`nv_hooks.c:46-50`）
   - 结构体：`struct vup_patch_info`（`nv_hooks.c:52-58`）
   - 宏：`VUP_PATCH_DEF` / `VUP_PATCH`（`nv_hooks.c:60-70`）
   - 入口：`vup_apply_patches()`（`nv_hooks.c:366-404`）

这是本次计划必须围绕的第一层分类轴。

#### D. `nv_hooks.c` 的目标 blob 基址来自 `rm_ioctl`

- `nv_hooks.c:233` / `311`：`#define RM_IOCTL_OFFSET 0xae2680`
- `nv_hooks.c:455-470`：
  - `u8 *blob = (u8 *)rm_ioctl - RM_IOCTL_OFFSET;`

这说明：

- 它不是通过文件 I/O patch 某个离线 blob
- 而是在驱动运行时，利用 `rm_ioctl` 已知偏移反推出目标 blob 映射基址
- 后续所有 `vup_hooks[]` / `vup_patches[]` 的 offset，都是围绕这个运行时 blob 基址来解释的

#### E. `vup_patch_item` 的 offset 很可能采用“blob diff 风格偏移”

- `vup_hooks_init()` 中：`vup_apply_patches(blob - 0x40);`（`nv_hooks.c:469`）

这意味着：

- `vup_patch_item.offset` 不是直接加在 `blob` 上
- 而是加在 `blob - 0x40` 上

这与此前 `blob-550.90.05.diff` 分析里得到的：

- `IDA_EA = DIFF_OFFSET - 0x40`

高度一致。

因此，当前高概率可以推断：

- `vup_patch_item` 里的 offset 使用的是**diff / 文件偏移风格**
- 后续正式分析必须逐项验证这一点，但它已经是当前最值得优先利用的映射起点

#### F. `vup_hook_info` 的 offset 很可能采用“blob 基址相对偏移”

- `vup_hooks_init()` 中：`vup_inject_hooks(blob);`（`nv_hooks.c:466`）
- `vup_inject_hooks()` 中访问方式是：`blob[hi->offset + j]`（`nv_hooks.c:347-358`）

这说明：

- `vup_hook_info.offset` 与 `vup_patch_item.offset` 不是同一种坐标系
- `vup_hook_info.offset` 更像是直接相对 `blob` 的偏移
- 也就是更接近 IDA 中 `.text` 起始基址下的 EA 相对值

后续正式分析必须逐项验证：

- `vup_hooks[]` 的 offset 是否可直接映射到 IDA 地址
- `vup_patches[]` 的 offset 是否要先减 `0x40`

#### G. `vup_inject_hooks()` 的等效 patch 规则已经可以明确写出来

根据 `nv_hooks.c:333-360`，每个 hook 的运行时改写规则已经很清楚：

1. 先读取 `hi->offset` 与 `pbytes[]`
2. 校验目标位置的原始字节是否与 `pbytes[]` 完全一致
3. 若不一致，记录失败并跳过
4. 若一致：
   - 设 `n = pbytes` 的字节数
   - 从 `offset + n - 5` 开始写入 `E8 <rel32>`
   - 把前面的 `n - 5` 个字节填成 `NOP`

因此，hook 项的字节层等效 patch 模板不是“old->new 表”，而是：

- 原始模板：`pbytes[0..n-1]`
- 改写结果：`90 ... 90 E8 <rel32-to-vup_hook_*_naked>`

这里的 `<rel32>` 指向的是内核模块中 `vup_hook_*_naked` 的运行时地址，因此它天然不是一个静态仓库内就能完全固定下来的 blob diff。

#### H. `vup_hook_*_naked` 会手工回放被覆盖语义

当前已能从三个 hook 看出统一模式：

- `vup_hook_cudahost_naked()`（`nv_hooks.c:88-112`）
- `vup_hook_vupdevid_naked()`（`nv_hooks.c:127-156`）
- `vup_hook_klogtrace_naked()`（`nv_hooks.c:193-216`）

它们的共同点是：

1. 先保存若干寄存器
2. 调用普通 C helper（`vup_hook_*`）
3. 恢复寄存器
4. 手工补回原控制流所需的关键指令
5. `ret`

这意味着正式分析必须回答两个问题：

- 被覆盖的原始指令是什么
- 这些指令是如何在 `*_naked` 中被“回放”或“语义等效替代”的

#### I. 当前已确认的 hook 项

`vup_hooks[]` 目前包含以下 hook；其中三处关键 hook 的 offset 与原始模板字节已经通过当前 IDA 中的 `nv-kernel.o_binary` 完成首轮验证。

##### 在 `NV_VGPU_KVM_BUILD` 下

- `cudahost`
  - offset：`0x00416B9C`
  - 原始模板字节数：8
  - `nv_hooks.c:221`
- `vupdevid`
  - offset：`0x0051E3A7`
  - 原始模板字节数：6
  - `nv_hooks.c:222`
- `klogtrace`
  - offset：`0x00016185`
  - 原始模板字节数：7
  - `nv_hooks.c:223`

##### 在非 `NV_VGPU_KVM_BUILD` 分支下

- `vupdevid`
  - offset：`0x0051E3A7`
  - `nv_hooks.c:225`
- `klogtrace`
  - offset：`0x00016185`
  - `nv_hooks.c:226`

这说明：

- `vupdevid` / `klogtrace` 是两边共有
- `cudahost` 只在 `NV_VGPU_KVM_BUILD` 下存在

#### J. 当前已确认的 patch 组

##### `NV_VGPU_KVM_BUILD` 分支下的 patch 组

- `vgpusig`：1 项（`nv_hooks.c:235-239`）
- `kunlock`：7 项（`nv_hooks.c:241-250`）
- `qmode`：2 项（`nv_hooks.c:252-256`）
- `merged`：2 项（`nv_hooks.c:258-262`）
- `swrlwar`：4 项（`nv_hooks.c:264-270`）
- `fbcon`：1 项（`nv_hooks.c:272-275`）
- `sunlock`：5 项（`nv_hooks.c:277-286`）
- `gspvgpu`：2 项启用 + 1 项注释（`nv_hooks.c:288-293`）

##### `NV_GRID_BUILD` 分支下的 patch 组

- `general`：4 项（`nv_hooks.c:312-321`）

#### K. 当前可见的与 `blob-550.90.05.diff` 的重叠线索

虽然本次分析对象不是 `blob-550.90.05.diff`，但从 offset 上已能看出至少两处强相关：

1. `kunlock` 中：
   - `0x0051DDDE`
   - `0x0051DDE2`
   - 与此前 `blob-550.90.05.diff` 报告中的 `_nv026411rm` 区域紧密相邻

2. `swrlwar` 中：
   - `0x0047CBBB`
   - `0x0047CC89`
   - `0x0047CC8A`
   - `0x0047CC8B`
   - 与此前 `_nv046497rm` / `objsched.c` 的 software runlist 逻辑区高度相邻

这意味着正式分析时，必须把 `nv_hooks.c` 与 `blob-550.90.05.diff` 区分对待，但也应显式说明二者在某些区域可能是：

- 对同一业务链的不同 patch 手段
- 或针对相邻控制流点的互补修改

#### L. `vup_patching_start()` / `vup_patching_done()` 说明它修改的是只读代码页

- `nv_hooks.c:417-453`

当前已可确认：

- patch 前会 `preempt_disable()`
- 清掉 `CR0.WP`
- 必要时暂时关掉 `CR4.CET`
- patch 完后再恢复

这说明 `nv_hooks.c` 的 hook / patch 不是逻辑层 API，而是对已经映射进内核地址空间的代码进行**原地字节级改写**。

这部分在正式文档里必须单独解释，因为它直接决定：

- 这些 patch 为何是运行时生效
- 为什么 hook 的等效 patch 不能简单用离线 diff 表示完

#### M. 双坐标系假设已经获得首轮 IDA 实证支持

目前已经通过当前 IDA 实例中的 `nv-kernel.o_binary` 对两套 offset 坐标系做了抽样验证。

##### 1) `vup_hook_info.offset` 直接映射到 blob EA

当前已确认样例：

- `0x0051E3A7` 处字节为：`4C 89 E0 44 89 FB`
  - 与 `VUP_HOOK(... vupdevid ...)` 中的原始模板完全一致
- `0x00016185` 处字节为：`48 81 ED 40 04 00 00`
  - 与 `VUP_HOOK(... klogtrace ...)` 中的原始模板完全一致
- `0x00416B9C` 处字节为：`41 80 BD 24 08 00 00 00`
  - 与 `VUP_HOOK(... cudahost ...)` 中的原始模板完全一致

这说明对当前版本而言，`vup_hook_info.offset` 可以直接当作 `blob` 基址下的相对地址来解释。

##### 2) `vup_patch_item.offset` 仍符合 `offset - 0x40` 规则

当前已确认样例：

- `0x0051DDDE -> 0x51DD9E`，旧字节：`07`
- `0x0051DDE2 -> 0x51DDA2`，旧字节：`00`
- `0x0047CBBB -> 0x47CB7B`，旧字节：`B1`
- `0x000D65C8 -> 0x0D6588`，旧字节：`75`
- `0x000B3AD9 -> 0x0B3A99`，旧字节：`97`
- `0x00033EE5 -> 0x033EA5`，旧字节首字节：`1B`

这说明 `vup_patch_item` 至少在当前版本上，与 `blob-550.90.05.diff` 采用的是同一套 diff/文件偏移坐标规则。

#### N. 三处 hook 的真实 IDA 落点已经确认，可作为正式分析第一优先级

##### 1) `vupdevid` hook

- offset：`0x0051E3A7`
- 所在函数：`_nv026445rm`
- 函数范围：`0x51E330 - 0x51E540`
- 原始模板长度：6 字节
- 被覆盖原始字节：`4C 89 E0 44 89 FB`
  - 对应两条指令：
    - `mov rax, r12`
    - `mov ebx, r15d`
- 运行时改写窗口：
  - `offset + 0`：`90`
  - `offset + 1`：`E8 <rel32-to-vup_hook_vupdevid_naked>`
- `vup_hook_vupdevid_naked()` 末尾明确回放：
  - `mov %r12, %rax`
  - `mov %r15d, %ebx`

这说明 `vupdevid` 是一个非常“干净”的运行时插桩例子：

- 覆盖窗口短
- 原始模板稳定
- 被覆盖语义在 naked hook 内有直接一一回放

因此它应成为正式文档里“hook 注入机制说明”的第一优先样例。

##### 2) `klogtrace` hook

- offset：`0x00016185`
- 所在函数：`_nv039916rm`
- 函数范围：`0x16170 - 0x16561`
- 原始模板长度：7 字节
- 被覆盖原始字节：`48 81 ED 40 04 00 00`
  - 对应指令：`sub rbp, 0x440`
- 运行时改写窗口：
  - `offset + 0..1`：`90 90`
  - `offset + 2`：`E8 <rel32-to-vup_hook_klogtrace_naked>`
- `vup_hook_klogtrace_naked()` 末尾明确回放：
  - `sub $0x440, %rbp`

这说明 `klogtrace` 也属于“原始语义可直接回放”的 hook。

从 helper 语义看，它更像一条**调试/可观测性插桩链**，而不是功能 unlock 主线。因此正式分析时可把它放在：

- 机制说明中的高价值样例
- 业务主线中的次优先项

##### 3) `cudahost` hook

- offset：`0x00416B9C`
- 所在函数：`_nv036968rm`
- 函数范围：`0x416AF0 - 0x416CDD`
- 原始模板长度：8 字节
- 被覆盖原始字节：`41 80 BD 24 08 00 00 00`
  - 对应指令：`cmpb $0, 0x824(%r13)`
- 运行时改写窗口：
  - `offset + 0..2`：`90 90 90`
  - `offset + 3`：`E8 <rel32-to-vup_hook_cudahost_naked>`
- `vup_hook_cudahost_naked()` 末尾明确回放：
  - `cmpb $0, 0x824(%r13)`
- helper 会先取：
  - `lea 0x50c(%rbx), %rdi`
  - 再调用 `vup_hook_cudahost(u8 *flag)`

这说明 `cudahost` hook 的关键点不是纯日志，而是：

- 在原始比较发生前，先对某个对象字段进行可选回写
- 再继续执行原本的条件比较

因此它很可能是一个“在不完全重写控制流的前提下，先改状态，再让原逻辑自然生效”的 hook 模式样例。

#### O. 已确认的重叠区域可以进一步细化为“相邻业务块”而不是泛泛相邻

##### 1) `swrlwar` 与 software runlist 逻辑区重叠已得到字节级支持

当前已确认：

- `0x0047CBBB -> 0x47CB7B`，旧字节为 `B1`
- `0x0047CC89 -> 0x47CC49`，旧字节为 `44`
- `0x0047CC8A -> 0x47CC4A`，旧字节为 `89`
- `0x0047CC8B -> 0x47CC4B`，旧字节为 `E8`

这意味着 `swrlwar` 不只是“接近 `_nv046497rm`”，而是直接命中了此前 `blob-550.90.05.diff` 已分析过的同一函数 `_nv046497rm`：

- `0x47CB7B` 落在原 `test/jnz` 所在早期块附近
- `0x47CC49-0x47CC4B` 命中原本的 `mov eax, r13d`

从字节替换看：

- `44 89 E8` 被改成 `90 31 C0`
- 等价于把“`mov eax, r13d`”改成“`nop; xor eax, eax`”

这说明 `swrlwar` 极可能是在同一错误/返回路径上，把原本返回某个错误值的逻辑改成返回 `0`，正式分析时必须与先前 `_nv046497rm` 的 runlist/timeslice 结论联动解释。

##### 2) `kunlock` 与 `_nv026411rm` 区域重叠已得到字节级支持

当前已确认：

- `0x0051DDDE -> 0x51DD9E`，旧字节 `07`
- `0x0051DDE2 -> 0x51DDA2`，旧字节 `00`

这与此前 `blob-550.90.05.diff` 对 `_nv026411rm` 的分析区完全重叠，说明 `kunlock` 不是泛泛“靠近 capability 聚合链”，而是直接落在同一能力聚合函数的同一 patch 子区域上。

因此正式分析时，`kunlock` 应优先与：

- `_nv026411rm`
- `_nv032674rm`
- `grid_features.c:isGridLicenseSupported()`
- `gpu_mgr.c` / `grid_features.c` 的 capability state 消费链

放在同一业务故事线中处理。

---

## 3. 正式文档的结构设计

### 3.1 文档开头：总体说明

正式文档开头需要包含以下内容：

1. `nv_hooks.c` 在仓库中的角色
2. 它与 `blob-*.diff` 的关系
3. 它作用的目标 blob 是什么
4. hook 项与 patch 组总数
5. 分析证据来源：
   - `nv_hooks.c`
   - `patch.sh`
   - IDA 反编译 / 汇编 / xref / caller / callee
   - `stage_rel` 源码
6. 本文采用的地址映射规则与置信度标准

### 3.2 文档主体：先分“机制层”，再分“项级层”

建议文档主体采用以下顺序，并贯彻“先 KVM 后 GRID”：

#### 第一部分：机制层说明

- `vup_hooks_init()` 的整体流程
- `blob = rm_ioctl - RM_IOCTL_OFFSET`
- 为什么 `vup_hooks[]` 与 `vup_patches[]` 的 offset 坐标系不同
- `vup_patching_start()` / `vup_patching_done()` 如何打开写保护并恢复

#### 第二部分：`NV_VGPU_KVM_BUILD` 下的 `vup_hooks[]` 逐项分析

建议优先顺序为：

1. `vupdevid`（模板短、回放直接、和 capability/设备 ID 主线强相关）
2. `cudahost`（会先修改状态字段，再继续原控制流）
3. `klogtrace`（更偏调试/可观测性插桩）

建议每个 hook 项统一采用以下模板：

##### Hook N：标题

###### A. 基本信息
- hook 名称
- build 条件
- offset
- 原始模板字节
- 原始模板长度

###### B. 运行时改写规则
- 校验原始字节方式
- `NOP + CALL rel32` 的改写窗口
- call 目标是哪个 `vup_hook_*_naked`
- 被覆盖原始指令如何在 naked hook 内回放

###### C. IDA 定位
- 落点函数
- 命中 basic block
- 周边关键汇编
- 原始控制流与被改写控制流

###### D. C helper / naked hook 语义
- `vup_hook_*` 做什么
- `vup_hook_*_naked` 保存/恢复什么
- 回放了哪些被覆盖语义

###### E. 源码映射
- 该 hook 命中的 **二进制函数名 / 地址**
- 该二进制函数对应的 `stage_rel` **源码文件 + 源码函数名**
- 必要时补行号范围
- 代码节选
- 映射依据
- 置信度
- 若无法收敛到单函数：说明当前最窄源码函数簇 / 调用链，以及不能继续收窄的原因

###### F. 等效 patch（双层）
- 业务层等效 patch
- 字节 / 插桩层等效 patch

###### G. patch 前后业务影响
- patch 前表现
- patch 后表现
- 影响调用链
- 潜在副作用

#### 第三部分：`NV_VGPU_KVM_BUILD` 下的 `vup_patches[]` 逐组分析

建议优先顺序为：

1. `kunlock`
2. `swrlwar`
3. `vgpusig`
4. `merged`
5. `qmode`
6. `sunlock`
7. `gspvgpu`
8. `fbcon`

其中前两组应优先，因为它们已经和既有 `blob-550.90.05.diff` 主分析链发生了可验证的重叠。

建议每个 patch 组统一采用以下模板：

##### Patch Group N：标题

###### A. 基本信息
- patch 组名称
- build 条件
- 默认值 / 启用值 / `ovgpu`
- 项数

###### B. 偏移清单
- 每项的 offset / old / new
- 与 diff 风格 offset 的关系
- 是否与既有 `blob-*.diff` 重叠或邻近

###### C. IDA 定位
- 每项命中函数
- 是否同属一个函数/业务块
- 周边关键汇编

###### D. 源码映射
- 每个 patch item 命中的 **二进制函数名 / 地址**
- 该二进制函数对应的 `stage_rel` **源码文件 + 源码函数名**
- 必要时补行号范围
- 代码节选
- 映射依据
- 置信度
- 若无法收敛到单函数：说明当前最窄源码函数簇 / 调用链，以及不能继续收窄的原因

###### E. 等效 patch（双层）
- 业务层等效 patch
- 字节层等效 patch（接近 `blob-*.diff` 形式）

###### F. patch 前后业务影响
- patch 前表现
- patch 后表现
- 上下游影响
- 可能副作用

### 3.3 文档结尾：整体总结

最终总结章节至少要回答：

1. `nv_hooks.c` 的整体目标是什么
2. 哪些功能是 hook 注入实现的，哪些是直接 patch 实现的
3. 它与 `blob-*.diff` 的边界和关系是什么
4. 对 vGPU 驱动整体行为的影响是什么
5. 是否存在版本耦合风险、原始字节模板失配风险、运行时副作用

### 3.4 后续版本快速定位与 patch 复用指南

正式文档必须单独包含一节“后续版本快速定位与 patch 复用指南”，并且要比 `blob-*.diff` 的复用章节多回答一类问题：

- 对于 hook 注入项，如何在新版本中找到“可被 `NOP + CALL` 重写”的同一业务位置

建议至少包含以下内容：

#### A. 双坐标系定位规则

必须明确写出：

1. `vup_patch_item.offset`
   - 采用 diff / 文件偏移风格
   - 需要结合 `blob - 0x40`
2. `vup_hook_info.offset`
   - 采用 blob 基址相对偏移风格
   - 更接近 IDA EA 相对值

#### B. Hook 项的复用锚点

对每个 hook 至少总结：

- 原始模板字节
- 模板长度 `n`
- `call` 的理论起始位置 `offset + n - 5`
- 被覆盖指令的语义
- naked hook 中回放的关键指令
- 调用前后寄存器约束
- 若有字符串/日志锚点，也要列出

#### C. 直接 patch 项的复用锚点

对每个 patch 组至少总结：

- 关键 offset
- old/new 字节
- 命中函数职责
- 控制流锚点
- 相邻字符串 / 错误码 / capability 判定锚点

#### D. 间接定位法

因为 `nv_hooks.c` 的某些目标点可能本身没有稳定字符串，正式报告必须明确加入“间接定位法”作为复用策略的一部分：

1. 看调用者是否有稳定日志字符串
2. 看被调 helper 是否有稳定字符串或错误码
3. 看谁消费它写回的状态位或返回值
4. 最后再回到目标 block 做控制流确认

#### E. 复用判定标准

至少区分：

- **可直接复用**
- **需要重定位后改写**
- **不建议直接复用**

---

## 4. 计划中的实际分析步骤

### 步骤 1：建立 `nv_hooks.c` 的机制模型

目标：先彻底分清“它在干什么”，而不是一上来就跳到单个 offset。

需要完成：

- 还原 `vup_hooks_init()` 的整体流程
- 区分 hook 注入与直接 patch 两套机制
- 确认 `blob`、`blob - 0x40`、`RM_IOCTL_OFFSET` 的坐标关系
- 记录 build 条件分支：
  - `NV_VGPU_KVM_BUILD`
  - `NV_GRID_BUILD`

输出物：

- 机制总览图
- 坐标系说明

### 步骤 2：枚举全部 hook 项与 patch 组

目标：把 `nv_hooks.c` 内全部可分析对象列成清单。

需要完成：

- 枚举 `vup_hooks[]`
- 枚举 `vup_patches[]`
- 统计每项的 offset / old / new / 原始模板字节 / build 条件
- 标记哪些项与既有 `blob-*.diff` 有重叠或相邻

输出物：

- hook 索引表
- patch 组索引表

### 步骤 3：在 IDA 中定位全部目标点

目标：先把 `NV_VGPU_KVM_BUILD` 主线的 hook offset 和 patch offset 落到当前 `nv-kernel.o_binary` 中，再补 `NV_GRID_BUILD` 分支，并为后续源码映射准备“函数级落点”。

需要完成：

- 确认当前 IDA 实例与目标二进制一致
- 逐项验证：
  - hook offset 是否可直接落到 blob EA
  - patch offset 是否需按 `offset - 0x40` 转换
- 记录命中函数、basic block、周边汇编
- 对每个目标点至少先收集：
  - 二进制函数名
  - 函数范围
  - 调用者 / 被调者线索

输出物：

- hook -> IDA 地址映射表
- patch item -> IDA 地址映射表
- 目标点 -> 二进制函数映射表

### 步骤 4：分析 hook 注入的字节级等效 patch

目标：把 `vup_hooks[]` 从源码描述翻译成可落地的“运行时 patch recipe”。

需要完成：

- 对每个 hook 统计 `pbytes` 长度
- 计算 `NOP + CALL rel32` 的覆盖窗口
- 对当前版本先固定三条已确认模板：
  - `vupdevid`：6 字节窗口 -> `1 字节 NOP + 5 字节 CALL`
  - `klogtrace`：7 字节窗口 -> `2 字节 NOP + 5 字节 CALL`
  - `cudahost`：8 字节窗口 -> `3 字节 NOP + 5 字节 CALL`
- 确认被覆盖原始指令是什么
- 确认 `*_naked` 中如何回放被覆盖语义
- 说明 call 目标为何不能直接静态固化成一个离线 diff

输出物：

- 每个 hook 的字节层等效 patch 模板
- 每个 hook 的业务层等效 patch 说明

### 步骤 5：分析直接 patch 组的静态等效 patch

目标：把 `vup_patches[]` 翻译成接近 `blob-*.diff` 风格的 patch 视图。

需要完成：

- 逐组整理 old/new 字节
- 与 IDA 原始字节对比验证
- 尽量归并出相邻/同函数的项
- 识别业务主题，例如：
  - signature check
  - kernel unlock
  - qmode
  - merged driver 行为
  - swrl workaround
  - fb console 相关
  - GSP/vGPU 相关

输出物：

- 每个 patch 组的静态等效 patch 清单
- patch 组业务主题表

### 步骤 6：在 `stage_rel` 中建立源码映射

目标：从 IDA 逻辑反推或比对到源代码位置，并尽可能把**每一处被 hook 或 patch 命中的二进制函数**落到 `stage_rel` 的**源码文件 + 源码函数**。

需要完成：

- 在 `stage_rel` 中搜索相关字符串、条件、调用链、状态位写回模式
- 先为每个目标点确认：
  - 命中的二进制函数是谁
  - 该二进制函数最可能对应哪个源码文件 / 源码函数
- 对每个 hook / patch 组，汇总其内部所有 target function 的源码归属
- 若公开源码里无法精确到单函数：
  - 收敛到最窄源码函数簇 / 调用链
  - 明确为什么不能继续收窄
- 对映射打上置信度标签

输出物：

- target function -> 源码文件 / 源码函数 / 置信度表
- hook / patch group -> 源码路径 / 行号 / 置信度表

### 步骤 7：解释 patch 前后逻辑变化

目标：把“源码 + 汇编 + 注入机制”翻译成“业务行为改变”。

需要完成：

- 判断 hook 改写的是哪段原控制流
- 判断 direct patch 改变了什么条件 / 返回值 / 状态位
- 区分：
  - 运行时插桩
  - 直接静态字节替换
- 说明它们各自的业务目的与副作用

输出物：

- 每项 patch / hook 的前后行为对比说明

### 步骤 8：生成正式 Markdown 文档

目标：按“先 KVM 后 GRID”的顺序，把全部证据与结论写成可复查的最终文档，并落实“每一处目标函数都要对应到源码函数”的要求。

需要完成：

- 先写机制层，再写项级层
- 对所有等效 patch 都坚持双层表达
- 对每个 hook / patch 组明确列出：
  - 命中的二进制函数
  - 对应的源码文件 / 源码函数
  - 如果只能落到源码函数簇，也要解释原因
- 标清直接证据、关联证据、推断结论
- 加入整体总结与复用指南

输出物：

- `patches/nv_hooks.c.md`

---

## 5. 证据采集与展示规则

### 5.1 源码引用格式

统一写成：

- `relative/path/to/file.c:123-145`

并附必要代码节选。

### 5.2 IDA 引用格式

每处关键证据尽量包含：

- 函数名
- 函数起始地址
- 命中地址
- basic block 地址

### 5.3 汇编引用格式

汇编节选必须带地址，例如：

```asm
0xXXXXXXXX  nop
0xXXXXXXXX  call ...
0xXXXXXXXX  mov ...
```

### 5.4 等效 patch 展示格式

对每个 hook / patch 项，都建议统一给出两段：

#### 业务层等效 patch

用自然语言说明控制流和业务变化。

#### 字节 / 插桩层等效 patch

- 对直接 patch：列出 `offset / old / new`
- 对 hook 注入：列出
  - 原始模板字节
  - 模板长度 `n`
  - 覆盖窗口
  - `NOP*(n-5) + CALL rel32(vup_hook_*_naked)`
  - call 起始偏移 `offset + n - 5`
  - 被覆盖指令回放位置

### 5.5 结论表达规则

每个分析结论都要尽量区分：

- 直接证据
- 关联证据
- 推断结论

不能把“推测”写成“事实”。

---

## 6. 重点检查项

正式分析时，应默认检查以下问题：

1. hook offset 与 patch offset 是否使用同一坐标系
2. hook 覆盖的原始字节是否稳定可匹配
3. naked hook 是否完整回放了被覆盖语义
4. direct patch 是否与 `blob-*.diff` 已知命中点重叠
5. patch 是否改动条件跳转、返回值、状态位或 capability gate
6. patch 是否只改一处，还是会改变整条调用链
7. 运行时注入是否可能引入寄存器/栈平衡风险
8. 版本变化后原始模板字节是否容易失配

---

## 7. 风险与不确定性处理策略

### 7.1 hook 注入项不能被误写成普通静态 diff

处理方式：

- 正式文档中明确区分“静态可表达 patch”与“运行时 patch recipe”
- 不把 `CALL rel32` 的最终地址伪装成仓库静态常量

### 7.2 同一业务链可能同时被 blob diff 与 nv_hooks.c 修改

处理方式：

- 对重叠区域显式标记“静态 patch / 动态 patch / 二者并存”
- 避免把两个来源的行为混写成一个补丁

### 7.3 某些源码映射可能只能落到“逻辑簇”，而不是单函数

处理方式：

- 对映射标置信度
- 必要时用“逻辑簇落点”替代“硬指认单函数”

---

## 8. 交付标准

当以下条件都满足时，认为正式文档达到可交付状态：

1. `vup_hooks[]` 与 `vup_patches[]` 的所有分析对象都已编号并说明
2. 每项都给出 IDA 地址定位结果
3. 每一处被 hook 或 patch 命中的二进制函数，都尽量给出对应的 `stage_rel` 源码文件与源码函数
4. 若无法对应到单函数，明确给出最窄源码函数簇 / 调用链与原因
5. 每项都给出双层等效 patch 说明
6. 每项都解释了 patch 前后行为变化
7. 文档中所有关键汇编都附地址
8. 对不确定映射明确标注了置信度
9. 最终有整体作用总结与复用指南

---

## 9. 下一步执行入口

按本计划执行正式分析时，建议直接从以下顺序开始：

1. 先验证 `nv_hooks.c` 的双坐标系假设
2. 再逐个 hook 定位 IDA 落点并还原插桩模板
3. 再逐个 patch 组验证 old/new 字节与业务落点
4. 再建立 `stage_rel` 源码映射
5. 最后写双层等效 patch 与整体业务影响
