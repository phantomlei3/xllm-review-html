# xLLM MR #636 — Review 意见与修复交接

- 来源：[MR #636](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636)
- 标题：`bugfix: reject oversized vlm inputs before worker dispatch.`
- 分支：`bugfix/vlm_oversize` → `main`
- 页面提交列表首项：`1ba509d230c3a878d42d9defb7f61c0f5c915716`；另一个提交为 `e34a33a3f081cbaf67613f9842b4d70f7f74a1a5`。R14 也将首项标为 head，未另查仓库确认。
- 页面状态：Open / Reviewing；2 commits、24 files changed；要求 2 人批准，3 项状态检查通过、1 项失败，尚未满足合并要求。失败项为 xLLM服务CI流水线 #9271455。
- 提取时间：2026-10-30 14:38:49（Asia/Shanghai）；通过已登录 Chrome 刷新页面、等待讨论加载后读取。提交前于 2026-10-30 14:39:46 再次刷新，核对 14 条根评论的正文、时间、锚点及讨论控件未变。
- 筛选：不按评审者、时间或解决状态过滤；包含作者说明、全部已发布行级意见及独立评审小结，不含提交或无评论正文的 reviewed 事件，不含草稿。
- 清单：14 条根评论（1 条作者说明、12 条行级意见、1 条 Review 小结），0 条回复；页面 Conversation 13 不作为根评论总数。
- 范围：已逐条核对会话中的讨论正文和回复位置，未见隐藏同类评论、过期线程、分页或加载更多讨论入口。另打开 Files Changed 核对为 24 个变更文件；该视图存在按视口加载及 Load Diff 入口，未逐一加载全部文件 diff，未独立验证是否存在会话外讨论。
- 状态说明：12 条行级讨论均只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件，逐条标为“状态未核实”；作者说明和独立小结未见解决状态标记。
- 原文说明：评审者的技术判断、严重性及测试声明未经导出者独立验证。R05 原文以未闭合的反引号和 `CHECK` 结束，未补全。R14 称“已有未解决 P2 讨论”，但当前可见的 12 条行级意见未明确标注 P2，且无法从各线程确认解决状态；保留该小结，不据此替各线程定级或判定状态。部分行级锚点位于 include 或函数签名，与评论讨论的实现位置不同，以下同时保留页面锚点及原文位置。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：VLM 超大输入预检、batch 拆分、取消传播与执行超时 | 未见解决状态标记 |
| R02 | 行级意见：正确性：预览阶段可能修改 Batch / Sequence 状态 | 状态未核实 |
| R03 | 行级意见：效率：预览仍构建 engram_history | 状态未核实 |
| R04 | 行级意见：健壮性：ControlMetadata 的原子操作与共享内存布局 | 状态未核实 |
| R05 | 行级意见：正确性：执行超时后的 promise / RPC 竞态 | 状态未核实 |
| R06 | 行级意见：important / perf：预估与执行重复构建 ForwardInput | 状态未核实 |
| R07 | 行级意见：important / correctness：ControlMetadata 扩容未升级 SHM 布局版本 | 状态未核实 |
| R08 | 行级意见：minor / dead-code：LOG(FATAL) 后的 std::abort() | 状态未核实 |
| R09 | 行级意见：minor / dead-code：update_last_step_result 重复校验 | 状态未核实 |
| R10 | 行级意见：minor / style：raw_output_read 超时默认值重复 | 状态未核实 |
| R11 | 行级意见：minor / license：两个新测试文件的 All Rights Reserved | 状态未核实 |
| R12 | 行级意见：minor / perf：forward_input_limit 重复读取和解析 gflags | 状态未核实 |
| R13 | 行级意见：minor / dead-code：split_by_request 的 mm_data_vec_ 守卫 | 状态未核实 |
| R14 | Review 小结：修复后再合并；声称已有未解决 P2 讨论 | 未见解决状态标记 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月29日 15:32:03｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

#### 修复 VLM 超大输入传输失败和分布式执行无限等待

当图片、视频等多模态输入超过共享内存或 RPC 传输容量时，原有流程可能出现传输失败、worker 执行状态不一致或请求持续等待。本 PR 在下发前检查输入大小，并完善通信失败和执行超时的处理。

- 下发前预估序列化输入大小，不推进序列状态、不复制多模态张量数据。

- 未开启调度重叠时，按请求拆分超限 batch；单个请求仍超限则返回明确错误。开启调度重叠时，直接拒绝超限 batch。

- 移除共享内存写入失败后单方面切换 RPC 的逻辑，明确传播通信失败并触发取消。

- 新增 forward_execution_timeout_ms，默认 300 秒，覆盖执行 RPC、共享内存输出和 VLM worker 结果等待。执行失败或超时后标记服务不健康并终止进程，避免继续使用不一致的状态。

新增回归测试覆盖容量边界、输入预估与序列状态保持、取消传播、RPC 失败和输出等待超时。

## R02 — 邓英旭 (dengyingxu1)｜2026/10/29 16:32:23｜xllm/core/distributed_runtime/vlm_engine.cpp:37｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484770`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3484770)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、函数签名或前一函数结尾，不能据此定位评论所述实现；未补造实现代码。

### 页面代码上下文（锚点附近节选）

```diff
 #include "common/interruption_bus.h"
 #include "common/metrics.h"
 #include "core/common/global_flags.h"
+#include "core/common/health_check_manager.h"
```

### 评论原文

**1. 正确性 — 预览阶段的潜在状态修改**
位置：`xllm/core/distributed_runtime/vlm_engine.cpp:37`

`forward_input_sizes` 函数 调用 `prepare_inputs` 时虽然传入了 `commit_sequence_state=false`，但 `prepare_inputs` 内部会调用 `dp_balance_shuffle_seqs()` 和 `refresh_output_targets()`。这些操作可能会修改 `Batch` 或 `Sequence` 的内部状态，导致预览产生副作用，影响后续的 `execute_batch`。

建议确认这些操作是幂等且无破坏性副作用的。如果它们会改变状态，需要将它们也置于 `commit_sequence_state` 的控制之下，或在预览时使用深拷贝。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/29 16:32:24｜xllm/core/framework/batch/batch_input_builder.cpp:216｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484771`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3484771)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、函数签名或前一函数结尾，不能据此定位评论所述实现；未补造实现代码。

### 页面代码上下文（锚点附近节选）

```diff
     int32_t cp_size,
-    ThreadPool* thread_pool)
-    : sequences_(sequences),
+    ThreadPool* thread_pool,
```

### 评论原文

**2. 效率 — 预览阶段的多余计算开销**
位置：`xllm/core/framework/batch/batch_input_builder.cpp:216`

`process_sequences` 中的 `engram_history` 构建 即使 `commit_sequence_state=false`，`engram_history` 的构建逻辑（`emplace_back`）仍然会执行，只是最后没有调用 `mark_engram_history_restored()`。对于极大的 batch，这会带来不必要的内存和 CPU 开销。

建议如果预览仅为了计算大小，可以在 `commit_sequence_state=false` 时直接跳过 `engram_history` 的构建逻辑。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/29 16:32:24｜xllm/core/runtime/forward_shared_memory_manager.cpp:3527｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484772`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3484772)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、函数签名或前一函数结尾，不能据此定位评论所述实现；未补造实现代码。

### 页面代码上下文（锚点附近节选）

```diff
   return filename;
 }

+uint64_t ForwardSharedMemoryManager::input_size(const ForwardInput& input) {
```

### 评论原文

**3. 健壮性 — 共享内存结构体的原子操作与布局**
位置：`xllm/core/runtime/forward_shared_memory_manager.cpp:3527`

`cancel()` 和 `cancelled()` `ControlMetadata` 中的 `cancelled` 字段使用 `__atomic_store_n` 和 `__atomic_load_n`。在共享内存中，不同进程的结构体布局必须严格一致。此外，裸用 `__atomic` 内建函数不如 `std::atomic` 安全。

建议确保 `ControlMetadata` 具有明确的内存布局（如静态断言大小和偏移）。考虑使用 `std::atomic<uint64_t>` 替代裸原子操作（前提是 C++ 实现保证其在共享内存中是锁-free的）。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/29 16:32:24｜xllm/core/distributed_runtime/vlm_engine.cpp:45｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484773`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3484773)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。
- 位置说明：页面锚点附近仅显示 include、函数签名或前一函数结尾，不能据此定位评论所述实现；未补造实现代码。

### 页面代码上下文（锚点附近节选）

```diff
 #include "core/framework/config/scheduler_config.h"
 #include "core/framework/config/service_config.h"
 #include "core/framework/prefix_cache/block_hasher.h"
+#include "core/runtime/forward_shared_memory_manager.h"
```

> 导出注：下方建议在页面中以未闭合的反引号和 CHECK 结束，原样保留，未猜测缺失内容。

### 评论原文

**4. 正确性 — 超时后的竞态条件**
位置：`xllm/core/distributed_runtime/vlm_engine.cpp:45`

`execute_batch` 中的 `within` 超时处理 `folly::collectAll` 超时后，底层的 promise 可能仍在运行。虽然调用了 `abort_execution` 和 `cancel_execution()`，但 RPC 调用可能尚未终止。如果 worker 随后处理新请求，可能产生竞态。

建议确保 cancel 机制能彻底停止 worker 的当前计算，或确保 worker 在处理下一个请求前会检查 cancel 标志并拒绝服务（如 `worker_service.cpp` 中的 `CHECK

## R06 — 冯艳 (fengyan.119)｜2026/10/29 17:44:57｜xllm/core/distributed_runtime/vlm_engine.cpp:438｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：important / perf。
- 线程锚点：[`discussion_r3485044`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485044)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+    limits[dp_rank] =
+        std::min(limits[dp_rank], worker_clients_[rank]->forward_input_limit());
+  }
+  const auto sizes = forward_input_sizes(batch);
```

### 评论原文

#### 【🟠 important】【⚡ perf】

#### 每步为预估 size 多做一次全量 ForwardInput 构建

---

**🔎 问题**

> step() 在 438 行无条件调用 forward_input_sizes(batch)，经 forward_input_sizes -> prepare_inputs(batch, commit_sequence_state=false) 完整走一遍 Batch::prepare_forward_input/BatchInputBuilder::build_forward_input，只为拿到每 rank 字节数；紧接着 444 行 execute_batch 在 546 行又执行一次 prepare_inputs 做同样全量构建。fits 为真的常规路径因此把 host 侧输入构建、tensor 拷贝与序列化布局计算做了两遍，等于在 decode 热路径上固定翻倍这部分 CPU 开销。

---

**🔧 修改建议**

让 size 检查复用同一次构建结果：execute_batch 内先 prepare_inputs 得到 forward_inputs，再用 ForwardSharedMemoryManager::input_size 判定是否超限；未超限直接下发，超限时才走 split 分支对分片 batch 重新构建。若必须保留下发前预判，则把 forward_input_sizes 改为只累加布局尺寸的轻量路径（不 materialize tensor），并让 execute_batch 复用该布局结果而非重建 ForwardInput。

## R07 — 冯艳 (fengyan.119)｜2026/10/29 17:44:57｜xllm/core/runtime/forward_shared_memory_manager.h:34｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：important / correctness。
- 线程锚点：[`discussion_r3485045`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485045)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff

 struct ControlMetadata {
   volatile uint64_t version;
+  uint64_t cancelled;
```

### 评论原文

#### 【🟠 important】【✅ correctness】

#### ControlMetadata 新增 cancelled 改变 SHM 载荷布局，但未提升布局版本号

---

**🔎 问题**

> ControlMetadata 由 8 字节扩到 16 字节，构造函数里 metadata_addr_ = base_address() + sizeof(ControlMetadata)，所有 pb/raw 载荷与 payload_size 字段整体后移 8 字节。而 SharedMemoryManager 只校验 SharedMemoryLayoutHeader 的 magic/version/header_size/payload_size：payload_size 用的是构造参数 size（input_shm_size 配置值，未变），kSharedMemoryLayoutVersion 仍是 2，所以新旧布局在同一份 SHM 上会被判定为兼容，payload 偏移错位无法被任何机制检出。

---

**🔧 修改建议**

把布局版本号一起提升：在 xllm/core/util/shared_memory_manager.cpp 将 kSharedMemoryLayoutVersion 从 2 改为 3（配合既有 magic/version CHECK，让旧对象在 attach 时直接失败并提示清理），使 ControlMetadata 的扩容成为可检出的不兼容变更。若希望优先保持兼容，则把 cancelled 移出载荷前缀（放进 SharedMemoryLayoutHeader 的 reserved 区或独立 control SHM），使 metadata_addr_ 偏移保持不变。

## R08 — 冯艳 (fengyan.119)｜2026/10/29 17:44:58｜xllm/core/distributed_runtime/vlm_engine.cpp:542｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3485046`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485046)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+  }
+  LOG(FATAL) << reason
+             << "; worker state is not recoverable, restart the worker group";
+  std::abort();
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### abort_execution 中 LOG(FATAL) 之后的 std::abort() 不可达

---

**🔎 问题**

> VLMEngine::abort_execution（vlm_engine.cpp:535-543）在 LOG(FATAL) 之后跟了一行 std::abort()。glog 的 LOG(FATAL) 是 noreturn 语义（打印后自身终止进程），因此 std::abort() 永远不会执行，是死代码；函数已声明 [[noreturn]]，不需要额外的兜底调用。

---

**🔧 修改建议**

删除 vlm_engine.cpp:542 的 std::abort() 行，保留 LOG(FATAL) 作为终止点。

## R09 — 冯艳 (fengyan.119)｜2026/10/29 17:44:59｜xllm/core/distributed_runtime/vlm_engine.cpp:664｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3485047`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485047)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
     if (result.has_value()) {
       raw_forward_outputs.emplace_back(std::move(result.value()));
     } else {
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### update_last_step_result 第二轮校验的 else 分支不可达

---

**🔎 问题**

> update_last_step_result（vlm_engine.cpp:653-667）先用一个循环对 last_step_results 逐项检查 hasException()/!has_value() 并 abort_execution（653-657），随后 659-667 的分发循环又对同样的条件写了 else { abort_execution("Failed to get last step results"); }。第一轮已保证所有结果非空且无异常，第二个 abort 分支永远不会执行，属于重复校验加死分支。

---

**🔧 修改建议**

删掉 661-666 的 has_value()/else 判断，直接 raw_forward_outputs.emplace_back(std::move(result.value()))；或去掉 653-657 的第一轮预检循环，只保留分发处的一次校验。二者取其一。

## R10 — 冯艳 (fengyan.119)｜2026/10/29 17:44:59｜xllm/core/runtime/forward_shared_memory_manager.h:145｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / style。
- 线程锚点：[`discussion_r3485048`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485048)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
-  void raw_output_read(RawForwardOutput& outputs);
+  bool raw_output_read(
+      RawForwardOutput& outputs,
+      std::chrono::milliseconds timeout = std::chrono::milliseconds(300000),
```

### 评论原文

#### 【🔵 minor】【🎨 style】

#### raw_output_read 默认参数硬编码 300000ms，与 ExecutionConfig 默认值重复

---

**🔎 问题**

> ForwardSharedMemoryManager::raw_output_read 的头文件默认参数 std::chrono::milliseconds(300000)（forward_shared_memory_manager.h:145）与 ExecutionConfig::forward_execution_timeout_ms 的 PROPERTY 默认 300000（execution_config.h:88）是两份独立的 300000。生产调用方（shm_channel.cpp 的 raw_output_read 路径）都显式传入 config 值，该默认值只在测试里生效；将来调整 config 默认时这个隐式副本不会被一起改，容易产生测试与生产行为漂移。

---

**🔧 修改建议**

去掉默认参数，要求调用方显式传入超时（生产路径已全部显式传 config 值）；测试里显式写 std::chrono::milliseconds 或读取 ExecutionConfig。单一来源保持在 ExecutionConfig。

## R11 — 冯艳 (fengyan.119)｜2026/10/29 17:45:00｜tests/core/framework/batch/batch_transport_test.cpp:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / license。
- 线程锚点：[`discussion_r3485049`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485049)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

### 评论原文

#### 【🔵 minor】【📄 license】

#### 新增测试文件版权头包含 All Rights Reserved，违反项目代码风格

---

**🔎 问题**

> 本 MR 新增的两个测试文件 tests/core/framework/batch/batch_transport_test.cpp:1 与 tests/core/distributed_runtime/comm_channel_failure_test.cpp:1 的版权头均为 "/* Copyright 2026 The xLLM Authors. All Rights Reserved."。项目 custom-code-style 第 2 条规定不得添加 All Rights Reserved。两个文件的版权行都在本 diff 内，属于新增内容。

---

**🔧 修改建议**

将两个新测试文件首行改为 "/* Copyright 2026 The xLLM Authors."，去掉 " All Rights Reserved."。

## R12 — 冯艳 (fengyan.119)｜2026/10/29 17:45:00｜xllm/core/distributed_runtime/comm_channel.cpp:34｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / perf。
- 线程锚点：[`discussion_r3485050`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485050)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+uint64_t CommChannel::forward_input_limit() const {
+  std::string value;
+  CHECK(gflags::GetCommandLineOption("max_body_size", &value));
+  const uint64_t max_body_size = std::stoull(value);
```

### 评论原文

#### 【🔵 minor】【⚡ perf】

#### forward_input_limit 每次调用都查 gflags 并解析字符串

---

**🔎 问题**

> forward_input_limit() 每次调用都执行 gflags::GetCommandLineOption("max_body_size", &value) 加一次 std::string 构造和 std::stoull 解析，但 max_body_size 是进程级常量。VLMEngine::step 在 436 行对每个 rank 调一次、execute_batch 在 553 行再对每个 rank 调一次，即每 step 有 2*worker_clients_num 次字符串解析；同时 CHECK(GetCommandLineOption(...)) 把 brpc 的 flag 名硬编码成容量契约，flag 缺失时会直接 abort 而不是回落到一个明确容量。

---

**🔧 修改建议**

在 CommChannel 初始化（init_brpc 或构造）时解析一次并缓存为成员（如 uint64_t forward_input_limit_），forward_input_limit() 直接返回缓存值；把 CHECK(gflags::GetCommandLineOption(...)) 改为带回退的读取（取不到时用明确的默认容量并 LOG(WARNING)），避免把 flag 注册情况变成进程级断言。

## R13 — 冯艳 (fengyan.119)｜2026/10/29 17:45:01｜xllm/core/framework/batch/batch.cpp:528｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / dead-code。
- 线程锚点：[`discussion_r3485051`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3485051)；未见 OutDated 标记。
- 状态证据：只见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+  // Block-copy operations may depend on sources belonging to another request.
+  // Such a batch must remain atomic rather than replay its copies per split.
+  if (requires_host_restore_ || !swap_block_transfer_infos_.empty() ||
+      !mm_data_vec_.empty()) {
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### split_by_request 的 mm_data_vec_ 守卫无写入方，恒不成立

---

**🔎 问题**

> split_by_request 用 !mm_data_vec_.empty() 作为“该 batch 不可按请求切分”的条件之一，但全树对 Batch::mm_data_vec_ 只有读取（batch.cpp 277/317/499/528 行与构造入参），没有任何 push_back/emplace/赋值，因此该条件永远为假，多模态 batch 的原子性实际只靠 requires_host_restore_ 与 swap_block_transfer_infos_ 兜住。若本意是“带多模态数据的 batch 不得切分”，当前实现并未生效；若本意是给未来写入留位，则这是一段不可达判断。

---

**🔧 修改建议**

二选一并删除歧义：若多模态 batch 必须保持原子，就从 Sequence 侧汇总（例如遍历 sequences_ 检查 sequence->mm_data().valid()）或补上 mm_data_vec_ 的实际填充点，让该条件真正生效；若不打算支持该约束，直接删掉 `!mm_data_vec_.empty()` 这个分支条件，只保留 requires_host_restore_ 与 swap_block_transfer_infos_ 两项。

## R14 — 邓英旭 (dengyingxu1)｜2026年10月29日 22:02:46｜MR 总体评论｜未见解决状态标记

- 类型：Review 小结；根评论，0 条回复。

### 评论原文

审查范围：MR !636 的两个提交，覆盖 24 个文件，重点检查 VLM 超限预检、batch 拆分、SHM/RPC 取消与执行超时路径，以及对应回归测试。精确 refs：base `bd223f66f356e61db669583caacd2b51f83a316e`，head `1ba509d230c3a878d42d9defb7f61c0f5c915716`，merge-base `47c03115ecaaf15051b1d3186874cd7f6b07e668`。已检查完整 diff、调用链、状态所有权和已有讨论；现有讨论中已覆盖的候选问题未重复发布。`git diff --check` 通过；未运行 C++ 测试或硬件/推理验证，当前环境缺少构建工具链。已有未解决 P2 讨论，影响传输容量边界处理，详见对应 inline 讨论。

合并建议：修复后再合并。
