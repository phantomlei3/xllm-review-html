# xLLM MR #543 — Review 意见与修复交接

- 来源：[MR #543](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/543)
- 标题：`perf: accelerate mlu dflash2 sampling and convolution.`
- 分支：`refactor/mlu-dflash2-bangc-only` → `main`
- Head：`191ab0c63c3b1983af46aa06a9ed2a3420640bb9`（页面提交链接）；未另查代码仓库。
- 页面状态：Open / Reviewing；13 commits、53 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。
- 提取时间：2026/10/20 18:54:12（Asia/Shanghai）；已登录 Chrome 刷新加载后读取。
- 筛选：不按作者、时间、解决状态过滤；包含作者说明、评审总结及会话中的全部已发布行级意见与回复；不含提交和 reviewed 事件。
- 清单：19 条根评论（1 条作者说明、1 条评审总结、17 条行级意见），0 条回复。页面 Conversation 17 不用作评论总数。
- 范围：已检查会话中的全部讨论，未见折叠讨论、隐藏同类评论、过期线程、分页或加载更多入口；未检查 Files Changed 中未在会话展示的内容，不含草稿。
- 状态说明：17 条行级讨论均仅见“Resolved”和“No Processing Required”操作按钮，未见明确解决事件，各线程均标为“状态未核实”。两条总体评论未见解决状态标记。
- 原文说明：作者和 reviewer 的技术判断、验证声明及性能数字未经导出者独立验证。R07 称“当前尚未到达”2026 年，与页面评论时间及提取日期（2026/10/20）冲突，原样保留。R08 的“本轮未解决 finding：无”是该次评审总结，不代表页面所有讨论的解决状态；其他评审的意见仍独立保留。部分评论标题在源页面中已截断（如 R03–R05），R09 的规范链接也由页面自动转换；均原样保留。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认这些技术问题成立；相近意见仍按原始线程分别保留。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：优化范围、验证说明及性能表 | 未见解决状态标记 |
| R02 | 正确性：ConvNorm 注册层 CHECK 与降级契约 | 状态未核实 |
| R03 | 健壮性：采样内核 NRAM 占用 | 状态未核实 |
| R04 | 健壮性：ConvNorm 内核 SRAM 占用 | 状态未核实 |
| R05 | 正确性：分组卷积转置及 cycle_add 边界 | 状态未核实 |
| R06 | 正确性：采样内核 top_k 除零防护 | 状态未核实 |
| R07 | 正确性：版权年份（原文称尚未到 2026，与页面日期冲突） | 状态未核实 |
| R08 | 评审总结：本轮未确认缺陷，有条件推荐合并 | 未见解决状态标记 |
| R09 | minor / license：删除 All Rights Reserved. | 状态未核实 |
| R10 | minor / reuse：grouped_conv 多层重复校验 | 状态未核实 |
| R11 | minor / style：kpool 魔法数字 | 状态未核实 |
| R12 | minor / style：fused_sampl 截断命名 | 状态未核实 |
| R13 | minor / reuse：apply_norm 重复实现 | 状态未核实 |
| R14 | P2：grouped_conv 门限后恒真 CHECK | 状态未核实 |
| R15 | P2：fused_sampl 统一更名 | 状态未核实 |
| R16 | P2：随机采样混合设备门限不完整 | 状态未核实 |
| R17 | P2：kpool B16 启发式影响共享路径，缺测试 | 状态未核实 |
| R18 | P2：三个算子 TORCH_LIBRARY 注册面缺测试 | 状态未核实 |
| R19 | Question：共享 kpool verify metadata 对 DFlash 一代的影响 | 状态未核实 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月20日 16:54:26｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

本 PR 优化 MLU 平台 DFlash2 推测解码的采样与卷积执行路径，并将后端优化封装到独立的 MLU 实现中。

* 新增 BANG C 分组卷积、随机路径采样及卷积 + 残差 + RMSNorm 融合算子，减少中间计算和算子调用开销；处理 BF16 中间舍入与残差原地更新语义。
* 引入 MLU 专用 DFlash2 Decoder，复用通用卷积模块的权重和投影逻辑，保持 checkpoint 参数名称及加载方式兼容。
* 将 DFlash/DFlash2 worker 整理到 runtime/dflash/，通过 CMake 在编译时选择融合采样实现。
* 为 GLM5 系列目标模型开放 DFlash2 验证阶段的图执行，补齐有状态目标模型的 kpool 验证元数据，并调整四 query 验证场景的 kpool 分块策略。
* 补充卷积、采样、权重加载及图规划与回放测试，覆盖混合采样、非连续输入、序列块边界和尾部计算等场景。

验证说明： 分支已新增或更新相关测试；本次仅生成 PR 描述，未执行编译、测试或性能基准。

| Batch | 预测token | 原Triton µs | BangC µs | 加速范围 |
| --- | --- | --- | --- | --- |
| 16 | 3 | 30.287 | 4.250 | 7.09–7.13× |
| 16 | 5 | 44.300 | 5.000 | 8.86–8.90× |
| 32 | 3 | 58.562 | 5.800 | 10.05–10.14× |
| 32 | 5 | 86.775 | 7.200 | 12.01–12.10× |
| 64 | 3 | 114.650 | 8.500 | 13.45–13.50× |
| 64 | 5 | 171.087 | 10.800 | 15.73–16.12× |
| 128 | 3 | 227.350 | 13.200 | 17.22–17.44× |
| 128 | 5 | 339.288 | 17.325 | 19.52–19.67× |

## R02 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469096`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors.
```

### 评论原文

**1. 正确性 — xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp**
位置：`xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp:1`

xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp 在 dflash2_conv_norm 的注册实现中，当 block_size 不等于 4 或 6 时，使用了 CHECK 宏导致进程直接崩溃。头文件注释声明“Unsupported inputs return nullopt without launching or modifying tensors”，意味着上层 API 设计意图是对不支持的输入优雅降级返回 std::nullopt，但注册层却直接 CHECK 崩溃，导致无法优雅降级，且 CHECK 宏不会抛出 Python 可见的异常栈

建议将 CHECK 替换为 TORCH_CHECK 或直接返回空值/抛出 c10::Error，以符合头文件声明的契约并提供更好的错误诊断信息。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469097`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

**2. 健壮性 — xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu 在**
位置：`xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu:1`

xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu 在 sample_path_kernel 中声明了多个大型 **nram** 数组，如 logs_storage[4096]、proposal_buf[4096]、edge_buf[16384]、scores[4096]、noise_buf[4096]、ids_buf[4096] 以及 token_buf[32]。粗略计算这些数组的总显存占用约为 160KB。在 MLU 硬件上（如 MLU370 通常有 256KB NRAM），考虑到对齐和系统保留空间，如此大的 NRAM 静态分配极易导致栈溢出或初始化失败

建议：核对目标硬件的 NRAM 容量限制，考虑将部分缓冲区合并复用，或改为动态分配（如果 MLU BANGC 支持），或使用 SRAM/DRAM 缓存部分数据，以降低 NRAM 峰值占用。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/src/dflash2_conv_norm/conv_norm.mlu:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469098`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

**3. 健壮性 — xllm/core/kernels/mlu/bangc/src/dflash2_conv_norm/conv_norm.**
位置：`xllm/core/kernels/mlu/bangc/src/dflash2_conv_norm/conv_norm.mlu:1`

xllm/core/kernels/mlu/bangc/src/dflash2_conv_norm/conv_norm.mlu 在 conv_norm_kernel 中声明了 **mlu_shared** 数组，其中 shared_residual 大小为 48 * 4096 个元素，shared_history 为 524288 字节，加上其他共享内存数组，SRAM 总占用超过 1MB。对于 SRAM 容量有限（如 1MB 或 2MB）的 MLU 架构，这种分配可能超出硬件限制，或者在多 Cluster 并发执行时引发显存分配失败

建议：检查目标 MLU 硬件的 SRAM 容量，如果超限，需将部分 shared_history 或 shared_residual 降级到 GDRAM 并通过 IO1 异步拷贝，或重构算法以分块处理，减少 SRAM 占用。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/src/dflash2_grouped_conv/kernel.mlu:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469099`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

**4. 正确性 — xllm/core/kernels/mlu/bangc/src/dflash2_grouped_conv/kernel.**
位置：`xllm/core/kernels/mlu/bangc/src/dflash2_grouped_conv/kernel.mlu:1`

xllm/core/kernels/mlu/bangc/src/dflash2_grouped_conv/kernel.mlu 在 grouped_conv_cached 函数中，当 count_groups >= 4 时，执行了 __bang_transpose 和 __bang_cycle_add 操作。__bang_cycle_add 的参数为 (transposed, transposed, tap_delta, count, count_groups)，其中 count 为 count_groups * group_size。如果 tap_delta 的实际有效长度小于 count_groups，或者 transposed 缓冲区大小不足以容纳转置后的矩阵（大小为 count），将会导致越界读写

建议：确保 tap_delta 数组的长度严格等于 count_groups，且 transposed 缓冲区分配的大小至少为 count 个元素，并在代码中添加相应的断言或边界检查。

## R06 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu:3｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469100`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
+Licensed under the Apache License, Version 2.0 (the "License"). */
+#include <bang.h>
```

### 评论原文

**5. 正确性 — xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu 在**
位置：`xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu:3`

xllm/core/kernels/mlu/bangc/src/dflash2/sample_path.mlu 在 sample_path_kernel 中，通过 const bool preload_noise = shape.nk == 1 && shape.ns == top_k && steps <= 4096 / top_k; 决定是否预加载噪声。虽然上层 API 对 top_k 做了 CHECK_GT(top_k, 0) 的保护，但内核本身作为独立的执行单元，直接进行 4096 / top_k 运算，若被其他路径直接调用且 top_k 为 0，将触发除零异常

建议：在内核入口处或计算 preload_noise 之前，增加对 top_k > 0 的断言检查，增强内核的健壮性。

## R07 — 邓英旭 (dengyingxu1)｜2026/10/20 17:14:58｜xllm/core/kernels/mlu/bangc/torch_api/dflash2_conv_norm.cpp:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469101`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors.
```

> 导出注：下文年份判断与该评论的页面时间（2026/10/20）冲突，未改动原文。

### 评论原文

**6. 正确性 — xllm/core/kernels/mlu/bangc/include/dflash2_conv_norm.h**
位置：`xllm/core/kernels/mlu/bangc/torch_api/dflash2_conv_norm.cpp:1`

xllm/core/kernels/mlu/bangc/include/dflash2_conv_norm.h 文件头部的版权年份标注为 2026 年。这通常是由于复制模板或笔误造成的，当前尚未到达该年份

建议：将版权年份修改为当前年份（如 2024 或 2025），以符合代码规范和实际情况

## R08 — 邓英旭 (dengyingxu1)｜2026年10月20日 17:37:19｜MR 总体评论｜未见解决状态标记

- 类型：评审总结；根评论，0 条回复。

### 评论原文

MR #543 审查：target/base `main`=`bf823cc776cae8d18b4b3edae10688335d1d9c8a`；source head=`191ab0c63c3b1983af46aa06a9ed2a3420640bb9`；merge-base=`3bb1e13500fef131ba592e3de5d9c69a97497672`。审查了完整 13 个提交、53 个改动文件及相关调用、张量/状态契约和测试，重点覆盖 MLU DFlash2 BANG C 分组卷积、采样与 ConvNorm 融合、专用 decoder、worker 采样路径、GLM5 图规划/kpool 及相应测试。另核对 target 后续对 DFlash CP 与采样接口的变更；最新 target 与 head 的 merge-tree 干净。

检查结果：`git diff --check`（MR diff 和最新 target diff）通过。未执行 C++/BANG C 构建、MLU 测试或设备运行：本地缺少 cncc、Neuware 和 Torch/torch_mlu。现有讨论已检查。本轮未发现可确认的 P0-P3 缺陷，因此未发布 inline finding；本轮未解决 finding：无。

合并建议：有条件推荐合并（条件：在 MLU mtp_592 目标环境完成 BANG C 构建，并运行 DFlash2 采样、分组卷积及 ConvNorm 测试）。

## R09 — 冯艳 (fengyan.119)｜2026/10/20 18:09:55｜xllm/core/kernels/mlu/dflash2_grouped_conv.cpp:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469218`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

#### 【🔵 minor】【📄 license】

#### 新增文件 copyright 头含 "All Rights Reserved."，与仓库规范及同 MR 其他新文件不一致

---

**🔎 问题**

> 本 MR 新增/迁移的多个文件首行为 `/* Copyright 2026 The xLLM Authors. All Rights Reserved.`，而仓库 license 头规范（[custom-code-style.md:74](http://custom-code-style.md:74)）明确“Do not add `All Rights Reserved` to the project copyright header”；同 MR 内 xllm/core/kernels/mlu/bangc/include/dflash2_conv_norm.h、xllm/core/layers/mlu/dflash2_decoder_layer.cpp 等新文件已是不带该短语的形式，同一 MR 内两种写法并存。

---

**🔧 修改建议**

统一把相关文件首行改为 `/* Copyright 2026 The xLLM Authors.`（删除 “ All Rights Reserved.”），与 dflash2_conv_norm.h、dflash2_decoder_layer.* 等同 MR 文件保持一致。

## R10 — 冯艳 (fengyan.119)｜2026/10/20 18:09:55｜xllm/core/kernels/mlu/dflash2_grouped_conv.cpp:42｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469219`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+      base.device() != hidden_states.device()) {
+ return std::nullopt;
+  }
+ CHECK(hidden_states.device().is_privateuseone());
```

### 评论原文

#### 【🔵 minor】【♻️ reuse】

#### try_dflash2_grouped_conv 重复执行 bangc 层已完整负责的 shape/dtype/device 校验集

---

**🔎 问题**

> try_dflash2_grouped_conv 在轻量 guard（31-41 行）之后，42-68 行再次执行与 bangc::dflash2_grouped_conv（torch_api/dflash2_grouped_conv.cpp:32-65）逐条对应的全套 CHECK；调用方 layer 层 mlu::dflash2_grouped_conv 35-55 行又先校验一遍 dim/block_size/num_groups/shape。同一 invariant 三层重复，后续修改 kernel 契约需同步三层，容易漂移。

---

**🔧 修改建议**

try_dflash2_grouped_conv 只保留决定能否走 fused 路径的前置 guard（31-41 行），guard 通过后直接 `return bangc::dflash2_grouped_conv(hidden_states, delta, base, block_size, group_size);`，删除 42-68 行与 bangc 层重复的整套 CHECK；shape/dtype/device 契约由 bangc 层唯一负责。layer 层 mlu::dflash2_grouped_conv 的 CHECK 集服务于其自身对 common eager 实现的公共入口，可保留。

## R11 — 冯艳 (fengyan.119)｜2026/10/20 18:09:55｜xllm/core/kernels/mlu/kpool.cpp:378｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469220`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
  if (four_query_verify) {
- constexpr int32_t kPoolsPerTile = 128;
+ int32_t pools_per_tile = 128;
+ if (table.size(0) == 16 && scores.size(1) >= 4096 &&
```

### 评论原文

#### 【🔵 minor】【🎨 style】

#### pools_per_tile 选择分支引入一批未命名的魔法数字

---

**🔎 问题**

> score_kpool 的 four_query_verify 分支（kpool.cpp:377-392）新增裸魔法数：table.size(0)==16、scores.size(1)>=4096、<=32768、(scores.size(1)+1023)/1024、pools_per_tile=256、默认 128。custom-code-style 要求魔法数字使用 constexpr 命名常量；同一函数内已有命名先例（394 行 kTilesPerProgram、424-425 行 kTile/kChunk），这些新常量却直接以字面量混入条件表达式。

---

**🔧 修改建议**

提取 constexpr 命名常量并替换全部字面量，例如：`constexpr int32_t kDefaultPoolsPerTile = 128; constexpr int32_t kLargePoolsPerTile = 256; constexpr int64_t kUnionLaunchTableRows = 16; constexpr int64_t kMinUnionLaunchScoreCols = 4096; constexpr int64_t kMaxUnionLaunchScoreCols = 32768; constexpr int64_t kScoreColTile = 1024;`（命名可按实际语义调整），条件与 ceil-div 全部改用常量表达。

## R12 — 冯艳 (fengyan.119)｜2026/10/20 18:09:55｜xllm/core/runtime/dflash/dflash2_worker_impl.h:105｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469221`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+    torch::Tensor dense_probs;
+  };
+
+ static std::optional<BlockSampleOutput> fused_sampl(
```

### 评论原文

#### 【🔵 minor】【🎨 style】

#### 新增静态方法名 fused_sampl 使用未解释的截断缩写

---

**🔎 问题**

> fused_sampl（dflash2_worker_impl.h:105 声明，mlu_fused_sampling.cpp:22 与 default_fused_sampling.cpp:21 两处定义，dflash2_worker_impl.cpp:743 调用）是 "fused sample" 的截断缩写 "sampl"。custom-code-style 禁止未解释缩写；同一 MR 相邻符号均用全称（try_dflash2_random_sample_path、sample_path、BlockSampleOutput）。

---

**🔧 修改建议**

改名为 fused_sample：同步更新 dflash2_worker_impl.h 声明、mlu_fused_sampling.cpp 与 default_fused_sampling.cpp 两处定义、dflash2_worker_impl.cpp:743 调用点，以及 tests/core/runtime/mlu_dflash2_sampling_test.cpp 中 DFlash2WorkerImplTestPeer::fused_sampl 转发。

## R13 — 冯艳 (fengyan.119)｜2026/10/20 18:09:56｜xllm/core/layers/mlu/dflash2_decoder_layer.cpp:96｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469222`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+}
+
+std::tuple<torch::Tensor, std::optional<torch::Tensor>>
+DFlash2DecoderLayerImpl::apply_norm(
```

### 评论原文

#### 【🔵 minor】【♻️ reuse】

#### DFlash2DecoderLayerImpl::apply_norm 逐字复制 Qwen2DecoderLayerImpl::apply_norm

---

**🔎 问题**

> 新增的 DFlash2DecoderLayerImpl::apply_norm（dflash2_decoder_layer.cpp:95-115）与既有 Qwen2DecoderLayerImpl::apply_norm（xllm/core/layers/qwen2_decoder_layer.cpp:100-120）逐字相同，包括注释（首层 residual 初始化、fp8 融合选择、fused add+norm）。RMSNorm+residual+fp8 的融合选择逻辑现在在两个 decoder layer 里各维护一份。

---

**🔧 修改建议**

把 apply_norm 提取为 layers/common 下的自由函数（其唯一依赖是 RMSNorm 接口，与两个使用方共同依赖的 rms_norm.h 同层），例如 `std::tuple<torch::Tensor, std::optional<torch::Tensor>> apply_rms_norm(RMSNorm& norm, torch::Tensor& input, std::optional<torch::Tensor>& residual, const std::optional<torch::Tensor>& fp8_scale);`，两个 decoder layer 各自调用，删除复制的私有静态实现。

## R14 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:40:24｜xllm/core/kernels/mlu/dflash2_grouped_conv.cpp:42｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469291`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+      base.device() != hidden_states.device()) {
+ return std::nullopt;
+  }
+ CHECK(hidden_states.device().is_privateuseone());
```

### 评论原文

[P2] try_dflash2_grouped_conv 早退门限后紧跟 8 处恒真 CHECK，构成不可达的防御性死代码

**后果**：`group_size > 2048`、设备/dtype 不匹配等条件任一成立时，函数在 `:35-41` 已 `return std::nullopt`。此后 `:42`（privateuseone）、`:46-:49`（delta/base 的 device 与 dtype）、`:50-:52`（dtype_supported）、`:55`（`CHECK_LE(group_size, 2048)`）共 8 处 CHECK 断言的正是门限里已排除的条件，在所有可达路径下恒真、永不触发。这是"复制 bangc 层 CHECK 契约块到 try_ 层再加门限"形成的 layer → try_ → bangc 三层重复校验，与门限重叠的部分退化为死代码。给作者"我在这里做了防护"的心理安慰，实际保护不了任何真错误；将来门限调整（如放宽 group_size 上限）时，死 CHECK 与门限会静默漂移出两份真相。

**触发条件**：任何执行路径都无法触发这些 CHECK（这正是问题——它们不可达）。

**贴码**（xllm/core/kernels/mlu/dflash2_grouped_conv.cpp:34-55 节选）：

```cpp
  if (group_size > 2048 || !hidden_states.device().is_privateuseone() ||
      !dtype_supported || delta.scalar_type() != hidden_states.scalar_type() ||
      base.scalar_type() != hidden_states.scalar_type() ||
      delta.device() != hidden_states.device() ||
      base.device() != hidden_states.device()) {
    return std::nullopt;                      // 早退门限
  }
  CHECK(hidden_states.device().is_privateuseone());   // :42 恒真
  CHECK_EQ(hidden_states.dim(), 2);                   // :43 活检查
  ...
  CHECK_EQ(delta.device(), hidden_states.device());   // :46 恒真
  CHECK_EQ(base.device(), hidden_states.device());    // :47 恒真
  CHECK_EQ(delta.scalar_type(), hidden_states.scalar_type());  // :48 恒真
  CHECK_EQ(base.scalar_type(), hidden_states.scalar_type());   // :49 恒真
  ...
  CHECK_LE(group_size, 2048);                         // :55 恒真
```

**根因**：try_ 包装是在 bangc 版 `dflash2_grouped_conv`（torch_api/dflash2_grouped_conv.cpp:32-65 的同款 CHECK 契约块）前加门限时，把 bangc 层的 CHECK 契约整段复制进了 try_ 层。对比姊妹接口 `try_dflash2_random_sample_path`，其门限干净、无此问题。

**最小修复**：删除 `:42`、`:46-:49`、`:50-:52`、`:55` 这 8 处与门限重复的 CHECK；保留 `:43-:45`、`:53-:54` 的 shape 活检查。或反向收敛：try_ 只做门限，shape CHECK 完全交给 bangc 层（后者已全部具备）。

**验证**：删除后 MLU 构建编译通过 + 既有 `MatchesEagerWithStridedProjectionAndHistory`（dflash2_test.cpp:178+，3 dtype × 4 taps 参数化）与 `UnsupportedFusionLeavesInputsForEagerNorm` 仍绿即证明行为不变。

## R15 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:43:20｜xllm/core/runtime/dflash/mlu_fused_sampling.cpp:22｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469295`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+namespace xllm {
+
+std::optional<DFlash2WorkerImpl::BlockSampleOutput>
+DFlash2WorkerImpl::fused_sampl(const torch::Tensor& candidate_ids,
```

### 评论原文

[P2] 静态方法名 `fused_sampl` 疑为 `fused_sample` 的截断笔误，7 处引用一致沿用

**后果**：纯命名问题，无功能影响。但该方法已是跨 4 个生产文件 + 测试 peer 的公共静态入口，7 处一致说明是系统性笔误而非局部疏漏，越晚改成本越高。仓库内同类命名（`fused_gdn_gating.cpp`、`fused_sigmoid_gating_delta_rule_update.cpp`、`fused_post_conv_prep.cpp`）均为完整拼写。commit 13d（"simplify sampling dispatch"，改名去 try 前缀）疑为 `try_fused_sample → fused_sample` 改名时的截断。

**触发条件**：所有构建（MLU 与非 MLU）——纯命名问题。

**贴码**（xllm/core/runtime/dflash/dflash2_worker_impl.h:104-110 声明；调用点 dflash2_worker_impl.cpp:743）：

```cpp
  // dflash2_worker_impl.h:104-110
  static std::optional<BlockSampleOutput> fused_sampl(
      const torch::Tensor& candidate_ids,
      const torch::Tensor& edge_log_probs,
      const torch::Tensor& gumbel_noise,
      const SamplingParameters& sampling_params,
      int64_t vocab_size);
```

7 处引用位置：dflash2_worker_impl.h:105（声明）、dflash2_worker_impl.cpp:743（调用）、mlu_fused_sampling.cpp:22、default_fused_sampling.cpp:21（两份定义）、tests/core/runtime/mlu_dflash2_sampling_test.cpp:36/41/174（测试 peer）。

**根因**：改名截断（推断，改名前拼写无法从 head 快照确认）。

**最小修复**：一次性重命名 7 处为 `fused_sample`（含测试 peer），无行为变化。

**验证**：编译 + `MluDFlash2SamplingTest` 全绿（测试 peer 通过静态方法名调用，改名后同步即可）。

## R16 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:45:37｜xllm/core/kernels/mlu/dflash2_sample_path_random.cpp:35｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469299`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+ const int64_t top_k = candidate_ids.size(2);
+ const int64_t num_steps = candidate_ids.size(1);
+ if (all_greedy_sample || top_k <= 0 || top_k > 4096 || num_steps <= 0 ||
+      !edge_log_probs.device().is_privateuseone()) {
```

### 评论原文

[P2] try_dflash2_random_sample_path 的设备门限只查 edge_log_probs，混合设备输入穿透到 bangc 层 CHECK 崩溃而非返回 nullopt

**后果**：`try_` 前缀的契约是"不支持的输入在 launch 前返回 std::nullopt，由上层回退 eager 路径"。但采样门限只检查 `edge_log_probs` 一个张量的设备；若 `candidate_ids`（或 `gumbel_noise`/`do_sample`）在 CPU 而 edge 在 MLU，门限放行后直接穿透到 bangc 层的 `CHECK_EQ(candidate_ids.device(), edge_log_probs.device())`，glog CHECK 进程崩溃、无 Python 可见异常栈——优雅降级契约对采样入口不完整。当前从 `DFlash2WorkerImpl::sample_path` 调用时三张量同源、现实中不可达，但 `fused_sampl` 是 static 公共入口，测试 peer 已直接调用它（HostInputsAreRejectedWithoutMutation），未来任何新调用方传混合设备输入即触发崩溃。

**触发条件**：`candidate_ids`/`gumbel_noise`/`do_sample` 与 `edge_log_probs` 设备不一致、且 edge 在 MLU 上。

**贴码**（xllm/core/kernels/mlu/dflash2_sample_path_random.cpp:32-38 门限；对照 bangc 层 torch_api/dflash2_sample_path.cpp:34-41）：

```cpp
  // 门限只检查 edge_log_probs 一个张量的设备
  if (all_greedy_sample || top_k <= 0 || top_k > 4096 || num_steps <= 0 ||
      !edge_log_probs.device().is_privateuseone()) {
    return std::nullopt;
  }
  return dflash2_random_sample_path(candidate_ids, ...);  // 穿透

  // bangc/torch_api/dflash2_sample_path.cpp:34-36 —— 混合设备在此 CHECK 崩溃
  CHECK_EQ(candidate_ids.device(), edge_log_probs.device());
  CHECK_EQ(candidate_ids.device(), gumbel_noise.device());
```

对照姊妹接口 `try_dflash2_grouped_conv`（kernels/mlu/dflash2_grouped_conv.cpp:35-39）：门限同时检查 `hidden_states/delta/base` 三个张量的设备与 dtype，采样版只查了一张。

**根因**：门限张量集合与被包装层的 CHECK 张量集合不对称——门限是"edge 在 MLU"，CHECK 是"四张量同设备"。

**最小修复**：门限补齐 `candidate_ids.device().is_privateuseone() && gumbel_noise.device().is_privateuseone()`（gumbel_noise 未定义时可一并查 `.defined()`），与 `try_dflash2_grouped_conv` 门限风格对齐。

**验证**：新增一个 host/mixed 设备输入的 peer 测试，断言返回 nullopt 且输入无突变（现成模板：mlu_dflash2_sampling_test.cpp 的 HostInputsAreRejectedWithoutMutation）。

## R17 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:47:56｜xllm/core/kernels/mlu/kpool.cpp:378｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469300`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
  if (four_query_verify) {
- constexpr int32_t kPoolsPerTile = 128;
+ int32_t pools_per_tile = 128;
+ if (table.size(0) == 16 && scores.size(1) >= 4096 &&
```

### 评论原文

[P2] kpool B16 tile 启发式改变共享 four_query_verify 路径的 launch grid（含既有 GLM5 MTP verify 负载），且该选择逻辑零测试覆盖

**后果**：`four_query_verify` 的判据（kpool.cpp:362-373）只看 query/cache 形状与 block_size==16，不含算法身份。因此任何命中 `table.size(0)==16`、`4096<=scores.size(1)<=32768`、`candidate_programs > core_count` 的负载——**包括 merge-base 之前就存在的 GLM5 MTP B16 verify 与 DFlash 一代 verify**——launch grid 第二维都会减半、每 program 处理的池数翻倍（128→256），改变其 launch 模式与性能特征（方向未测）。MR 主题是"MLU-only、不污染通用代码"，这是三处共享文件行为改变中影响面最大的一处；且 `candidate_programs > core_count` 依赖运行时设备属性，不同核数的 MLU 型号行为分叉，无日志可观测。同时该选择逻辑完全无测试锁定。

**触发条件**：`four_query_verify` 为真 + `table.size(0)==16` + `4096<=scores.size(1)<=32768` + `candidate_programs > core_count`——不限于 DFlash2。

**贴码**（xllm/core/kernels/mlu/kpool.cpp:377-395，本 MR 唯一被修改的既有内核文件，+25/-18；原为 constexpr kPoolsPerTile=128）：

```cpp
  if (four_query_verify) {
    int32_t pools_per_tile = 128;
    if (table.size(0) == 16 && scores.size(1) >= 4096 &&
        scores.size(1) <= 32768) {
      ...
      if (candidate_programs > core_count) {
        pools_per_tile = 256;      // B16 大 scores 时翻倍
      }
    }
```

**根因**：共享内核的调优门限以 workload 形状（table.size(0)==16）而非算法身份为条件，影响面超出 DFlash2。commit 1 的意图是"Tune KPool scoring tiles for B16 verification loads"，对 B16 负载是有意为之，但未说明是否预期覆盖 MTP/GLM5 一代负载。

**最小修复**：① 补一个单测锁定选择逻辑（构造 table.size(0)==16 且 scores.size(1)=16384 的输入，断言 JITKernel 收到的 pools_per_tile；若不可注入，至少在 256 分支加一次 VLOG 便于线上观测）；② 在 kpool.cpp 注释或 MR 描述中明确"该启发式同时作用于 GLM5 MTP verify"，避免后续归因混乱。

**验证**：GLM5 MTP B16 verify 在目标设备（mtp_592）上 A/B 对比 pools_per_tile=128/256 的延迟，确认无回退。

## R18 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:50:15｜xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp:47｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469301`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+}
+}  // namespace
+
+TORCH_LIBRARY_FRAGMENT(xllm_mlu, m) {
```

### 评论原文

[P2] 三个 DFlash2 算子的 TORCH_LIBRARY 注册面（register/ 层）完全无测试覆盖

**后果**：12 个测试文件、约 40 个 TEST 全部直接调用 C++ API（`kernel::mlu::bangc::dflash2_conv_norm` 等），没有任何测试经过 TORCH_LIBRARY 注册面。注册层独有的三段逻辑因此全部未被测试锁定：① 适配层的 CHECK 崩溃路径（block_size≠4/6 时 CHECK 进程崩溃而非可捕获异常——已有讨论线程 1 的本体，此处只报覆盖缺口不重复定性）；② schema 参数收窄（`int block_size` 按 int64_t 进 dispatcher 后 `static_cast<int32_t>`，越界 int 会静默截断后才命中 CHECK）；③ `Tensor(a!) residual` 可变参数标注与返回值第二元素的别名契约是否与 dispatcher 语义一致。注册面是这三个算子对 Python/外部消费者的唯一公开契约，将来若改 schema（加参数、改标注），现有测试全绿也不会发现注册面损坏。

**触发条件**：任何经 `torch.ops.xllm_mlu.*`（Python 或 C++ dispatcher）调用这三个算子的路径。

**贴码**（xllm/core/kernels/mlu/bangc/register/dflash2_conv_norm.cpp:47-54；dflash2_grouped_conv.cpp 与 dflash2_sample_path.cpp 同款）：

```cpp
  TORCH_LIBRARY_FRAGMENT(xllm_mlu, m) {
    m.def(
        "dflash2_conv_norm(Tensor hidden, Tensor delta, Tensor base, "
        "Tensor(a!) residual, Tensor weight, int block_size, float eps) "
        "-> (Tensor, Tensor(a!))");
  }
  TORCH_LIBRARY_IMPL(xllm_mlu, PrivateUse1, m) {
    m.impl("dflash2_conv_norm", TORCH_FN(dflash2_conv_norm));
  }
```

适配层（register/dflash2_conv_norm.cpp:30-44）含 `CHECK(block_size == 4 || block_size == 6)` 与 `static_cast<int32_t>(block_size)` 收窄。

**根因**：测试与实现同源于 C++ API 层，注册层被视为纯转发而未纳入契约测试。

**最小修复**：在 mlu 测试集中加一个 RegisterSurfaceTest：经 `torch::dispatch` 或 Python 子进程调用 `xllm_mlu.dflash2_grouped_conv` 一次（复用现成测试张量），断言与直调结果一致；再各加一条不支持输入验证行为（返回 nullopt 或 CHECK 崩溃，按实际契约断言）。

**验证**：新测试在 MLU CI 绿。

## R19 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 18:52:54｜tests/core/runtime/mlu_dflash2_sampling_test.cpp:387｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3469303`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+  }
+};
+
+TEST(MluDFlash2VerifyInputTest, ReplacesDecodeSpansOnlyForStatefulTargets) {
```

### 评论原文

[Question] kpool verify metadata 填充位于 DFlash/DFlash2 共享基类，门控是 target_is_hybrid_recurrent_ 而非 DFlash2 算法——DFlash 一代 verify 行为同样被改变，请确认意图

**说明**：merge-base 上此处不设置 kpool verify metadata；本 MR 后，DFlash 一代的 verify 输入也会被填 `kpool_query_lens = num_speculative_tokens()+1`（测试 ReplacesDecodeSpansOnlyForStatefulTargets 断言了该行为，但只以 "DFlash2" 算法实例化 worker，mlu_dflash2_sampling_test.cpp:1157）。commit 1 的措辞是 "Fix GLM5 DFlash2 target verification admission and KPool metadata"，未提一代。

**触发条件**：compressed kpool cache 平台 + 有 request state cache 的 target + 任何 hybrid recurrent verify（DFlash 一代或 DFlash2 均命中，门限不含算法判断）。

**贴码**（xllm/core/runtime/dflash/dflash_worker_impl.cpp:1311-1324，所在函数为 DFlash 一代与 DFlash2 共用的基类 `DFlashWorkerImpl::prepare_validate_inputs`）：

```cpp
  const bool use_linear_spec_verify = target_is_hybrid_recurrent_;   // :1311
  prepared_input.input_params.is_spec_verify = use_linear_spec_verify;
  if (use_linear_spec_verify) {
    ModelInputParams& input_params = validate_input.input_params;
    if (Platform::uses_compressed_kpool_cache() &&
        impl_->has_request_state_cache()) {
      specBuilder::set_kpool_verify_metadata(                    // :1320 新增
          input_params,
          std::vector<int32_t>(input.input_params.meta.num_sequences,
                               options_.num_speculative_tokens() + 1));
    }
```

**问题**：如果这是修 DFlash 一代同样存在的元数据缺失（fix），请在 commit/MR 描述补一句"同样修复 DFlash 一代"；如果只想影响 DFlash2，则门限应收窄（加 is_dflash2_algorithm 判断），否则一代负载在 compressed cache 平台获得未经验证的行为变化。

**建议验证**：用 `speculative_algorithm("DFlash")` 实例化同一测试（现有模板把算法写死为 "DFlash2"），断言 kpool_query_lens 行为与预期一致。
