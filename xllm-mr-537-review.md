# xLLM MR #537 — Review 意见与修复交接

- 来源：[MR #537](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/537)
- 标题：`bugfix: fix mtp sampling logits publication.`
- 分支：`bugfix/fix-mtp-logits` → `main`
- Head：`b8fb9d45f315d90ba955621a2a382b484240652e`（页面提交链接）；未另查代码仓库。
- 页面状态：Open / Reviewing；1 commit、7 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。
- 提取时间：2026/10/20 14:22:08（Asia/Shanghai）；已登录 Chrome 刷新加载后读取。
- 筛选：不按作者、时间、解决状态过滤；包含作者说明、评审总结、全部会话中已发布行级意见与回复；不含提交和 reviewed 事件。
- 清单：15 条根评论（1 条作者说明、1 条评审总结、13 条行级意见），1 条回复。页面 Conversation 14 不用作评论总数。
- 范围：检查了会话中的全部讨论，未见折叠讨论、隐藏同类评论、过期线程、分页或加载更多入口；未检查 Files Changed 中未在会话展示的内容，不含草稿。
- 状态说明：各行级讨论仅见“Resolved”和“No Processing Required”操作按钮，未见明确解决事件，均标为“状态未核实”。两条总体评论未见解决状态标记。
- 原文说明：作者和 reviewer 的技术判断及测试声明未经导出者独立验证。R02 的回复对根评论作了复核收敛，须连同阅读。最后一条 refactor 建议的单行伪代码含 `/dim=/`，且 `sample_idxes` 未在签名中给出；原样保留，不替 reviewer 补写。

## 行动索引（导出者摘要）

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：根因、修复内容及验证 | 未见解决状态标记 |
| R02 | 正确性：全行 publish 行序；含马晓龙复核回复 | 状态未核实 |
| R03 | 正确性：未定义 sample_idxes | 状态未核实 |
| R04 | 正确性：worker logits 赋值路径 | 状态未核实 |
| R05 | 正确性：移除 dense mask 重复叠加的前提 | 状态未核实 |
| R06 | 健壮性：dtype 转换与旧引用 | 状态未核实 |
| R07 | 测试缺口：等长非顺序索引 | 状态未核实 |
| R08 | 评审总结：未发现阻塞项，4 个 P2 与 2 个 Question | 未见解决状态标记 |
| R09 | P2：dense mask applied 契约断言 | 状态未核实 |
| R10 | P2：rec_sampler 平行实现回写 | 状态未核实 |
| R11 | P2：suffix_worker 丢弃返回值 | 状态未核实 |
| R12 | P2：CPU 与 dflash 回归覆盖 | 状态未核实 |
| R13 | Question：ForwardOutput.logits dtype 提升 | 状态未核实 |
| R14 | Question：MTP logprobs 口径变化 | 状态未核实 |
| R15 | minor / refactor：复用 use_sample_indices | 状态未核实 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月20日 10:28:03｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

#### 问题根因

MLU 的 top-k/top-p 算子返回新 Tensor，但 Sampler::forward 在全行采样时未将其回写给调用方；同时，worker 在采样前已将旧 Tensor 句柄保存到 ForwardOutput.logits。因此，target 正常采样使用过滤后的分布，MTP 验证却读取旧 logits，使

本应为零概率的候选重新获得非零概率。

受控用例中，top_k=20、top_p=0.95 应仅保留 18 个候选，修复前 verify 实际使用全部 64 个候选，过滤集合外仍有约 23.23% 的概率质量。

#### 修改内容

• 统一回写 Sampler 处理后的 logits，覆盖全行、子集及 greedy 采样路径，并保持返回概率的原始 dtype。

• 将 LLM/VLM 的 ForwardOutput.logits 赋值移至采样完成后。

• 移除 MTP 验证阶段重复叠加的 dense mask。

• 新增回归测试，覆盖过滤候选拒绝、接受前缀、bonus token，以及 logits 回写和子集行保留行为。

#### 验证结果

• 当前 main 基线分支：MLU 编译通过，采样及 JSON 相关测试 74/74 通过，无跳过。

• 原 Qwen 集成分支：模型测试 24/24 通过；原异常题完成 15 题 × 5 次回放，人工审读范围内未确认同类文字混入或机械循环。

• 标准 64 题 E2E：58/64（90.625%），对照基线为 55/64。

本次修复确认消除了采样分布传递错误；尚不能据此认定历史所有生成异常均由该缺陷唯一导致。CUDA/NPU 后端未运行验证。

## R02 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:13｜xllm/core/framework/sampling/sampler.cpp:31｜状态未核实

- 类型：行级意见；根评论，1 条回复。
- 线程锚点：`discussion_r3467606`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
 #include "sampling_params.h"

 namespace xllm {
+namespace {
```

### 评论原文

**1. 正确性 — sampler.cpp publish_logits 函数中，当**
位置：`xllm/core/framework/sampling/sampler.cpp:31`

sampler.cpp publish_logits 函数中，当 selected_token_idxes.numel() == sample_idxes.numel() 时直接执行 logits = sample_logits，存在行顺序错乱的风险。如果 sample_idxes 是一个非排序的排列（例如 {2, 0, 1}），sample_logits 的行顺序与原始 logits 不同，直接赋值会导致行对应关系被打乱。原代码使用 index_copy_ 按索引写回，能保证行顺序正确

建议：在判断条件中增加对 sample_idxes 是否为连续递增序列（即 {0, 1, ..., n-1}）的校验，或者直接始终使用 index_copy_ 以确保行顺序的正确性。

### 回复 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:09:10

- 回复 R02；锚点：`discussion_r3467675`；所属线程状态未核实。

补充新证据，支持您对「numel 相等直接赋值」路径的关心，并把结论收敛到可文档化的措辞：

**复核结论**：非排序排列打乱行序在现结构不可达——`use_sample_indices` 与 `publish_logits` 用同一 numel 判据（sampler.cpp:69-70 与 :36），numel 相等 ⇒ 从未 index_select（:71-72 只有子集才 index_select）⇒ 无乱序路径。

**新证据**：numel 相等谓词的真实语义是「sample_idxes 覆盖 logits 的全部行」而非「sample==selected」，且有在库调用方依赖这一点——dflash 构造 draft query 时 selected 是稀疏集、sample 是 arange（dflash_worker_impl.cpp:1412-1417）：

```cpp
query_input.sampling_params.selected_token_idxes =
    safe_to(specBuilder::make_cpu_int_tensor(selected_idxes), idx_options, true);
query_input.sampling_params.sample_idxes =
    torch::arange(static_cast<int64_t>(selected_idxes.size()), idx_options);
```

两者 numel 相等 ⇒ 全行采样 lm_head 输出的所有行。因此「恒用 index_copy_」的修法会给全行热路径加一次全量拷贝；更轻的修法是把前提显式化：在 sampler.cpp:36 或 sampler.h 注释写明「全行分支前提：sample_idxes 覆盖全部行（numel 相等即视为全行）」。

**替代测试建议**（也回应 #3467611）：与其构造不可达的乱序等长 sample_idxes，不如断言「全行 publish 不改变行序」——forward 前记录 `logits.index_select(0, torch::randperm(n))` 的内容指纹，forward 后比对（现有 SubsetSamplingPreservesOtherRows 已覆盖子集侧）。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:13｜xllm/core/framework/sampling/sampler.cpp:32｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467607`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff

 namespace xllm {
+namespace {
+
```

### 评论原文

**2. 正确性 — sampler.cpp 在 forward 函数的 all-greedy 快速路径中新增了**
位置：`xllm/core/framework/sampling/sampler.cpp:32`

sampler.cpp 在 forward 函数的 all-greedy 快速路径中新增了 publish_logits(logits, sample_logits, params) 调用。如果 use_sample_indices 为 false，params.sample_idxes 可能是未定义的 Tensor。在 publish_logits 内部调用 params.sample_idxes.numel() 时，未定义 Tensor 返回 0。若此时 selected_token_idxes.numel() 也为 0，会触发 logits = sample_logits；若 selected_token_idxes.numel() 不为 0，则会进入 index_copy_ 分支并传入未定义的 sample_idxes，可能导致崩溃

建议：在 publish_logits 开头增加对 sample_idxes 是否 defined 的检查，若未定义则直接 return，避免对未定义 Tensor 进行操作。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:13｜xllm/core/framework/sampling/sampler.cpp:33｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467608`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
 namespace xllm {
+namespace {
+
+void publish_logits(torch::Tensor& logits,
```

### 评论原文

**3. 正确性 — llm_worker_impl.cpp 将 output.logits = logits; 从原先的 if**
位置：`xllm/core/framework/sampling/sampler.cpp:33`

llm_worker_impl.cpp 将 output.logits = logits; 从原先的 if (sampling_params.selected_token_idxes.defined()) 块中移除，并移至后续的另一个条件块中。如果存在 selected_token_idxes.defined() 为 true 但后续条件块未执行的分支，output.logits 将不会被赋值，导致上层获取不到采样后的 logits

建议：确认所有需要输出 logits 的分支都能执行到 output.logits = logits;，或者将其放在更外层的公共路径中，确保不会遗漏赋值。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:13｜tests/core/framework/sampling/sampler_filter_mask_test.cpp:18｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467609`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff

 #include <gtest/gtest.h>

+#include <limits>
```

### 评论原文

**4. 正确性 — mtp_worker_impl.cpp 移除了 target_logits = target_logits +**
位置：`tests/core/framework/sampling/sampler_filter_mask_test.cpp:18`

mtp_worker_impl.cpp 移除了 target_logits = target_logits + target_filter_mask.view(...) 操作。如果 target_logits 在后续流程中未经过 sampler 的 apply_logits_processors（该函数内部会应用 filter_mask），则 target_logits 将不再被 filter_mask 过滤，可能导致被屏蔽的 token 仍然参与计算

建议：确认移除该手动加法后，target_logits 确实会流入 sampler 并由 apply_logits_processors 统一处理 filter_mask；如果存在跳过 sampler 的路径，需要保留该手动过滤逻辑。

## R06 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:13｜xllm/core/framework/sampling/sampler.cpp:34｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467610`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+namespace {
+
+void publish_logits(torch::Tensor& logits,
+ const torch::Tensor& sample_logits,
```

### 评论原文

**5. 健壮性 — sampler.cpp publish_logits 函数在 selected_token_idxes.numel()**
位置：`xllm/core/framework/sampling/sampler.cpp:34`

sampler.cpp publish_logits 函数在 selected_token_idxes.numel() != sample_idxes.numel() 分支中执行 logits = [logits.to](http://logits.to)(sample_logits.scalar_type())。如果两者 dtype 不同，.to() 会创建一个新的 Tensor 并将引用重绑定。此时后续的 index_copy_ 修改的是新 Tensor，而原始 Tensor 数据未变。如果调用方或其他地方仍持有旧 logits 的引用，将无法看到更新

建议：在 sampler.h 的注释中进一步明确 publish_logits 可能导致 Tensor 底层存储变更（不仅仅是 rebind），调用方必须在函数返回后重新读取 logits，避免使用缓存的旧引用。

## R07 — 邓英旭 (dengyingxu1)｜2026/10/20 10:42:14｜xllm/core/framework/sampling/sampler.cpp:35｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467611`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+
+void publish_logits(torch::Tensor& logits,
+ const torch::Tensor& sample_logits,
+ const SamplingParameters& params) {
```

### 评论原文

**6. 测试缺口 — sampler_test.cpp 中的 SubsetSamplingPreservesOtherRows**
位置：`xllm/core/framework/sampling/sampler.cpp:35`

sampler_test.cpp 中的 SubsetSamplingPreservesOtherRows 测试用例，sample_idxes 设置为 {2, 0}（非连续、非递增），但 selected_token_idxes.numel() (3) 不等于 sample_idxes.numel() (2)，因此未覆盖到 publish_logits 中 numel 相等且 sample_idxes 非顺序排列的场景

建议：增加一个 sample_idxes 为非顺序排列（如 {2, 0, 1}）且其 numel 与 selected_token_idxes.numel() 相等的测试用例，以验证该边界条件下 logits 的行顺序是否正确。

## R08 — 马晓龙 (maxiaolong.maxwell)｜2026年10月20日 11:13:16｜MR 总体评论｜未见解决状态标记

- 类型：评审总结；根评论，0 条回复。

### 评论原文

#### Review 总结（bugfix: fix mtp sampling logits publication）

审查范围：base 51e15239e → head b8fb9d45f，7 文件 +242/-14；逐行对照 head 物化源码验证。

**总体结论：未发现阻塞项。** 修复主链路逐条核实成立：

* 三出口 publish 齐全（greedy 快路径 sampler.cpp:126 / greedy+return_probs :143 / 主路径 :150），reranker 早退在 publish 之后，无第四出口；
* llm/vlm worker 赋值后移无漏赋值（386→455 / 238→242 之间无 return；skip_sampling_for_logits_only 路径仍赋原始 logits，与旧行为一致）；
* MTP 去双叠前置条件成立：validate 的 filter_mask 与 target 采样共用同一份 sampling_params，经 run_worker_no_sync → LLMWorkerImpl 采样时应用并经 publish 发布；
* prob_dtype 快照为纯健壮性增强（与旧 `.to(logits.dtype())` 在旧代码路径下等价）。

**做得好**：

1. 把「processors 可能返回新 Tensor」升格为 sampler.h 显式契约注释，方向正确；
2. 量化描述精确（top_k=20/top_p=0.95 → 18 候选、集合外 23.234%，独立手算复核吻合）；
3. 测试同时夹击契约层（3 个 publish 测试）与语义层（2 个 rejection 测试）；
4. dtype 提升行为写进了测试断言，有锁定。

**建议按优先级处理**（均非阻塞）：P2 x4——dense mask 契约缺 applied 断言 / rec_sampler 平行代码保留旧回写模式 / suffix 丢弃返回值与新契约矛盾 / 4/5 新测试 MLU-gated（CPU CI 仅 1 条覆盖）；Question x2——ForwardOutput.logits MLU 下静默 fp32 提升 / MTP logprobs 口径变化未声明。详见各条评论。

另有若干证据不足的疑点未列出（如 MLU 闭源 kernel 的 top-p 边界语义、物化集外消费者 dtype 假设），如需可单独讨论。

## R09 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:24:35｜xllm/core/runtime/mtp_worker_impl.cpp:4503｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467721`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
           << "MTP JSON filter mask row count mismatch";
  CHECK_EQ(target_filter_mask.size(1), vocab_size)
```

### 评论原文

[P2] dense mask「只加一次」契约缺少 bitmask 路径已有的 applied 断言

**后果**：删除 validate 侧双叠后，dense `filter_mask`「已被 target sampler 应用」仅靠约定，无运行时守卫、无回归测试。未来任何改动让携带 dense mask 的 target 前向绕过 sampler（如 `skip_sampling_for_logits_only=true` 或新增旁路），validate 不会失败——被屏蔽 token 静默不过滤，回到本 MR 修复前的错误接受行为。

**不对称证据**（mtp_worker_impl.cpp:4489-4505，所在函数 `MTPWorkerImpl::validate`）：

```cpp
if (target_filter_bitmask.defined()) {
  CHECK(target_output.filter_bitmask_applied_to_logits)   // bitmask 路径有契约断言
      << "packed target logits must be filtered by the target sampler";
} else if (target_filter_mask.defined()) {
  CHECK_EQ(target_filter_mask.dim(), 2) ...               // dense 路径只剩形状检查
}
```

bitmask 的标志位由 llm_worker_impl.cpp:436-437 在采样时设置；`ForwardOutput` 只定义了 bitmask 版标志（forward_params.h:658-661 `bool filter_bitmask_applied_to_logits = false;`）。

**根因**：本 MR 把 dense mask 的应用责任从 validate 下沉到 sampler（语义迁移正确），但只迁移了行为，没有迁移 bitmask 路径已有的契约执行机制；被删除的 3 行双叠代码也没有回归测试守护——重新引入双叠或漏叠都不会被测试抓住。

**最小修复**：仿照 `filter_bitmask_applied_to_logits` 增加 dense 版标志（llm_worker 设置、mtp validate CHECK），或复用同一标志改名 `filter_applied_to_logits` 两条路径都置位。

**验证**：单测构造 `filter_mask` + `skip_sampling_for_logits_only=true` 的 target 输出喂给 validate，断言 CHECK 触发（fail-fast 而非静默）。

（当前主链路 MTP validate 的 target 恒走采样，已核实不会触发，故为防御缺口而非现行 bug。）

## R10 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:30:08｜xllm/core/framework/sampling/sampler.cpp:150｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467735`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
- if (use_sample_indices) {
-    logits.index_copy_(/*dim=*/0, params.sample_idxes, sample_logits);
-  }
+ publish_logits(logits, sample_logits, params);
```

### 评论原文

[P2] rec_sampler 内联复制仍是修复前的旧回写模式——同类 bug 在平行代码存活

**后果**：OneRec 管线走 beam-search+logprobs 分支时，`apply_top_k_top_p` 过滤后的 sample_logits 不回写给调用方——正是本 MR 在 `Sampler::forward` 里修掉的 bug 原样保留；子集回写前还缺 dtype 对齐（MLU kernel 输出恒 fp32，apply_topkp.cpp:46-47）。

**代码对照**（rec_sampler.cpp:256-260，本 MR 未同步这份重复实现）：

```cpp
apply_top_k_top_p(
    sample_logits, sample_temperatures, sample_top_k, sample_p);
if (use_sample_indices) {
  logits.index_copy_(/*dim=*/0, params.sample_idxes, sample_logits);
}
```

而修复后的 `Sampler::forward` 同位置（sampler.cpp:148-150）已是 `publish_logits(logits, sample_logits, params)`——全行 `logits = sample_logits`，子集先 `.to()` 对齐再 `index_copy_`。

**触发条件**：`use_beam_search && all_random_sample && logprobs && max_top_logprobs > 0` 的反面组合（rec_sampler.cpp:204-207），且 top_k/top_p 生效（MLU 任意组合，或任意后端仅定义其一）。rec 侧调用方是否在 forward 后读 logits 需作者确认。

**最小修复**：把 rec_sampler.cpp:258-260 替换为复用 `publish_logits` 语义（可将 `publish_logits` 提为 `Sampler` 静态方法供两处共用），消除双份实现漂移。

**验证**：把 sampler_filter_mask_test.cpp 的 publish 断言用例套到 `RecSampler::forward`（OneRec 策略 + top_k/top_p）上跑一遍。

## R11 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:30:52｜xllm/core/framework/sampling/sampler.h:48｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467737`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
  // bitmask or the dense filter mask) and returns the row subset to sample
- // from: `logits` itself unless the parameters select sample indices.
+ // from. Masks or sample indices may produce a new Tensor; callers must use
+ // the returned scores.
```

### 评论原文

[P2] suffix_worker 丢弃 apply_logits_processors 返回值，与本 MR 新写的 sampler.h 契约直接矛盾

**后果**：suffix（vlm greedy verify）路径的 `target_output.logits` 只吃到 in-place 部分（penalties、bitmask），dense filter mask 与行子集的效果只存在于被丢弃的返回值里——随后 validate（suffix_worker_impl.cpp:613-614）用未过滤 logits 做 argmax bonus 与 greedy 比对。

**代码**（suffix_worker_impl.cpp:512-513）：

```cpp
Sampler::apply_logits_processors(target_output.logits,
                                 validate_input.sampling_params);   // 返回值被丢弃
```

而本 MR 在 sampler.h:44-48 新写的契约：`Masks or sample indices may produce a new Tensor; callers must use the returned scores.`——dense mask 分支只改局部变量（sampler.cpp:92 `sample_logits = sample_logits + effective_filter_mask;` 仅 return，不写回引用参数）。

**触发条件**：suffix 路径 `validate_input.sampling_params` 中 `filter_mask` 有定义，或 selected/sample numel 不等（index_select 子集）。

**最小修复**：`target_output.logits = Sampler::apply_logits_processors(target_output.logits, validate_input.sampling_params);`（一行）；若确认 suffix 路径永无 dense mask/子集，则收窄 sampler.h 注释措辞。二者取一，消除注释契约与在库调用方行为的矛盾。

**验证**：单测构造 filter_mask + `apply_logits_processors`，断言返回值与被引用参数在 dense mask 下不再相等（该差异即本条根因的可执行定义）。

## R12 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:31:37｜tests/core/framework/sampling/rejection_sampler_test.cpp:51｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467738`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
 }  // namespace

+TEST(RejectionSamplerTest, FilteredTargetRejectsTailDraft) {
+ if (!Platform::is_mlu() || Platform::device_count() == 0) {
```

### 评论原文

[P2] 5 个新测试中 4 个 MLU-gated：publish 契约在无 MLU 的 CI 仅剩 1 条用例，「bug 非 MLU-only」无测试守护

**后果**：MR 核心 claim（三出口回写）在无 MLU 设备的 CI 环境零回归保护；dflash/dflash2 的 validate 被本 MR 静默修复（修复前同样读未过滤 logits，dflash_worker_impl.cpp:1193-1194 直接 view 进 rejection sampling）也无对应测试。

**证据**：4/5 测试的设备门（rejection_sampler_test.cpp:51-52、sampler_filter_mask_test.cpp:103-105/140-142）：

```cpp
if (!Platform::is_mlu() || Platform::device_count() == 0) {
  GTEST_SKIP() << "MLU top-k sampling requires a device";
}
```

但触发 publish 修复的「重绑新 Tensor」并非 MLU 专属——仅定义 top_k 或 top_p 之一时走通用 else 分支，**所有后端**都重绑（logits_utils.cpp:170-173）：

```cpp
logits = torch::empty_like(sorted_logits)
             .scatter_(/*dim=*/-1, /*index=*/logits_idx,
                       /*core=*/sorted_logits.to(logits.scalar_type()));
```

**最小修复**：新增一条 CPU 用例——greedy 参数改 random 采样 + 仅 `params.top_p`（不设 top_k），断言 forward 后调用方 logits 已被过滤（softmax 后集合外为 0）；顺带为 dflash validate 的过滤语义补一条经 `spec_verify::run_rejection_sampling` 的用例。

**验证**：CPU 构建跑该用例（无需设备）；在未打本 MR 的 base 上跑应失败（证明确为回归测试）。

## R13 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:31:51｜xllm/core/framework/sampling/sampler.cpp:37｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467739`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+ const torch::Tensor& sample_logits,
+ const SamplingParameters& params) {
+ if (params.selected_token_idxes.numel() == params.sample_idxes.numel()) {
+    logits = sample_logits;
```

### 评论原文

[Question] ForwardOutput.logits 在 MLU 下被静默提升为 fp32，ForwardOutput 处无契约注释、物化集外消费者 dtype 假设未审计

**现象**：全行 publish 直接重绑 MLU kernel 输出（sampler.cpp:37 `logits = sample_logits;`），该输出恒 fp32（apply_topkp.cpp:46-47）；子集路径同样整体提升（sampler.cpp:40 `logits = logits.to(sample_logits.scalar_type());`）。测试已把提升断言为预期（sampler_filter_mask_test.cpp:175 `EXPECT_EQ(logits.scalar_type(), torch::kFloat32);`，输入含 bf16 变体）。

**影响**：MLU + top_k/top_p + bf16 输入时 `ForwardOutput.logits` dtype 变 fp32，[rows, vocab] 每 decode 步内存翻倍。物化集内消费者已逐一核实安全（mtp/dflash validate 的 `softmax(..., kFloat32)`、adaptive_pruning 的 `.options()` 保 dtype 拷贝），但 forward_params.h:671 仅 `torch::Tensor logits;` 无注释，物化集外（logits 回传客户端的序列化路径）dtype 假设未审计。

**建议**（二选一）：(a) 接受提升——在 forward_params.h:671 注释文档化「post-sampling、processors 已应用、dtype 可能提升」，并确认集外消费者无 dtype 假设；(b) 不接受——publish 时 `.to(prob_dtype)` 压回输入 dtype（与 probs 的处理对齐，代价一次转换）。

## R14 — 马晓龙 (maxiaolong.maxwell)｜2026/10/20 11:32:03｜xllm/core/framework/sampling/sampler.cpp:176｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467741`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+  output.probs = probs.to(prob_dtype);
   output.next_tokens = sample_indices;

  if (params.logprobs) {
```

### 评论原文

[Question] MTP spec-decode 返回的 logprobs 口径变化：现在在 top-k/top-p 截断分布上计算（与非 spec 对齐，方向正确），但用户可见数值变化未在 MR 描述声明

**现象**：RejectionSampler 的 logprobs 直接对 validate 传入的 target_logits 做 log_softmax（rejection_sampler.cpp:190-191）。修复前该 logits 来自未过滤分布（+ dense mask）；修复后是 publish 出来的截断 logits——截断位 logprob 为 -inf，保留 token 在截断支撑集上重归一化。非 spec 路径的 logprobs 本来就在过滤后分布上算（sampler.cpp:190-195）——即 spec 与非 spec 的口径由「不一致」变为「一致」，与 vLLM 语义对齐。

**用户可见变化**：MTP 投机解码 + logprobs/top_logprobs + top_k/top_p 生效时，被过滤 token 的 logprobs 变为 -inf、保留 token 概率重归一化后变大。

**建议**：确认该对齐是有意为之，并在 MR 描述补一句；可选：为既有 LogProbs 用例（rejection_sampler_test.cpp:469）补一条带 top_k 的变体，锁定新口径。

## R15 — 冯艳 (fengyan.119)｜2026/10/20 11:36:32｜xllm/core/framework/sampling/sampler.cpp:36｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 线程锚点：`discussion_r3467751`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点前节选）

```diff
+void publish_logits(torch::Tensor& logits,
+ const torch::Tensor& sample_logits,
+ const SamplingParameters& params) {
+ if (params.selected_token_idxes.numel() == params.sample_idxes.numel()) {
```

### 评论原文

#### 【🔵 minor】【🔧 refactor】

#### publish_logits 内部重新推导 use_sample_indices，应直接复用 forward 已计算的标志

---

**🔎 问题**

> publish_logits 用 params.selected_token_idxes.numel() == params.sample_idxes.numel() 第三次推导同一个派生标志：apply_logits_processors（sampler.cpp:70）和 forward（sampler.cpp:109）已各自计算 use_sample_indices。且 :126、:143 两处调用点所在分支的条件已保证 !use_sample_indices，helper 内部的比较在这两处是死逻辑，只有 :150 的调用真正需要分支。重复推导同一谓词有漂移风险（一处改成显式 defined() 判断后另一处容易被遗漏）。

---

**🔧 修改建议**

把标志作为参数传入，删除 helper 内部的重复推导：

void publish_logits(torch::Tensor& logits,
const torch::Tensor& sample_logits,
bool use_sample_indices) {
if (!use_sample_indices) {
logits = sample_logits;
return;
}
logits = [logits.to](http://logits.to)(sample_logits.scalar_type());
logits.index_copy_(/*dim=*/0, sample_idxes, sample_logits);
}

调用点改为 publish_logits(logits, sample_logits, use_sample_indices)；:126/:143 分支内条件已保证该值为 false，语义不变。sample_idxes 仍需 params 传入或在 helper 内取用，保持 index_copy_ 现有行为。

---

**🔬 复现步骤**

未执行；静态证据来自 head b8fb9d45f315d90ba955621a2a382b484240652e 的 sampler.cpp:33-42/70/109/126/143/150。
