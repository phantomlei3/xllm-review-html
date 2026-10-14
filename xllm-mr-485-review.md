# xLLM MR #485 — 12:20 之后的 Review 意见

- 来源：[MR #485](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485)
- 标题：`perf: optimize glm5-next prefill state on mlu`
- 分支：`perf/mlu-glm5-next-prefill-state-io` → `main`
- 页面最近一次 force-push 的目标 head：`acc300979d2ef153bd5fcd998cc275b5678f8879`（2026/10/14 11:59:56）。
- 页面状态：Open / Reviewing；要求 2 人通过 review，当前尚未满足合并要求。
- 提取时间：2026/10/14 16:22（Asia/Shanghai）；通过已登录 Chrome 刷新后读取，并展开隐藏的同类信息。
- 筛选：仅按时间筛选，收录 **2026/10/14 12:20:00 之后**所有作者发表的评论及 review 意见；不指定 reviewer，不收录更早内容，未按解决状态过滤。
- 数量：**4 条根评论（3 条行级、1 条总体评论），0 条回复**。reviewed 事件不重复计作评论，页面 Conversation 数字 5 不作为筛选后的评论数。
- 范围核查：展开隐藏记录后，未发现其他符合时间条件的内容，未见分页或“加载更多”入口。本次记录页面已发布内容，不包含未提交的 review 草稿。

以下为 reviewer 原文，未独立验证其中的技术判断。

## 行动索引（导出者摘要）

| 编号 | Reviewer | 意见 |
| --- | --- | --- |
| R01 | zengyuting12 | High：宽度为 5 的 prefill 卷积使用错误 tap |
| R02 | fengyan.119 | minor / license：删除新增文件版权头中的 All Rights Reserved. |
| R03 | fengyan.119 | minor / refactor：复用 KPOOL ring 定位与写入公式 |
| R04 | fengyan.119 | 总体评论：MR 标题不符合规范 |

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

## R02 — 冯艳（fengyan.119）｜2026/10/14 12:36:03｜causal_conv1d.py:1｜状态未核实

- 类型：行级根评论；无回复。
- 严重程度：`minor / license`（原文标注）。
- 文件：`xllm/core/kernels/mlu/triton_kernel/causal_conv1d.py`；评论紧随新增行 1。
- 原讨论：[discussion_r3461253](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485#discussion_r3461253)
- 状态核查：仅见处理动作按钮，未见明确已解决记录，无法确认是否已解决。

### 评论原文

**【🔵 minor】【📄 license】**

**新增文件版权头含 "All Rights Reserved."，违反项目 header 规则**

---

**🔎 问题**

custom-code-style §2 明确规定 "Do not add `All Rights Reserved` to the project copyright header"。本文件是本次 diff 新增的生产文件，版权行 `# Copyright 2026 The xLLM Authors. All Rights Reserved.` 不应携带该短语；同一 diff 内新增的三个测试文件存在相同问题。

---

**🔧 修改建议**

删除版权行中的 `All Rights Reserved.`，保留 `# Copyright 2026 The xLLM Authors.`，后续 Apache 许可声明保持不变；三个新增测试文件同步删除。

### 页面可见代码摘录（新增代码，非完整上下文）

```python
# Copyright 2026 The xLLM Authors. All Rights Reserved.
```

## R03 — 冯艳（fengyan.119）｜2026/10/14 12:36:04｜glm5_next_kpool.py:630｜状态未核实

- 类型：行级根评论；无回复。
- 严重程度：`minor / refactor`（原文标注）。
- 文件：`xllm/core/kernels/mlu/triton_kernel/glm5_next_kpool.py`；评论紧随新增行 630。
- 原讨论：[discussion_r3461254](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485#discussion_r3461254)
- 状态核查：仅见处理动作按钮，未见明确已解决记录，无法确认是否已解决。

### 评论原文

**【🔵 minor】【🔧 refactor】**

**`_stash_from_source` 重复了 `kpool_prefill_stash` 的 ring 定位与写入公式**

---

**🔎 问题**

本次新增的 `_stash_from_source`（line 647-680）与既有 `kpool_prefill_stash`（line 477-510）各自维护同一套 ring 公式：尾部 trim 的 `while (end > start) & (last < 0)` 循环、`source = end - 1 - (last % T - r + T) % T` 定位、`source_pos >= 0` 有效位掩码和 `tail + (id * 2 * T + r) * D` 写入，约 12 行重复索引数学。当 `read_tail_ids == tail_ids` 时 `_stash_from_source` 的 `inherit` 掩码恒为 false，输出与 `kpool_prefill_stash` 完全一致（纯整数索引计算，无数值差异），两份公式今后漂移会直接造成 prefill 普通/带源两条 stash 路径行为分叉。`_compress_decode_four` 已示范同文件内 `@triton.jit` helper 复用模式。

---

**🔧 修改建议**

将 `kpool_prefill_stash` 改为转发调用 `_stash_from_source(k, gate, tail, tail_ids, /*read_tail_ids=*/tail_ids, positions, starts, ...)`（`xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer.cpp:270` 的 launch grid 补第二维 `(T + 15) / 16` 以匹配 `kpool_stash_from_source` 的 ring-tile 划分），或提取共享的 ring 定位 helper 供两个 kernel 调用。分块粒度差异只影响并行度，输出等价；`Glm5NextKPoolIndexerTest.PrefillMatchesPagedPathAcrossRaggedTiles` 等既有测试可复核等价性。若担心分块对既有路径的并行度影响，至少应让 `_stash_from_source` 的公式与 `kpool_prefill_stash` 保持单一来源。

### 页面可见代码摘录（新增代码，非完整上下文）

```python
# Source slots remain immutable while unique destination slots receive the
# updated ring. A destination must not alias another request's live source.
@triton.jit
def _stash_from_source(
```

## R04 — 冯艳（fengyan.119）｜2026/10/14 12:36:04｜MR 总体评论｜无行级线程状态

- 类型：独立总体评论；无回复，无代码行锚点。
- 严重程度：原文未标注。
- 来源：[MR #485 会话页](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/485)；以作者、时间及正文定位。

### 评论原文

⚠️ **MR 标题不符合规范** `#title`

squash-merge 会用 MR 标题作合并 commit 名，需遵循 `<type>: <subject>`（type ∈ feat|bugfix|docs|test|refactor|chore|style|revert|perf|model|build|release；subject 英文、动词开头、≥4 词、句号结尾）。

当前标题：`perf: optimize glm5-next prefill state on mlu`
