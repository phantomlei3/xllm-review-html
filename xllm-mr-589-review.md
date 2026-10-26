# xLLM MR #589 — Review 意见与修复交接

- 来源：[MR #589](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/589)
- 标题：`bugfix: fix linear state allocation and shared kpool budgets.`
- 分支：`bugfix/fix-linear-state` → `main`
- 页面最后展示的提交：`eb9aa7c56ed6e43dc233c40d7471dc132fdb63a2`；另一个提交为 `e691039d81282a46ba5870e49dae1ebbde99e8ff`。未另查仓库确认 head。
- 页面状态：Open / Reviewing；2 commits、8 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。
- 提取时间：2026-10-26 11:00:38（Asia/Shanghai）；已登录 Chrome 刷新后读取并展开作者说明。提交前于 2026-10-26 11:01:24 再次刷新，核对作者说明及 5 条行级意见正文、时间和状态未变。
- 筛选：不按作者、时间、解决状态过滤；包含作者说明、全部已发布行级意见及回复，不含提交和 reviewed 事件。
- 清单：6 条根评论（1 条作者说明、5 条行级意见），0 条回复；页面 Conversation 5 不作为评论总数。
- 范围：已检查会话讨论及其包含的评论正文，未见隐藏同类评论、过期线程、分页或加载更多入口。未另外检查 Files Changed 中未在会话展示的讨论，不含草稿。
- 状态说明：5 条行级讨论均仅见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件，逐条标为“状态未核实”；作者说明未见解决状态标记。
- 原文说明：作者和评审者的技术判断、测试声明及数据未经导出者独立验证。R06 引用的 head 为 `e691039`，与页面最后展示提交 `eb9aa7c` 不同，原样保留。R05 赞成保留兼容 wrapper，R06 建议删除，仅记录不同意见。R02 关于负数转换安全性的说法及 R05 结尾“一切看起来都很好。准备输出”均为原文，未作修正。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：LINEAR 分配失败、共享 KPool 预算修复及八卡验证 | 未见解决状态标记 |
| R02 | 正确性：认可 SWA 下溢修复，建议确认参数类型和负数校验 | 状态未核实 |
| R03 | 正确性：建议解释 draft 继承 target 容量属性的原因 | 状态未核实 |
| R04 | 正确性：建议确认所有调用路径正确处理 std::nullopt | 状态未核实 |
| R05 | 测试缺口：建议为硬编码预期值添加计算注释，附整体检查结论 | 状态未核实 |
| R06 | minor / dead-code：建议删除仅供测试调用的 estimate_shared_kpool_blocks | 状态未核实 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月26日 10:08:25｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

#### Description

修复 GLM-5.3-Flash 在 MLU 上启用 MTP 和 graph 时的两处缓存正确性问题：必需 LINEAR 状态槽分配失败被误判为成功，以及同设备 target/draft KPool 联合预算与实际分配参数不一致。

修复后，现有 warmup 容量探测能够识别真实可分配序列数，target/draft 使用同一份最终容量计划完成预算计算和缓存分配。

#### 故障现象

在以下配置下部署 GLM-5.3-Flash-W4A8：

- 8 卡，ep_size=8、cp_size=8。

- 启用 MTP，num_speculative_tokens=3。

- 启用 graph、chunked prefill 和 schedule overlap。

- max_seqs_per_batch=32、max_memory_utilization=0.9。

主模型自动规划出 29 个物理状态槽，扣除 padding 后实际最多可分配 28 个基本请求状态。但 graph warmup 的容量探测错误地报告可分配 32 个序列，随后在构造输入时触发：

```text
BatchInputBuilder::append_linear_state_row()
Check failed: !linear_blocks.empty()
```

同时，draft KPool tail 分配了 1,543,178 个状态槽，占用约 8.83 GiB，与联合预算采用的状态槽数量不一致。

#### 根因一：必需状态槽分配失败返回了成功语义

LinearStateBlockManager::allocate_prefill() 在申请状态槽失败时返回空 vector。

在复合分配协议中：

- std::nullopt 表示分配失败。

- 空 vector 表示操作成功，但没有新增资源。

因此，当新请求尚无 LINEAR 状态槽且申请失败时，复合管理器仍然报告分配成功。现有完整性检查又将 LINEAR 作为请求级资源单独处理，没有拦截缺失状态的请求。

这导致容量探测将缺少必需状态的请求计入成功数量，最终在构造模型输入时崩溃。

#### 根因二：联合预算假设没有落实到实际分配参数

原有 estimate_shared_kpool_blocks() 按 target 的请求槽数和验证窗口计算 draft 状态开销，但只返回公共 KV block 数。

调用方仅同步 target/draft 的 n_blocks，没有同步 draft 的状态槽数量、tail 窗口和派生字节数。因此，实际创建 KVCacheShape 时仍使用 draft 独立估算的容量，造成预算与实际分配不一致。

#### 修复方式

- 正确传播必需 LINEAR 状态槽分配失败。

当请求尚无 LINEAR 状态槽且申请失败时，返回 std::nullopt，由现有复合管理器回滚已暂存资源并向调用方报告失败。

保留已有状态可以继续使用时的空 vector 语义，使可选状态分配失败不会错误地阻止请求继续执行。

- 使用完整的 target/draft 联合容量计划。

新增 SharedKPoolPlan 和 plan_shared_kpool()，在同设备 KPool MTP 路径中统一产出两份最终 capacity：

- draft 继承 target 的物理请求槽数量。

- draft tail 窗口覆盖 target 验证窗口及自身 index_kpool 要求。

- 保留 draft 自身的层数、head 维度和每 block 开销。

- 根据最终布局重新计算 draft 单槽字节数和状态总字节数。

- 扣除双方状态开销后，计算公共 KV block 数。

- 实际分配直接使用返回的两份 capacity。

原有仅返回 block 数的接口复用联合规划，保持既有调用方式可用。

#### 八卡端到端验证

使用上述 GLM-5.3-Flash-W4A8 部署配置完成验证：

| 检查项 | 修复前 | 修复后 |
| --- | --- | --- |
| 主模型物理状态槽 | 29 | 29 |
| warmup 探测可分配序列数 | 错误报告 32 | 正确报告 28 |
| draft 状态槽 | 1,543,178 | 29 |
| draft 状态预留 | 约 8.83 GiB | 174 KiB |
| draft tail shape | [1543178 2 12 128] | [29 2 12 128] |
| graph warmup | 触发状态槽断言 | 9 个 bucket 全部完成 |

## R02 — 邓英旭 (dengyingxu1)｜2026/10/26 10:13:44｜xllm/core/framework/block/sliding_window_block_manager.cpp:118-119｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477370`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
  CHECK_EQ(compressed_unit_tokens % block_size, 0);
  const size_t blocks_per_unit =
      static_cast<size_t>(compressed_unit_tokens / block_size);
```

### 评论原文

**1. 正确性 — 修复滑动窗口块管理器中的无符号整数下溢问题**
位置：`xllm/core/framework/block/sliding_window_block_manager.cpp:118-119`

原代码中 `valid_per_unit` 直接取 `options_.swa_blocks_per_seq()`，当该值大于 `blocks_per_unit` 时，后续的 `blocks_per_unit - valid_per_unit` 会导致无符号整数下溢，产生一个极大的错误值。

建议当前的 `std::min` 修复非常正确且必要。建议确认 `options_.swa_blocks_per_seq()` 的返回值类型，如果是有符号类型，`static_cast<size_t>` 处理负数也是安全的；如果可能为负数且业务逻辑上不应为负，可以考虑在 `Options` 的设置处增加断言或校验。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/26 10:13:44｜xllm/core/framework/kv_cache/kv_cache_estimation.h:118｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477371`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+SharedKPoolPlan plan_shared_kpool(KVCacheCapacity target_cap,
+                                  KVCacheCapacity draft_cap,
+                                  const ModelArgs& draft_args);
```

### 评论原文

**2. 正确性 — 优化共享 KPool 预算规划逻辑**
位置：`xllm/core/framework/kv_cache/kv_cache_estimation.h:118`

将 `estimate_shared_kpool_blocks` 重构为 `plan_shared_kpool` 并返回 `SharedKPoolPlan` 结构体，使得 target 和 draft 的容量规划更加内聚，避免了调用方需要手动修改容量对象。逻辑清晰，减少了不一致的风险。

建议在 `plan_shared_kpool` 函数内部，对 `draft_cap` 的属性修改（如 `num_linear_state_blocks`, `kpool_tail_len` 等）基于 `target_cap` 的继承逻辑，建议添加简短注释说明为何 draft 需要继承 target 的这些属性（例如：“Draft 引擎的请求槽位和验证窗口继承自 Target 引擎”），以提高代码可读性。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/26 10:13:44｜xllm/core/framework/block/linear_state_block_manager.cpp:230-232｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477372`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
  std::vector<Block> allocated = BlockManagerImpl::allocate(1);
  if (allocated.empty()) {
```

### 评论原文

**3. 正确性 — 修复线性状态块分配失败时的重试逻辑**
位置：`xllm/core/framework/block/linear_state_block_manager.cpp:230-232`

当底层 `BlockManagerImpl::allocate(1)` 返回空时，如果当前已分配的 `blocks` 也为空，原代码返回空 `vector`，导致上层无法区分“完全无内存可重试”和“部分分配失败”。修改为返回 `std::nullopt` 后，上层可以据此决定是否重试。

建议逻辑正确。建议确认所有调用该分配方法的代码路径都能正确处理 `std::nullopt` 返回值，确保重试机制按预期工作。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/26 10:13:44｜tests/core/framework/kv_cache/kv_cache_test.cpp:1690｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477373`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+          std::move(target_cap), std::move(draft_cap), draft_args);
+      target_cap = std::move(plan.target);
+      draft_cap = std::move(plan.draft);
+      const int64_t n_blocks = target_cap.n_blocks();
```

### 评论原文

**4. 测试缺口 — 完善相关测试覆盖**
位置：`tests/core/framework/kv_cache/kv_cache_test.cpp:1690`

针对上述 bugfix 和 refactor，新增和修改了相应的测试用例。特别是 `SharedKPoolPlanMatchesAllocatedStateAndBudget` 测试不仅验证了计划的一致性，还验证了实际分配内存不超过预算，非常严谨。

建议测试用例质量很高。在 `SharedKPoolPlanMatchesAllocatedStateAndBudget` 中使用了硬编码的期望值（如 `63488`, `58496`），如果未来底层张量布局或数据类型大小发生变化，可能需要同步更新。建议在硬编码值旁添加计算过程的注释，便于后续维护。

检查是否有遗漏：

- `speculative_engine.cpp` 中移除了 `estimate_kv_cache_blocks` 中的 if 块，因为现在在 `allocate_kv_cache` 中提前处理了。这是合理的，因为 `estimate_kv_cache_blocks` 现在只负责估算，不再负责修改容量。

- `kv_cache_estimation.cpp` 中 `estimate_shared_kpool_blocks` 仍然保留，作为 `plan_shared_kpool` 的 wrapper，保持了向后兼容。

一切看起来都很好。准备输出

## R06 — 冯艳 (fengyan.119)｜2026/10/26 10:49:10｜xllm/core/framework/kv_cache/kv_cache_estimation.h:109｜状态未核实

- 类型：行级意见；根评论，0 条回复。严重性：minor；类别：dead-code。
- 线程锚点：`discussion_r3477433`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
-// Full-attention KPool drafts inherit the target's request-state slots/window.
-// Both capacities retain their own model's layer counts and page costs.
 int64_t estimate_shared_kpool_blocks(const KVCacheCapacity& target_cap,
```

### 评论原文

#### 【🔵 minor】【🧹 dead-code】

#### estimate_shared_kpool_blocks 在生产侧已无调用者，仅剩测试引用，应删除并将测试迁移到 plan_shared_kpool

---

**🔎 问题**

> 本次 diff 把 speculative_engine.cpp 中的共享 KPool 块数计算改为直接调用 plan_shared_kpool（xllm/core/distributed_runtime/speculative_engine.cpp:221），并在 xllm/core/framework/kv_cache/kv_cache_estimation.cpp:916-920 把 estimate_shared_kpool_blocks 退化为一行转发 plan_shared_kpool(target_cap, draft_cap, draft_args).target.n_blocks()。全仓搜索（head e691039）显示该符号的生产调用者为零，仅剩 tests/core/framework/kv_cache/kv_cache_test.cpp:1733/1736 两处 EXPECT_DEATH 断言。项目规则明确：生产 API 只剩测试消费者时要删除该 API 并让测试直接消费实际生产入口。

---

**🔧 修改建议**

删除 kv_cache_estimation.h:109-111 的声明和 kv_cache_estimation.cpp:916-920 的定义；SharedKPoolBudgetRejectsInsufficientMemory 测试改用 plan_shared_kpool(capacity, capacity, draft_args).target.n_blocks() 触发同样的 CHECK 死亡断言（两条死亡消息均来自 plan_shared_kpool 内部的 CHECK_GT(budget, state_bytes) 和 CHECK_GT(n_blocks, 0)，语义不变）。
