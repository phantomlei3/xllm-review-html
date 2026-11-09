# xLLM MR #739 — Review 意见

- 来源：[MR #739](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739)
- 标题：`feat: enable deepseek v4.1 pd, dp and dspark graphs on mlu.`
- 分支：`feat/dsv4.1_pd` → `main`；7 commits、43 files changed。
- 页面状态：Open / Reviewing；要求 2 人批准，等待人工评审通过；状态检查 3 items passed。
- 当前 head：未从独立 head 字段核实；最新提交事件及提交列表首项为 `bc71062ccb9f4da15e7ef17e14c74f6a39b627c9`。
- 提取时间：2026-11-09T20:36:25+08:00；通过已登录 Chrome 刷新页面后读取。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评论及作者说明。
- 清单：5 条行级根评论、0 条已发布回复；另有作者说明 1 条，共 6 条根内容。页面 Conversation 5 不作为内容总数。
- 线程状态：5 条均只见 No Processing Required / Resolved 操作按钮，未见明确解决事件，实际状态未核实。R01 标记 OutDated。
- 范围限制：已读取五个评审事件对应的五条评论，包括最初未显示正文的 R04；未发现评论分页或隐藏同类评论控件。View Changes 进入文件差异后存在 Load Diff 控件，未完整加载所有文件差异，不能独立排除未加载差异中的额外讨论；不含草稿。
- 原文说明：技术判断均为评审者意见，未经导出者独立验证。R02、R03 的页面渲染将部分比较符号呈现为高亮，导致可见文本如 `q_max_seq_len0`、`ratio1`、`CHECK(raw.scalar_type()torch::kFloat32)` 缺少符号；下文保留可见文字，不推断修复。原文中的 `latent.to` 被页面自动转成链接，导出为普通文本。
- 位置说明：各标题位置照录评论中的文件/行号；差异片段只收录评论旁已读取的代码，不补全缺失行。

## 行动索引（导出者摘要）

| 评论 | 作者 | 类别 | 主题 | 回复数 | 状态 |
| --- | --- | --- | --- | --- | --- |
| R01 | 邓英旭 | 正确性（未标严重性） | 校验 hash routing 的 input_ids 元素数 | 0 | 未核实 / OutDated |
| R02 | 邓英旭 | 正确性（未标严重性） | 明确空行 dummy 输出的不变量 | 0 | 未核实 |
| R03 | 邓英旭 | 正确性（未标严重性） | 检查 ratio 分支的 dtype 与 wgate | 0 | 未核实 |
| R04 | 邓英旭 | 正确性（未标严重性） | 对齐 norm 输出与 cache dtype，覆盖精度行为 | 0 | 未核实 |
| R05 | 邓英旭 | 正确性（未标严重性） | 核对 linear state 来源并记录图回退原因 | 0 | 未核实 |

## R01 — 邓英旭｜2026/11/9 19:02:22｜xllm/core/kernels/mlu/fused_moe.cpp:75｜状态未核实 / OutDated

- 类型：行级根评论，0 条已发布回复；账号 `dengyingxu1`；原文类别为正确性，未标严重性。
- 线程：[discussion_r3501640](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739#discussion_r3501640)
- 状态证据：仅见处理与解决操作按钮，未见明确状态事件。

### 根评论原文

**1. 正确性 — hash routing 忽略 tid2eid 之外的 bias 语义且 index_select 对 flatten 结果的 shape 恢复依赖 input.sizes()**
位置：`xllm/core/kernels/mlu/fused_moe.cpp:75`

当 tid2eid 存在时，expert_id 直接由 index_select 得到并 view 为 out_shape，但 out_shape 由 input.sizes() 推导（最后一维换为 topk）。若 input 不是 [tokens, hidden] 形态（如 3D），input_ids->flatten() 长度与 out_shape 前缀乘积是否一致未做 CHECK，错误的 input_ids 形状会在 view 处以难以定位的方式失败。

建议在 view 之前 CHECK input_ids->numel() 与 input 去掉最后一维后的元素数相等，并注明 input_ids 的期望形状。

### 页面差异片段

```diff
+  } else {
+    torch::Tensor selection = bias.has_value() ? scores + *bias : scores;
+    expert_id = std::get<1>(selection.topk(topk, /*dim=*/-1));
+  }
```

## R02 — 邓英旭｜2026/11/9 20:00:24｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_decoder_layer.cpp:147-160｜状态未核实

- 类型：行级根评论，0 条已发布回复；账号 `dengyingxu1`；原文类别为正确性，未标严重性。
- 线程：[discussion_r3501731](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739#discussion_r3501731)
- 状态证据：仅见处理与解决操作按钮，未见明确状态事件。

### 根评论原文

**1. 正确性 — 空行早退路径 dummy 输出仅取第 0 个 token 再 expand，丢失原 hidden 语义**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_decoder_layer.cpp:147-160`

新增的 q_max_seq_len0 / num_sequences0 分支直接用 hidden.select(1,0) 构造 dummy 并 expand_as(hidden) 返回，未保留 hidden 中除第 0 列外的其它列；且 attention_input 被写为 dummy 而非真实的 attention 输入。若调用方（如 DSpark 的 capture 边界）依赖该层输出的逐 token 值或 attention_input 的真实内容，会得到错误结果。

建议明确该路径仅用于 MoE 通信占位，并在调用方确认不读取这些输出；或返回与 hidden 形状一致且语义正确的张量（例如对 hidden 本身做 MoE 而非仅第 0 列再 expand），并在注释中写明不变量。

### 页面差异片段

```diff
     DeepseekV41EngramImpl::HashState* hash_state,
     const mlu_v4_cp::DeepseekV4CpContext* cp_context,
     torch::Tensor* attention_input) {
```

## R03 — 邓英旭｜2026/11/9 20:00:24｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:205-214｜状态未核实

- 类型：行级根评论，0 条已发布回复；账号 `dengyingxu1`；原文类别为正确性，未标严重性。
- 线程：[discussion_r3501732](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739#discussion_r3501732)
- 状态证据：仅见处理与解决操作按钮，未见明确状态事件。

### 根评论原文

**2. 正确性 — ratio1 与 ratio!=1 分支中 raw 的 dtype 不一致，后续 CHECK 会在 ratio1 时必然失败**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:205-214`

新代码对 ratio1 用 dense_projection(x, wkv) 得到 BF16 raw，但下方 compress 路径新增 CHECK(raw.scalar_type()torch::kFloat32)，而 ratio1 时 latent=raw 走的是 CHECK(raw.scalar_type()torch::kBFloat16) 分支，二者虽自洽，但 ratio2/4 的 scores 与 raw 都是 FP32 且在 compress 状态写入中被直接使用；对 ratio!=1 的 scores 未对 wgate 为空做防护（仅在 ratio2 时加载 wgate，ratio==4 时 scores 仍用 weights_.at("compressor.wgate.weight") 会抛异常）。

建议对 ratio4 与 ratio2 统一判断 wgate 是否存在；或在 ratio!=1 时显式 CHECK wgate 存在，避免 at() 抛出不友好错误。

### 页面差异片段

```diff
-    x.to(torch::kFloat32),
-    weights_.at("compressor.wgate.weight").to(torch::kFloat32));
+    CHECK(x.scalar_type() == torch::kBFloat16);
+    if (ratio_ == 1) {
```

## R04 — 邓英旭｜2026/11/9 20:00:24｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:617｜状态未核实

- 类型：行级根评论，0 条已发布回复；账号 `dengyingxu1`；原文类别为正确性，未标严重性。
- 线程：[discussion_r3501733](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739#discussion_r3501733)
- 状态证据：仅见处理与解决操作按钮，未见明确状态事件。

### 根评论原文

**3. 正确性 — norm 前把 latent 转为 BF16 时丢弃了 FP32 状态，与 CHECK 的 latent 来源不一致**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:617`

ratio==1 分支 latent=raw（BF16），而 ratio!=1 分支 latent 来自 FP32 state 池化结果；新代码统一执行 latent.to(torch::kBFloat16) 后 norm，但原代码是 norm(latent) 后再 .to(main_cache.scalar_type())。若 main_cache 为其他 dtype（如 FP8/INT8 路径），输出 dtype 与后续写 cache 的 target 可能不匹配，且 norm 输入改为 BF16 后精度行为变化未经验证。

建议保持输出 dtype 与 main_cache.scalar_type() 对齐的显式转换，并在测试中覆盖非 BF16 的 main_cache 场景。

### 页面差异片段

```diff
   if (committed > first) {
-    latent = norm(latent, weights_.at("compressor.norm.weight"), eps)
-                 .to(main_cache.scalar_type());
+    // Pooling and Norm each have a BF16 rounding point; state stays FP32.
```

## R05 — 邓英旭｜2026/11/9 20:00:24｜xllm/core/runtime/mlu_graph_executor_impl.cpp:768-770｜状态未核实

- 类型：行级根评论，0 条已发布回复；账号 `dengyingxu1`；原文类别为正确性，未标严重性。
- 线程：[discussion_r3501734](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/739#discussion_r3501734)
- 状态证据：仅见处理与解决操作按钮，未见明确状态事件。

### 根评论原文

**4. 正确性 — linear_state_required 仅依赖 index_kpool 配置，未考虑 graph state 是否真正被使用**
位置：`xllm/core/runtime/mlu_graph_executor_impl.cpp:768-770`

新增 planner_config.linear_state_required = has_linear_attention_layers(args_) || (args_.index_kpool()>1 && index_kpool_compress())。当模型同时满足 index_kpool 条件但 linear_state_ids 实际为空时，get_graph_layout 会因 linear_state_required 而直接返回 nullopt → EagerReason::INVALID_LAYOUT，导致 DSpark/V4.1 的 graph 静默退化为 eager，性能回退且无告警。

建议在 planner 中记录被拒绝的原因并在日志中输出一次，或在构造 planner 时与模型实际需要的 linear state 来源保持一致并加单测覆盖该条件。

### 页面差异片段

```diff
   planner_config.options = options_;
   planner_config.capabilities = mtp_capabilities_;
   planner_config.state_required = model_->requires_graph_forward_metadata();
+  planner_config.linear_state_required =
```

## 作者说明 — ext.luolei15｜2026/11/9 18:59:18｜MR 级说明｜不适用解决状态

- 类型：作者对变更及验证情况的说明，非独立评审结论；0 条已发布回复。

MLU：支持 DeepSeek V4.1 PD、DP 与 Dspark 图执行

PR 描述

本 PR 完善 DeepSeek V4.1 在 MLU 上的 PD 分离、DP 推理和 Dspark 图执行支持，并修复相关精度与状态管理问题。

- 统一压缩缓存状态分配，MLU 使用连续存储，支持 PD 状态传输和融合压缩算子。

- 支持 DP 空闲分片参与 MoE 通信，按配置补齐 dummy token，跳过注意力及压缩缓存写入。

- 启用 Dspark 分段图捕获与重放，支持动态 block table，并区分图元数据与线性注意力状态需求。

- 对齐 compressor 的 BF16/FP32 计算与舍入路径，修复负 logits 下 MoE 路由权重的数值问题。

- 统一 BangC Torch API 的张量校验，补充压缩精度、DP 空闲分片、图重放和非法输入回归测试。

验证：本次仅分析代码差异，未运行编译或测试。
