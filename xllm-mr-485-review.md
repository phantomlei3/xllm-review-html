# xLLM MR #485 — zengyuting12 12:20 之后的 Review 意见

- 来源：[MR #485](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485)
- 标题：`perf: optimize glm5-next prefill state on mlu`
- 分支：`perf/mlu-glm5-next-prefill-state-io` → `main`
- 页面最近一次 force-push 的目标 head：`acc300979d2ef153bd5fcd998cc275b5678f8879`（2026/10/14 11:59:56）。
- 页面状态：Open / Reviewing；要求 2 人通过 review，当前尚未满足合并要求。
- 提取时间：2026/10/14 14:51（Asia/Shanghai）；通过已登录 Chrome 刷新后读取，并展开隐藏的同类信息。
- 筛选：仅收录曾玉婷（`zengyuting12`）在 **2026/10/14 12:20:00 之后**发表的评论及 review 意见；不收录更早内容及其他作者的评论。未按解决状态过滤。
- 数量：**1 条行级根评论，0 条回复，0 条独立总体 review 总结**。对应一条 `reviewed` 事件（12:20:45），该事件不是额外评论。页面 Conversation 数字 5 不作为筛选后的评论数。
- 范围核查：展开隐藏记录后，未发现其他符合作者与时间条件的内容，未见分页或“加载更多”入口。本次记录页面已发布内容，不包含未提交的 review 草稿。

以下为 reviewer 原文，未独立验证其中的技术判断。

## R01 — 曾玉婷（zengyuting12）｜2026/10/14 12:20:45｜causal_conv1d.py:431｜状态未核实

- 类型：行级根评论；无回复。
- 严重程度：`High`（原文标注）。
- 文件：`xllm/core/kernels/mlu/triton_kernel/causal_conv1d.py`；页面评论紧随新增行 431，代码片段为 428–431 行。
- 原讨论：[discussion_r3461249](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485#discussion_r3461249)
- 状态核查：线程仅显示处理动作按钮，未见明确已解决事件或状态标记，因此无法确认是否已解决。

### 评论原文

**[High] 宽度为 5 的 prefill 卷积使用错误 tap**

**问题：** 统一 prefill kernel 只为 KERNEL_WIDTH=2、3、4 编写了卷积 tap 和状态轮转分支。宽度为 5 时，循环中的 matrix_w/matrix_x 始终保持 col0/w_col0，实际重复累加第一个 tap；状态更新也没有 KERNEL_WIDTH=5 分支，因此不会正确滑动历史窗口。C++ 入口对非 distinct 路径没有宽度上限，GLM5-Next 仅检查宽度大于 0，Qwen3.5 也直接采用配置值，故该配置会进入此 kernel。

**触发条件：** 在 MLU 上加载 linear_conv_kernel_dim=5 的 GLM5-Next 或 Qwen3.5 配置，执行普通 PREFILL，使 causal_conv1d_fn 进入 causal_conv1d_prefill_kernel。

**影响：** 当 MLU 模型配置 linear_conv_kernel_dim=5 时，prefill 输出和写回的卷积状态都会错误，导致后续 KDA/GDN 推理结果确定性偏离。

**建议：** 为宽度 5 及其他允许宽度补全 tap、状态轮转和状态写回逻辑，或在 dispatch 前明确拒绝并回退到仍支持该宽度的实现；同时为宽度边界增加测试。

### 页面可见代码摘录（diff，非完整上下文）

```diff
+                                matrix_w = w_col2
+                                x_ptrs_1d = x_base_1d + idx_token * stride_x_token  # [BLOCK_N]
+                                matrix_x = tl.load(x_ptrs_1d, mask=mask_x_1d)
+                        elif KERNEL_WIDTH == 4:
```
