# xLLM MR #590 — Review 意见与修复交接

- 来源：[MR #590](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/590)
- 标题：`bugfix: remove mlu graph history admission limits.`
- 分支：`bugfix/glm53-graph` → `main`
- 页面提交：`e30be3ad640d830c7d6e88f736405a1f5ec88bb7`、`77f5d51e5c5d95fb1f82424b96b648dd98975dde`（最后展示）；未另查仓库确认 head。
- 页面状态：Open / Reviewing；2 commits、16 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。冯艳 (fengyan.119) 于 2026年10月26日 11:37:18 approved changes，另于 12:21 再次 approved changes；批准事件未附评论正文。
- 更新时间：2026-10-26 17:47:52（Asia/Shanghai）；刷新已登录 Chrome 的 MR #590 页面后提取，展开全部已解决讨论核对线程和回复关系。提交前于 17:48:24 再次刷新，3 条保留行级意见正文及 4 条已解决状态未变。
- 筛选：不按评审者过滤；排除已明确标记 Resolved 的整条讨论及其所有回复。保留作者说明、未见已解决标记的独立小结和其余行级意见。
- 清单：保留 6 条根评论（1 条作者说明、1 条自动审查小结、3 条行级意见、1 条 Review 小结），0 条回复；排除 4 条已解决根评论及其 6 条回复。页面 Conversation 14 不作为根评论总数。
- 范围：已核对会话中全部讨论及回复，未见隐藏同类评论、过期线程、分页或加载更多入口；未另查 Files Changed 中未在会话展示的讨论，不含草稿。
- 状态说明：排除线程均有明确 “This conversation was maked as "Resolved" by 罗磊” 记录；保留的 3 条行级意见仅见处理按钮、未见明确解决记录，状态逐条注明未核实。
- 原文说明：评审者的技术判断、严重性及测试声明未经导出者独立验证。马晓龙的独立 Review 小结仍未见解决标记，原样保留；其引用的 F1/F4 所属讨论已解决并排除，不另重建已排除评论正文。小结称共 5 条 finding，与本次保留 3 条行级意见的筛选口径不同。

## 行动索引（导出者摘要）

以下仅概括保留评论主题，不代表导出者确认问题成立。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：Graph 历史准入与 MTP draft Graph 修复及验证 | 未见解决状态标记 |
| R02 | 自动审查小结：本轮未发现需修改代码的问题 | 未见解决状态标记 |
| R03 / F2 | P1：DFlash/DFlash2 物理 page 表宽与图内容量不符 | 状态未核实 |
| R04 / F3 | P1：历史超容量时由 eager 降级变为 replay CHECK abort | 状态未核实 |
| R05 / F5 | P2：等价性测试对 block table 与 max_seq_len 不敏感 | 状态未核实 |
| R06 | Review 小结：原始 5 条 finding 及证据边界 | 未见解决状态标记 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月26日 10:26:21｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

**PR 描述**

#### 修改说明

修复 GLM-5.3 在 MLU 上启用 Graph 和 MTP 后，部分合法输入无法进入 Graph 的问题，支持首步 Top-k producer 和后续 Top-k reuse 的 capture/replay。

本 PR 覆盖当前分支相对 `main` 的两个提交：Graph 历史长度准入修复，以及 MTP draft Graph 修复。

#### 故障原因

- **历史长度准入未正确考虑 KV split**

原逻辑使用逻辑 `block_size` 判断 block table 能否覆盖当前历史长度，没有使用 KV split 后每列实际覆盖的 token 数。例如 `block_size=16、kv_split_size=4` 时，一列实际覆盖 64 tokens，旧检查可能错误拒绝合法输入。此外，block table 拷贝使用较小列宽截断，存在静默丢失数据的风险。

- **真实模型缺少 draft Graph 能力注册**

`glm_moe_dsa_mtp` 没有注册 grouped MTP Graph 能力。已有回归使用测试内注册的模型名，因此未覆盖生产模型注册项中的遗漏。

- **首步 draft 扩展输入后遗留无用状态元数据**

首步 draft 会将 N 行输入扩展为 2N 行，但继承的 `linear_state_ids` 仍只有 N 项。GLM-5.3 本身没有 linear attention，不过通用 batch builder 仍会生成值为 `-1` 的占位字段，导致 Graph 布局检查因行数不匹配而拒绝 capture。

#### 修复方案

- 根据模型上下文长度、实际 KV-split token span 和模型 padding 要求分配持久化 block table。

- 每次执行更新当前历史长度，完整拷贝源 block table，并增加容量边界检查，移除历史长度准入策略。

- 为 `glm_moe_dsa_mtp` 注册 grouped MTP Graph 和 native index-sharing 能力，继续使用 full Graph。

- 当 draft 没有 request-state cache 时，在首步输入准备中复用已有的 `clear_draft_state()`，清理无用的 linear-state 和 accepted-token 输入元数据。该操作不清理 KV cache，也不修改 target 状态。

- 扩展回归测试，使用真实模型注册项验证 producer/reuse 的 capture/replay、输入与 Top-k 状态更新，以及 replay 不再次调用模型 forward。

- 同时覆盖首步和后续无状态 draft 的元数据清理，并验证有状态 draft 保留所需状态。

#### 验证结果

- MLU 编译闸口通过。

- 13 个相关回归用例全部通过。

- clang-format 20.1.6、`git diff --check` 和提交时的 pre-commit 检查通过。

- 使用原 GLM-5.3-W4A8 checkpoint，保持 MTP3、DP2、EP8、KV split4 和 schedule overlap 配置。

- Eager 和 Graph 各完成一次单请求、两轮 8 并发，均使用 seed 42、`temperature=0`、`max_tokens=128`。

- 固定数列续写的所有响应逐字一致，均连续输出 51 至 114。

- 确认 `draft=1`、`mtp_topk_reuse=0/1` 均完成 full Graph capture 和后续 replay，MTP proposal/acceptance 指标非零。

- 未观察到请求挂起或设备错误。

历史长度准入修复的提交另记录了 98 个 Graph 用例和 13 个 DP 用例通过；本次最终 draft 修复未重新运行这两组完整测试。


## R02 — 冯艳 (fengyan.119)｜2026年10月26日 12:21:03｜MR 总体评论｜未见解决状态标记

- 类型：自动审查小结；根评论，0 条回复。

### 评论原文

### 🤖 自动审查小结：**LGTM，这代码看着真舒服！**

本轮静态代码审查未发现需要修改代码的问题。

## R03 — 马晓龙 (maxiaolong.maxwell)｜2026/10/26 12:46:23｜xllm/core/runtime/mlu_graph_executor_impl.cpp:740｜状态未核实

- 类型：行级意见；根评论，0 条回复。严重性：P1。
- 线程锚点：`discussion_r3477670`。未见明确解决事件；“Resolved”为处理按钮，解决状态未核实。

### 评论原文

#### F2 [P1] DFlash/DFlash2 独立 drafter 的 block table 被展开成物理 page（列数 × kv\_split），超出按 token span 分配的图内列数 → replay 期 `CHECK` 中止

**位置（索引）**：`xllm/core/runtime/mlu_graph_executor_impl.cpp:735-742`（列数分配）与 `:312-313`（越界 CHECK）；
展开侧 `xllm/core/runtime/dflash/dflash_worker_impl.cpp:1391-1403`、`:1420`；几何定义
`xllm/core/distributed_runtime/llm_engine.cpp:692-694`；语义注释 `xllm/core/runtime/draft_model_spec_worker_impl.h:99-108`

**几何先摆出来（宽度是怎么错开的）**

```text
同一 batch，max_pos = M，block_size = B，kv_split = K，一列的逻辑覆盖 = B*K 个 token

  scheduler/BlockManager 的 logical block_size = B*K          （llm_engine.cpp:692-694）
        │
        ▼  builder 产出「逻辑表」：宽 = W = ceil(运行历史 / (B*K))
  ┌───────────────────────── W 列 ─────────────────────────┐
  │ logical_block_id …                                     │      ← 目标侧 / MTP draft 直接用
  └────────────────────────────────────────────────────────┘
        │
        │  DFlash/DFlash2 draft 且 K>1：expand_draft_block_tables()
        │  每一列再展开成 K 个物理 page（id*K + 0..K-1）
        ▼
  ┌──────────────────── W*K 列（物理 page） ────────────────┐
  │ p0 p1 p2 p3 │ p0 p1 p2 p3 │ …                          │      ← 送进 draft 的 executor
  └────────────────────────────────────────────────────────┘

  MLU graph 持久 buffer（本 MR 之后）：
  ┌──────── ceil(M/(B*K)) + extra 列 ────────┐
  │                                          │  ← 只有 W 量级，容不下 W*K
  └──────────────────────────────────────────┘
```

**代码（逐字）**

1）本 MR 的列数分配（`xllm/core/runtime/mlu_graph_executor_impl.cpp:735-742`）：

```cpp
  const int64_t token_span = options_.block_table_token_span() > 0
                                 ? options_.block_table_token_span()
                                 : options_.block_size();
  CHECK_GT(token_span, 0);
  CHECK_GE(mtp_capabilities_.graph_block_table_extra_columns, 0);
  main_block_table_columns_ =
      (args_.max_position_embeddings() + token_span - 1) / token_span +
      mtp_capabilities_.graph_block_table_extra_columns;
```

`token_span` 的来源（`xllm/core/runtime/worker_impl.cpp:2236-2237`，对每个 worker 都写）：

```cpp
  options_.block_table_token_span(static_cast<int64_t>(options_.block_size()) *
                                  parallel_args_.kv_split_size_effective());
```

2）越界判定（`xllm/core/runtime/mlu_graph_executor_impl.cpp:307-318`，本 MR 新增）：

```cpp
  if (source.block_tables.defined()) {
    CHECK(buffer.block_tables.defined());
    CHECK_EQ(source.block_tables.dim(), 2);
    CHECK(buffer.block_tables.device() == source.block_tables.device())
        << "block_tables graph input device changed";
    CHECK_LE(source.block_tables.size(1), buffer.block_tables.size(1))
        << "block_tables graph input exceeds column capacity";
```

调用点在 replay 分支（`:856-860`）：

```cpp
  if (action == GraphAction::REPLAY) {
    CHECK(it != graphs_.end());
    it->second->update_input_buffer(model_, tokens, positions, params, layout);
```

3）draft 侧把表**扩宽 K 倍**（`xllm/core/runtime/dflash/dflash_worker_impl.cpp:1391-1403`）：

```cpp
void DFlashWorkerImpl::expand_draft_block_tables(
    ModelInputParams& input_params) const {
  const int64_t replicated_block_pages = draft_replicated_block_pages();
  if (replicated_block_pages <= 1) {
    return;
  }
  input_params.attention.host.block_tables =
      layer::expand_replicated_block_table(
          input_params.attention.host.block_tables, replicated_block_pages);
  input_params.attention.device.block_tables =
      layer::expand_replicated_block_table(
          input_params.attention.device.block_tables, replicated_block_pages);
}
```

它在 draft 的 query 输入上被调用（`xllm/core/runtime/dflash/dflash_worker_impl.cpp:1417-1420`），而这份输入正是 draft executor 的入参
（`dflash_worker_impl.cpp:797-801`）：

```cpp
  ForwardInput query_input;
  prepare_query_inputs(input, query_input);

  ForwardOutput draft_output =
      speculative_worker::run_worker_no_sync(
          *draft_impl_, query_input, *prepare_stream_, *compute_stream_)
          .value();
```

展开本身（`xllm/core/layers/common/kv_shard_batch_metadata.cpp:87-96`）把列维乘上页数：
`expanded = logical_block_table.unsqueeze(-1) * replicated_block_pages + page_offsets`，随后
`reshape({rows, columns * replicated_block_pages})`。

4）为什么"逻辑表一列 = `B*K` token"是仓库自己的定义（`xllm/core/runtime/draft_model_spec_worker_impl.h:99-108`）：

```cpp
  // Tokens covered by one column of the TARGET's block table. The table holds
  // logical blocks, so under kv_split one column spans block_size *
  // kv_split_size tokens. Every conversion between a KV length and a column
  // count against that table has to use this value; the draft's own table is
  // expanded to physical pages instead (DFlashWorkerImpl::
  // expand_draft_block_tables). SuffixWorkerImpl has no draft and derives from
  // SpeculativeWorkerImpl directly, so it keeps the physical block size.
```

以及 `replicated_block_pages` 的取值规则（`xllm/core/framework/config/speculative_config.h:138-142`：
`kv_split_size > 1 && owns_replicated_kv_cache(algorithm) ? kv_split_size : 1`，
`owns_replicated_kv_cache` = DFlash / DFlash2，见同文件 `:131-134`）。

**触发条件**：MLU + `enable_graph()==true` + `speculative_algorithm ∈ {DFlash, DFlash2}` + `kv_split_size_effective() > 1`；
draft 侧历史长度满足 `ceil(h/(B*K))*K > ceil(M/(B*K)) + extra`，即约 `h > M/K + B*K`。
手算（M=202752、B=16、K=4、extra=1，与 GLM-5.3 同量级）：buffer = 3169 列，物理页宽 = `ceil(h/64)*4`，
**h 超过约 50,704 个 token 就越界**；而模型上下文是 202,752，也就是说"任何超过 1/K 上下文的请求"都会踩到。

**当前行为**：base 侧列数按 `block_size` 计（`ordinary_graph_capacity`/`history_capacity` 决定，
`src/base/xllm/core/runtime/mlu_graph_executor_impl.cpp:106-115`、`:752-753`、`:868`），buffer ≈ `M/B` 列，正好 ≥ 物理页宽 `h/B`，
且旧拷贝用 `std::min(dst, src)` 截断（`diff.patch` 中 `- std::min(...)` 两处），**不会崩**。
本 MR 把列数缩小 K 倍，同时把截断换成 `CHECK_LE`，于是同一条路径从"能跑"变成"replay 中途 abort"。

**期望行为**：图内列数必须按"送给这个 executor 的表的真实寻址单位"来分配。对独立 drafter，
一列 = 1 个物理 page，列数应为 `ceil(M/(B*K)) * replicated_block_pages + extra`；
或显式让此类 worker 退出图模式（planner 层返回 `EagerReason`），不要让它走到 replay 才 abort。

**影响**：operability/correctness —— 进程级 abort（glog `CHECK` 在 release 也 FATAL），发生在长跑服务的
replay 路径，等价于"上下文一长就整实例挂掉"。

**根因**：`block_table_token_span` 假设"所有 worker 的 block table 都是逻辑块寻址"，
而 DFlash/DFlash2 的 draft 池是**复制型池**（`replicated_block_pages = kv_split`），
其表已在进 executor 前被换成物理页寻址（仓库注释明确写了两种寻址单位并存）。

**建议（最小修法）**：在 `MluGraphExecutorImpl` 构造列数时引入"该 executor 的表的寻址单位"这一因子：
worker 侧（`WorkerImpl::init_model` 已经把 span 写进 options）再写一个
"列数放大因子 = `replicated_block_pages`"（DFlash/DFlash2 且 kv\_split>1 时为 kv\_split，否则 1），
executor 用 `ceil(M/span) * factor + extra`；短期替代方案：在 `MluGraphPlanner` 里对
"draft 池复制"的 worker 直接返回 eager，避免 abort。

**验证**：`mlu_graph_executor_test` 里加一个用例：draft executor + `replicated_block_pages=4`，
源表宽度取 `ceil(h/64)*4`、buffer 按 `ceil(M/64)+1` 构造，断言 `update_input_buffer` 不 abort；
或直接跑 DFlash2+kv\_split=4 的两请求长上下文回归。

**证据边界**：已证实 = 宽度算术与调用链（纯仓库内代码，无厂商依赖）。
未证实 = "MLU 上 DFlash/DFlash2 + kv\_split>1 + graph" 是否为受支持/已上线组合（`unverified.md` U3）：
`master.cpp:170-171` 的 `kv_split>1` 限制只在 `cp_size>1` 时生效，`cp_size==1` 的独立 DCP 场景没有模型/算法约束。

---

## R04 — 马晓龙 (maxiaolong.maxwell)｜2026/10/26 12:48:34｜xllm/core/runtime/mlu_graph_executor_impl.cpp:312｜状态未核实

- 类型：行级意见；根评论，0 条回复。严重性：P1。
- 线程锚点：`discussion_r3477671`。未见明确解决事件；“Resolved”为处理按钮，解决状态未核实。

### 评论原文

#### F3 [P1] 历史超容量从"退回 eager"变成"replay 中途 `CHECK` abort"，而被删掉的两个用例正是这条契约的唯一守卫

**位置（索引）**：`xllm/core/runtime/mlu_graph_executor_impl.cpp:312-313`（abort 点）、`:856-860`（replay 调用点）；
planner 侧现状 `xllm/core/runtime/mlu_graph_planner.cpp:334-354`；旧守卫
`src/base/xllm/core/runtime/mlu_graph_planner.cpp:199-212`、`:368-370`

**代码（逐字）**

1）新的越界形态是一条 `CHECK`（`xllm/core/runtime/mlu_graph_executor_impl.cpp:312-313`）：

```cpp
    CHECK_LE(source.block_tables.size(1), buffer.block_tables.size(1))
        << "block_tables graph input exceeds column capacity";
```

它位于 `update_input_buffer`，而 `update_input_buffer` 在 **replay** 每一步都跑（`:856-860`）：

```cpp
  if (action == GraphAction::REPLAY) {
    CHECK(it != graphs_.end());
    it->second->update_input_buffer(model_, tokens, positions, params, layout);
    result = it->second->replay();
```

2）planner 已经完全不看历史长度，只剩下 token 上限与 key（`xllm/core/runtime/mlu_graph_planner.cpp:334-354`）：

```cpp
  if (layout->padded_num_tokens > config_.max_tokens) {
    return EagerReason::TOKEN_LIMIT;
  }
  if (layout->cache_pad_slot == 0) {
    layout->cache_pad_slot = config_.capabilities.graph_cache_pad_slot;
  }
```

旧代码在这里还有一道降级（`src/base/xllm/core/runtime/mlu_graph_planner.cpp:368-370`）：

```cpp
  if (!supports_graph_history(params, capacity, options.block_size())) {
    return EagerReason::HISTORY_LIMIT;
  }
```

3）本 MR 删掉的两个用例（`diff.patch`，`tests/core/runtime/mlu_graph_executor_test.cpp` 的 hunk）正是这条降级契约：

```cpp
-TEST_F(MluGraphExecutorTest, QwenHistoryBeyondContextFallsBackThenReplays) {
```

```cpp
-  input.input_params.meta.kv_max_seq_len = 8193;
-  impl_->run(input.token_ids, input.positions, kv_caches_, input.input_params);
-  EXPECT_EQ(model_->forward_cnt(), 2);
```

替代用例 `QwenHistoryGrowthReplaysWithoutRecapture`（`tests/core/runtime/mlu_graph_executor_test.cpp:1829-1835`）
只把历史拉到 8192 就停手，**没有任何用例**再触碰"超过容量"这一侧：

```cpp
  for (const int32_t history : {16, 4097, 8192, 16}) {
    input.input_params.meta.kv_max_seq_len = history;
```

**容量账（手算）**：`capacity = (ceil(M/span) + extra) * span`。

| 模型 | M | span | extra | 图容量 | 旧行为起点 | 新行为起点 |
| --- | --- | --- | --- | --- | --- | --- |
| `qwen3_5_text`（测试 8192） | 8192 | 16 | 1（默认） | 8208 | 8193 起退回 eager | 8209 起 abort |
| `glm_moe_dsa`（GLM-5.3 target） | 202752 | 64 | 1（默认） | 202,816 | 202753 起退回 eager | 202,817 起 abort |
| `glm5_next`/`glm5_next_mtp` | 8192 | 16 | **0**（`glm5_next_mtp.h:94`） | **8192** | 8193 起退回 eager | **8193 就 abort（零余量）** |

**触发条件**：图已 capture，之后某一步 `kv_max_seq_len` 使 `ceil(h/span) > buffer 列数`
（≈ `h > capacity - span + 1`，见上表）。

**当前行为**：replay 期 `CHECK` FATAL，进程退出；同一情形在 base 是 planner 返回 `HISTORY_LIMIT` → 本步 eager。
**期望行为**：至少保持"宁可慢不可挂"的降级语义，或在 **plan 阶段**就 fail fast —— planner 能看到
`params.attention.device.block_tables.size(1)`，但看不到 buffer 列数，需要把容量透传给它。

**影响**：operability。单实例长跑中被一个超长请求打断，且发生在 replay 而非启动期，排障成本高。

**根因**：移除准入时把"准入 + 降级"一起删了，只补了"越界就崩"的断言（`:312`），
没有补"到不了图就退回 eager"。

**建议（最小修法）**：`GraphPlannerConfig` 增加一个只读的 `main_block_table_columns`（构造期已知），
plan 时若 `ceil(kv_max_seq_len/token_span) > 容量` 直接返回 `EagerReason`（可复用 `INVALID_LAYOUT`
或新增 `HISTORY_EXCEEDED`）；这样 `:312` 的 CHECK 退回"不该发生"的断言，语义与 base 一致。

**验证**：恢复一个负例用例（参数化 `qwen` 与 `glm5_next`，两者 extra 不同），断言超过容量后
`forward_cnt` 递增（= 走 eager）且回到容量内仍只 capture 一次；同时保留一条 death test 覆盖"真的拼错宽度"。

**证据边界**：已证实 = 行为反转与契约删除。**可达性未完全证实**：请求侧有上下文校验
（`xllm/core/framework/request/llm_request_factory.cpp:294` 取 `max_position_embeddings()`；
`:338` 构造 `StoppingChecker` 时把上限设为 `max_context_len - num_speculative_tokens`；
`stopping_checker.cpp:115-118` 到点即结束），因此**正常 HTTP + MTP 投机路径**大概率到不了上表阈值 ——
见 `unverified.md` U2 列出的其余入口（PD decode、graph warmup、beam/`best_of`、`max_tokens` 覆盖、
以及 `extra=0` 模型的零余量）。若作者确认所有入口都保证 `h ≤ capacity`，本条可降为 P2（至少补注释与断言消息），
但**删除负例**这一点仍应保留为测试缺口（见 F5）。

---

## R05 — 马晓龙 (maxiaolong.maxwell)｜2026/10/26 12:53:13｜tests/core/runtime/mlu_graph_executor_test.cpp:1510｜状态未核实

- 类型：行级意见；根评论，0 条回复。严重性：P2。
- 线程锚点：`discussion_r3477673`。未见明确解决事件；“Resolved”为处理按钮，解决状态未核实。

### 评论原文

#### F5 [P2] 新增/改写的"图 vs eager 等价"断言对 block table 与 `max_seq_len` 完全不敏感，F1/F2 可以在测试全绿下存在

**位置（索引）**：`tests/core/runtime/mlu_graph_executor_test.cpp:133-153`（mock 输出）、`:1510-1512`（新增等价断言）、
`:1826-1834`（qwen 成长用例只断言 capture 次数）

**代码（逐字）**

mock 的输出只依赖 `input_embedding`（`tests/core/runtime/mlu_graph_executor_test.cpp:133-153`）：

```cpp
  ModelOutput forward(const torch::Tensor& tokens,
                      const torch::Tensor& positions,
                      std::vector<KVCache>& kv_caches,
                      const ModelInputParams& params) override {
    (void)tokens;
    (void)positions;
    (void)kv_caches;
    ++forward_cnt_;
    last_tokens_size_ = tokens.size(0);
    last_dp_token_nums_ = params.parallel.dp_global_token_nums;
    last_kv_max_seq_len_ = params.meta.kv_max_seq_len;
    last_block_table_width_ = params.attention.device.block_tables.defined()
                                  ? params.attention.device.block_tables.size(1)
                                  : 0;
    auto hidden_states = params.embedding.input_embedding.matmul(weight_);
```

新增用例的等价断言正是比这个输出（`:1510-1512`）：

```cpp
    auto expected = model_->forward(
        input.token_ids, input.positions, kv_caches_, input.input_params);
    EXPECT_TRUE(torch::equal(actual, expected.hidden_states));
```

qwen 用例只比 capture 次数（`:1826-1834`）：

```cpp
  for (const int32_t history : {16, 4097, 8192, 16}) {
    input.input_params.meta.kv_max_seq_len = history;
    input.input_params.attention.device.block_tables = torch::zeros(
        {3, (history + 15) / 16 + 1}, tensor_options_.dtype(torch::kInt32));
    impl_->run(
        input.token_ids, input.positions, kv_caches_, input.input_params);
    EXPECT_EQ(model_->forward_cnt(), 1);
  }
```

`grep -n "attn_metadata" tests/core/runtime/mlu_graph_executor_test.cpp` 显示：除 `kpool_batch_metadata`
与 `has_initial_states`（`:394`、`:440`、`:455`、`:480-481`）外，**没有任何 mock 读
`attn_metadata.max_seq_len` / `block_table`**。

**触发条件**：任何"图内常量被 capture 时机决定"或"block table 列/内容拷错"的回归
（F1、F2 就属此类）落在上述断言覆盖范围内时，用例仍然全绿。

**当前行为**：`torch::equal(actual, expected)` 的两侧都只差 embedding 的 matmul 结果；
`last_kv_max_seq_len()` 只记录 `params.meta.kv_max_seq_len`（host 侧每步刷新的那个值），
不记录 `attn_metadata.max_seq_len` —— 两者在本 MR 之后**已经不是同一个数**（F1）。

**期望行为**：断言至少对"图内实际送进 kernel 的量"敏感（`attn_metadata.max_seq_len`、
`block_table` 的整行/列宽），这样 F1/F2 这类回归会转红。

**影响**：测试有效性 —— 现有 98 graph 用例的"全绿"不能作为本 MR 两个核心改动（几何换算 + metadata 刷新）
的正确性证据。**根因**：mock 的 forward 与 `AttentionMetadata` 完全解耦，等价性只落在 embedding 通路上。

**建议**：把 mock 的 `hidden_states` 改成对 `params.attn_metadata->max_seq_len`、`block_table.sum()`
各加权一次（仓库里已有先例：`StatefulVerifyModel` 在本 MR 中把 `block_tables.select(1,0)` 改成
`sum(1)`，`tests/core/runtime/mlu_graph_executor_test.cpp:493-494`，正是一次"让断言对整行敏感"的改进），
并对 `KvSplitDecodeReplaysAcrossLogicalBlocks` 保留等价断言。

**验证**：改造后重跑该文件；再故意把 capture 期的 `kv_max_seq_len` 改回"首步历史"，用例应转红。

**证据边界**：已证实 = mock 输出与断言口径（源码直读）。未证实 = 是否有其它测试文件覆盖了这些量
（本文件外的覆盖未逐文件穷举，见 `unverified.md` U8）。

---

## R06 — 马晓龙 (maxiaolong.maxwell)｜2026年10月26日 12:55:30｜MR 总体评论｜未见解决状态标记

- 类型：Review 小结；根评论，0 条回复。原始清单含已解决讨论的引用，按独立小结保留原文；不代表这些讨论仍待处理。

### 评论原文

# Review 小结（pr-teach-dag 流水线）

本 MR 深度教学式审查完成（教学书 6 章 + 审查轨 5 条 finding，P1×3 + P2×2），行级评论清单：

| # | 级别 | 锚点 | 一句话 |
| --- | --- | --- | --- |
| F1 | P1 | mlu\_graph\_executor\_impl.cpp:292（回复 note:3477387） | capture 期把首步历史烘成图内 max\_seq\_len，回放不重建 |
| F2 | P1 | mlu\_graph\_executor\_impl.cpp:740 | DFlash/DFlash2 独立 drafter 的表已展开成物理 page（列数×kv\_split），图 buffer 按 span 分配容不下 → replay CHECK abort |
| F3 | P1 | mlu\_graph\_executor\_impl.cpp:312 | 历史超容量从"退回 eager"变成 replay 中途 CHECK abort；被删的两个负例是唯一守卫 |
| F4 | P2 | tests/.../mlu\_graph\_executor\_test.cpp:529（回复 note:3477388） | 新 option 与 SpeculativeWorkerImpl::logical\_block\_size() 同规则第二份实现 |
| F5 | P2 | tests/.../mlu\_graph\_executor\_test.cpp:1510 | 新增等价断言建立在只吃 input\_embedding 的 mock 上，对表内容/max\_seq\_len 零敏感 |

说明：note:3477385（ODR 说法）经源码核实不成立（namespace-scope const 内部链接 + 全树仅 1 个 TU 引入 + 注册为覆盖式写 map），未重复提交；note:3477386 的形状断言诉求已由 :312 新增 CHECK 提供，未新开。另有 9 条未验证项（kernel max\_seq\_len 语义、DFlash+kv\_split>1+graph 组合支持性、全入口可达性等）证据不足，未发布。
