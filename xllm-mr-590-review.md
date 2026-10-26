# xLLM MR #590 — Review 意见与修复交接

- 来源：[MR #590](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/590)
- 标题：`bugfix: remove mlu graph history admission limits.`
- 分支：`bugfix/glm53-graph` → `main`
- 页面提交：`e30be3ad640d830c7d6e88f736405a1f5ec88bb7`、`77f5d51e5c5d95fb1f82424b96b648dd98975dde`（最后展示）；未另查仓库确认 head。
- 页面状态：Open / Reviewing；2 commits、16 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。冯艳 (fengyan.119) 于 2026年10月26日 11:37:18 approved changes，该事件未附评论正文。
- 提取时间：2026-10-26 11:56（Asia/Shanghai）；通过已登录 Chrome 刷新后读取，展开收起内容。提交前于 2026-10-26 11:57:02 再次刷新，核对作者说明与 4 条行级意见正文未变。
- 筛选：不按作者、时间或解决状态过滤；包含作者说明及全部已发布行级意见和回复；不含提交、reviewed 和 approved 事件正文。
- 清单：5 条根评论（1 条作者说明、4 条行级意见），0 条回复。页面 Conversation 4 不作为评论总数；隐藏预览中重复的评论不重复计数。
- 范围：检查会话及其中的行级讨论，未见更多评论分页、加载更多或 OutDated 标记；未另外检查 Files Changed 中未在会话展示的讨论，不含草稿。
- 状态说明：4 条行级讨论仅见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件，逐条标为“状态未核实”。
- 原文说明：技术判断和测试声明均为评论作者原文，未经导出者独立验证。R03 标题止于“的”，R05 标题止于文件名，保留原文不补写。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：Graph 历史长度准入及 MTP draft Graph 修复、验证结果 | 未见解决状态标记 |
| R02 | 正确性：头文件全局变量的 static / inline 与 ODR 问题 | 状态未核实 |
| R03 | 健壮性：移除源 tensor narrow 后的形状断言 | 状态未核实 |
| R04 | 正确性：无条件更新 kv_max_seq_len 对 capture / replay 的影响 | 状态未核实 |
| R05 | 健壮性：Zero uses block_size 注释与无条件覆盖实现不符 | 状态未核实 |

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

## R02 — 邓英旭 (dengyingxu1)｜2026/10/26 10:34:18｜xllm/models/llm/mlu/glm5_mtp.h:42｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477385`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点节选）

```diff
+const bool glm5_mtp_capabilities_registered = []() {
```

### 评论原文

**1. 正确性 — glm5_mtp.h 中全局变量 glm5_mtp_capabilities_registered 缺少 static**
位置：`xllm/models/llm/mlu/glm5_mtp.h:42`

glm5_mtp.h 中全局变量 glm5_mtp_capabilities_registered 缺少 static 或 inline 关键字，在头文件中定义会导致多重定义的链接错误（ODR 违规）。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/26 10:34:18｜xllm/core/runtime/mlu_graph_executor_impl.cpp:292｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477386`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点节选）

```diff
-  if (params_.graph.eager_context != nullptr) {
-    params_.meta.kv_max_seq_len = params.meta.kv_max_seq_len;
-  }
+  params_.meta.kv_max_seq_len = params.meta.kv_max_seq_len;
```

### 评论原文

**2. 健壮性 — mlu_graph_executor_impl.cpp 中 copy_ 操作移除了对源 tensor 的**
位置：`xllm/core/runtime/mlu_graph_executor_impl.cpp:292`

mlu_graph_executor_impl.cpp 中 copy_ 操作移除了对源 tensor 的 narrow，虽然形状匹配时行为一致，但显式 narrow 可以提供更好的错误信息和安全性

建议保留或添加形状断言。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/26 10:34:18｜xllm/core/runtime/mlu_graph_executor_impl.cpp:308｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477387`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点节选）

```diff
   if (source.block_tables.defined()) {
+    CHECK(buffer.block_tables.defined());
```

### 评论原文

**3. 正确性 — mlu_graph_executor_impl.cpp 中 update_input_buffer 无条件赋值**
位置：`xllm/core/runtime/mlu_graph_executor_impl.cpp:308`

mlu_graph_executor_impl.cpp 中 update_input_buffer 无条件赋值 params_.meta.kv_max_seq_len，需确认在 graph capture 阶段更新此值是否会影响 replay 时的正确性。

## R05 — 邓英旭 (dengyingxu1)｜2026/10/26 10:34:18｜tests/core/runtime/mlu_graph_executor_test.cpp:529｜状态未核实

- 类型：行级意见；根评论，0 条回复。原文未标 P 级严重性。
- 线程锚点：`discussion_r3477388`；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点节选）

```diff
   options.block_size(16)
+      .block_table_token_span(/*value=*/64)
```

### 评论原文

**4. 健壮性 — options.h 中注释 "Zero uses block_size" 与 worker_impl.cpp**
位置：`tests/core/runtime/mlu_graph_executor_test.cpp:529`

options.h 中注释 "Zero uses block_size" 与 worker_impl.cpp 中的无条件覆盖逻辑不符

建议更新注释或修改实现逻辑。
