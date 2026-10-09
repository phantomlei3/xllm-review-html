# xLLM MR #442 — Review 意见与修复交接

- 来源：[MR #442](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/442)
- 标题：feat: add glm5 flash prefill context parallelism on mlu.
- 仓库：xLLM_AI/xllm（ID 958063）
- 分支：`feat/glm53_flash_cp` → `main`
- 页面快照：Open；Conversation 显示 2；6 commits；44 files changed；Reviewing，要求 2 人批准；状态检查显示 4 项通过。
- 当前 head（审查总结所述）：`c816136517bff85fe45649afd43e0f6e52bed631`
- 导出时间：2026-10-09（Asia/Shanghai）
- 提取范围：刷新已登录 Chrome 页面并展开“同类信息已隐藏”区域后，读取全部已加载评论、回复、行级意见、审查总结及状态。Review 内容包括一条 Medium 行级意见、审查总结，以及 F1–F8、Q1–Q3、T1 等顶层意见。
- 忠实性说明：评论部分保留页面原文和可见署名/时间；代码事实及建议未另行验证。行号基于各评论中标注的审查版本。

## 给修复 agent 的交接索引

| 优先级 | 评论 | 主题 | 位置/关注点 |
| --- | --- | --- | --- |
| Medium | zengyuting12 行级意见 | 回归测试无外部基线时无条件失败；缺少环境变量应跳过或提供基线，并保证设备不足时能到达 skip。 | `tests/models/glm5_next_mlu_pcp_regression_test.cpp:1211` |
| P1 | F1 | KPool prefill 选路改写影响所有 glm5_next（含 cp=1），缺少旧新选块等价测试。 | `glm5_next_kpool_indexer.cpp:625-641` |
| P2 | F2 | `chunk_count < cp_size` 时 PCP 静默回退，建议加日志或 counter。 | `glm5_next.h:75-81`；PCP geometry 分配逻辑 |
| P2 | F3 | DSA/KPool PCP 中空行保护分支受准入门控保证不可达。 | attention / KPool PCP 实现 |
| P2 | F4 | `local_causal_conv` 按请求和前序 rank 循环 `torch::cat`，可能产生大量小 kernel launch。 | `glm5_next_kda_pcp.cpp:73-92` |
| P2 | F5 | 模型出口对 normalized/residual 分别 gather，可考虑合并一次通信。 | `glm5_next.h:102-113` |
| P2 | F6 | 两个 Triton kernel 的 constexpr `N` 未使用，增加 JIT 特化。 | `chunk_kda_pcp.py:31,94` |
| P2 | F7 | PCP×TP>1 缺少模型级/collective 组合回归。 | MoE、DSA 和 DenseMLP 的 TP/CP 组合 |
| P2 | F8 | `kv_split==1` 由 KPool 通用 CHECK 间接保证，报错未提 CP 限制。 | `glm5_next_kpool_indexer.cpp:449-450` |
| Question | Q1–Q3 | deepseek_v4 mHC 形状门、DFlash 准入缺口、描述中的修复/性能数字缺少可定位证据。 | 详见原文 |
| Question | T1 | PCP 每个 CP rank 写全量 cache 副本，prefill cache 内存可能按 cp_size 放大。 | DSA/KPool PCP cache 写入路径 |

审查总结建议：合入前优先补 F1 等价性验证与 F2 回退日志；F3–F8 可后续处理，Q1/Q2 建议作者先在 MR 中说明。另有一条既有 Medium 意见指出测试基线环境变量导致常规 CTest 稳定失败。

## MR 作者描述（页面原文）


为 MLU 上的 GLM-5.3-Flash（glm5_next）增加 Prefill Context Parallelism（PCP）：cp_size > 1 时，prefill / chunked-prefill 的 token 按 KDA chunk 边界切分到各 CP rank 上并行计算，层间只交换必要的中间量，模型出口再恢复为全局行序。

主要改动
准入与上下文
is_mlu_model_cp_capable 增加 glm5_next。
新增 glm5_next_pcp::Geometry / Context，负责按 chunk 切分、行分片（shard_rows）和 gather 后恢复（gather_restore）。
dummy、mixed、spec-verify、decode 批次，以及存在空 rank 的批次不走 PCP，回退到原有的全序列路径。
KDA（线性注意力）
新增 chunk_kda_pcp kernel（Triton + C++ 封装）：各 rank 先算本地的仿射摘要 [E; M]，all-gather 后由 chunk_kda_pcp_merge 合并前序 rank 的摘要得到本 rank 的初始状态，再 replay 出输出。
最后一个 rank 持有每条请求的尾部，由它把最终的卷积状态和循环状态广播给其他 rank。
卷积 cache 视图只覆盖已提交的历史，不触碰 MTP 的 speculative checkpoint 列。
DSA / KPool
新增 DeepseekV2AttentionImpl::forward_glm5_next_pcp 和 Glm5NextKPoolIndexer::forward_pcp：K 在各 rank 本地计算后 gather 成全局 K 并写入 cache，query 侧只算本地行。
MoE
DeepseekV4SparseMoEBlock 抽出 forward_cp，供 glm5_next 复用；shared expert 改为在 gather 之前对本地行计算。
通信与计算重叠
ProcessGroup 新增 broadcast_async，KDA 的广播、摘要 all-gather 以及 DSA/MoE 的 gather 都改为先发起、算完独立部分后再等待。
顺带的性能与修复
swiglu_limit 路径的 split + clamp + concat 合并为单个 Triton kernel（fused_moe_clamp）。
fused mHC kernel 现在在 prefill 中也启用，不再只用于 decode。
16 个本地头的批量 KDA decode 把 block_v 降到 64，避免超出 MLU NRAM。
修复 KPool metadata 透传、Triton 模块查找和测试链接问题。
对现有路径的影响

以下改动不限于 cp_size > 1，请重点 review：

fused mHC 在 prefill 中启用，影响所有使用该 kernel 的 GLM-5.3-Flash 部署。
fused_moe_clamp 替换了 swiglu_limit 的原实现。
DeepseekV4SparseMoEBlock 的 CP 路径被重构，deepseek_v4 的 MLU CP 也会走到新代码。
批量 KDA decode 的 block_v 调整与 main 上 a855ef3e 的 split_small_kda_batch 会在 TP4 decode 场景叠加，正确性已验证（见下），性能未单独测。
已知限制
MLU 上 glm5_next + cp_size > 1 只在 dp_size == 1、kv_split_size == 1、ep_size == world_size 下可用。
spec-verify 批次不做 PCP。
glm5_next + DFlash + cp_size > 1 能通过 master 的 MLU 准入，但会在 DFlashWorkerImpl 构造时 CHECK 失败（该豁免仅对 NPU 生效）。这是 MLU 既有的缺口，本 PR 未处理。
验证
MLU 全量编译通过。
新增单测：
chunk_kda_pcp_test：摘要、合并与 replay 对照 FP32 CPU 递推，覆盖多序列、尾部不满 chunk、2 个和 3 个前序 rank。
fused_moe_clamp_test、glm5_next_pcp_context_test、parallel_state_test（broadcast_async）。
glm5_next_mlu_pcp_regression_test：前 8 层、8 rank（cp_size=8、TP=1、EP=8），连续 chunked-prefill、decode 和复用 cache 的第二轮，与冻结基线对比输出、hidden 和 cache，并检查 speculative checkpoint 未被改写。基线在仓库外，由 XLLM_GLM5_NEXT_PCP_BASELINE_DIR 指定。
rebase 到 main（74cfdb32）后，FusedSigmoidUpdateTest 24 个用例全部通过，包括 main 新增的两个 TP4 decode graph 用例。
8 节点 CP=4 服务启动成功。
fused_moe_clamp：16K 输入、1 token 输出的 TTFT 三轮 A/B，中位数从 1170.44 ms 降到 1086.02 ms（-7.21%），24/24 请求成功。
Related Issues

无

Change Type
 Bug fix
 New feature
 Performance improvement
 Refactor
 Documentation
 Test
 Build or CI
Pull Request Checklist
PR Title and Commit Messages
 The PR title and each commit message follow the xLLM commit format: <type>: <subject>.
Pre-commit Checks
 I have installed pre-commit by running pip install pre-commit or an equivalent command.
 I have installed the hooks with pre-commit install.
 I have run pre-commit run --all-files and fixed any reported issues.
Self Review
 I have self-reviewed the code according to .agents/skills/code-review/references/custom-code-style.md, especially code written or assisted by AI.

## 全部 Review 与 comments 原文

以下保留页面中展开后的全部 review/comment 内容；其中含页面的文件锚点、状态标签及评论正文。

zengyuting12

reviewed    10:29
  View Changes
tests/models/glm5_next_mlu_pcp_regression_test.cpp
View file
...
...
@@ -0,0 +1,1283 @@
1211
+
TEST(Glm5NextMluPcpRegressionTest, EightRanksMatchFrozenBaseline) {
1212
+
  std::string error;
1213
+
  const std::optional<BaselineOptions> baseline = read_baseline_options(error);
1214
+
  ASSERT_TRUE(baseline.has_value())
曾玉婷  5 小时前

[Medium] 未配置外部基线时新增测试必然失败

问题： 该测试已由 cc_test 无条件注册，但 CMake 没有为其设置 XLLM_GLM5_NEXT_PCP_BASELINE_DIR，也没有携带基线数据。测试入口却在检查设备数量之前断言该环境变量必须存在，因此默认测试环境必然失败。

触发条件： 以 USE_MLU=ON、BUILD_TESTING=ON 构建后，直接运行该 CTest，且未设置 XLLM_GLM5_NEXT_PCP_BASELINE_DIR。

影响： 任何启用 USE_MLU 和 BUILD_TESTING 的常规 ctest 运行，只要没有额外配置该环境变量，就会稳定失败；即使机器不足 8 张 MLU，本应执行的 GTEST_SKIP 也无法到达。

建议： 若这是依赖外部基线的可选回归测试，在环境变量未设置时使用 GTEST_SKIP；或者将冻结基线作为测试 DATA 提供，并在 CMake 的 ENVIRONMENT 中设置目录。还应确保设备不足时能够先进入跳过路径。

No Processing Required
Resolved
dengyingxu1
 commented 11:13

审查范围：MR !442，base 74cfdb3202771dedf5a4d4777e64232e007da68a，merge base 74cfdb3202771dedf5a4d4777e64232e007da68a，head c816136517bff85fe45649afd43e0f6e52bed631；检查 6 个提交、44 个变更文件，覆盖 GLM5 MLU PCP 上下文/KDA、并行通信、DSA/KPool、MoE、fused kernel 及新增单测/回归测试。

检查结果：git diff --check 通过；已完成受影响调用链、状态/张量契约及新增测试的静态检查。未运行 C++/MLU 构建与测试，未进行 MLU/NPU、ATB、ACL Graph、HCCL、CUDA 或实际推理验证。

当前仍有 1 个未解决的既有 inline finding：tests/models/glm5_next_mlu_pcp_regression_test.cpp:1211 的外部基线环境变量未配置时测试无条件失败，导致常规 ctest 稳定失败；建议未配置基线时显式 GTEST_SKIP，或随测试提供并配置基线数据。未发现其他可确认的 P0-P3 缺陷，因此未发布新的 inline finding。

合并建议：修复后再合并

maxiaolong.maxwell
 commented 11:27
F1 [P1] KPool prefill 选路重写波及所有 glm5_next 部署（含 cp=1），选择语义零测试覆盖，新旧等价性无验证
位置：src/head/xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer.cpp:625-641（索引；证据链涉及 :603-665 与被删除的 base 同文件 select_prefill/launch_prefill_logits）
代码（head，select_projected_pools 的 prefill 分支）：
if (execution.prefill && !execution.fused_update) {
  // Score directly from the paged pool cache.  The previous prefill path
  // gathered every request's history into dense tensors and then reran an
  // FP32 matmul for each query tile.  The paged score kernel already applies
  // the request-local causal limit from positions and row_batch, so it is
  // equivalent while avoiding the dense gather and host-side grouping loops.
  return glm5_next_kpool_select(query,
                                weights,
                                positions,
                                execution.batch->row_batch,
                                index_cache,
                                execution.batch->block_table,
                                execution.batch->max_kv_len,
                                block_size_,
                                index_kpool_,
                                index_topk_,
                                softmax_scale_);
}

（base 同分支调用的是被整段删除的 select_prefill：逐请求 gather_prefill_cache 把历史 pool 收成 dense keys → score_prefill（PAGED=0）FP32 打分 → select_topk/select_topk_streaming。select_projected_pools 同时被非 PCP 的 select_pools（:603-616）与 PCP 的 forward_pcp（kpool_indexer_pcp.cpp:102-107）调用，因此该重写对 cp=1 部署同样生效。）
触发条件：任何 glm5_next MLU 部署（cp=1 或 cp>1）执行 max_seq_len > 32 的 prefill/chunked-prefill（fused_update 门在 :548-550）。
当前行为：打分-选块从"dense gather + 专用 prefill kernel 链"整体切换到既有 paged kernel 链（score_kpool+select_kpool，kpool.cpp 本 MR 零改动）。等价性依据只有上面引用的代码注释。
期望行为：对既有部署的输出改变（若有）被显式验证：新旧两条链在相同输入上选出相同 pool 集合，或至少 E2E logits 对比 base 构建。
影响：correctness 风险（低概率、高隐蔽）+ 测试缺口。两条链的数学目标相同，但 tile 划分与 top-k 平局裁决顺序不同，边界分数相近的 pool 可能翻转选择，进而改变 DSA 注意力输出。这不是理论洁癖：MR 的全部测试都没有覆盖该分支的区分性行为——
模型级回归测试刻意中和了选择：glm5_next_mlu_pcp_regression_test.cpp:33-34 明文 "sequences shorter than index_topk, so sparse selection keeps every block"——所有块都保留时，任何打分/选块差异都不可见；
仓内不存在任何 kpool 选择单测（git grep -i kpool -- tests/ 在 base 与 head 均无命中）；
冻结基线由本 MR 代码自身 record，不构成新旧对照。
根因：性能重写搭在 PCP MR 里顺带发布，影响面（所有部署）与验证投入（零）不匹配；"equivalent" 声明停留在注释层。
建议（最小修复，任选其一或组合）：
补一个 kernel 级等价测试：构造多请求、含平局分数、pool 数跨 streaming 阈值的输入，分别跑旧链（可在测试内重建 dense gather + score_prefill + select_topk 的参考实现，或直接以 base 行为为参考）与新 glm5_next_kpool_select，断言选中 pool 集合一致或在平局时断言确定性；
或补一个 E2E 对比：同一权重与请求集，base 构建与 head 构建的 prefill 输出 logits 全量对比（容差内一致）。
验证：上述测试在 MLU 上跑通；另建议在 MR 描述中把这条重写从"Optimize prefill paths"扩写为显式条目（它和 fused mHC 一样属于"影响面超出 CP"的改动）。
证据边界：两条链的数学等价性（含 causal 掩码方式）已由代码审读确认方向一致；未证明的是平局裁决与浮点累加顺序下的选择一致性——这正是需要测试的原因。MLU 实测未做。
maxiaolong.maxwell
 commented 11:27
F2 [P2] chunk_count < cp_size 时 PCP 整批静默回退，无日志/指标，chunked-prefill 尾 chunk 高发
位置：src/head/xllm/models/llm/mlu/glm5_next.h:75-81；根因在 src/head/xllm/core/layers/mlu/glm5_next/glm5_next_pcp_context.cpp:62-68
代码：
// glm5_next.h:75-81
const bool all_ranks_have_tokens =
    std::all_of(context.geometry.tokens_per_rank.begin(),
                context.geometry.tokens_per_rank.end(),
                [](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/blob/master/int32_t count) { return count > 0; });
if (!all_ranks_have_tokens) {
  return std::nullopt;
}

// glm5_next_pcp_context.cpp:62-68（整数除法分配）
const int64_t chunk_count =
    (static_cast<int64_t>(query_length) + chunk_size - 1) / chunk_size;
const int32_t begin = static_cast<int32_t>(std::min<int64_t>(
    query_length, chunk_size * (chunk_count * rank / cp_size)));
const int32_t end = static_cast<int32_t>(std::min<int64_t>(
    query_length, chunk_size * (chunk_count * (rank + 1) / cp_size)));

触发条件：手算例（context-pack ⑥.4 例 B，本审查复算一致）：单请求 L=8、chunk_size=4（chunk_count=2）、cp=4 → begin/end 为 r0=[0,0)、r1=[0,4)、r2=[4,4)、r3=[4,8) → rank 0/2 空 → 整批回退。真实量纲：批内总 chunk 数不足时（如 chunked-prefill 的最后一个短 chunk、或单条短请求的整批），高 rank 拿 0 chunk。
当前行为：整批在所有 rank 上一致回退到非 PCP 路径（每 rank 全量冗余计算，输出仍正确），无任何日志、计数器或告警。回退决策由确定性几何驱动，各 rank 一致，无 collective 死锁风险（已核验）。
期望行为：正确性已满足；缺的是可观测性——CP 部署的收益会在部分 batch 上静默消失，线上无法定位"为什么 CP 没生效"。
影响：performance / operability。
根因：回退是静默控制流，无遥测点。
建议：回退处加 LOG_EVERY_N(WARNING, n) << "GLM5-Next PCP skipped: empty rank (chunk_count=" << ... << " < cp_size=" << cp_size << ")" 或暴露 counter；顺带可在注释中写明尾 chunk 高发这一运营特征。
验证：单测构造 chunk_count < cp_size 批次，断言回退路径 + 日志触发。
证据边界：回退正确性与一致性已证明；发生频率是推断（依赖 chunked prefill 预算与 cp_size 配置），需线上数据确认。
maxiaolong.maxwell
 commented 11:27
F3 [P2] DSA/KPool PCP 路径存在不可达的空行防御分支（0.1 死代码）
位置：src/head/xllm/core/layers/mlu/deepseek_v2_attention_glm5_next_pcp.cpp:31,37-40,43,56-61,82-84；src/head/xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer_pcp.cpp:51,57-60,73,97-101
代码（DSA，:82-84）：
if (local_hidden_states.size(0) == 0) {
  return torch::empty_like(local_hidden_states);
}

触发条件：无——prepare_context 的 all_ranks_have_tokens 门（glm5_next.h:75-81）保证进入 PCP 路径时 local_hidden_states.size(0) == tokens_per_rank[cp_rank] > 0（shard_rows 于 :97-98 按 rows_by_rank 取行）。
当前行为：四个函数里成对的空张量构造与提前返回永远不会执行。
期望行为：按 0.1 规则：删除，或保留但注明 rationale（哪个未来改动——如逐 rank 混合 PCP——会让它有意义）。
影响：maintainability（读者会误以为空 rank 可达，与 F2 的整批回退语义混淆——事实上一旦某 rank 空，整批根本不进这些函数）。
根因：防御性代码与上游门控重复确认同一 invariant。
建议：最小修法：在这四处 if 上方加一行注释 // Unreachable: prepare_context gates on all_ranks_have_tokens.；或直接删除分支。
验证：现有测试全绿即可（分支删除不影响覆盖）。
证据边界：不可达性由调用链静态证明；无运行动态证据。
maxiaolong.maxwell
 commented 11:27
F4 [P2] local_causal_conv 逐（请求×前序 rank）host 端 torch::cat 链，每 KDA 层 O(R·cp_rank) 次小 kernel launch
位置：src/head/xllm/core/layers/mlu/glm5_next/glm5_next_kda_pcp.cpp:73-92
代码：
for (int64_t request = 0; request < requests; ++request) {
  torch::Tensor history = initial_history.select(/*dim=*/0, request);
  for (int32_t rank = 0; rank < context.cp_rank; ++rank) {
    ...
    history = torch::cat({history, earlier}, /*dim=*/0)
                  .narrow(/*dim=*/0, count, history_size);
  }
  local_history.emplace_back(std::move(history));
}

（history/earlier 为 [history_size(=conv_kernel_size−1), channels] 的小张量；initial_history.select 是视图，torch::cat 每次发射一个 device kernel。）
触发条件：每次 KDA 层 PCP 前向；代价 ∝ 请求数 R × 本 rank 的 cp_rank。
当前行为：以 rank 7 视角、R=16 个请求、cp=8 为例：每 KDA 层 16×7=112 次 cat launch（外加每请求一次 select 视图与收尾的 stack/contiguous/cat），~30 个 KDA 层的一次 prefill 累计数千次微 kernel launch。功能正确（滑窗语义本审查逐行手算核对无误），纯 launch 开销。
期望行为：hot path 上的 host 循环合并为少量向量化操作。
影响：performance（估算每次 launch 数 μs 级，累计可达每 iteration 数十 ms 量级；未实测——证据边界）。
根因：逐请求逐 rank 的标量循环直接映射成逐次 kernel 发射。
建议：最小改法：对每个前序 rank，一次性用 index_select/narrow+copy 把所有请求的 earlier 段拼成一个 [R, count, ch]，再对 history 做一次批量滑窗（或预计算每请求的全局 token 偏移，用一次 index_select 从 cat({initial_history, gathered_tails}) 中按行号取窗口）。
验证：加 profile（一次 8-rank prefill 的 kernel launch 计数/耗时 A/B）。
证据边界：launch 次数是静态推算；绝对耗时未测，作者 TTFT A/B 为净效果（含此开销后仍为正收益），故仅列为改进项。
maxiaolong.maxwell
 commented 11:27
F5 [P2] 模型出口 restore_output 对 normalized/residual 各发一次 gather，可合并为 1 次
位置：src/head/xllm/models/llm/mlu/glm5_next.h:102-113
代码：
normalized = layer::glm5_next_pcp::gather_restore(normalized, *context);
if (residual.has_value()) {
  residual = layer::glm5_next_pcp::gather_restore(residual.value(), *context);
}

触发条件：每次 PCP forward 的模型出口（residual 通常有值）。
当前行为：两次独立的 variable-length allgather（各含 pad 到 max）。
期望行为：torch::cat({normalized, residual}, -1) 一次 gather 后 chunk 拆回，集合通信 2→1（0.2 第 3 问的"多次 gather 同 shape host 数据→一次 gather 后 unpack"）。
影响：performance（每 iteration 省 1 次集合通信；尾部同步点减少）。
根因：逐张量恢复的直写。
建议：如上合并；注意两个张量 dtype 需一致（norm 输出与 residual 同 dtype，成立）。
验证：回归测试基线比对不变。
证据边界：收益量级未测。
maxiaolong.maxwell
 commented 11:27
F6 [P2] 两个 PCP Triton kernel 声明但未使用 constexpr N，徒增 JIT 特化基数
位置：src/head/xllm/core/kernels/mlu/triton_kernel/chunk_kda_pcp.py:31（summary kernel 形参 N: tl.constexpr）与 :94（merge kernel 形参 N: tl.constexpr）
代码：
def kda_pcp_summary_kernel(
    ..., H: tl.constexpr, N: tl.constexpr, BT: tl.constexpr, D: tl.constexpr, BV: tl.constexpr,
) -> None:
    job = tl.program_id(0)
    row_blocks: tl.constexpr = triton.cdiv(D, BV)
    row_block = job % row_blocks
    head = (job // row_blocks) % H
    sequence = job // (row_blocks * H)   # 全函数体无任何 N 的引用

merge kernel（:89-119）同样声明 N 但函数体只用 H/D/P/RANK/BV。C++ 侧把 sequences 作为 N= 传入（chunk_kda_pcp.cpp:176, 288）。
触发条件：批内请求数（N）变化的线上流量——每个新出现的 N 值都会生成新的 kernel 特化并触发一次 JIT 编译。
当前行为：N 不参与任何计算，却进 kernel cache key；首次遇到新 batch 组合时付编译延迟。
期望行为：删除该形参（或改为非 constexpr 普通参数）。
影响：performance（首次延迟）/ maintainability。
根因：形参抄自调用约定但未接线。
建议：最小修法：两个 kernel 签名与两处 launch 参数中删掉 N；顺带可给 merge 的 RANK: tl.constexpr 加注释说明"每 rank 一份特化，cp_size 固定时仅 cp_size 个变体"（这个是有语义的，保留）。
验证：chunk_kda_pcp_test 全绿。
证据边界：JIT 行为依 Triton 语义推定，未在 MLU 上实测编译缓存行为。
maxiaolong.maxwell
 commented 11:27
F7 [P2] PCP×TP>1 组合无模型级回归；MoE forward_cp 的 TP 切分路径无 CP 组合测试
位置：测试配置证据 src/head/tests/models/glm5_next_mlu_pcp_regression_test.cpp:1149-1156（"dp=1, cp=8, tp=1, ep=8 … single-rank TP groups"）；src/head/tests/core/layers/mlu/deepseek_v4_sparse_moe_collective_test.cpp:327-377（collective 用例均为 world=2、单一模式，无 cp×tp 复合）
代码（forward_cp 的 TP 切分逻辑，无 CP 组合测试覆盖）：
// deepseek_v4_sparse_moe_block.cpp:235-250
const std::pair<int32_t, int32_t> local_range =
    split_range(local_tokens, tp_size, tp_rank);
...
for (int32_t ep_rank = 0; ep_rank < ep_size; ++ep_rank) {
  const int32_t cp_rank = ep_rank / tp_size;
  const int32_t attention_tp_rank = ep_rank % tp_size;
  const int32_t cp_tokens = tokens_per_rank[static_cast<size_t>(cp_rank)];
  unique_tokens_per_ep_rank.emplace_back(
      split_range(cp_tokens, tp_size, attention_tp_rank).second);
}

触发条件：TP>1 且 CP>1 的 glm5_next / deepseek_v4 部署。
当前行为：MoE 的 (ep_rank → cp_rank,tp_rank) 映射、DSA 的 reduce(output, tp_group_)（deepseek_v2_attention_glm5_next_pcp.cpp:93-95）、DenseMLP 的 TP 归约在 PCP 下均无集成测试。
期望行为：至少一个 cp×tp>1 的模型级（或 MoE collective 级）用例锁住该组合。
影响：测试缺口（组合乘法路径只被各自单测覆盖，交叉正确性靠推理）。
根因：回归测试为控制成本固定 tp=1。
建议：在 collective 测试加一个 use_cp=true, tp_size=2（world=4，或以 mock group 降低设备需求）的用例；或在 MR 描述已知限制中显式声明"TP×CP 组合未经回归验证"。
验证：新用例通过。
证据边界：代码审读未发现 TP×CP 的正确性疑点（split_range 映射自洽、DSA reduce 位置正确）；纯覆盖缺口。
maxiaolong.maxwell
 commented 11:27
F8 [P2] kv_split==1 限制仅由 KPool 构造的通用 CHECK 间接保障，报错不提 CP
位置：src/head/xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer.cpp:449-450（base 已有，非本 MR 引入）
代码：
CHECK_EQ(parallel_args.kv_split_size_effective(), 1)
    << "KPool does not support DCP cache sharding.";

触发条件：glm5_next（含 cp=1）配置 kv_split>1 时于构造期失败；MR 描述把 "kv_split_size==1" 列为 PCP 已知限制。
当前行为：限制成立（glm5_next 的 DSA 层必带 KPool indexer，构造期 fail-fast），但报错文案指向 KPool/DCP，排障者需自行联想到 CP 准入。
期望行为：要么在 CP 准入（model_registry / parallel args 校验）处显式拒绝并提及 CP，要么在 MR 描述注明该限制由 KPool CHECK 代持。
影响：operability（低）。
根因：限制的实际执行者（KPool）与文档归属（PCP）分离。
建议：最小修法：MR 描述/注释注明；或准入期加一条带 "context parallelism" 字样的 CHECK。
验证：配置 kv_split=2 + cp=2 走查报错路径。
证据边界：glm5_next 是否必然构造 KPool indexer 已由 layer role 解析确认（DSA 层存在即构造）；若存在全 KDA 变体配置则守卫失效——未见此类配置。
maxiaolong.maxwell
 commented 11:27
Q1 deepseek_v4 是否存在 hc_mult4 && hidden_size4096 的配置命中 fused mHC 形状门
位置：src/head/xllm/core/layers/mlu/hyper_connection.h:89（bool supports_fused_mhc() const { return hc_mult_ == 4 && dim_ == 4096; }）；调用方 src/head/xllm/core/layers/mlu/deepseek_v4/deepseek_v4_decoder_layer.cpp:169-176
问题：mHC prefill 翻转的真正门是形状而非模型类型。deepseek_v4 与 glm5_next 共用 resolve_mhc_fusion；若任何 deepseek_v4 配置恰好 hc_mult4 且 hidden4096，其 prefill 也从本 MR 起走 fused mHC + pending 链，而本 MR 未更新任何 deepseek_v4 的 prefill+mHC 测试（deepseek_v4_hyper_connection_test 不在 44 文件内）。
关闭条件：作者确认现网/计划内的 deepseek_v4 配置均不命中该形状门；或补一个命中形状门的 deepseek_v4 prefill 用例（fused vs 非 fused 数值对照）。
maxiaolong.maxwell
 commented 11:28
Q2 glm5_next + DFlash + cp>1 的 MLU 准入缺口（作者自认，diff 外）
问题：MR 描述自述 "glm5_next + DFlash + cp>1 通过 master MLU 准入但会在 DFlashWorkerImpl 构造 CHECK 失败（豁免仅对 NPU 生效）"。DFlashWorkerImpl 与准入豁免逻辑均不在本 MR 44 文件内，无法核验，也无法确认失败发生在服务启动的哪个阶段（预检期 vs 首请求）。
关闭条件：diff 外核对准入代码路径；若属实建议在准入期拒绝并给出可理解报错，而不是留到 worker 构造。
maxiaolong.maxwell
 commented 11:28
Q3 MR 描述三处声明无法在 diff 内定位/验证
(a) "fix KPool metadata forwarding"：最接近的对应物是 prepare_context 里 kpool_query_lens/q_seq_lens_vec/slot_mapping 转发 + kpool_batch_metadata.reset()（glm5_next.h:85-94），但无法确认为同一修复；
(b) "Triton module lookup" 修复：候选是 chunk_kda_pcp.cpp:30-32 的模块路径常量或 c81613651 把 Python 仿射检查改用 importlib 文件路径加载后迁 gtest，无法定位；
(c) TTFT 1170.44→1086.02 ms（−7.21%）三轮中位数：diff 内无数据或脚本。
关闭条件：作者指认对应 commit/代码位置；性能数字附可复现脚本（固定硬件/shape/轮次/统计口径，按提示词性能证据分级目前只能支持"机制生效"不能支持"端到端收益"结论）。
7. 性能专项
机制	理论收益	隐藏成本	当前证据	需要的实验
KDA [E;M] 摘要 CP	通信 ∝ O(状态)（N·H·D²），与序列长度解耦；线性注意力段间可分	每 KDA 层 4 次集合通信；F4 的 host cat 链	结构+单测；无 profile	8-rank prefill 的通信/计算占比 profile
DSA/KPool 全量 K gather	正确性必需（稀疏注意力看全局 K）	通信 ∝ 本地 token × K 维，与 CP 规模线性叠加	结构	同上
MoE shared expert 本地化	消除 ×EP 冗余计算 + 与 gather 重叠	无（collective 数不变）	结构+collective 测试	大 EP 下 shared 计算占比测量
fused mHC prefill	每层少一次 MHCPost+MHCPre+Norm 分离 kernel 链	pending 链的张量保活（4 张量跨层）	kernel 级测试	cp=1 prefill A/B
fused_moe_clamp	3 kernel/算子 + 中间张量 → 1 kernel	无（等价替换）	等价性测试 ✅；收益为作者声明	公开 A/B 脚本（Q3c）
block_v=64	修复 16 头批量 decode NRAM 超限（base 不可用 → 可用）	BV 减半的并行度损失（仅该子路径）	回归测试 ✅	该子路径 micro-bench

⚠️ 证据边界：除 fused_moe_clamp/block_v 的机制有测试背书外，端到端收益当前只能确认机制，不能确认收益（提示词性能证据分级第 1–2 级）。

8. 测试与验收矩阵（增量建议）
风险	测试场景	指标/断言	优先级
F1	kpool 新旧选路等价（多请求、平局分数、跨 streaming 阈值）	选中 pool 集合一致	P1
F2	chunk_count < cp_size 批次	回退路径 + 日志/counter	P2
F7	cp×tp>1 模型级或 collective 用例	输出等价	P2
Q1	deepseek_v4 命中 mHC 形状门的 prefill	fused vs 非 fused 数值	P2
KDA PCP	cp∈{2,4}（当前仅 cp=8）	冻结基线	P3
混合批次	长短请求混合（部分 rank 对某请求 0 长）已有单测覆盖，建议 E2E 加一档	基线比对	P3
9. 做得好的地方（附证据）
[E;M] 仿射摘要设计：summary kernel 对零长请求段初始化为恒等映射（chunk_kda_pcp.py:44-45 + 无条件 store :84-86），使 ragged 切分天然正确；EmptyRankKeepsPrefixState/MergeComposesEveryPrecedingRankSummary/LongShardsMatchUnshardedKDA 把代数性质直接钉成测试。
ac5d81ead conv checkpoint 视图收窄（glm5_next_kda_pcp.cpp:139-144）：注释讲清 why（committed history vs speculative checkpoints），回归测试逐步断言 checkpoint 列未被改写（glm5_next_mlu_pcp_regression_test.cpp:641-667）。
broadcast_async 契约（process_group.h:105-108）：null-work 单 rank 语义、输入存活要求均成文，并有双向单测（parallel_state_test.cpp:592-598）。
MoE shared expert 本地化：消除 base 的 ×EP 冗余计算（逐 token 独立性论证 + ragged/空 rank 测试），是真实的去冗余而非搬家。
block_v 修复的手术式精确：全文件 diff 仅 6 行（kSplitBlockV 常量 + 一个 if），注释写明 NRAM 根因，配 shape 回归 + 数值回归双测。
每 forward 的 Context 值语义：无跨调用缓存状态，从结构上消灭了 0.4 类 stale-cache 风险。
10. 最终建议
合入条件：F1 补等价性验证（测试或 E2E 对照，二选一）；F2 加最低限度的回退日志。
可后续处理项：F3–F8、Q1–Q3（Q1/Q2 建议在 MR 描述中先行回答）。
建议作者优先修改顺序：F1（影响面×零覆盖）→ F2（一行日志）→ Q1（一句确认）→ 其余。
一句话结论：PCP 主体工程质量和测试针对性是高的，作者自标的三个风险面全部经得起推敲；真正需要作者再补一步的是藏在"Optimize prefill paths"一句话里的 KPool 选路重写——它影响每一个 glm5_next 部署，却还没有任何测试能看见它的行为。
11. 发布结果
模式：draft（本文件为审查轨产出；发布永远单独走用户显式批准，不在 DAG 内）。
去重跳过：note:3455145（回归测试 baseline 环境变量问题）及其 context-pack ⑦.3 同根因相邻项。
maxiaolong.maxwell
 commented 11:28
T1 [Question]（补充）PCP 下每 rank 写全量 cache 副本，prefill 期 cache 内存 ×cp_size，请确认是否为有意的容量取舍
位置：
src/head/xllm/core/layers/mlu/deepseek_v2_attention_glm5_next_pcp.cpp:77-81
update_mla_k_cache(global_k,
                   global_metadata,
                   kv_cache,
                   kv_cache.get_k_cache_scale(),
                   /*is_prefill_phase=*/true);

src/head/xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer_pcp.cpp:88-95
update_cache(global_raw_k,
             global_gate,
             cache_positions,
             index_cache,
             tail_cache,
             global_execution.batch->tail_indices,
             global_execution.batch->block_table,
             global_execution);

问题：两处均以全局 K + 全局 metadata/positions 写入本 rank 自己的 kv_cache / index_cache / tail_cache。prefill 结束后每个 CP rank 持有整批请求的完整 cache 副本——这与 KDA 侧"末 rank 广播终态"同构，换来 decode 阶段零 CP 通信的简单性，代价是 prefill 期间的 cache 内存 = 单 rank 部署 × cp_size。
影响：容量规划（非正确性）。长上下文 × 大 cp_size 组合可能先撞内存墙；建议在部署文档/容量规划中注明该放大系数。
建议：若为有意取舍（TTFT 换内存），补一行注释或文档说明即可关闭；若非有意，可评估 gather 后 reduce-scatter、各 rank 只写本 rank decode 所需分片（需同步评估 decode 侧行路由改动量）。
来源说明：本条来自独立教学审查轨的源码推演（非自动评审），供容量规划参考。
