# xLLM MR #636 — Force-push 后的 Review 意见与修复交接

- 来源：[MR #636](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636)
- 标题：`bugfix: reject oversized vlm inputs before worker dispatch.`
- 分支：`bugfix/vlm_oversize` → `main`
- 时间边界：罗磊 (ext.luolei15) 于 **2026-10-30 15:39:59（Asia/Shanghai）** force-pushed，分支从 [`1ba509d`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/commit/1ba509d230c3a878d42d9defb7f61c0f5c915716) 更新到 [`514fd75`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/commit/514fd75378cbf5bec6618b2010d7f04a9ebb0d13)。仅收录严格晚于该事件发布的 review 评论。
- 当前页面提交列表首项：`514fd75378cbf5bec6618b2010d7f04a9ebb0d13`；其余为 `9f576b804a7c916e615cc0c3f5558ac52a788223`、`875b51ad36bdbbb2d72cb0dabbb107d647145a29`、`7cd3848dcea5a00f0864dacfc761dbabeae1a717`。已打开 Commits 页核对，未另查仓库确认 head。
- 页面状态：Open / Reviewing；4 commits、27 files changed；要求 2 人批准，3 项状态检查通过、1 项运行中（xLLM服务CI流水线 #9271455），尚未满足合并要求。
- 提取时间：2026-10-30 17:39:45（Asia/Shanghai）；通过已登录 Chrome 刷新页面，点击 show more 加载后续记录，并展开全部 Show conversation 后读取。提交前于 2026-10-30 17:41:26 再次刷新、加载并展开讨论，核对评论正文、时间、锚点和解决标记未变。
- 筛选：移除该 force-push 之前的全部 14 条根评论（包括旧作者说明、旧行级意见及旧评审小结），不附旧评论正文；本轮不按评审者或解决状态过滤，保留已解决讨论。旧评论后来被标为已解决不会使其进入本轮范围。
- 清单：页面已加载正文共 26 条根评论、0 条回复；本次纳入 **12 条根评论，全部为行级意见，0 条回复**，其中邓英旭 6 条、冯艳 6 条；无本轮独立评审小结。页面 Conversation 25 不作为根评论总数。
- 状态说明：R02、R05、R06、R07、R08 的线程明确显示由罗磊标为 Resolved，记为“已解决”；其余 7 条只有处理动作按钮，未见明确解决事件，记为“状态未核实”。
- 重复发布说明：本轮 R07 与 R09、R08 与 R10 分别正文相同，但为不同时间和锚点的独立评论，前者明确已解决、后者未见解决事件，因此分别保留。它们均晚于时间边界，不属于已移除的旧版评论。
- 范围：已加载会话全部记录并展开折叠讨论，核对行级正文、锚点、时间、状态及回复位置；最终未见 show more、隐藏同类评论或其他分页入口。本轮纳入线程未见 OutDated 标记。未逐一打开 27 个文件 diff，无法独立排除会话外讨论；不含草稿、提交事件或无正文的 reviewed 事件。
- 原文说明：以下严重性、技术判断和修复建议均为评审者原文，未经导出者独立验证。R01/R02/R03/R04/R06 标题以“在”“使用”“硬编码”或“auto”等词结束，按页面原样保留，未补全。部分页面锚点位于 include、签名或版权头，与评论讨论的实现位置不同，以下保留两者，不推断缺失代码。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。R07/R09 与 R08/R10 各为同文的独立发布记录。

| 评论 | 评审者 | 主题 / 原文标签 | 状态 |
| --- | --- | --- | --- |
| R01 | 邓英旭 | 正确性：swap_blocks 按值拷贝是否破坏原始排序不变性 | 状态未核实 |
| R02 | 邓英旭 | 健壮性：初始化中 std::stoull 异常未捕获 | 已解决 |
| R03 | 邓英旭 | 健壮性：kEnvelopeReserve = 64 的信封容量假设 | 状态未核实 |
| R04 | 邓英旭 | 效率：candidate_batch 拷贝开销与共享状态 | 状态未核实 |
| R05 | 邓英旭 | 正确性：拆分执行后刷新原始 batch 的目的与同步 | 已解决 |
| R06 | 邓英旭 | 健壮性：延迟更新捕获 Sequence 裸指针的生命周期 | 已解决 |
| R07 | 冯艳 | minor / dead-code：execute_batch 预检后遗留 else 分支（第一次发布） | 已解决 |
| R08 | 冯艳 | minor / perf：最终容量守卫重复计算同一 dp 输入大小（第一次发布） | 已解决 |
| R09 | 冯艳 | minor / dead-code：execute_batch 预检后遗留 else 分支（再次发布） | 状态未核实 |
| R10 | 冯艳 | minor / perf：最终容量守卫重复计算同一 dp 输入大小（再次发布） | 状态未核实 |
| R11 | 冯艳 | minor / perf：DECODE 判定前无条件拷贝 MMData | 状态未核实 |
| R12 | 冯艳 | minor / dead-code：prepare_forward_input 的预览分支仅测试使用、入口重复 | 状态未核实 |

## R01 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜xllm/core/framework/batch/batch_input_builder.cpp:216｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486880`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486880)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
     int32_t cp_size,
-    ThreadPool* thread_pool)
-    : sequences_(sequences),
+    ThreadPool* thread_pool,
```

### 评论原文

**1. 正确性 — xllm/core/framework/batch/batch_input_builder.cpp 在**
位置：`xllm/core/framework/batch/batch_input_builder.cpp:216`

xllm/core/framework/batch/batch_input_builder.cpp 在 build_forward_input 中，旧代码使用 auto& swap_blocks = *swap_block_transfer_infos_; 原地排序，新代码改为 auto swap_blocks = *swap_block_transfer_infos_; 按值拷贝。当 commit_sequence_state_ 为 true 时，原始的 swap_block_transfer_infos_ 不再被排序。如果后续逻辑依赖原始 swap_block_transfer_infos_ 的有序状态，这会破坏原有不变性，导致行为不一致

建议如果原始的 swap_block_transfer_infos_ 需要在 commit_sequence_state_=true 时被排序，应保留引用或显式赋值回去；如果不需要排序，请添加注释说明为什么按值拷贝是安全的。

## R02 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜xllm/core/distributed_runtime/comm_channel.cpp:26｜已解决

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486881`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486881)；未见 OutDated 标记。
- 状态证据：线程明确显示 “This conversation was maked as "Resolved" by 罗磊”；已解决，页面未显示解决时间。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
 #include <cstdint>
 #include "common/global_flags.h"
+#include "core/common/health_check_manager.h"
```

### 评论原文

**2. 健壮性 — xllm/core/distributed_runtime/comm_channel.cpp 在通道初始化时使用**
位置：`xllm/core/distributed_runtime/comm_channel.cpp:26`

xllm/core/distributed_runtime/comm_channel.cpp 在通道初始化时使用 std::stoull(value) 解析 gflag max_body_size。如果该 gflag 被配置为非数字字符串或超出 unsigned long long 范围，std::stoull 会抛出 std::invalid_argument 或 std::out_of_range 异常。如果调用方未捕获此异常，会导致通道初始化时发生未处理异常而崩溃

建议使用 try-catch 块包裹 std::stoull，或在转换前验证字符串格式，若无效则返回错误或 CHECK 失败并输出清晰的错误信息。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜xllm/core/distributed_runtime/comm_channel.cpp:27｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486882`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486882)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
 #include "common/global_flags.h"
+#include "core/common/health_check_manager.h"
+#include "core/framework/config/execution_config.h"
```

### 评论原文

**3. 健壮性 — xllm/core/distributed_runtime/comm_channel.cpp 硬编码**
位置：`xllm/core/distributed_runtime/comm_channel.cpp:27`

xllm/core/distributed_runtime/comm_channel.cpp 硬编码 kEnvelopeReserve = 64 用于计算 forward_input_limit_，假设 brpc/protobuf 的信封大小永远不会超过 64 字节。如果实际传输时的信封（如头部、元数据）大于 64 字节，计算出的 forward_input_limit_ 会高估实际可用容量。这会导致原本应该被拒绝的超大输入通过了预览检查，但在实际发送时仍然失败，违背了本 MR 在分发前拒绝超大输入的初衷

建议验证 64 字节是否为 brpc/protobuf 信封的严格上限。如果可能，从相关 API 动态获取信封大小，或使用更安全的保守常量。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜xllm/core/distributed_runtime/vlm_engine.cpp:28｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486883`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486883)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
 #include <chrono>
-#include <cstdlib>
 #include <functional>
+#include <limits>
```

### 评论原文

**4. 效率 — xllm/core/distributed_runtime/vlm_engine.cpp auto**
位置：`xllm/core/distributed_runtime/vlm_engine.cpp:28`

xllm/core/distributed_runtime/vlm_engine.cpp auto candidate_batch = batch; 对 std::vector 进行了拷贝构造。如果 Batch 的拷贝构造是深拷贝（如拷贝张量或大向量），在每次预览时都会产生较大的性能开销；如果是浅拷贝（如仅拷贝指针），后续 prepare_inputs 修改 candidate_batch 的内部结构时，可能会与 batch 产生意外的共享状态问题

建议确认 Batch 的拷贝语义对此操作是安全的。如果深拷贝开销过大，考虑使用指针或移动语义优化；如果存在浅拷贝别名风险，确保修改 candidate_batch 不会污染 batch 的状态。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜xllm/core/distributed_runtime/vlm_engine.cpp:36｜已解决

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486884`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486884)；未见 OutDated 标记。
- 状态证据：线程明确显示 “This conversation was maked as "Resolved" by 罗磊”；已解决，页面未显示解决时间。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
 #include "common/interruption_bus.h"
 #include "common/metrics.h"
 #include "core/common/global_flags.h"
+#include "core/common/health_check_manager.h"
```

### 评论原文

**5. 正确性 — xllm/core/distributed_runtime/vlm_engine.cpp**
位置：`xllm/core/distributed_runtime/vlm_engine.cpp:36`

xllm/core/distributed_runtime/vlm_engine.cpp 在拆分执行路径的循环结束后，调用了 for (auto& dp_batch : batch) { dp_batch.refresh_sequences_from_groups(); }。但在拆分路径中，实际执行的是 sub_batch，原始 batch 对象并未直接参与执行。如果 batch 在此之后不再被使用，此操作属于无效的死代码；如果被使用，由于 sub_batch 与 batch 共享底层 Sequence 指针，需要确认 refresh_sequences_from_groups() 是否能正确同步 sub_batch 执行后的状态到 batch

建议明确此处刷新原始 batch 对象的目的。如果是死代码则移除；如果需要同步状态，请确保 refresh_sequences_from_groups() 的逻辑能正确反映 sub_batch 执行后的序列状态。

## R06 — 邓英旭 (dengyingxu1)｜2026/10/30 16:03:23｜tests/core/framework/batch/batch_transport_test.cpp:1｜已解决

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3486885`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3486885)；未见 OutDated 标记。
- 状态证据：线程明确显示 “This conversation was maked as "Resolved" by 罗磊”；已解决，页面未显示解决时间。
- 位置说明：页面锚点附近仅显示 include、签名或版权头，未显示评论所述实现；下方只保留页面实际代码节选。

### 页面代码上下文（锚点附近节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

**6. 健壮性 — xllm/core/framework/batch/batch_input_builder.cpp 在**
位置：`tests/core/framework/batch/batch_transport_test.cpp:1`

xllm/core/framework/batch/batch_input_builder.cpp 在 process_single_sequence 中，apply_or_defer 的 lambda 捕获了 sequence 的原始指针。当 commit_sequence_state_ 为 false 时，该 lambda 会被存储在 sequence_updates 中延迟执行。如果在 commit() 实际调用前，sequence 对象被销毁或生命周期结束，将导致 use-after-free

建议确保 Sequence 对象的生命周期严格长于 PreparedForwardInput 的生命周期。如果无法保证，考虑使用智能指针或在提交时重新获取序列引用。

## R07 — 冯艳 (fengyan.119)｜2026/10/30 17:19:56｜xllm/core/distributed_runtime/vlm_engine.cpp:603｜已解决

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3487172`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487172)；未见 OutDated 标记。
- 状态证据：线程明确显示 “This conversation was maked as "Resolved" by 罗磊”；已解决，页面未显示解决时间。
- 发布关系：与 R09 正文相同，属于不同时间和锚点的独立评论；不据其中一条的状态推断另一条。

### 页面代码上下文（锚点附近节选）

```diff
+  }
+  auto results = std::move(completion).value();
+  for (const auto& result : results) {
+    if (result.hasException() || !result.value().has_value()) {
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### execute_batch 结果预检循环使分发循环的 else 分支不可达

---

**🔎 问题**

> execute_batch 在 602-606 行新增的结果预检循环先对所有 futures 检查 hasException()/!has_value() 并 abort_execution，随后 610 行起的分发循环里保留了既有的 else { LOG(FATAL) << "Failed to execute model, result has no value"; }（638-640 行）。预检已保证进入分发循环时每个结果非空且无异常，该 else 分支永远不会执行，属于预检引入后遗留的死分支。同一 MR 中 update_last_step_result（672-676 预检 + 678-682 无条件 emplace）已经按去掉 else 的方式修复过完全相同的模式，此处漏改。

---

**🔧 修改建议**

删除分发循环中对 result 的 if/else 判断，直接以 result.value() 消费结果（与 update_last_step_result 678-682 行的写法一致，例如直接调用 batch[dp_rank].process_sample_output(result.value(), ...)），去掉 638-640 行的 else 死分支。

## R08 — 冯艳 (fengyan.119)｜2026/10/30 17:19:56｜xllm/core/distributed_runtime/vlm_engine.cpp:570｜已解决

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / perf。
- 线程锚点：[`discussion_r3487173`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487173)；未见 OutDated 标记。
- 状态证据：线程明确显示 “This conversation was maked as "Resolved" by 罗磊”；已解决，页面未显示解决时间。
- 发布关系：与 R10 正文相同，属于不同时间和锚点的独立评论；不据其中一条的状态推断另一条。

### 页面代码上下文（锚点附近节选）

```diff
+  // Last guard before any rank can observe an input. A mismatch with the
+  // preview is an invariant failure, not a reason to switch transports.
+  for (uint32_t rank = 0; rank < worker_clients_num_; ++rank) {
+    if (ForwardSharedMemoryManager::input_size(
```

### 评论原文

#### 【🔵 minor】【⚡ perf】

#### 最终容量守卫对同一 dp 输入重复计算 dp_local_tp_size_ 次 input_size

---

**🔎 问题**

> execute_batch 的最终容量守卫循环（569-575 行）遍历全部 worker_clients_num_ 个 rank，每次调用 ForwardSharedMemoryManager::input_size(forward_inputs[rank / dp_local_tp_size_])。同一个 dp 组的 ForwardInput 是同一份对象，其完整布局尺寸（遍历所有 tensor 的 numel*dtype 与序列化开销）被重复计算 dp_local_tp_size_ 次；该守卫在常规 decode 热路径每步执行。step() 中 limits 向量（431-435 行附近）已经按 dp 维度构建并复用，此处未采用同样的去重。

---

**🔧 修改建议**

按 dp 维度计算：循环 dp_size_ 次（或对 forward_inputs 的去重序列）调用 input_size 并与对应 channel 的 limit 比较，同一 dp 组内所有 rank 复用该结果，与 step() 里 limits 向量的构建方式一致。

## R09 — 冯艳 (fengyan.119)｜2026/10/30 17:39:10｜xllm/core/distributed_runtime/vlm_engine.cpp:603｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3487215`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487215)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。
- 发布关系：与 R07 正文相同，属于不同时间和锚点的独立评论；不据其中一条的状态推断另一条。

### 页面代码上下文（锚点附近节选）

```diff
+  }
+  auto results = std::move(completion).value();
+  for (const auto& result : results) {
+    if (result.hasException() || !result.value().has_value()) {
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### execute_batch 结果预检循环使分发循环的 else 分支不可达

---

**🔎 问题**

> execute_batch 在 602-606 行新增的结果预检循环先对所有 futures 检查 hasException()/!has_value() 并 abort_execution，随后 610 行起的分发循环里保留了既有的 else { LOG(FATAL) << "Failed to execute model, result has no value"; }（638-640 行）。预检已保证进入分发循环时每个结果非空且无异常，该 else 分支永远不会执行，属于预检引入后遗留的死分支。同一 MR 中 update_last_step_result（672-676 预检 + 678-682 无条件 emplace）已经按去掉 else 的方式修复过完全相同的模式，此处漏改。

---

**🔧 修改建议**

删除分发循环中对 result 的 if/else 判断，直接以 result.value() 消费结果（与 update_last_step_result 678-682 行的写法一致，例如直接调用 batch[dp_rank].process_sample_output(result.value(), ...)），去掉 638-640 行的 else 死分支。

## R10 — 冯艳 (fengyan.119)｜2026/10/30 17:39:10｜xllm/core/distributed_runtime/vlm_engine.cpp:570｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / perf。
- 线程锚点：[`discussion_r3487216`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487216)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。
- 发布关系：与 R08 正文相同，属于不同时间和锚点的独立评论；不据其中一条的状态推断另一条。

### 页面代码上下文（锚点附近节选）

```diff
+  // Last guard before any rank can observe an input. A mismatch with the
+  // preview is an invariant failure, not a reason to switch transports.
+  for (uint32_t rank = 0; rank < worker_clients_num_; ++rank) {
+    if (ForwardSharedMemoryManager::input_size(
```

### 评论原文

#### 【🔵 minor】【⚡ perf】

#### 最终容量守卫对同一 dp 输入重复计算 dp_local_tp_size_ 次 input_size

---

**🔎 问题**

> execute_batch 的最终容量守卫循环（569-575 行）遍历全部 worker_clients_num_ 个 rank，每次调用 ForwardSharedMemoryManager::input_size(forward_inputs[rank / dp_local_tp_size_])。同一个 dp 组的 ForwardInput 是同一份对象，其完整布局尺寸（遍历所有 tensor 的 numel*dtype 与序列化开销）被重复计算 dp_local_tp_size_ 次；该守卫在常规 decode 热路径每步执行。step() 中 limits 向量（431-435 行附近）已经按 dp 维度构建并复用，此处未采用同样的去重。

---

**🔧 修改建议**

按 dp 维度计算：循环 dp_size_ 次（或对 forward_inputs 的去重序列）调用 input_size 并与对应 channel 的 limit 比较，同一 dp 组内所有 rank 复用该结果，与 step() 里 limits 向量的构建方式一致。

## R11 — 冯艳 (fengyan.119)｜2026/10/30 17:39:11｜xllm/core/framework/batch/batch_input_builder.cpp:1608｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / perf。
- 线程锚点：[`discussion_r3487217`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487217)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
-  MMData& mm_data = sequence->mutable_mm_data();
+  // The scheduling visitor updates item indices and ranges. Stage those
+  // updates on a shallow metadata copy, sharing the original tensor storage.
+  MMData mm_data = sequence->mm_data();
```

### 评论原文

#### 【🔵 minor】【⚡ perf】

#### DECODE 步在判定 stage 前无条件拷贝整份 MMData

---

**🔎 问题**

> process_multi_modal_inputs 现在先执行 `MMData mm_data = sequence->mm_data();`（1608 行），再在 1609 行用 `sequence->stage() != SequenceStage::DECODE` 决定是否使用它。改动前（old 1531 行）这里取的是 `MMData& mm_data = sequence->mutable_mm_data();` 引用，只在需要时才对序列的 mm_data 做原地更新，不产生拷贝。MMData 持有 `std::variant<MMItemVec, MMDict>`（mm_data.h:59、187），拷贝会复制 variant 内的 vector/unordered_map 结构（每个 MMDataItem/MMValue 还会各自复制其张量与元数据成员）。该函数由 process_sequences 对每个序列逐个调用（884-885 行），走的是默认 decode 热路径；纯文本 decode 时 `mm_data.valid()` 为 false，这次拷贝完全无用，每步每序列都要付一次堆分配与整表复制的代价。

---

**🔧 修改建议**

把拷贝移入需要它的分支，decode 路径保持零成本：先判 stage，再在分支内构造拷贝，例如

```cpp
if (sequence->stage() != SequenceStage::DECODE && sequence->mm_data().valid()) {
  MMData mm_data = sequence->mm_data();
  ...
}
```

或先取 `const MMData& src = sequence->mm_data();` 做 stage/valid 判定，仅在进入分支后拷贝一次。

## R12 — 冯艳 (fengyan.119)｜2026/10/30 17:39:11｜xllm/core/framework/batch/batch.cpp:494｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3487218`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487218)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
-                                          int32_t cp_size) {
+                                          int32_t cp_size,
+                                          bool commit_sequence_state) {
+  if (!commit_sequence_state) {
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### commit_sequence_state=false 的预览分支无生产调用者，且与 candidate 路径重复

---

**🔎 问题**

> Batch::prepare_forward_input 新增的 `bool commit_sequence_state` 参数只在 494-499 行分支里起作用：置 false 时值拷贝一个 Batch，走 prepare_forward_input_candidate 后直接返回 `std::move(prepared.input())`，把准备好的 PreparedForwardInput 连同其 sequence_updates_ 一起丢弃（预览阶段的延迟更新永远不会提交），得到的是 meta 占位输入。生产路径全部使用默认 true：llm_engine.cpp:1430 与 vlm_executor_impl 都不传该参数；唯一传 false 的是 batch_transport_test.cpp（68/198/224/293）。也就是这条预览分支只在测试里被使用，功能上等价于 prepare_forward_input_candidate().input()，等于同一能力维护了两套入口，且误用时不会报错、只会静默丢弃 sequence 更新。

---

**🔧 修改建议**

删除 Batch::prepare_forward_input 的 commit_sequence_state 参数与 494-499 行的预览分支（batch.h:100-103 同步去掉该形参），预览语义统一由 prepare_forward_input_candidate() + commit_forward_input() 承担；测试侧的预览调用改成 `batch.prepare_forward_input_candidate(args, nullptr).input()`。注意 BatchInputBuilder 自身的 commit_sequence_state 参数仍需保留，prepare_forward_input_candidate（batch.cpp:541）正是靠它构造预览。
