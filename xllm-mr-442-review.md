# xLLM MR #442 — Review 意见与修复交接

- 来源：[MR #442](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/442)
- 标题：feat: add glm5 flash prefill context parallelism on mlu.
- 仓库：xLLM_AI/xllm（ID 958063）
- 分支：`feat/glm53_flash_cp` → `main`
- 页面快照：Open；6 commits；44 files changed；Review Required；规则要求 2 人批准；状态检查显示 3 项通过、1 项运行中。
- 导出时间：2026-10-09（Asia/Shanghai）
- 提取范围：从已登录 Chrome 的 MR 页面读取 Conversation、提交摘要、变更数量及 review 状态区域。
- Review 结果：截至导出时，页面 Conversation 中仅有 MR 作者的描述，没有审查者 review 评论、回复或行级意见。因此本文件记录当前“暂无已发布 review 意见”的状态，并保留作者描述作为修复 agent 的背景。后续页面新增意见时需要重新导出。

## 给修复 agent 的交接说明

此快照没有可执行的 reviewer 意见。请在开始修复前重新查看 MR 页面是否已有新评论；本文件中的 MR 描述、限制与验证记录来自作者提交内容，不代表审查结论。

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

## Review 意见

截至导出时，页面未显示任何审查者评论或行级 review 意见（0 条）。MR 仍为 Open / Review Required，要求 2 人批准；状态检查为 3 项通过、1 项运行中。
