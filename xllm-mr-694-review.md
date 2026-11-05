# xLLM MR #694 — Review 意见

- 来源：[MR #694](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694)
- 标题：`refactor: shard glm5-next mlu kda heads across cp and tp.`
- 分支：`feat/glm53_next_cp_refactor` → `main`；7 commits、44 files changed。
- 页面状态：Open / Reviewing；评审规则要求 2 人批准，页面提示等待人工评审通过。
- 当前 head：未从独立 head 字段核实；页面提交首项链接为 `63747c0acbf961b219c84a1ac2e866cba4d4610c`。
- 提取时间：2026-11-05T14:31:40+08:00；通过已登录 Chrome 刷新页面，逐一切换全部 7 个 View Changes 评审事件。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评论和回复。
- 清单：7 条行级根评论、0 条已发布回复；另有作者说明、马晓龙评审总结、冯艳自动审查小结各 1 条。共 10 条根评论、0 条回复。页面 Conversation 8 不作为评论总数；空白回复编辑器不计为回复。
- 线程状态：仅见 No Processing Required / Resolved 操作按钮，未见明确解决事件，逐线程实际状态未核实。
- 范围限制：已核对全部 7 个评审事件的已加载讨论，未发现讨论分页、加载更多、隐藏同类评论或过时线程控件；部分完整文件 diff 未加载，不能独立排除其他或过时讨论。不含草稿。
- 位置说明：标题采用评论前页面差异行的新侧行号，无新侧时注明旧侧；正文中的位置照录。差异片段仅包含页面已有代码。
- 原文说明：技术判断均为评审者意见，未经导出者独立验证。保留自动审查 LGTM 与人工评审意见的差异。当前标题为 refactor，R06 与总结提及 perf 标题；提交列表中确有 perf 提交，未独立核实标题历史。R03 正文称 build_draft_kv_cache_shape，总结称 build_speculative_draft_kv_cache_shape，按原文保留，不裁定名称。

## 行动索引（导出者摘要）

| 评论 | 作者 | 主题 | 回复数 | 状态 |
| --- | --- | --- | --- | --- |
| R01 | 邓英旭 | 删除测试及内核的原因与替代覆盖 | 0 | 未核实 |
| R02 | 邓英旭 | kv_split_size 为零时的除零风险 | 0 | 未核实 |
| R03 | 马晓龙 | DFlash2 draft 缓存形状漏传 head 域 | 0 | 未核实 |
| R04 | 马晓龙 | max(1, ...) 掩盖非整除配置 | 0 | 未核实 |
| R05 | 马晓龙 | CP 分片语义的混版兼容与升级提示 | 0 | 未核实 |
| R06 | 马晓龙 | PCP 全行 allgather 的性能数据与通信权衡 | 0 | 未核实 |
| R07 | 马晓龙 | 副本发送者可达性与控制面链路契约 | 0 | 未核实 |

## R01 — 邓英旭｜2026/11/5 10:49:00｜tests/core/kernels/mlu/kpool/score_test.cpp:24｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496030](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496030)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

**1. 测试缺口 — 删除大量测试文件未说明原因**
位置：`tests/core/kernels/mlu/kpool/score_test.cpp:24-27`

本 MR 删除了 670 行针对 `chunk_kda_pcp` 的单元测试，同时删除了相关的 MLU kernel 文件。如果这些 kernel 被废弃或重构到了其他地方，删除测试是合理的，但 MR 描述中未说明。如果功能仍存在，删除测试会大幅降低代码覆盖率。

建议在 MR 描述中补充删除这些测试和 kernel 的原因。如果功能迁移，请确保有对应的新测试覆盖。

### 页面差异片段

```diff

 #include <algorithm>
 #include <limits>
+#include <utility>
```

## R02 — 邓英旭｜2026/11/5 10:49:00｜xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:312｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496031](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496031)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

**2. 正确性 — reshard\_planner.cpp 中存在潜在的除零风险**
位置：`xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:312-316`

`coordinates.cp_rank % (coordinates.cp_size / coordinates.kv_split_size)` 表达式中，如果 `coordinates.kv_split_size` 为 0，会导致除零异常。

建议确保进入此逻辑前 `coordinates.kv_split_size` 一定大于 0，或增加显式的条件判断以避免除零。

### 页面差异片段

```diff
+ // replicated MLA/index tensors even for an already selected source peer.
+  SpanWriters writers = include_replicas && !has_cp_shards
+                            ? SpanWriters::everything()
+                            : source_writers(source.coordinates);
```

## R03 — 马晓龙｜2026/11/5 11:07:31｜xllm/core/framework/kv_cache/kv_cache_shape.h:48｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496043](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496043)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

##### 【🔴 important】【✅ correctness】

##### DFlash2 draft 的 conv/ssm 缓存形状未随 CP×TP head 域更新：新放开的 MLU glm5\_next + DFlash2 + cp>1 组合在 draft 侧无法正确运行

---

**🔎 问题**

> 本 MR 打开了 MLU glm5\_next + DFlash2 的 CP 组合，draft worker 与 target 共用同一份 `parallel_args`（cp\_size > 1），draft KDA 层因此把 head 域扩成 world 组、每 rank 只持 `H/(tp*cp)` 个 head（`glm5_next_kda.cpp:274-302`）。但 draft 的 KV cache 形状构造走 `DraftModelSpecWorkerImpl::build_draft_kv_cache_shape`（主现场 `draft_model_spec_worker_impl.cpp:183`，不在本 MR diff 内），其 `draft_world_size = world/(dp*cp) = tp`，且是 `KVCacheShape` 新增参数 `linear_head_tp_size` **唯一漏传的调用点**——缺省 0 静默回退到 attention-TP 口径，conv/ssm 缓存按 `H/tp` 个本地 head 分配。target 侧 `llm_engine.cpp:648-654` 已正确传入 head 域宽度，只有 draft 侧漏了。

```cpp
// xllm/core/framework/kv_cache/kv_cache_shape.h:44-48
KVCacheShape(const KVCacheCapacity& kv_cache_cap,
             const ModelArgs& model_args,
             int64_t world_size,
             int64_t linear_head_tp_size = 0);   // ← 缺省 0 = 静默回退旧口径

// xllm/core/framework/kv_cache/kv_cache_shape.cpp:139-143
if (enable_linear_attention) {
  const int64_t head_tp_size =
      linear_head_tp_size > 0 ? linear_head_tp_size : world_size;  // 回退 = tp
  init_conv_cache_shape(kv_cache_cap, model_args, head_tp_size);
  init_ssm_cache_shape(kv_cache_cap, model_args, head_tp_size);
}
```

**后果**（cp=2、tp=2、H=64、dp=1，即本 MR 新放开的精确配置）：

* 开 PD 传输：draft 以 `is_spec_draft=true` 注册 SPEC\_DRAFT 缓存时，`describe_ssm` 对实际宽度 `H/tp` 的张量调 `get_head_placement`，命中 `cache_layout_builder.cpp:60-67` 的整除校验 → "global head count is not evenly sharded by TP" → 布局注册失败，PD 链路建不起来（启动期 loud，但报错不指向真正根因）。
* 不开 PD：`allocate_pools` 直接按错误形状分配、无布局校验；draft 前向 `ssm_cache.index_put_`（`glm5_next_kda.cpp:662-663`）source 形状 `[B, H/(tp*cp), kdim, vdim]` 与被索引行 `[B, H/tp, kdim, vdim]` 不一致 → 运行期 shape 错误或内核行宽错位（内核级失败模式无 MLU 环境未复现）。至少浪费 cp 倍 draft 线性状态内存。

**触发条件**：MLU + glm5\_next + DFlash2 + `cp_size > 1`。不触及既有可用组合（base 中该组合被 `CHECK_LE(cp_size, 1)` 拒绝）。

**根因**：`KVCacheShape` 构造函数新增参数带缺省值 0（静默回退），多调用点契约被破坏时编译期无感——「缺省参数 + 唯一漏改点」组合让 draft 侧无人校验。

**🔧 修改建议**

最小修复：`build_draft_kv_cache_shape` 从 draft 的 `ModelRegistry::get_linear_attention_policy(...)` 与 draft parallel args 计算 `linear_head_tp_size(policy, tp, cp)` 传入。防回归建议：去掉第 4 参缺省值（或改负值哨兵 + CHECK），迫使所有调用点显式表态。

**✅ 验证**

单测：构造 cp=2、tp=2、CP\_TP policy 的 draft 形状，断言 `conv_cache_shape()/ssm_cache_shape()` 与 `Glm5NextKDA` 同 parallel args 下的 `local_num_heads_` 一致（现有 `kv_cache_test.cpp` 的形状断言模式可复用；当前无任何测试覆盖 draft 形状构造器）。集成：MLU glm5\_next + DFlash2 + cp=2 冒烟（PD 与非 PD 各一次）。

### 页面差异片段

```diff
  const ModelArgs& model_args,
- int64_t world_size);
+ int64_t world_size,
+ int64_t linear_head_tp_size = 0);
```

## R04 — 马晓龙｜2026/11/5 11:09:47｜xllm/core/distributed_runtime/llm_engine.cpp:320｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496044](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496044)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

##### 【🔵 minor】【🟠 operability】

##### 引擎侧 n\_local\_linear heads 的 max(1, ...) 钳制掩盖非整除配置，失败延后到模型构造且容量日志先打出错误数字

---

**🔎 问题**

> `LLMEngine::init_model` 把分母从 `world_size` 换成 `head_tp_size`（含 CP 扩张）时沿用了旧的 `max(1, ...)` 钳制。`H < cp*tp` 时整除结果为 0 被钳成 1；`H % (cp*tp) != 0` 时向下取整静默丢 head——两种情况引擎都先按错误 head 数完成 KV 容量推导并打出误导性 block info 日志，随后才在模型构造处 `CHECK_EQ(args.linear_num_key_heads() % head_tp_size, 0)`（`glm5_next.h:154`）以不相关措辞崩溃，失败点与根因（head\_tp 组合）脱节。

```cpp
// xllm/core/distributed_runtime/llm_engine.cpp:313-322
if (has_linear_attention_layers(args_)) {
  const int64_t head_tp_size = linear_head_tp_size(
      linear_attention_policy_, world_size, options_.cp_size());
  const int64_t linear_n_k_heads = args_.linear_num_key_heads();
  const int64_t linear_n_v_heads = args_.linear_num_value_heads();
  n_local_linear_k_heads_ =
      std::max<int64_t>(1, linear_n_k_heads / head_tp_size);   // ← 钳制
  n_local_linear_v_heads_ =
      std::max<int64_t>(1, linear_n_v_heads / head_tp_size);   // ← 钳制
}
```

**触发条件**：`linear_num_key_heads < cp*tp`（如 H=32、cp=8、tp=8）或 `H % (cp*tp) != 0`（如 H=48、cp=4、tp=8）。

**根因**：`max(1, ...)` 旧语义服务于「合法但 head 数少于 TP 数的 replicated KV head」（GQA 场景）；线性 head 域恒为 SHARDED，钳制在新分母下失去合法语义，变成纯掩盖。

**🔧 修改建议**

在 sizing 处直接 `CHECK_EQ(linear_n_k_heads % head_tp_size, 0) << ...`（错误信息带上 head\_tp\_size、cp、tp 数值），把失败提前到第一次使用处。

**✅ 验证**

单测构造 H=48、cp=4、tp=8 的 args，断言 init 路径在 sizing 处即失败且错误信息包含 head\_tp\_size=32。

### 页面差异片段

```diff
  const int64_t linear_n_v_heads = args_.linear_num_value_heads();
     n_local_linear_k_heads_ =
-        std::max<int64_t>(1, linear_n_k_heads / world_size);
+        std::max<int64_t>(1, linear_n_k_heads / head_tp_size);
```

## R05 — 马晓龙｜2026/11/5 11:12:02｜xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:702｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496046](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496046)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

##### 【🔵 minor】【🔗 compatibility】

##### partitioned\_across\_cp 语义无版本协商：PD 滚动升级「旧 decode + 新 prefill」会在 coverage 校验处以不含版本信息的报错拒绝建链

---

**🔎 问题**

> proto 新字段 `partitioned_across_cp` 两端序列化透传（`cache_layout.cpp:186,217`），protobuf 前向兼容让旧 decode 读到该字段时直接忽略。但旧行为是「每组只选最低 CP rank 当 writer」；新 prefill 发出的 manifest 带 `partitioned_across_cp=true`（KDA SSM/CONV 分片横跨所有 CP rank），旧 decode 的 planner 不会为非首 CP rank 的 KDA head 分片选 writer，随后 `validate_writer_coverage` 以 "source DP rank X, KV-split rank Y cannot cover destination" 拒绝建链——报错不含任何版本或 partitioned 语义提示，现场会误判为容量/布局配置问题。反向（新 decode + 旧 prefill）安全：新 planner 退化为旧行为。

```cpp
// xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:700-714（新增循环）
// Ordinary caches are CP replicas. Recurrent head shards may instead
// span all CP ranks; select those writers without duplicating other roles.
for (const auto& [cp_rank, indices] : cp_workers) {
  for (size_t index : indices) {
    if (cp_rank != cp_workers.begin()->first &&
        !has_cp_sharded_resources(sources[index])) {
      continue;
    }
    if (has_logical_overlap(sources[index], destination_groups,
                            collapsed_source_writers(sources[index]))) {
      writers->emplace_back(index);
    }
  }
}
```

base 同位置只取 `cp_workers.begin()->second`（最低 CP rank 为唯一 writer）。

**触发条件**：PD 分离集群滚动升级期间 decode=base / prefill=head，模型 glm5\_next、cp>1、发生 reshard/PD 传输。

**根因**：proto 字段演进只保证了序列化兼容，没有为「reader 端行为依赖该字段」建立版本协商；布局 fingerprint（`mooncake_kv_cache_transfer.cpp:235-237`）未纳入 partitioned 语义。

**🔧 修改建议**

把 `partitioned_across_cp` 的存在性并入布局 fingerprint 或 `same_instance_layout` 比对字段（一处改动），使混版 pair 在 `link_cache_peer` 处得到「peer planner 不支持 CP head 分片，请先升级 X 侧」的明确报错；发布说明注明升级顺序约束。

**✅ 验证**

单测：head 端 manifest（partitioned=true）+ 模拟 base 端 planner 行为，断言链接失败且错误信息含升级提示；或直接对 fingerprint 断言混版 pair 不匹配。

### 页面差异片段

```diff
-        writers->emplace_back(index);
+ // Ordinary caches are CP replicas. Recurrent head shards may instead
+ // span all CP ranks; select those writers without duplicating other roles.
+ for (const auto& [cp_rank, indices] : cp_workers) {
```

## R06 — 马晓龙｜2026/11/5 11:14:19｜xllm/core/layers/mlu/glm5_next/glm5_next_kda_pcp.cpp:29｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496049](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496049)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

##### 【❓ question】【⚡ perf】

##### PCP 每个 KDA 层一次全行 allgather：相对被删除的 token-PCP 尾巴交换，prefill 通信量上升 2-3 个数量级——perf 标题的 MR 是否有 A/B 数字支撑？

---

**🔎 问题**

> 新路径在每个 KDA 层前向做 `gather_restore`（全 CP 组 allgather 恢复完整请求序列）→ 常规 forward → `shard_rows` 切回本地行。被删除的 token-PCP 内核（本 MR 删 4409 行）跨 rank 只交换每请求 conv 尾巴（conv\_kernel-1=3 行 × channels）与每请求终态（per-head 状态矩阵）。两者通信量差「token 数 vs 常数」量级。

```cpp
// xllm/core/layers/mlu/glm5_next/glm5_next_kda_pcp.cpp:26-32
// mHC/DSA retain CP-local rows. KDA consumes complete request sequences
// with heads sharded over CP x TP, including its persistent conv/SSM state.
const torch::Tensor global_hidden =
    glm5_next_pcp::gather_restore(hidden_states, context);
const torch::Tensor global_output =
    forward(global_hidden, attn_metadata, kv_cache, input_params);
return glm5_next_pcp::shard_rows(global_output, context);
```

**机制推算**（cp=2、hidden=4096、bf16、单请求 8k token prefill）：每 KDA 层每 rank 接收 `(cp-1)/cp × 8k × 4096 × 2B ≈ 32MB`；按 full\_attention\_interval 交替估算 KDA 层占多数（GLM5-Next 约 3/4 层）→ 单请求 prefill 每 rank 额外接收约 1GB 量级。base 路径为常数级小交换。交换所得：单卡 KDA 权重与 conv/ssm 状态内存降为 1/cp，且删掉 3300+ 行专用内核——通信-计算权衡方向反转了，但 MR 描述与 commit message 中没有任何 prefill TTFT / 吞吐 / 显存 A/B 数字（仅正确性回归）。

**两个具体追问**：

1. 请补一组固定硬件 + 固定 cp/tp + 固定请求集的 prefill TTFT / tokens/s A/B（base token-PCP vs head CP×TP）与单卡显存前后对比，支撑 perf 结论。
2. 相邻两个 KDA 层之间 `shard_rows(global_output)` 之后下一层又 `gather_restore` 同一批行——这对 gather/shard 是否可抵消或已被 stream 异步掩盖？值得确认。

**✅ 验证**

`XLLM_MLU_KDA_CHUNK_SIZE` 固定 + 相同请求集，cp=2 与 cp=1 的 prefill wall-time；通信量可用 HCCL/MLU 通信计数器与 `launch_gather` 调用次数 × 字节数静态推算交叉验证。

### 页面差异片段

```diff
+ // mHC/DSA retain CP-local rows. KDA consumes complete request sequences
+ // with heads sharded over CP x TP, including its persistent conv/SSM state.
+ const torch::Tensor global_hidden =
+      glm5_next_pcp::gather_restore(hidden_states, context);
```

## R07 — 马晓龙｜2026/11/5 11:16:36｜xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:319｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3496051](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/694#discussion_r3496051)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

##### 【❓ question】【✅ correctness】

##### without\_cp\_replicas 屏蔽非首 CP rank 的 MLA/KPool 副本后，副本字节的唯一发送者是首个 CP rank——目标端可达性依赖控制面全连接扇出，请确认链路契约

---

**🔎 问题**

> MLU glm5\_next CP 场景（kv\_split=1）下，`cp_rank % cp_size != 0` 使所有非首 CP rank 的源进入 `without_cp_replicas` 分支：只发送自己的 SSM/CONV head 分片，MLA/KPool/INDEX 副本被屏蔽——副本字节的唯一发送者是「本 KV shard 的首个 CP rank」。

```cpp
// xllm/core/framework/kv_cache_transfer/reshard_planner.cpp:305-322
if (has_cp_shards && !kv_split_spans_cp_and_tp(coordinates) &&
    coordinates.cp_rank % (coordinates.cp_size / coordinates.kv_split_size) != 0) {
  // Later CP ranks still own recurrent heads, but their MLA/KPool replicas
  // must not overwrite the copy sent by the first CP rank of this KV shard.
  return writers.without_cp_replicas();
}
```

**为什么是问题而非确定缺陷**：`validate_writer_coverage` 校验的是全部候选源的并集覆盖 destination；实际字节流向由 per-pair 的 `build_outgoing_plan`（每个已建链 source→destination 对一份）决定。若控制面只让某个 destination 与部分源建链（而非每个 tp\_rank 的首个 CP rank 都在 peer 列表内），被屏蔽的副本在该 destination 上无人发送。`link_sessions(cluster_ids, remote_addrs)` 只是被动链接给定列表，扇出策略在控制面（etcd 发现 / master 编排），不在本 MR 物化范围内，无法判定是缺陷还是既有契约。

**触发条件**：MLU glm5\_next CP + PD/reshard 传输 + 控制面链路非全连接。

**🔧 修改建议**

作者/控制面负责人确认：PD 与 reshard 场景下每个 destination rank 的 peer 列表是否保证覆盖所有源 rank（至少每个 (tp\_rank, 首个 cp\_rank) 组合）。若不保证，在 `select_collapsed_writers` 选出 writers 后增加副本发送者可达性断言/日志（现有测试 `CollapsesCpHeadShardsWithoutDuplicatingMlaReplicas` 隐式覆盖了全连接假设）。

**✅ 验证**

控制面代码或部署文档确认链路扇出；补一条「部分链接」场景的 planner 测试，断言失败信息明确指出缺失的副本发送者 rank。

### 页面差异片段

```diff
+ 0) {
+ // Later CP ranks still own recurrent heads, but their MLA/KPool replicas
+ // must not overwrite the copy sent by the first CP rank of this KV shard.
+ return writers.without_cp_replicas();
```

## A01 — 罗磊 (ext.luolei15) 作者说明｜2026年11月5日 10:43:07｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

refactor: 重构 MLU GLM5-Next 上下文并行并支持 DFlash2。

#### 改动说明

将 MLU GLM5-Next 的 KDA 改为在 CP×TP 范围内按 head 分片，统一权重、计算及 Conv/SSM 状态的分片方式，并支持 DFlash2 上下文并行。

* KDA 在 PCP 阶段恢复完整请求序列，复用常规计算路径后切回本地 token；移除原有 token-PCP 专用内核及测试。
* 引入线性注意力并行策略，分离后端 KV 状态约束与模型 head 分片配置，统一模型初始化、缓存尺寸和传输布局的处理。
* 扩展缓存分片描述与重分片规划，传输各 CP rank 持有的状态分片，避免重复传输 MLA/KPool 副本。
* 开放 MLU GLM5-Next + DFlash2 的 CP 组合，为 draft context KV PUSH 使用独立的逐层同步器并等待传输完成。
* 提前拒绝 MLU 线性注意力的有效 kv\_split\_size > 1 配置；修复 KPool 选择内核在 k=1 时的崩溃。

#### 测试与验证

* 新增或更新并行策略、缓存布局、传输规划、KPool 小规模输入及双卡 KDA 一致性测试，覆盖 prefill、续算、decode 和推测验证。
* 提交记录显示：MLU 编译通过，82 项相关回归测试通过。

## S01 — 马晓龙 (maxiaolong.maxwell) 评审总结｜2026年11月5日 11:19:05｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

#### MR694 Review 总结

对本 MR（KDA head 分片 + DFlash2 CP）做了一轮完整审查，共 5 条意见，均附 file:line 与代码原文证据：

1. **[P1] DFlash2 draft 缓存形状漏传 head 域**：`build_speculative_draft_kv_cache_shape` 是 `KVCacheShape` 新参数 `linear_head_tp_size`（缺省 0 静默回退）唯一漏传的调用点，本 MR 新放开的 MLU glm5\_next + DFlash2 + cp>1 组合 draft 侧布局注册失败或前向 shape 崩溃。（锚点：kv\_cache\_shape.h）
2. **[P2] `max(1, H/head_tp)` 钳制掩盖非整除配置**：失败延后到模型构造、容量日志先错。（锚点：llm\_engine.cpp）
3. **[P2] `partitioned_across_cp` 无版本协商**：滚动升级混版 pair 在 coverage 校验处以误导性报错拒绝建链，建议并入布局指纹。（锚点：reshard\_planner.cpp）
4. **[Question] PCP 全行 allgather 通信量**：较被删 token-PCP 尾巴交换升 2-3 个数量级，perf 标题缺 A/B benchmark；另请确认相邻 KDA 层 gather/shard 是否可抵消。（锚点：glm5\_next\_kda\_pcp.cpp）
5. **[Question] 副本发送者可达性**：`without_cp_replicas` 屏蔽后副本唯一发送者是首个 CP rank，目标端可达性依赖控制面全连接扇出，请确认链路契约。（锚点：reshard\_planner.cpp）

已核验无问题的重点区：KPool k=1 修复（最小且正确，仅 row\_count=1 时 nb 0→64）、kv\_split>1 三处前置拒绝 + master 白名单无绕过、CP\_TP 注册的 USE\_MLU 门控（NPU 构建行为不变）、mooncake/KDA/get\_head\_placement 三口径 head rank 一致、被删 4409 行 token-PCP 无残留引用、PCP 退化路径（rank 分不到 chunk → 全序列）正确性成立。

## S02 — 冯艳 (fengyan.119) 自动审查小结｜2026年11月5日 11:47:32｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

##### 🤖 自动审查小结：**LGTM，这代码看着真舒服！**

本轮静态代码审查未发现需要修改代码的问题。
