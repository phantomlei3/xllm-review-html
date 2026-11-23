# xLLM MR #943 — Review 意见

- 来源：[MR #943](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943)
- 标题：`feat: support yarn for qwen3.8 on mlu.`
- 分支：`feat/qwen3.8_yarn` → `main`。页面显示 2 个提交、13 个变更文件、195 行新增和 50 行删除。
- 当前版本：Commits 列表首项为 `9ade95ccf1bf21a025251a0426ded7aa593798ac`，与 Conversation 中 17:16:30 的 force-push 记录一致。
- 页面状态：Open / Reviewing，需 2 人批准。状态检查显示 2 项通过、1 项运行中（xLLM 服务 CI #9271455）。
- 提取时间：2026-11-23 19:29（UTC+08:00）。通过已登录的 Chrome 刷新读取，并核对了 Commits 与 Files Changed。
- 筛选与计数：收录全部可见评论，涵盖 2 条行级根评论、0 条已发布回复、0 条独立评审小结。另保留 1 条 MR 作者说明。Conversation 显示 2。
- 范围：Conversation 未见隐藏评论组、分页、加载更多评论或过期差异标记。两条线程只有处理操作入口，解决状态均未核实。未逐一检查历史差异版本。
- 以下技术判断来自评审者，未经独立验证。

## 行动索引（导出者摘要）

| 评论 | 严重性 / 类别 | 建议 |
| --- | --- | --- |
| R01 | important / perf | 让 target 和 MTP draft 通过 `CosSinCacheManager` 共享 YaRN 缓存，保留现有窗口长度与张量布局。 |
| R02 | minor / style | 将 YaRN correction helper 中的整数参数、返回值以及调用处的局部变量统一为 `int64_t`。 |

## 作者说明 — ext.luolei15｜2026/11/23 14:59:39｜MR 描述

### Bugfix

honor qwen yarn settings in target and draft models.

### Feature

support yarn for qwen3.8 on mlu.

## R01 — 冯艳（fengyan.119）｜2026/11/23 18:04:50｜`rotary_embedding_util.cpp:371`｜解决状态未核实

- 位置：`xllm/core/layers/common/rotary_embedding_util.cpp:371`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943#discussion_r3531862)

【🟠 important】【⚡ perf】

### get_qwen_mrope_cache 绕过 CosSinCacheManager，target 与 MTP draft 重复构建同一份 YaRN cos/sin 缓存

**🔎 问题**

同文件的 get_deepseek_rotary_embedding 经 compute_cos_sin_cache 使用 CosSinCacheManager 按 inv_freq_hash、device、dtype 和窗口长度共享缓存，而新增 get_qwen_mrope_cache 的 YaRN 分支直接执行 outer/cos/sin 并返回。MLU Qwen3.5 的 target 与 MTP draft 都调用该函数，draft loader 保留相同 rope 参数和 max_position_embeddings，因此同一 YaRN 配置会重复计算并常驻两份大缓存；例如百万位置、rotary_dim=64 时，每份 bf16 结果约 256MB。默认 rope（factor==0）路径不在本条范围内。

**🔧 修改建议**

在 get_qwen_mrope_cache 的 YaRN 分支复用现有 CosSinCacheManager，以实际 magnitude、inv_freq_hash、device、dtype、rotary_dim 和 max_positions 构造 CosSinCacheDesc，并用自定义 factory 保持当前数值与布局：

```cpp
const auto inv_freq = compute_yarn_inv_freq(...);
const float magnitude = configured_attn_factor > 0.0f
                            ? configured_attn_factor
                            : yarn_get_mscale(factor, /*mscale=*/1.0f);
CosSinCacheDesc desc{rotary_dim,
                     max_positions,
                     /*interleaved=*/false,
                     factor,
                     magnitude,
                     /*mscale=*/1.0f,
                     /*mscale_all_dim=*/1.0f,
                     compute_inv_freq_hash(inv_freq),
                     options.device(),
                     options.dtype().toScalarType()};
auto& cache_manager = CosSinCacheManager::get_instance();
return cache_manager.get_or_create_cache(desc, [&](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/blob/master/)%20%7B
  const auto positions = torch::arange(max_positions, torch::kFloat32);
  const auto freqs = torch::outer(positions, inv_freq);
  const auto cos = (freqs.cos() * magnitude).to(options);
  const auto sin = (freqs.sin() * magnitude).to(options);
  return torch::cat({cos, cos, sin, sin}, /*dim=*/-1);
});
```

不要改调 create_cos_sin_tensor：它把位置长度写成 max_position_embeddings*scaling_factor，与本函数只 materialize serving 窗口的契约不同。

**🔬 复现步骤**

未执行。静态调用链：Qwen3_5ModelImpl 与 Qwen3_5MtpModelImpl 均调用 get_qwen_mrope_cache，后者不经过现有缓存管理器。

代码示例中的 lambda 行含异常 URL 和百分号编码，按页面原样保留。

页面差异摘录

```diff
   return compute_rotary_embedding(dim, seq_len, rope_theta, options, true);
 }
+torch::Tensor get_qwen_mrope_cache(const ModelArgs& args,
```

## R02 — 冯艳（fengyan.119）｜2026/11/23 18:04:50｜`rotary_embedding_util.cpp:91`｜解决状态未核实

- 位置：`xllm/core/layers/common/rotary_embedding_util.cpp:91`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943#discussion_r3531863)

【🔵 minor】【🎨 style】

### 本次已改签名的 yarn_find_correction_dim/_range 仍使用裸 int 和 std::tuple<int, int>

**🔎 问题**

本 MR 将 num_rotations/low_rot/high_rot 改为 float，但同一签名中的 dim、max_position_embeddings 仍为裸 int，返回类型仍为 std::tuple<int,int>，调用点也保留 int low/high；compute_yarn_inv_freq 的 int64_t rotary_dim 和 old_context_len 因此发生隐式窄化。这些是文件内 inline helper，不存在外部 API 要求裸 int 的例外，违反项目的定宽整数规则。

**🔧 修改建议**

将已修改 helper 的整数参数、返回值和调用点一起改为 int64_t：

```cpp
inline double yarn_find_correction_dim(float num_rotations,
                                       int64_t dim,
                                       float theta,
                                       int64_t max_position_embeddings) {
```

```cpp
inline std::tuple<int64_t, int64_t> yarn_find_correction_range(
    float low_rot,
    float high_rot,
    int64_t dim,
    float theta,
    int64_t max_position_embeddings) {
  int64_t low = std::floor(
      yarn_find_correction_dim(low_rot, dim, theta, max_position_embeddings));
  int64_t high = std::ceil(
      yarn_find_correction_dim(high_rot, dim, theta, max_position_embeddings));
  return std::make_tuple(std::max<int64_t>(low, 0),
                         std::min(high, dim - 1));
}
```

```cpp
  int64_t low, high;
  std::tie(low, high) = yarn_find_correction_range(
      beta_fast, beta_slow, rotary_dim, theta, old_context_len);
```

页面差异摘录

```diff
 // Inverse dim formula to find dim based on number of rotations
-inline double yarn_find_correction_dim(int num_rotations,
+inline double yarn_find_correction_dim(float num_rotations,
```
