# xLLM MR #681 — Review 意见

- 来源：[MR #681](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681)
- 标题：`feat: support deepseek v4.1 mlu context parallel and dspark.`
- 分支：`feat/dsv4.1-cp-dspark` → `main`；6 commits、33 files changed。
- 页面状态：Open / Reviewing；要求 2 人批准；状态检查 3 项通过，人工评审未通过，阻止合并。
- 当前 head：未从独立 head 字段核实；页面提交首项链接为 `57fe948da27b0c3a735d5d64ab557edfc1ccbee0`；评审总结报告 base=`b4d4316`、head=`57fe948`。
- 提取时间：2026-11-04T14:33:02+08:00；通过已登录 Chrome 刷新页面后，逐一切换全部 11 个 View Changes 评审事件并读取已加载讨论。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评论。
- 清单：11 条行级根评论（邓英旭 1 条、马晓龙 10 条），2 条独立评审总结（冯艳 1 条、马晓龙 1 条），另有 1 条作者说明；共 14 条根评论、0 条已发布回复。页面 Conversation 12 不作为评论总数。
- 线程状态：所有行级线程仅见 No Processing Required / Resolved 操作按钮，未见明确解决事件，均标为“状态未核实”；回复输入框不计为回复。
- 范围限制：已核对全部 11 个评审事件的已加载讨论，未发现分页、加载更多、隐藏同类评论或 OutDated 控件；未逐一加载全部 33 个文件的完整 diff，不能独立排除其他或过时讨论。不含草稿。
- 位置说明：各标题位置为评论前差异行的新侧行号（无新侧则注明旧侧），原文报告的位置另行保留，二者不一定相同。差异片段只保留页面已有代码。
- 原文说明：技术判断均为评审者意见，未经独立验证。R06 原文含 `:unknow_emoji:`，照录。马晓龙总结报告 +1612/-362，而页面 Files Changed 显示 +1539/-289，未核实差异原因；其“无 P0/P1/P2”结论不代表邓英旭另发的正确性评论已关闭。

## 行动索引（导出者摘要）

| 评论 | 作者 | 主题 | 状态 |
| --- | --- | --- | --- |
| R01 | 邓英旭 | CP window 切片负起点 | 状态未核实 |
| R02 | 马晓龙 | 非 MLU native 投机准入 | 状态未核实 |
| R03 | 马晓龙 | Engram all-reduce 通信量 | 状态未核实 |
| R04 | 马晓龙 | gather.mlu 重复扫描 | 状态未核实 |
| R05 | 马晓龙 | commit message 与 diff 不一致 | 状态未核实 |
| R06 | 马晓龙 | draft 投影输入宽度校验 | 状态未核实 |
| R07 | 马晓龙 | grouped verify executor 集成测试 | 状态未核实 |
| R08 | 马晓龙 | draft 每层重复同步及过滤 | 状态未核实 |
| R09 | 马晓龙 | CP+Engram 多 rank 等价测试 | 状态未核实 |
| R10 | 马晓龙 | topk 二维形状契约 | 状态未核实 |
| R11 | 马晓龙 | CP 源层两次 all-gather | 状态未核实 |

## R01 — 邓英旭 (dengyingxu1)｜2026/11/4 10:38:53｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:21｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3493757](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3493757)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

**1. 正确性 — deepseek\_v41\_attention.cpp 在 forward\_cached\_attention 的 CP**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:21`

deepseek\_v41\_attention.cpp 在 forward\_cached\_attention 的 CP 上下文处理循环中，计算 window.slice(0, segment\_old\_begin - old\_begin) 时，segment\_old\_begin 是通过 std::max&lt;int64\_t&gt;(0, segment\_begin - args\_.window\_size() + 1) 计算的，可能为 0；而 old\_begin 是针对整个序列计算的，可能大于 0。当 segment\_old\_begin < old\_begin 时，切片的起始索引为负数。在 PyTorch 中，负的 start 索引会从张量末尾开始计算，导致错误地截取 window 末尾的数据，而不是预期的空切片或从头开始

建议将切片起始索引限制为非负数，改为 window.slice(0, std::max&lt;int64\_t&gt;(0, segment\_old\_begin - old\_begin))。

### 页面差异片段

```diff
 #include <algorithm>
 #include <cmath>
 #include <limits>
+#include <numeric>
```

## R02 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 11:56:24｜xllm/core/distributed_runtime/master.cpp:628｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494017](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494017)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] master 准入简化后，非 MLU 平台的 native V4.1 投机配置失去早期精确拒绝（fail-late + 诊断信息降级）

**后果**：native model\_impl + V4.1 + 投机的配置在非 MLU 平台（NPU/GPU）穿过 master 准入后，在更晚的模型解析处以不相关报错失败；native + 非 DSpark 算法的报错从"点名实现腿的精确信息"退化为只报算法腿。

**触发条件**：任一非 MLU 平台 + native model\_impl + V4.1 checkpoint + speculative\_algorithm != DSpark（或 = DSpark）。

**代码**（master.cpp:627-629，新）：

```cpp
if (v41_speculative_requested) {
  CHECK(SpeculativeConfig::is_dspark_algorithm(
      options_.speculative_algorithm()))
      << "DeepSeek-V4.1 speculative decoding requires DSpark.";
}
```

base（b4d4316）此处原有两个 CHECK：`is_python_model_impl` 实现腿（报错点名 grouped KV C1/C2 未在 native speculative path 实现）+ `is_v41_python_dspark_spec` 谓词腿。57fe948 拆掉了实现腿——MLU 上这是本 MR 特性（V4.1 的 C++ 注册只在 `xllm/models/llm/mlu/deepseek_v41.h:390`，mlu 命名空间），但其他平台 native+DSpark 也被放行。

**根因**：准入谓词按"实现 × 算法"两腿设计，简化时按 MLU 视角拆实现腿，未按平台收窄拆除范围。

**最小修复**：把实现腿改为平台限定，如 `CHECK(Platform::is_mlu() || ModelConfig::is_python_model_impl(model_impl))`，或在报错文案注明 native 执行器仅 MLU 支持。

**验证**：config\_json\_test 补"非 MLU 平台 + native impl + V4.1 + DSpark"准入用例。

（证据边界：MLU 放行路径有源码与测试证据；"NPU 上更晚失败"由注册表推导（`git grep REGISTER_CAUSAL_MODEL` 仅 mlu 命中 v41），未实测 NPU 构建。）

### 页面差异片段

```diff
- "for the DSpark algorithm: the grouped KV (C1/C2) is not "
- "implemented in the other speculative paths.";
+ CHECK(SpeculativeConfig::is_dspark_algorithm(
+ options_.speculative_algorithm()))
```

## R03 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 11:58:55｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:384｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494023](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494023)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] CP 下 Engram lookup 对全局行 all-reduce 后只保留 1/cp，通信量约为所需的 2×cp 倍

**后果**：cp>1 的 V4.1 prefill 每个 engram 层多付出约 2cp 倍的集合通信量（性能，无正确性问题）。

**触发条件**：cp\_size>1 + 该层在 engram\_layer\_ids 中；每个 prefill chunk、每个 engram 层一次。

**代码**（deepseek\_v41\_engram.cpp:382-387）：

```cpp
rows = rows.view({hashes.size(0), cols, dim}) * owned.unsqueeze(-1);  // hashes.size(0)=全局行数
rows = parallel_state::reduce(rows, lookup_group_);   // cp>1 时 lookup_group_=moe_ep_group_（cp×tp 世界）
if (row_indices.defined()) {
  rows = rows.index_select(0, row_indices);           // 只保留本 rank 的 1/cp 行
}
```

模型层把全局 tokens 传入 decoder layer（deepseek\_v41.h:240-243 传 `tokens` 而非 `local_tokens`），每个 rank 对全部 T 行算哈希、对自己的表分片产出稀疏贡献，all-reduce 求和后各自丢弃 (cp-1)/cp 的行。

**量级**：每次 all-reduce 载荷 = T × cols × dim × 2B（bf16）；理想（每 rank 只收本 rank 行）为 T/cp × cols × dim，比例 ≈ 2cp。

**根因**：正确性要求"每 rank 的局部行拿到全量表分片求和"，all-reduce 是最简单实现，未做通信量裁剪。cp=1 时 lookup\_group\_ 仍为 tp\_group\_，与非 CP 路径完全一致，无回归。

**建议**：先补一条量化日志（T、cols、dim、组大小）再评估 reduce\_scatter 变体（zigzag 行不连续，需二次置换）；若 engram 层数少且 T 中等，可维持现状并在注释记录取舍。

**验证**：cp=4/8 prefill 通信量 profile，对比 all-reduce 实测字节与公式。

（证据边界：通信量公式与丢弃比例由源码推导；无端到端带宽实测，故定 P3。）

### 页面差异片段

```diff
   rows = rows.view({hashes.size(0), cols, dim}) * owned.unsqueeze(-1);
   rows = rows.to(hidden.scalar_type());
-  rows = parallel_state::reduce(rows, tp_group_);
+  rows = parallel_state::reduce(rows, lookup_group_);
```

## R04 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:01:51｜xllm/core/kernels/mlu/bangc/src/deepseek_v41/gather.mlu:62｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494061](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494061)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] gather.mlu 每 task 全量扫描 topk 索引流，taskDim×count×8B 冗余 GDRAM 读

**后果**：decode/verify 热路径上每次 `deepseek_v41_decode_gather` 调用多 ~count×8B×taskDim 的 GDRAM 读；index\_topk=2048 时约 512KB 额外读（对比有用行数据约 (win+2048)×1KB），约 25% 额外流量。

**触发条件**：每次 decode\_gather 调用（graph 路径每 query 行每层每 decode/verify 步，attention.cpp:506-522 按行循环调用）。

**代码**（gather.mlu:51-78；launch 网格 gather.mlu:104 `grid={32,1,1}` → taskDim=32）：

```cpp
int32_t valid_count = 0;
for (int64_t begin = 0; begin < count; begin += kIndexTileSize) {
  ...
  __memcpy(index_tile, topk + begin, size * sizeof(int64_t), GDRAM2NRAM);
  for (int32_t index = 0; index < size; ++index) {
    const int64_t logical = index_tile[index];
    if (logical < 0) { continue; }
    const int32_t col = win + valid_count++;
    if (col % taskDim != taskId) { continue; }
    ...
  }
}
```

32 个 block task 各自把整条 topk 流搬入 NRAM 全量扫描，只为写出属于自己的 ~count/32 行。

**根因**：以全量重复扫描换取无 barrier、无 scratch 的正确性——`valid_count` 在所有 task 上一致、`col%taskDim` 稳定分工（该正确性已被 sparse\_verify\_test 的 12 种 count 边界 × 4 种 pattern 覆盖验证）。注释已声明取舍（"Block tasks cannot share a prefix scan… without scratch or barriers"），属知情选择。

**建议**：保留正确结构；有 profile 证据时再做"taskId 0 单独前缀扫描写 GDRAM scratch、其余 task 读 scratch"的优化；当前至少在注释补上冗余流量公式（count×8×taskDim）。

**验证**：cnrt counter 对比优化前后 gather kernel 的 GDRAM 读字节。

（证据边界：冗余量由源码与网格常量推导；未实测带宽影响。）

### 页面差异片段

```diff
+ __memcpy(index_tile, topk + begin, size * sizeof(int64_t), GDRAM2NRAM);
+    for (int32_t index = 0; index < size; ++index) {
+ const int64_t logical = index_tile[index];
+ if (logical < 0) {
```

## R05 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:04:24｜xllm/core/kernels/mlu/bangc/src/deepseek_v41/gather.mlu:51｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494064](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494064)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] commit 264b5f907 第三条 bullet 声称的改动在本 MR 中不存在（commit message 与 diff 失真）

**后果**：误导变更追溯（reviewer、后续 cherry-pick/回滚、release note）；若该改动本应随移植落地（cherry-pick 丢失），DiT scheduler 测试可能存在未被满足的链接依赖。

**证据**（commit 264b5f907 message 第三条 bullet）：

```text
- Include request_timing in the DiT scheduler test link group.
```

而 `git show --stat 264b5f907` 实测仅改 3 个文件（CMakeLists.txt / deepseek\_v41\_sparse\_verify\_test.cpp / gather.mlu，+269/-20）；整个 MR 33 个变更文件中不存在任何 DiT scheduler / request\_timing / scheduler 测试链接改动（`git diff --name-only b4d4316..57fe948 | grep -i "dit\|request_timing\|scheduler"` 无命中）。

**根因**：多分支并行开发时 commit message 未随 cherry-pick/拆分裁剪（264b5f907 无 cherry-pick 标注，最可能是源分支上下文带入）。

**最小修复**：在 MR 描述中声明该 bullet 不属于本 MR（或 rebase 时修正 message）；若确属丢失改动，补上对应提交。

**验证**：对照源分支确认 request\_timing 改动的真实去向。

（证据边界：message 与 MR 内容的不一致为双证据事实；"cherry-pick 丢失 vs 上下文残留"成因未定。）

### 页面差异片段

```diff
  __memcpy(output + col * 512, row, 512 * sizeof(int16_t), NRAM2GDRAM);
   }
+
+  // Block tasks cannot share a prefix scan. Scan the small index stream once
```

## R06 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:06:59｜xllm/models/llm/mlu/deepseek_v41_dspark.h:50｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494067](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494067)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] draft main\_proj 输入宽度（hidden×dspark\_num\_layers）与 target 捕获层数之间无启动期一致性校验

**后果**：checkpoint 的 `dspark_target_layer_ids` 数量 ≠ `dspark_num_layers` 时，在首次 `write_context_kv` 的 matmul 处以形状错误崩溃——fail-fast 但晚，且报错与根因距离远（不点名两个配置项）。

**触发条件**：如 2 个捕获层 + 3 个 draft 层（draft\_model\_config\_test.cpp:662-703 的测试夹具正是这个组合）。

**代码**（deepseek\_v41\_dspark.h:47-62 构造；:131-142 消费）：

```cpp
const int64_t capture_count = args.dspark_num_layers();
main_proj_ = register_module(
    "main_proj",
    layer::ReplicatedLinear(args.hidden_size() * capture_count,
                            args.hidden_size(), /*bias=*/false, ...));
...
torch::Tensor projected = main_proj_->forward(target_hidden);  // target 的 aux_hidden_states
```

target 侧宽度 = `layers_to_capture.size()`（checkpoint 缺省回退 EAGLE-3 默认 3 层，aux\_hidden\_capture.h:93-96）；draft 侧 `configure_block_diffusion_model` 还会清空 draft args 的 `layers_to_capture`（draft\_model\_config.cpp:321 `args.layers_to_capture({})`），使 draft 无法自行核对。`verify_dspark_weights` 只查 main\_proj 是否加载，不查宽度匹配。

**根因**：跨引擎契约（target 捕获宽度 :unknow\_emoji: draft 投影输入宽度）没有单一 owner 校验，只靠 checkpoint 作者自觉。

**最小修复**：`write_context_kv` 增加 `CHECK_EQ(target_hidden.size(1), args.hidden_size() * capture_count)`，报错文案点名 `dspark_target_layer_ids` 数量与 `dspark_num_layers` 需一致；更优为 draft engine 初始化时用 target args 交叉校验。

**验证**：构造 layers\_to\_capture={0,1} + dspark\_num\_layers=3 的 checkpoint，断言启动期报错而非首次前向报错。

（证据边界：不一致组合的晚期失败由形状推导；是否存在真实 checkpoint 触发未知。）

### 页面差异片段

```diff
+ const int64_t capture_count = args.dspark_num_layers();
+    CHECK_GT(capture_count, 0)
+        << "DeepSeek-V4.1 DSpark requires dspark_num_layers > 0.";
+    CHECK_EQ(args.n_layers(), args.dspark_num_layers())
```

## R07 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:09:23｜tests/core/runtime/mlu_graph_executor_test.cpp:599｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494070](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494070)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] V4.1 grouped verify 缺 executor 级集成测试（现有 MTP/DFlash2 executor 用例不经过 group\_verify\_input 路径）

**后果**：「CP+DSpark 组合被放行」这一核心 claim 的最外层（executor 与模型分组逻辑之间的接缝）未被直接证明；接缝错位只能在真机 serving 暴露。

**证据**（mlu\_graph\_executor\_test.cpp 两条既有用例的算法参数，:599 / :677）：

```cpp
.speculative_algorithm("MTP")     // :599 用例
.speculative_algorithm("DFlash2") // :677 用例
```

这两条验证的是通用 verify graph 的 executor 接线；V4.1 特有的 `spec_verify_width>1 → group_verify_input 重组 + HashState query_width 分组`（deepseek\_v41.h:280-305 + :138-143）只在 planner 测试（FlatVerifyBucketsPreserveWholeRequests）、layer 级 graph replay（VerifyRejectionRetainsInteriorCompressorRows 后半段）与 model 级元数据测试（VerifyMetadataGroupsRequestsAndReusesGatherStorage）分片覆盖——三段拼起来没有一条端到端「executor 驱动的 V4.1 DSpark verify graph 回放」用例。

**根因**：测试按层分片编写，未按端到端风险面收口。

**建议**：在 mlu\_graph\_executor\_test 增加 V4.1 DSpark verify graph 用例（可复用 MTP 用例骨架换 model\_type 与 spec\_verify\_width；最小覆盖：捕获 + 两次 replay + 拒绝后 resume）。

**验证**：新用例本身。

（证据边界：这是覆盖缺口而非已证缺陷——三段分片测试当前均通过。）

### 页面差异片段

```diff
       .block_table_token_span(/*value=*/64)
```

## R08 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:11:53｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:162｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494106](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494106)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] write\_context\_kv 在每个 draft 层重复执行 nonzero 同步与行过滤（同一 cache\_slots 计算 N 次）

**后果**：draft 热路径上每层一次隐式设备同步 + 冗余过滤（典型 3 层 = 3 次同步）；正确性无误（各层结果一致）。

**触发条件**：每次 draft 上下文写入（每个被接受的 verify 块一次），N 个 draft 层各一遍。

**代码**（deepseek\_v41\_attention.cpp:160-176；层循环调用点 deepseek\_v41\_dspark.h:145-152）：

```cpp
void DeepseekV41AttentionImpl::write_context_kv(...) {
  auto valid = torch::nonzero(cache_slots.ge(0)).flatten();   // 数据依赖形状 → 隐式设备同步
  auto rows = hidden.index_select(0, valid);
  ...
  flat_cache(cache.get_swa_cache())
      .index_copy_(0, cache_slots.index_select(0, valid).to(torch::kInt64), kv);
}
```

`DeepseekV41DSparkModelImpl::write_context_kv` 对所有 draft 层循环调用它，而 `cache_slots`/`positions` 在层间完全相同。

**根因**：接口按"每层自足"设计，未把 batch 级不变量（同一 cache\_slots）外提。

**最小修复**：在 `DeepseekV41DSparkModelImpl::write_context_kv` 先算一次 valid 索引并 `.to(kInt64)`，作为参数传入每层。

**验证**：draft 前向 profile 中同步次数对比（cnrt stream sync counter）。

（证据边界：冗余由源码直接证明（同一 slots、层循环）；性能影响未实测，量级小故 P3。）

### 页面差异片段

```diff
-                                                Csa2Context& shared) {
+void DeepseekV41AttentionImpl::write_context_kv(
+ const torch::Tensor& hidden,
+ const torch::Tensor& positions,
```

## R09 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:14:19｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:45｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494112](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494112)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[Question] CP+Engram 组合的前向正确性没有任何多 rank 测试

**问题**：CP 下 Engram lookup 的正确性链——全局 tokens 哈希（每 rank 复算）→ 每 rank 只对自有表分片产出贡献（`owned` 掩码，engram.cpp:367）→ EP 组 all-reduce 求和 → `index_select(local_row_indices)` 取本 rank 行 → `view_as(hidden)` 对齐局部行数——目前只被静态审查覆盖，没有测试背书。

**现状**：executor/planner 的 cp 用例全部用 MTP/DFlash2 模型；engram\_test 的 query\_width 用例是单 rank（world\_size=1）。

**需要什么证据关闭**：一条 cp>1、含 engram 层的 V4.1 多 rank 前向等价测试（actual vs cp=1 参考输出 allclose）。

（静态审查已逐步核对 shape 传导，未发现破绽；此条是验证缺口，不是已证缺陷。）

### 页面差异片段

```diff
- tp_group_(context.get_parallel_args().tp_group_),
+ lookup_group_(context.get_parallel_args().cp_size() > 1
+                        ? context.get_parallel_args().moe_ep_group_
+                        : context.get_parallel_args().tp_group_),
```

## R10 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 12:16:47｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:508｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494114](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494114)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[Question] graph 路径 `shared.topk.at(seq)[row]` 依赖 select\_\* 写入 2D 形状——当前正确，但契约未在头文件声明

**消费侧**（deepseek\_v41\_attention.cpp:507-509）按 2D 索引取 `shared.topk.at(seq)[row].flatten()`；写入侧实测（git show 核对）：`select_decode_indices` 写 `indices.view({1, -1}).slice(1, 0, k)`（{1,k}）、`select_indices` 写 `{count, k}`、`select_candidate_indices` 同族——三个写入点当前均 2D。

**风险**：旧代码用 `.flatten()` 对 1D/2D 都兼容，新代码收紧为必须 2D——属隐性契约收紧；后续新增写入点若给 1D 会静默错位。

**最小修复**：在 `Csa2Context::topk` 声明处注释"每个元素必须是 {rows, k} 二维"，或消费侧退回兼容写法。

### 页面差异片段

```diff
-          lengths.slice(0, seq, seq + 1));
+ for (int64_t row = 0; row < count; ++row) {
+ auto topk = ratio_ > 0
+                        ? shared.topk.at(seq)[row].flatten()
```

## R11 — 马晓龙 (maxiaolong.maxwell)｜2026/11/4 13:02:39｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:210｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3494190](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/681#discussion_r3494190)
- 状态证据：仅见处理与解决操作按钮，实际解决状态未核实。

### 评论原文

[P3] CP 下每个 kv-source 层发起两次背靠背 all-gather（kv 行 + compressor 投影行），可合并为一次

**后果**：CP prefill 的每个压缩源层每 chunk 多一次集合通信（性能）。

**触发条件**：`source_==layer_id_ && ratio_>0` 的层（V4.1 压缩源层），每 chunk 每层两次。

**代码**（deepseek\_v41\_attention.cpp:207-216）：

```cpp
if (cp_context != nullptr) {
  kv_gather = mlu_v4_cp::launch_gather_rows(kv, *cp_context);
  if (raw.defined()) {
    torch::Tensor projection =
        scores.defined() ? torch::cat({raw, scores}, -1) : raw.contiguous();
    compressor_gather =
        mlu_v4_cp::launch_gather_rows(projection, *cp_context);
  }
}
```

kv（行数=本地行，宽=head\_dim×heads）与 projection（同行数）分两次 launch\_gather，各自走 `launch_gather + finish_gather + index_select(restore_indices)`；两次的行集合与 rank 布局（tokens\_per\_rank）完全相同。

**根因**：复用 V4 既有 gather 原语时按张量逐个发起，未利用"同 leading dim 的 tensor 拼一次 gather 再 unpack"。

**最小修复**：`torch::cat({kv, projection}, 0)` 一次 all-gather，restore 后按行数切回两段（注意 dtype 一致、均为本地行连续）；或给 `launch_gather_rows` 加多张量拼接重载。非源层只有 1 次 gather，不受影响。

**验证**：cp=4 单层 micro-benchmark，合并前后每层 all-gather 次数与耗时。

（证据边界：两次 gather 行集合一致性由源码证明（同一 cp\_context、同为本地行张量）；性能收益未实测。）

### 页面差异片段

```diff
+  std::optional<parallel_state::GatherAsyncCtx> kv_gather, compressor_gather;
+ if (cp_context != nullptr) {
+    kv_gather = mlu_v4_cp::launch_gather_rows(kv, *cp_context);
+ if (raw.defined()) {
```

## A01 — 作者 ext.luolei15 说明｜2026年11月4日 10:16:36｜全局评论｜无明确解决状态

- 类型：独立根评论，非行级线程回复。

为 DeepSeek-V4.1 原生 MLU 实现接入上下文并行（CP）和 DSpark 投机解码，复用现有投机执行框架，并修复验证阶段的缓存与特征捕获问题。

* 上下文并行：支持 prefill query 的 zigzag 切分，适配全局 KV、压缩和索引计算及 MoE 路由；Engram 在 CP 场景下按 EP 分片，并在归约后选取本地行。
* DSpark 接入：新增原生 BF16 draft 模型，支持目标模型特征投影、上下文 KV 写入、Markov head 和权重加载；将 CP 目标模型输出及辅助特征恢复为全局顺序后交给 draft。
* 验证正确性：支持按请求分组的因果验证与 Engram 历史计算，保留候选拒绝后所需的压缩状态；过滤 KV gather 中被屏蔽的索引，返回实际有效长度；修正辅助特征捕获时机，在 Engram 注入前采集层输入。
* Graph 与配置：支持 CP 下的 grouped graph，适配多组 block table 和验证宽度；统一 V4.1 的 DSpark 校验，并在 MLU 初始化时拒绝自适应投机解码配置。

新增和更新测试，覆盖 draft 配置、多种 CP 大小下的 Graph 规划、验证分组、候选拒绝、稀疏 gather、Graph replay 及 Engram 特征捕获；同步更新 Engram host offload 中英文文档。本次仅生成 PR 描述，未执行构建及测试。

当前 MLU DSpark draft 使用 eager 执行；DeepSeek-V4.1 投机解码仅支持 DSpark，MLU 暂不支持自适应投机解码。

## S01 — 冯艳 (fengyan.119) 自动审查总结｜2026年11月4日 11:15:04｜全局评论｜无明确解决状态

- 类型：独立根评论，非行级线程回复。

### 🤖 自动审查小结：**LGTM，这代码看着真舒服！**

本轮静态代码审查未发现需要修改代码的问题。

## S02 — 马晓龙 (maxiaolong.maxwell) 评审总结｜2026年11月4日 12:19:16｜全局评论｜无明确解决状态

- 类型：独立根评论，非行级线程回复。

### 教学式深度审查小结（MR#681 · DeepSeek v4.1 CP + DSpark on MLU）

对 base b4d4316 → head 57fe948（33 files，+1612/-362，6 commits）做了源码级静态审查 + git 语义考古，逐条发布行级评论。

**总体结论：建议合入，无阻塞项**（P0/P1/P2 均未发现；8 条 P3 建议改进 + 2 条 Question）。

重点面结论：

* 三个 bugfix（264b5f907 masked index、9a4f288bc capture 时序、850700146 adaptive 拒绝）经反例构造逐一验证为**修复彻底**；
* 57fe948「simplify speculative validation」经三代语义考古确认为**真死代码删除**（`is_v41_python_dspark_spec` head 零引用），行为差异仅为 F1 所述的准入放宽；
* 拒绝路径 parked 行回滚正确（state\_rows=32 全块暂存 + CHECK 自守）；
* MLU adaptive 门禁位置覆盖 flags+json 两入口。

建议作者优先处理顺序：F1（准入平台限定，一行改动）→ F5（MR 描述声明 bullet 失真）→ Q1（补 CP+Engram 多 rank 等价测试）→ 其余按需。

评论列表：F1 准入外溢 / F2 Engram all-reduce 通信量 / F3 双 all-gather 可合并 / F4 gather.mlu 冗余扫描 / F5 commit message 失真 / F6 draft 投影宽度无校验 / F7 executor 级测试缺口 / F8 write\_context\_kv 重复同步 / Q1 CP+Engram 多 rank 测试 / Q2 topk 2D 隐性契约。
