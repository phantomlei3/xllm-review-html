# xLLM MR #943 — Review 意见

- 来源：[MR #943](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943)
- 标题：`feat: support yarn for qwen3.8 on mlu.`
- 分支：`feat/qwen3.8_yarn` → `main`。页面显示 3 个提交、13 个变更文件、281 行新增和 80 行删除。
- 当前版本：Commits 列表首项为 `fa999914b3eb35798c442705489faf812e0fef32`。
- 页面状态：Open / Reviewing，需 2 人批准。时间线显示冯艳于 11 月 24 日 13:17 批准，3 项状态检查均已通过。
- 提取时间：2026-11-24 14:20（UTC+08:00）。通过已登录的 Chrome 刷新读取 Conversation，并核对了 Commits 和 Files Changed。
- 筛选与计数：仅收录本轮新增的 2 条行级根评论、0 条已发布回复、0 条独立评审小结，另保留 MR 作者说明。已移除上次导出的两条旧意见。
- 范围：Conversation 显示 6，时间线含历史评审和审批事件。新意见的解决状态均未核实。此次核对了新意见对应的差异，历史差异未逐一核对。
- 以下技术判断来自评审者，未经独立验证。

## 行动索引（导出者摘要）

| 评论 | 严重性 / 类别 | 建议 |
| --- | --- | --- |
| R01 | important / architecture | 移除共享 Qwen3.5 YaRN 配置加载器中的 MLU 平台断言，保留参数解析与校验。 |
| R02 | minor / reuse | 让两条 RoPE 路径共用一个缓存管理器，保留 Qwen 自有的缓存构建逻辑。 |

## 作者说明 — ext.luolei15｜2026/11/23 14:59:39｜MR 描述

### Bugfix

honor qwen yarn settings in target and draft models.

### Feature

support yarn for qwen3.8 on mlu.

## R01 — 冯艳（fengyan.119）｜2026/11/24 13:25:53｜`qwen3_5.h:226`｜解决状态未核实

- 位置：`xllm/models/vlm/qwen3_5.h:226`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943#discussion_r3533153)

【🟠 important】【🏛️ architecture】

### 共享 Qwen3.5 配置加载器中的 MLU 断言使其他后端的 YaRN 配置加载失败

**🔎 问题**

load_qwen3_5_rope_scaling 被 NPU、MLU、MUSA、DCU 的 Qwen3.5 模型参数注册共同使用，但新增 CHECK(Platform::is_mlu()) 在非 MLU 编译中恒为 false。这些后端加载 rope_type=yarn 的同一配置时会在参数解析阶段直接退出；基线没有该断言，且 DCU 的 Qwen3.5 decoder 使用自己的 MRotaryEmbedding，并不依赖 MLU 的 get_qwen_mrope_cache。后端消费策略不应阻断共享配置解析。

**🔧 修改建议**

删除第 226 行的 CHECK(Platform::is_mlu())，保留后续 YaRN 参数解析、数值校验和 rope_type 归一化，使非 MLU 后端维持基线的参数加载行为。

页面差异摘录

```diff
+  CHECK(Platform::is_mlu()) << "Qwen3.5 YaRN requires the MLU backend";
```

## R02 — 冯艳（fengyan.119）｜2026/11/24 13:25:53｜`rotary_embedding_util.cpp:398`｜解决状态未核实

- 位置：`xllm/core/layers/common/rotary_embedding_util.cpp:398`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/943#discussion_r3533154)

【🔵 minor】【♻️ reuse】

### 新增 Qwen RoPE 路径为同一 CosSinCacheDesc 键域创建第二个静态缓存管理器

**🔎 问题**

compute_cos_sin_cache 在 336 行使用一个 static CosSinCacheManager，而 get_qwen_mrope_cache 在 398 行又创建独立 static 实例。两者使用相同的 CosSinCacheDesc 和 get_or_create_cache 键语义；相同描述符无法跨两个 manager 去重，会分别初始化并保留缓存。

**🔧 修改建议**

在文件作用域提供一个返回同一 CosSinCacheManager 实例的 accessor，让 336 行与 398 行共同使用。保留 get_qwen_mrope_cache 自己的 factory：通用 create_cos_sin_tensor 会按 scaling_factor 扩展 positions，而 Qwen factory 只物化 serving window，不能直接改调 compute_cos_sin_cache。

页面差异摘录

```diff
+  static CosSinCacheManager qwen_cache;
```
