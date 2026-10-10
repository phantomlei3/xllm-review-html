# xLLM MR #414 — dengyingxu1 未解决 Review 意见

- 来源：[MR #414](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/414)
- 仓库：xLLM_AI/xllm（ID 958063）
- 分支：`feat/kv-cache-store-opt` → `main`
- 页面快照：Open；5 commits；36 files changed；Reviewing；4 项状态检查通过。
- 导出时间：2026-10-10（Asia/Shanghai）
- 筛选范围：只保留 dengyingxu1 发布且未标记为 resolved 的评论，其他作者的内容均已删除。

## dengyingxu1 — Review 总结（10月2日）

dengyingxu1
 commented 10月 2

Review 总结（head b424dc1d5396a172137bea8033abb41c11216d92，base ba50a61c525d74554ee46ef775a5c82766d09219，merge base 159a1ba427d643e60459b83d52e14eaf41ee590b）：已检查 2 个提交、33 个变更文件，重点覆盖 Mooncake Store 预取统计、worker 会话生命周期、请求状态透传、Store/层级缓存契约及新增 CPU 单测。git diff --check 通过；已完成静态调用链和测试覆盖检查，未运行 C++ 构建、pytest、NPU、ATB、ACL Graph、HCCL、CUDA 或推理运行。当前讨论中已有 4 条 P2 行级意见（CHECK_EQ 使诊断路径可崩溃、probe 区间把已读取未发布 unit 计入、worker 失败路径丢失统计、Store/会话测量缺少 CPU 覆盖），本次不重复发布；另有既有关于 stats/copy 线程池析构竞态的意见，均待作者处理。合并建议：修复后再合并。
