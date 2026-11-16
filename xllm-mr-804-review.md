# xLLM MR #804 — Review 意见

- 来源：[MR #804](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804)
- 标题：`feat: support mlu native deepseek v4.1 vlm inference.`
- 分支：`feat/dsv4.1_vlm` → `main`；2 commits、40 files changed。
- 页面状态：Open / Reviewing；要求 2 人批准，等待人工评审；3 items passed。
- 当前 head：页面未显示独立 head 字段，未核实；提交列表展示 `cd45369` 和 `8822941`，不据此推断 head。
- 提取时间：2026-11-16T10:12:18+08:00；通过已登录 Chrome 刷新并等待加载后读取；写入后再次刷新核对，全部评论正文一致。
- 筛选：未指定作者或解决状态过滤；收录全部已读取评论、回复及作者说明。
- 清单：5 条行级根评论、1 条标题更正回复、1 条独立自动审查小结；另有 MR 作者说明 1 条，共 7 条根内容及 1 条回复。页面 Conversation 7 不包含作者说明。
- 解决状态：5 个行级线程均只见 No Processing Required / Resolved 操作按钮，未见明确解决事件，逐线程标为未核实。独立小结未显示解决状态。
- 范围限制：正文从已加载 DOM 中读取，包含页面折叠隐藏的行级正文；未发现评论分页、load more 或隐藏评论组控件，未发现 OutDated 标记。Files Changed 的全部历史版本与过期线程未独立遍历，不能排除未加载内容；不含草稿。
- 原文说明：以下技术判断属于评审者，未经导出者独立验证。R01 标题含平台占位符 `:unknow_emoji:`，更正保留在其回复中；R01 代码中 lambda 捕获被平台转为异常链接，按页面原样保留，未推断修复。

## 行动索引（导出者摘要）

| 评论 | 作者 | 严重性 | 主题 | 回复数 | 线程状态 |
| --- | --- | --- | --- | --- | --- |
| R01 | 马晓龙 | P2 | 增加 Python/C++ 视觉塔数值对拍证据 | 1 | 未核实 |
| R02 | 马晓龙 | P2 | 降低视觉 MoE 偏置逐 token 物化成本 | 0 | 未核实 |
| R03 | 马晓龙 | P2 | 拆分 bias 权重校验并说明兼容性变化 | 0 | 未核实 |
| R04 | 马晓龙 | P2 | 为图元数据透传补基类回落或注释 | 0 | 未核实 |
| R05 | 马晓龙 | P2 | 解释视觉 RMSNorm eps 的独立契约 | 0 | 未核实 |
| S01 | fengyan.119 | 自动小结 | 静态审查未发现需修改问题 | 0 | 未显示 |

## 作者说明 — ext.luolei15｜2026/11/13 18:04:58｜MR 描述

## 描述

为 DeepSeek-V4.1 新增 MLU 原生 VLM 实现，接入图像编码、图文 embedding 融合、上下文并行及 DSpark 投机解码。

## 主要改动：

- 实现视觉编码器和投影层，复用现有多模态处理器，将图像特征接入语言模型；视觉 RMSNorm 权重保留 FP32 精度。
- 支持原生 VLM 的上下文并行（CP），调整 DP/CP/TP 分组、输入分发及容量检查。
- 接入图像 token 专用 MoE 路由偏置，屏蔽图像及填充 token 的 Engram 更新；复用模型级 CP token 分片，保留全局 token 用于哈希。
- 透传 VLM 图执行元数据，支持 VLM 目标模型与 LLM 草稿模型组合使用 DSpark，并允许模型按能力省略未使用的 cache slots。
- 补充视觉精度、路由、Engram、CP 和图捕获/重放回归测试。

## S01 — fengyan.119｜2026/11/13 19:27:49｜独立自动审查小结｜状态未显示

### 🤖 自动审查小结：**LGTM，这代码看着真舒服！**

本轮静态代码审查未发现需要修改代码的问题。

## R01 — 马晓龙｜2026/11/14 10:33:34｜tests/core/layers/mlu/deepseek_v41_vision_test.cpp:95｜状态未核实

- 类型：行级根评论；账号 `maxiaolong.maxwell`；P2；1 条已发布回复。
- 线程：[discussion_r3512189](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512189)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：标题位置取自页面差异锚点；正文保留评审者提供的范围。

### 根评论原文

[P2] 视觉塔数值路径缺 Python:unknow_emoji:C++ 对拍测试：布局契约有 LOCKED PAIR 双侧 pin，数值契约没有

**后果**：视觉数学的未来漂移（一侧单独改动、kernel 级数值差异）没有回归防线。本 MR 的 bugfix commit（cd45369 "preserve vision norms"）正是实例——首版 C++ 把 norm 权重落成 BF16、偏离 Python 参考的 FP32 约定，靠人工发现而非测试拦截。

**触发条件**：未来任何一侧单独改动视觉数学（RoPE/aligner/norm/eps）；或两侧 kernel 实现差异（cnnl SDPA vs NPU fusion attention 等）只能在数值对拍中暴露。

**现状**：本 PR 的 12 个测试文件全部是自洽性/能力位/边界证据。vision_test 的 `reference()`（tests/core/layers/mlu/deepseek_v41_vision_test.cpp:95 起）是同 PR 手写、逐算子复刻被测 C++ 代码——它证明 C++ 与「作者对设计的复述」自洽，不构成与 Python 参考的执行级对拍：

```cpp
// tests/core/layers/mlu/deepseek_v41_vision_test.cpp:95-127（节选）
torch::Tensor reference(const torch::Tensor& patches, int64_t h, int64_t w) {
  const auto norm = [&](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/blob/master/const torch::Tensor& x, const std::string& name) {
    const torch::Tensor xf = x.to(torch::kFloat32);
    return (xf * torch::rsqrt(xf.square().mean(-1, true) + 1e-6) *
            weights_.at(name))
        .to(x.scalar_type());
  };
  ...
}
```

仓库里存在 Python 参考视觉塔（xllm/python/models/deepseek_v41.py:3976 起，"Ported from the vllm DSV4 vision reference"）。布局契约已有跨语言双侧 pin（prompt 处理器 LOCKED PAIR 注释：tests/python/test_deepseek_v41_vision_layout.py 与 tests/processors/deepseek_v41_prompt_processor_test.cpp 两侧 pin 同一组 golden role sequences），但数值契约（encode 的浮点输出）没有任何一条测试把 C++ `DeepseekV41VisionImpl::encode` 与 Python `DeepseekV41VisionTower/Aligner` 接到同一组输入上比对。端到端测试 deepseek_v41_vlm_graph_test.cpp:105-106 用随机权重只验证内部一致性。

**根因**：数值等价契约只存在于「作者逐行移植的自觉」中，没有落成可执行断言；与 LOCKED PAIR 布局测试的防护级别不对称。

**最小修复**：在 tests/python/ 侧导出固定输入+权重的 golden tensors（或直接在同一测试里 import Python tower），C++ 侧 deepseek_v41_vision_test 增加 allclose 用例；或在 MR 描述补一次真机 Python vs native logits 对拍结果。

**验证**：新用例在 MLU 单卡 CI 可跑；故意扰动一侧（如 eps 改 1e-20）应能被测试抓住。

**证据边界**：本次审查已对 Python 参考做逐机制静态比对（RoPE 频率构造、neox 半宽旋转、SDPA scale、aligner zero-pad+unfold、gelu、行对交错、pad token 的 MoE bias 选择、norm eps/FP32 权重），静态语义全部对齐，未发现现有偏差；剩余风险 = kernel 级数值差异 + 未来漂移无回归防线。想跟作者确认：是否在 MR 之外做过真机对拍？若有，请把结果或脚本入库。

### 回复 — 马晓龙｜2026/11/14 10:48:16

- 账号：`maxiaolong.maxwell`；父评论：R01；[discussion_r3512194](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512194)。

标题更正：本条评论标题应为「[P2] 视觉塔数值路径缺 Python<->C++ 对拍测试：布局契约有 LOCKED PAIR 双侧 pin，数值契约没有」。发布时平台把标题里的双向箭头字符渲染成了占位符 :unknow_emoji:（仅此一处字符清洗），评论正文其余内容与原稿逐字一致，无需其他更正。

## R02 — 马晓龙｜2026/11/14 10:35:47｜xllm/core/layers/mlu/moe_softplus_topk.cpp:72｜状态未核实

- 类型：行级根评论；账号 `maxiaolong.maxwell`；P2；0 条已发布回复。
- 线程：[discussion_r3512190](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512190)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：标题位置取自页面差异锚点；正文保留评审者提供的范围。

### 根评论原文

[P2] MoE 视觉偏置每次 gate 前向物化 [N, E] 逐 token 偏置张量：hot-path 瞬时分配与逐元素算子放大

**后果**：performance。每个 MoE 层每次前向分配并计算 3 个 [N, E] 级张量（eq → [N,E] bool、where → [N,E] FP32、kernel 内 scores + bias → 再一个 [N,E]）。以 DSV4.1 生产配置量级估算：N=8k prefill tokens × E=256 专家 × 4B ≈ 8MB/张量，~60 个 MoE 层 × 每个 prefill chunk。文本 token 占绝对多数（纯文本请求占 100%）时，这些计算产出与 bias_ 广播相同的结果。

**触发条件**：任何启用 visual bias 的前向。注意 `use_visual_bias` 在构造期由「模型族 + mm 层数」决定（moe_gate.cpp:45-47），不感知请求内容——用同一 VLM checkpoint 以纯文本 LLM 形态 serving 的既有部署也会在每个 MoE 层付这笔成本，即使整个请求没有一个图像 token。decode 图捕获路径只捕获一次、replay 零成本；但 eager prefill / spec-verify / CP prefill 每层每 chunk 都付。

**代码**（xllm/core/layers/mlu/moe_softplus_topk.cpp:68-75，被 MoEGateImpl::forward 每层调用；kernel 侧消费在 core/kernels/mlu/fused_moe.cpp:80 `selection = bias.has_value() ? scores + *bias : scores;`）：

```cpp
std::optional<torch::Tensor> bias = bias_opt_;
if (use_visual_bias_ && input_ids.has_value()) {
  CHECK(bias_vl_is_loaded_) << "Missing visual router bias_vl";
  CHECK_EQ(input_ids->numel(), scores.size(0));
  bias = torch::where(
      input_ids->reshape({-1, 1}).eq(image_token_id_), bias_vl_, bias_);
}
auto [weights, indices] = xllm::kernel::mlu::moe_softplus_topk(
    scores, topk_, input_ids, tid2eid_opt_, bias, route_scale_);
```

**根因**：把「逐 token 选偏置向量」实现为「物化逐 token 偏置矩阵」，而非在稀疏行上做修正。图像 token 是稀疏事件（每请求几百到几千个）。Python 参考同样是 torch.where 全量物化（deepseek_v41.py:1995-1997），语义一致、性能同样不优——但 C++ 侧作为新写的性能敏感路径有条件做得更好。

**最小修复**（二选一）：

1. 保持 1-D 广播 + 稀疏修正：`selection = scores + bias_;` 然后对图像行 `selection.index_put_({image_rows}, scores.index_select(0, image_rows) + bias_vl_)`（image_rows = `input_ids->eq(image_token_id_).nonzero()`，[K,1] 级小张量）；
2. 更彻底：把 bias/bias_vl/image_token_id_ 三个参数传进 moe_softplus_topk kernel，在 kernel 内按行选择（kernel 已经拿到 input_ids）。

**验证**：现有 moe_gate_test.cpp:332 VisualBiasRespectsCapabilityAndTokenType 直接可作为等价性回归（两种实现应产出相同 indices/weights）；补一个 prefill 尺寸（如 4k tokens）的微基准对比。

**证据边界**：代码路径、调用频度、kernel 消费方式均已核实。想跟作者确认：是否有 prefill 性能数据表明该成本可接受（MR 无任何 benchmark）。

## R03 — 马晓龙｜2026/11/14 10:38:05｜xllm/core/layers/mlu/moe_softplus_topk.cpp:94｜状态未核实

- 类型：行级根评论；账号 `maxiaolong.maxwell`；P2；0 条已发布回复。
- 线程：[discussion_r3512191](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512191)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：标题位置取自页面差异锚点；正文保留评审者提供的范围。

### 根评论原文

[P2] verify 级联 CHECK 条件与动机不匹配：visual 模型缺 bias_vl 启动失败（描述未提），而「sqrtsoftplus 缺 bias → 用未初始化内存路由」的静默漏洞只堵住了一半

**后果**：分两种。

1. `use_visual_bias_=true`（DSV4.1 家族 + mm 层数 > 0 自动启用）且 checkpoint 缺 bias_vl（或 bias）→ 启动即 CHECK 失败。这是本 MR 新增行为，MR 描述未提；对不含 bias_vl 键但 config 带视觉层的旧 DSV4.1 checkpoint 导出，从「能起」变「起不来」。
2. `use_visual_bias_=false` 的其他 sqrtsoftplus 非 hash 模型缺 bias → `!use_visual_bias_ || ...` 短路为真，校验放行，`bias_opt_` 仍是 torch::empty 未初始化内存，静默参与专家路由（base 已有静默漏洞，本 MR 差一步没堵全——顺带提醒，不算本 MR 回归）。

**代码**（xllm/core/layers/mlu/moe_softplus_topk.cpp:93-95，新增）：

```cpp
void MOESoftPlusTopKImpl::verify_loaded_weights() const {
  CHECK(!use_visual_bias_ || (bias_is_loaded_ && bias_vl_is_loaded_))
      << "Visual routing requires both bias and bias_vl";
}
```

级联链：模型 verify → FusedMoEImpl::verify_loaded_weights（fused_moe.cpp:790 新增首行 gate_->verify_loaded_weights()）→ MoEGateImpl::verify_loaded_weights → 此处。而构造/load 的 else 分支（moe_softplus_topk.cpp:50-61, 84-91，既有代码）里 `bias_ = register_parameter("bias", torch::empty({n_routed_experts_}, ...))` 是未初始化内存、`bias_opt_ = bias_` 无条件赋值，forward 直接传给 kernel 相加。

**根因**：校验条件把两个独立前提（非 hash 路由需要 bias；visual 路由还需要 bias_vl）用一个 `||` 短路耦合，第二前提的失败被第一前提的通过掩蔽。

**最小修复**：拆成两条 CHECK：`CHECK(use_hash_ || bias_is_loaded_) << "sqrtsoftplus routing requires bias";` 与 `CHECK(!use_visual_bias_ || bias_vl_is_loaded_) << "Missing visual router bias_vl";`；并在 MR 描述记录「缺 bias/bias_vl 的 checkpoint 现在加载期失败」。

**验证**：构造三个 state dict（缺 bias / 缺 bias_vl / 齐全）分别加载，断言前两个 CHECK 失败且消息可定位到具体权重名。

**证据边界**：torch::empty 未初始化、bias_opt_ 无条件赋值、CHECK 短路逻辑均引自 head 源码原文。想跟作者确认：是否存在无 bias_vl 键但 config 含 mm 层的 DSV4.1 checkpoint 部署（决定情形 1 是否实际触发）；校验条件本意是哪一种。

## R04 — 马晓龙｜2026/11/14 10:40:18｜xllm/core/framework/model/causal_vlm.h:86｜状态未核实

- 类型：行级根评论；账号 `maxiaolong.maxwell`；P2；0 条已发布回复。
- 线程：[discussion_r3512192](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512192)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：标题位置取自页面差异锚点；正文保留评审者提供的范围。

### 根评论原文

[P2] CausalVLMImpl::prepare_graph_forward_metadata 是五个图元数据/能力透传方法中唯一不回落基类的：基类默认一旦非空，VLM 包装将静默丢失该行为

**后果**：maintainability。今天无实际错误（CausalLM::prepare_graph_forward_metadata 当前是空实现，causal_lm.h:101-103），语义等价。但未来任何一个提交给基类默认加实质行为（通用 hash-state 刷新、日志、metric），所有未实现该方法的 VLM 包装会静默跳过——requires_graph_forward_metadata() 回落却返回 true 的模型组合会拿着空 state 进 capture。

**代码**（xllm/core/framework/model/causal_vlm.h:80-86，本 MR 新增）：

```cpp
void prepare_graph_forward_metadata(ModelGraphMetadataState* state,
                                    const torch::Tensor& positions,
                                    ModelInputParams& parameters) override {
  if constexpr (detail::has_prepare_graph_forward_metadata<Model>::value) {
    model_->prepare_graph_forward_metadata(state, positions, parameters);
  }
  // <-- 没有 else / 没有 CausalLM::prepare_graph_forward_metadata(...)
}
```

对照同文件四个兄弟透传（:64-78 requires_graph_forward_metadata、:96-101 create_graph_forward_metadata_state、:103-110 supports_mla_graph_kv_bucketing、pooler），全部带 `return CausalLM::...` 基类回落，唯独这个没有。

**根因**：SFINAE 透传样板复制时漏掉了 fallback 分支。

**最小修复**：一行——补 `else { CausalLM::prepare_graph_forward_metadata(state, positions, parameters); }`，或至少注释说明「基类默认为空，故省略回落」。无行为变化。

**验证**：现有 deepseek_v41_vlm_graph_test 全绿即可（等价重构）。

## R05 — 马晓龙｜2026/11/14 10:42:32｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_vision.cpp:192｜状态未核实

- 类型：行级根评论；账号 `maxiaolong.maxwell`；P2；0 条已发布回复。
- 线程：[discussion_r3512193](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/804#discussion_r3512193)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：标题位置取自页面差异锚点；正文保留评审者提供的范围。

### 根评论原文

[P2] 视觉 RMSNorm 的 eps 硬编码 1e-6 且无出处注释：值正确，但与 LM 侧 1e-20 差 14 个数量级的反差缺一行「为何不同」的防护说明

**后果**：maintainability（防未来静默回归）。未来维护者做「一致性清理」：看到 LM 侧都从 args.rms_norm_eps() 读（DSV4.1 默认 1e-20，deepseek_v41.py:316/340）、vision 侧却是魔法数字，把它「统一」成 1e-20——视觉输出系统性偏差且无测试拦截（对拍缺口同另一条 finding）。

**代码**（xllm/core/layers/mlu/deepseek_v41/deepseek_v41_vision.cpp:175-180，新增，eps 字面量，无注释）：

```cpp
torch::Tensor DeepseekV41VisionImpl::rms_norm(
    const torch::Tensor& x,
    const torch::Tensor& weight) const {
  torch::Tensor xf = x.to(torch::kFloat32);
  return (xf * torch::rsqrt(xf.square().mean(-1, /*keepdim=*/true) + 1e-6) *
          weight)
      .to(x.scalar_type());
}
```

**值是对的**：Python 参考 docstring 明确记录（xllm/python/models/deepseek_v41.py:4044）：`"""RMSNorm with fp32 weight and the vision eps (1e-6, not the text 1e-20)."""`——C++ 硬编码与之一致。但 Python 侧有防护注释，C++ 侧既没有 mm_rms_norm_eps 配置位，也没有任何注释解释这个反差是刻意的。

**根因**：移植时只搬了值，没搬 Python 侧的防护性文档。

**最小修复**：一行注释 `// Vision RMSNorm eps is 1e-6 by the vision config contract (Python: DeepseekV41VisionNorm), NOT the text-side rms_norm_eps (1e-20).`；或提为 `constexpr double kVisionRmsNormEps = 1e-6;` 常量。

**验证**：无行为变化；现有 Bfloat16PreservesFloat32NormWeights 继续覆盖数值路径。
