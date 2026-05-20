# `blob-550.90.05.diff` 深度逆向分析计划

## 1. 目标

产出一份放在 `patches/` 同目录下的中文 Markdown 深度分析文档，系统说明 `patches/blob-550.90.05.diff` 的作用、命中位置、源码映射关系与业务影响。

目标分析链路为：

`diff 字节变化 -> IDA 地址/函数/伪代码/汇编 -> stage_rel 源码相对路径/行号/代码节选 -> patch 前后业务影响`

最终正式文档建议文件名：

- `patches/blob-550.90.05.diff.md`

当前文件是执行前的计划留档：

- `patches/blob-550.90.05.diff.plan.md`

---

## 2. 已明确的需求边界

### 2.1 交付深度

本次交付采用**深度版**。

这意味着最终文档除了基础说明外，还需要尽量覆盖：

- 每个 patch hunk 的地址级映射
- 函数调用链与交叉引用
- 条件分支、状态位、返回值变化
- 关键常量、结构体字段、全局变量的推断
- patch 可能影响的上下游路径
- 所有 hunk 合并后的整体业务影响

### 2.2 文档组织方式

最终正式文档采用**按 hunk 展开**的结构。

也就是说，`blob-550.90.05.diff` 中的每个 hunk 都需要单独成节，分别完成：

- patch 点定位
- IDA 证据提取
- 源码映射
- 行为解释
- 业务影响说明

### 2.3 分析对象

本次默认只围绕以下对象展开：

- `patches/blob-550.90.05.diff`
- 该 diff 所作用的目标二进制文件
- 已在 IDA 中打开的对应二进制，当前已知分析目标为 `nv-kernel.o_binary`
- `NVIDIA-Linux-x86_64-stage_rel` 源码树

默认不把以下文件扩展为主线分析对象：

- `patches/vgpud-550.90.05.diff`
- `patches/libnvidia-ml.so.550.90.diff`
- 其他 patch 文件

如果分析过程中发现某个 hunk 必须借助其他补丁才能解释，会在正式文档中以“关联说明”的方式提及，但不展开成独立主线。

### 2.4 当前已确认的技术事实（基于 IDA 与源码）

以下内容是当前已经通过 IDA MCP 和本地源码检索确认、可直接作为正式分析起点的事实：

#### A. IDA 实例与目标二进制已确认

- 当前目标 IDA 实例端口：`10002`
- 当前打开文件：`NVIDIA-Linux-x86_64-550.90.05-vgpu-kvm/kernel/nvidia/nv-kernel.o_binary`
- `get_metadata` 返回 hash：
  - `0884124c5e623e8c71779b20cf196176ba9dccc19024936b9fc5e23517f52d14`
- 该 hash 与 `patches/blob-550.90.05.diff` 第一行完全一致，说明当前 IDA 打开的正是该 patch 对应的原始 blob。

#### B. `blob-550.90.05.diff` 的结构已确认

该文件不是 unified diff，而是仓库自定义的字节差分格式：

- 第一行：原始文件 sha256
- 中间若干行：`偏移: 原字节 新字节`
- 最后一行：打补丁后的目标 sha256

当前已识别到 5 组 patch 命中位置：

- `0x000D65A4`
- `0x0047CBB7`
- `0x0047CC70`
- `0x0051DDDB`
- `0x02EADC58`

#### C. diff 偏移到 IDA EA 的映射规律已初步确认

当前已对多个 patch 点做了字节级验证，结果表明：

- 对本文件而言，`blob-550.90.05.diff` 中记录的偏移，与 IDA 中 `nv-kernel.o_binary` 的 EA 之间，当前可见规律为：
  - `IDA_EA = DIFF_OFFSET - 0x40`

已验证样例：

- `0x000D65A4 -> 0x000D6564`
- `0x0047CBB7 -> 0x0047CB77`
- `0x0047CC70 -> 0x0047CC30`
- `0x0051DDDB -> 0x0051DD9B`
- `0x02EADC58 -> 0x02EADC18`

后续正式分析时，仍需把这一规律对所有 patch 字节再逐项复核一遍，并在正式文档中写明这是“文件偏移到装载地址的映射关系”，不要只把 diff 偏移误写成 IDA 地址。

#### D. 目标 blob 的段布局已确认

`nv-kernel.o_binary` 当前 IDA 段布局如下：

- `.text`: `0x0 - 0xBEC164`
- `.data`: `0xBEC180 - 0xC9FE10`
- `.rodata`: `0xC9FE20 - 0x307EB80`
- `.bss`: `0x307EB80 - 0x3139368`

这意味着：

- 前 4 个 patch 点落在 `.text`
- 最后 1 个 patch 点落在 `.rodata`

#### E. 当前已确认的函数级命中关系

##### 1) `0x000D65A4 -> IDA 0x000D6564`

- 所在函数：`_nv032674rm`
- 函数范围：`0xD6560 - 0xD6CCA`
- 推断签名：`__int64 __fastcall nv032674rm(__int64 a1)`
- 当前已验证字节：
  - IDA `0xD6564` 处字节为 `41 56 41 55 53 48 83 ED`
  - 与 diff 中 `0x000D65A4` 开始的旧字节前缀一致
- 该函数已知调用者之一：`_nv026411rm`，调用点 `0x51DE43`
- 从 patch 字节形态看，该点很可能是把函数开头替换成“直接返回常量”的 early-return stub，需要在正式分析中重点确认其返回值语义与上游判断关系。

##### 2) `0x0047CBB7 -> IDA 0x0047CB77`

- 所在函数：`_nv046497rm`
- 函数范围：`0x47CB60 - 0x47CC78`
- 调用者：`_nv046455rm`，调用点 `0x477756`
- 当前已验证字节：
  - IDA `0x47CB77` 处字节为 `85 ED 0F 85 B1 00 00 00`
- 当前可见 patch 形态表明：
  - 该点会把一个 `test/jnz` 风格的条件分支改写为“清零寄存器 + NOP 掉跳转”的形式
- `get_basic_blocks` 显示该 patch 点实际落在 basic block：
  - `0x47CB60 - 0x47CB7F`，其中包含实际命中 EA `0x47CB77`
- 同函数内后续还存在一个独立 basic block：
  - `0x47CBB7 - 0x47CBD8`
  - 该块不包含本 patch 命中地址，但可能属于同一逻辑链的后续路径，正式文档中需要与 `0x47CC30` 一并判断是否应作为关联证据引用

##### 3) `0x0047CC70 -> IDA 0x0047CC30`

- 仍位于函数：`_nv046497rm`
- 当前已验证字节：
  - IDA `0x47CC30` 处字节为 `41 BD 40 00 00 00 48 C7`
- 该点与上一 patch 点明显属于同一函数内的同一逻辑链，应在正式文档中合并解释“为何需要两个 patch 点配合改变该控制流程”。

##### 4) `0x0051DDDB -> IDA 0x0051DD9B`

- 所在函数：`_nv026411rm`
- 函数范围：`0x51DD40 - 0x51DF9D`
- 推断签名：`__int64 __fastcall nv026411rm(__int64 a1, __int64 a2, int a3, int a4, int a5, int a6)`
- 当前已验证字节：
  - IDA `0x51DD9B` 处字节为 `85 C0 74 07 C7 45 0C 00`
- 该函数的已确认 callee：
  - `_nv032674rm`，调用点 `0x51DE43`
  - `_nv026628rm`，调用点 `0x51DDE0`
  - `_nv039914rm`
  - `_nv039090rm`
- 该函数会设置一组连续状态字节（反编译可见偏移 `a1 + 18010 ~ 18017` 一带），说明它很可能是“能力探测 / 状态聚合 / 特性结果回填”类函数，后续源码映射应优先在 virtualization / gpu_mgr 相关代码里搜索。

##### 5) `0x02EADC58 -> IDA 0x02EADC18`

- 落在 `.rodata`
- 当前已验证字节：
  - IDA `0x2EADC18` 处字节前缀为：`63 6F 75 6E 74 2E 0A 00`
  - 即字符串尾部 `count.\n\0`
- 这说明最后一个 patch 点不是代码，而是日志字符串本体的一部分。

#### F. 已确认的源码高置信映射起点

当前已经确认一组高置信映射，可作为正式文档优先落地的第一章：

##### `_nv046497rm` 对应 `drivers/resman/src/physical/gpu/fifo/objsched.c`

已确认依据：

- IDA 字符串 `0x2EADBF0`：
  - `"NVRM: Can't change software runlist max count.\n"`
- IDA 字符串 `0x2EADC20`：
  - `"NVRM: Software scheduler timeslice set to %uuS.\n"`
- 对应 xref：
  - `0x47CC36 -> 0x2EADBF0`
  - `0x47CBF1 -> 0x2EADC20`
- `stage_rel` 已确认源码位置：
  - `drivers/resman/src/physical/gpu/fifo/objsched.c:3737-3817`
- 该源码片段中同时出现：
  - `portDbgPrintf("NVRM: Can't change software runlist max count.\n");`
  - `portDbgPrintf("NVRM: Software scheduler timeslice set to %duS.\n", ...);`

因此，`0x0047CBB7`、`0x0047CC70`、`0x02EADC58` 这三个 patch 点应优先作为一组联合分析对象处理。

#### G. 当前正式分析的优先源码范围

结合已有证据，后续源码映射优先级建议如下：

1. `drivers/resman/src/physical/gpu/fifo/objsched.c`
   - 已高置信命中 `_nv046497rm`
   - 直接覆盖 3 个 patch 点

2. `drivers/resman/src/kernel/virtualization/`
   - 重点关注：
     - `vgpu_mgr.c`
     - `kernel_vgpu_mgr.c`
     - `grid/grid_features.c`
   - 主要用于给 `_nv032674rm` / `_nv026411rm` 寻找源码候选

3. `drivers/resman/src/kernel/gpu_mgr/`
   - 重点关注：
     - `gpu_mgr.c`
     - `gpu_group.c`
   - 用于比对设备能力、状态聚合、设备 ID / 子设备 ID / policy 判定相关逻辑

#### H. 当前已确认的业务语义起点

以下内容是已经能够从源码与 IDA 双方提炼出的“业务语义起点”，后续正式报告必须围绕这些语义展开，而不是只停留在字节/指令层。

##### 1) `_nv046497rm` 对应的是 software runlist scheduler 的运行时配置逻辑

结合 `drivers/resman/src/physical/gpu/fifo/objsched.c:3726-3817` 与 `drivers/resman/src/physical/gpu/fifo/objschedmgr.c:935-968`，当前已可确认：

- `schedMgrSwrlSetCountMax_IMPL()` 会遍历有效 runlist，并把 `swrlCountMax` 下发给 `schedSwSwrlSetCountMax()`
- `schedSwSwrlSetCountMax_IMPL()` 负责真正修改 software runlist scheduler 的“最大虚拟 runlist 数量”与相关 timeslice 配置
- 业务上，这不是一个普通调试函数，而是 software scheduling / runlist 配置链路中的实际运行时控制点

因此，正式报告在分析 `0x0047CBB7`、`0x0047CC70`、`0x02EADC58` 时，必须解释：

- `swrlCountMax` 在 scheduler 中控制什么
- 为什么源码里原本禁止“在 SWRL 正在运行时修改 count max”
- 修改该限制对运行中软件调度器意味着什么
- timeslice 计算与 `swrlCountMax` 的关系为何构成业务影响

##### 2) `_nv046497rm` 相关 patch 的业务变化不只是“分支改了”，而是“运行时策略限制被放宽”

根据当前已确认的反编译与源码：

- 原始源码语义：
  - 如果 `pSchedSw->swrlCount != 0`，则打印
    - `NVRM: Can't change software runlist max count.`
  - 并返回 `NV_ERR_INVALID_STATE`
- 这表明原始业务规则是：
  - **software runlist scheduler 一旦已经在运行，就不允许动态修改最大虚拟 runlist 数量**

结合当前 patch 字节形态，正式报告中必须重点确认并说明：

- patch 是否实际把这条“运行中不可修改”的保护逻辑绕开
- patch 前：调用方会因状态不合法而失败
- patch 后：调用方是否能够在 scheduler 已运行时继续修改 `swrlCountMax`
- 这种变化会不会影响：
  - 调度时序
  - runlist 布局
  - timeslice 分配
  - 运行中 VM / vGPU 的公平性或稳定性

##### 3) `.rodata` patch 需要按“可观测性变化”来分析

当前已确认：

- `0x02EADC58 -> 0x02EADC18` 命中的是字符串尾部 `count.\n\0`
- 它位于 `_nv046497rm` 使用的日志字符串区域
- `0x0047CC70 -> 0x0047CC30` 同时修改了原本给 `nv_printf` 准备参数的代码

因此，正式报告不能把这个 `.rodata` patch 当作孤立字符串改动，而必须解释：

- 字符串 patch 是否是在配合代码 patch 改变日志格式
- 原始日志输出表达了什么
- patch 后日志是否改成携带更多上下文（例如当前值 / 目标值）
- 即使该路径后续可能变成弱可达或不可达，也要说明它在“错误诊断/调试可观测性”上的潜在含义

##### 4) `_nv032674rm` 很可能是某种“设备支持 / 特性支持”布尔判定函数

虽然目前还没有源码级 100% 锚定，但从反编译可见：

- 函数内部存在大量按设备 ID / 子设备 ID 的白名单或特判逻辑
- 返回值语义接近布尔量
- 它被 `_nv026411rm` 调用，并直接影响后续状态位聚合路径
- 它的语义特征与 `drivers/resman/src/kernel/virtualization/grid/grid_features.c:isGridLicenseSupported()` 存在较强相似性，需要在正式分析中重点对比

因此，正式报告中必须把 `0x000D65A4` 这组 patch 作为“业务开关”类 patch 来审视，重点回答：

- patch 是否把原本依赖设备白名单/平台状态的支持判定强行改成恒成立
- 若恒成立，被放开的到底是：
  - vGPU software licensing 支持判定
  - 某类 GRID feature 支持判定
  - 或更底层的 capability gate
- patch 前后，哪些本来不应通过的 GPU / SKU / 配置可能被视为“支持”

##### 5) `_nv026411rm` 应按“能力探测结果聚合”来写报告

当前已确认：

- `_nv026411rm` 会调用 `_nv032674rm`
- 还会调用 `_nv026628rm` 以及若干函数指针回调
- 最终会写入一组连续状态字节（`a1 + 18010 ~ 18017` 一带）
- 这些字节显然不是临时变量，而更像是被后续流程消费的 capability / feature state 缓存

因此，正式报告中不能只写“它设置了一些字节”，而必须尽量解释：

- 这些状态字节分别代表什么业务状态
- `_nv032674rm` 的返回值如何改变这些状态的最终组合
- patch 前后，这条聚合链会让后续上层逻辑认为 GPU 具备了哪些能力 / 许可 / 支持状态
- 哪些调用方或控制命令可能直接消费这些状态

##### 6) 报告必须显式区分“已确认业务事实”和“待验证业务推断”

本次 patch 业务分析里，至少有两类内容：

- **已确认业务事实**
  - 例如 `_nv046497rm` 命中 `objsched.c` 的 runlist/timeslice 控制逻辑
  - 例如 `_nv046497rm` 原始源码明确禁止运行中修改 `swrlCountMax`
- **待验证业务推断**
  - 例如 `_nv032674rm` 是否可一一对应到 `isGridLicenseSupported()`
  - 例如 `_nv026411rm` 写入的状态字节分别对应哪个外部能力位

正式报告必须把这两者明确分开，避免把尚未完成源码对位的业务结论写成既定事实。

#### I. 对正式分析步骤的直接影响

基于当前发现，后续执行不应再从“盲搜全仓库”开始，而应按以下顺序推进：

1. 先把 5 个 patch 点全部换算为 `IDA_EA = DIFF_OFFSET - 0x40`
2. 先完成 `_nv046497rm` 对 `objsched.c` 的完整地址级映射
3. 再把 `_nv046497rm` 的报告写成“运行中修改 SWRL 配置限制被放宽”的业务故事线
4. 再围绕 `_nv026411rm -> _nv032674rm` 调用链寻找 virtualization / gpu_mgr / grid_features 源码落点
5. 把 `_nv032674rm` 的设备白名单语义与 `isGridLicenseSupported()` / 相关 capability gate 做对照
6. 最后处理 `.rodata` 字符串 patch 与代码 patch 的联动关系

---

## 3. 正式文档的结构设计

## 3.1 文档开头：总体说明

正式文档开头需要包含以下内容：

1. `blob-550.90.05.diff` 的用途概览
2. 它 patch 的目标二进制是什么
3. IDA 中该二进制对应的模块信息
4. patch hunk 总数
5. 分析证据来源
   - diff 文件
   - IDA 反编译
   - IDA 汇编
   - xref / caller / callee
   - `NVIDIA-Linux-x86_64-stage_rel` 源码
6. 本文采用的映射方法与置信度标准

## 3.2 文档主体：按 hunk 逐项分析

每个 hunk 都要使用统一的小节模板。

建议模板如下：

### Hunk N：标题

#### A. 基本信息
- diff 中的偏移
- patch 前字节
- patch 后字节
- 文件偏移与虚拟地址换算结果

#### B. IDA 定位
- 所在函数名
- 函数起始地址
- 命中 basic block
- 附近关键汇编
- patch 前后关键指令变化

#### C. 伪代码说明
- 与 patch 命中逻辑直接相关的伪代码节选
- 条件判断/返回值/状态位变化的解释

#### D. 调用链与引用关系
- callers
- callees
- 直接相关 xrefs
- 上下游依赖关系

#### E. 源码映射
- `NVIDIA-Linux-x86_64-stage_rel` 相对路径
- 行号
- 代码节选
- 映射依据
- 映射置信度

#### F. patch 作用
- 该 hunk 实际改变了什么逻辑
- 绕过了什么检查或放宽了什么限制
- 影响了什么分支/状态/能力判定

#### G. patch 前后业务影响
- patch 前表现
- patch 后表现
- 受影响的业务路径
- 可能副作用

## 3.3 文档结尾：整体总结

在所有 hunk 分析结束后，再写一个总体总结章节，至少回答：

1. 所有 hunk 合起来的整体目标是什么
2. 它们分别属于哪些逻辑类别，例如：
   - 能力判定修正
   - 校验/签名/限制绕过
   - 初始化流程修正
   - 错误路径屏蔽或改写
   - 兼容性分支修正
3. 对 vGPU 驱动行为的整体影响
4. 可能存在的风险点或副作用

---

## 4. 计划中的实际分析步骤

## 步骤 1：解析 `blob-550.90.05.diff`

目标：先把 patch 文件本身拆开，建立 hunk 清单。

需要完成：

- 识别 diff 文件格式
- 统计 hunk 数量
- 提取每个 hunk 的偏移、旧字节、新字节
- 记录每个 hunk 的长度与连续性
- 初步判断哪些 hunk 可能落在同一函数或相邻 basic block

输出物：

- hunk 索引表
- 每个 hunk 的基础字段清单

## 步骤 2：在 IDA 中定位 patch 命中地址

目标：把每个 hunk 从 diff 偏移映射到 IDA 里的具体地址。

需要完成：

- 确认当前 IDA 实例与目标二进制一致，优先核对当前打开对象是否为 `nv-kernel.o_binary`
- 获取 image base、segments、模块元数据
- 记录 `nv-kernel.o_binary` 在 IDA 中的装载信息，作为后续地址映射基准
- 将每个 hunk 对应到 IDA 地址
- 识别命中位置是否位于函数内
- 若位置尚未成函数，按需补做代码识别或函数边界确认

输出物：

- hunk -> IDA 地址映射表
- 所属函数初步清单

## 步骤 3：提取每个 hunk 的 IDA 证据

目标：为每个 hunk 采集足够的逆向证据。

需要完成：

- 提取目标地址附近的汇编
- 获取函数反编译伪代码
- 获取 basic block 信息
- 抓取 callers / callees / xrefs
- 查找相关常量、字符串、全局变量、结构体线索

输出物：

- 每个 hunk 的汇编节选
- 伪代码节选
- 调用链信息

## 步骤 4：在 `stage_rel` 中建立源码映射

目标：从 IDA 逻辑反推或比对到源代码位置。

需要完成：

- 在 `NVIDIA-Linux-x86_64-stage_rel` 中搜索函数名、常量、日志、条件结构和关键调用
- 根据控制流、条件分支、局部变量关系进行比对
- 找到最可能的源文件相对路径与行号
- 提取必要源码节选
- 对每个映射打上置信度标签

置信度建议分级：

- **高置信映射**：控制流、常量、调用关系和语义都能稳定对上
- **中置信映射**：核心语义能对上，但局部细节或命名存在偏差
- **推测映射**：只能根据局部模式和业务语义推断，缺少足够直接证据

输出物：

- hunk -> 源码相对路径/行号/节选/置信度表

## 步骤 5：解释 patch 前后逻辑变化

目标：把“字节改动”翻译成“业务逻辑变化”。

需要完成：

- 判断是否改变了条件跳转方向
- 判断是否强制让某个条件恒真/恒假
- 判断是否修改了返回值或错误码
- 判断是否影响 capability / policy / mode 判定
- 判断是否影响初始化、签名校验、兼容性判断或错误处理路径

输出物：

- 每个 hunk 的 patch 前后行为对比说明

## 步骤 6：整理整体影响与关联关系

目标：把单点分析整合为完整故事线。

需要完成：

- 识别多个 hunk 是否共同服务于同一业务目标
- 标出 hunk 之间的上下游关系
- 说明 patch 的整体作用是否是“解锁能力”“绕过限制”“修复初始化”“屏蔽错误”或其组合
- 评估潜在副作用

输出物：

- 总体结论章节

## 步骤 7：生成正式 Markdown 文档

目标：把全部证据与解释写成可复查的最终文档。

需要完成：

- 按 hunk 组织章节
- 统一源码引用格式、IDA 地址格式、汇编格式
- 对所有推断结论写清依据
- 标出映射置信度
- 补上总览与总结章节

输出物：

- `patches/blob-550.90.05.diff.md`

---

## 5. 证据采集与展示规则

为了保证正式文档可复查，计划采用以下证据规则：

### 5.1 源码引用格式

统一写成：

- `relative/path/to/file.c:123-145`

并附必要代码节选。

### 5.2 IDA 引用格式

每处关键证据尽量包含：

- 函数名
- 函数起始地址
- hunk 命中地址
- 关键 basic block 地址

### 5.3 汇编引用格式

汇编节选必须带地址，例如：

```asm
0xXXXXXXXX  test ...
0xXXXXXXXX  jnz  ...
0xXXXXXXXX  mov  ...
```

### 5.4 伪代码引用规则

只节选与 patch 直接相关的局部逻辑，不默认贴整函数；除非该函数本身很短，或者整函数上下文对理解 patch 必不可少。

### 5.5 结论表达规则

每个分析结论都要尽量区分：

- 直接证据
- 关联证据
- 推断结论

不能把“推测”写成“事实”。

---

## 6. 重点检查项

正式分析时，应默认检查以下问题，避免遗漏：

1. patch 是否改动条件跳转方向
2. patch 是否强制某个分支恒成立或恒不成立
3. patch 是否修改返回值或错误码
4. patch 是否改写 capability / mode / policy 判定
5. patch 是否绕过某种校验、兼容性限制或签名检查
6. patch 是否影响结构体字段、全局变量或状态位
7. patch 是否只影响单函数，还是影响整条调用链
8. patch 是否可能引入副作用或兼容性问题

---

## 7. 风险与不确定性处理策略

在正式分析过程中，可能出现以下情况：

### 7.1 二进制与源码并非一一直接对应

如果 `stage_rel` 中的源码与当前 blob 版本存在命名差异、内联差异、宏展开差异或编译器优化差异，则不能只靠函数形状简单断定对应关系。

处理方式：

- 结合控制流、常量、日志、调用关系和业务语义综合判断
- 对每个源码映射打置信度标签
- 对不确定处明确写出理由

### 7.2 某些 hunk 可能落在缺少符号信息的区域

处理方式：

- 优先使用邻近函数、basic block、xref 与常量线索辅助定位
- 必要时以“地址级 patch 点”先行分析，再补源码推断

### 7.3 多个 hunk 可能共同构成一个完整逻辑改动

处理方式：

- 虽然正式文档按 hunk 展开，但在每个 hunk 小节中要明确标出“与其他 hunk 的关系”
- 在最终总结中再把这些 hunk 串成完整业务链路

---

## 8. 交付标准

当以下条件都满足时，认为正式文档达到可交付状态：

1. `blob-550.90.05.diff` 的所有 hunk 都已被编号并分析
2. 每个 hunk 都有 IDA 地址定位结果
3. 每个 hunk 都尽量给出函数级和源码级映射
4. 每个 hunk 都解释了 patch 前后行为变化
5. 文档中所有源码引用都附相对路径和行号
6. 文档中所有关键汇编都附地址
7. 对不确定映射明确标注了置信度
8. 最终有整体作用总结与风险分析

---

## 9. 下一步执行入口

按本计划执行正式分析时，建议直接从以下顺序开始：

1. 解析 `patches/blob-550.90.05.diff`
2. 连接并确认当前 IDA 实例
3. 确认目标二进制模块信息
4. 逐个 hunk 建立地址映射
5. 逐个 hunk 提取 IDA 证据
6. 逐个 hunk 对回 `NVIDIA-Linux-x86_64-stage_rel`
7. 汇总业务影响并撰写正式文档
