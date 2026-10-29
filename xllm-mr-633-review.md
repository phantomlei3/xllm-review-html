# xLLM MR #633 — Review 意见与修复交接

- 来源：[MR #633](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633)
- 标题：`perf: accelerate mlu kda and kpool with bangc kernels.`
- 分支：`perf/mlu-kda-kpool-bangc` → `main`
- 页面提交列表首项：`87d2af311272bbb6fd6e0cbc6fd82060b3db09ef`；R05 也将其标为 head。未另查仓库确认 head。
- 页面状态：Open / Reviewing；25 commits、59 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。曾玉婷 (zengyuting12) 于 2026年10月29日 14:05:27 approved changes，事件未附评论正文。
- 提取时间：2026-10-29 14:55:12（Asia/Shanghai）；通过已登录 Chrome 刷新页面后读取。首次读取 Conversation 10，期间新增 R12，刷新后为 Conversation 11，已纳入本次导出。提交前于 2026-10-29 14:56:23 再次刷新，核对 12 条根评论的正文、时间及讨论状态未变。
- 筛选：不按评审者、时间或解决状态过滤；包含作者说明、全部已发布行级意见、独立评审小结和回复，不含提交、reviewed 或无正文的批准事件。
- 清单：12 条根评论（1 条作者说明、10 条行级意见、1 条 Review 小结），0 条回复；页面 Conversation 11 不作为根评论总数。
- 范围：已核对会话中全部讨论正文及回复位置；未见隐藏同类评论、过期线程、分页或加载更多讨论入口。另打开 Files Changed 核对为 59 个变更文件，但未逐一加载全部文件 diff，未独立验证其是否包含会话外讨论；不含草稿。
- 状态说明：10 条行级讨论均仅见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件，逐条标为“状态未核实”；作者说明和独立小结未见解决状态标记。
- 原文说明：评审者的技术判断、严重性、测试声明及作者验证数据未经导出者独立验证。R05 称“未发布 inline finding”，但页面另有同一作者更早发布的 R02–R04，保留两者，不推断其关系。作者说明与 R10 对 chunk size 作用域的陈述也存在差异，原样保留。R03/R04 标题结尾不完整、R07 的 `static_cast(scale)`、R09 的“生产入口”表述和 R12 的 `forward_pcp.cpp` 文件名均保留原文，未擅自修正。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。

| 评论 | 类型 / 主题 | 状态 |
| --- | --- | --- |
| R01 | 作者说明：KDA/KPool BangC 迁移、配置约束与验证 | 未见解决状态标记 |
| R02 | 行级意见：健壮性：extern "C" 与 BangcKdaPcpParams ABI | 状态未核实 |
| R03 | 行级意见：健壮性：建议核算 kernel NRAM 占用 | 状态未核实 |
| R04 | 行级意见：复用：建议补充 `__bang_fusion` 数学语义注释 | 状态未核实 |
| R05 | Review 小结：有条件推荐合并；要求目标 MLU/BangC 环境验证 | 未见解决状态标记 |
| R06 | 行级意见：important / style：false 返回契约与 CHECK 调用方不一致 | 状态未核实 |
| R07 | 行级意见：minor / style：提取 scale 与 gate 下界的具名常量 | 状态未核实 |
| R08 | 行级意见：P2：bangc_kpool_gather 及 device kernel 无调用方 | 状态未核实 |
| R09 | 行级意见：P2：stash_kpool_from_source 公共 API 变为仅测试使用 | 状态未核实 |
| R10 | 行级意见：P2：chunk size 文档与实际作用域不符，chunk_indices 数据流未收敛 | 状态未核实 |
| R11 | 行级意见：P2：KPool indexer 的 Triton / tie 顺序注释过期 | 状态未核实 |
| R12 | 行级意见：P2：PCP prefill 尚未接入 norm/gate 融合 | 状态未核实 |

## R01 — 罗磊 (ext.luolei15)｜2026年10月29日 12:04:26｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，0 条回复。

### 评论原文

#### 描述

将 GLM5-Next 的 MLU KDA 和 KPool 算子迁移到 BangC，优化 prefill、decode 和 MTP 验证性能，并修复长上下文和批量执行中的正确性问题。

- KDA：实现 recurrent、chunk prefill 和 PCP BangC 路径，融合输出 RMSNorm 与 sigmoid 门控，移除旧 Triton 实现。

- KPool：实现缓存更新、打分、Top-K 和索引展开的 BangC 内核，通过流水线和按有效上下文选择减少计算与访存。

- 修复 KDA 分块求逆的数值不稳定、PCP 大规模元数据读取，以及 KPool Top-K 零值边界问题。

- 扩展长上下文、变长批次、padding、状态继承、MTP checkpoint 和 Graph replay 回归覆盖。

- 调整 CTest 环境变量与串行执行配置，单目标测试通过标签运行，确保运行约束生效。

#### 配置约束

- KDA prefill 默认 chunk size 为 64，支持通过 `XLLM_MLU_KDA_CHUNK_SIZE=16` 切换。

- MLU KPool 限定为 BF16、32 heads、head\_dim=128、pool\_size=4、block\_size=16、index\_topk=2048，不支持的配置在初始化时拒绝。

#### 验证

- 最新提交记录：MLU 编译通过，56 次相关内核、层级、chunk override 和双设备 PCP 状态测试执行通过，无跳过。

- 本次仅整理 PR 和 commit 文案，未重新执行编译或测试。

## R02 — 邓英旭 (dengyingxu1)｜2026/10/29 12:18:37｜xllm/core/kernels/mlu/bangc/src/kda/bangc_kda_pcp_prepare.mlu:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484075`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484075)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。
- 代码上下文：页面锚点仅显示文件开头的版权注释或空行，没有与意见对应的实现片段；未据评论补造代码。

### 评论原文

**1. 健壮性 — xllm/core/kernels/mlu/bangc/bangc\_kda\_pcp.h 接口 extern "C"**
位置：`xllm/core/kernels/mlu/bangc/src/kda/bangc_kda_pcp_prepare.mlu:1`

xllm/core/kernels/mlu/bangc/bangc\_kda\_pcp.h 接口 extern "C" void bangc\_kda\_pcp\_prepare\_canonical(cnrtQueue\_t queue, const BangcKdaPcpParams\* params); 使用了 extern "C" 链接，但其参数包含了 C++ 结构体 BangcKdaPcpParams。虽然该结构体目前是 POD 类型，但跨 ABI 边界传递 C++ 结构体指针是不安全的，尤其是在不同编译器或 ABI 环境下，结构体的内存布局和对齐方式可能不一致，且 extern "C" 并不能保证 C++ 结构体布局的完全一致

建议如果该接口仅供 C++ 代码调用，移除 extern "C" 以避免潜在的 ABI 风险；如果必须提供 C 语言接口，建议改为传递基本类型参数，或提供 C 兼容的创建/销毁/访问接口来操作该结构体。

## R03 — 邓英旭 (dengyingxu1)｜2026/10/29 12:18:37｜xllm/core/kernels/mlu/bangc/src/kda/bangc_kda.mlu:1｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484076`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484076)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。
- 代码上下文：页面锚点仅显示文件开头的版权注释或空行，没有与意见对应的实现片段；未据评论补造代码。

### 评论原文

**2. 健壮性 — xllm/core/kernels/mlu/bangc/src/kda/bangc\_kda.mlu 在**
位置：`xllm/core/kernels/mlu/bangc/src/kda/bangc_kda.mlu:1`

xllm/core/kernels/mlu/bangc/src/kda/bangc\_kda.mlu 在 kda\_chunk32\_prefill 和 kda\_prepare\_blocks 等 kernel 中，声明了大量位于 **nram** 的局部数组（如 gram\_k, inverse, power, snapshots 等）。这些数组的总大小可能接近甚至超出 MLU 硬件的 NRAM 容量限制（例如 MLU100 通常为 256KB）。当 NRAM 空间不足时，会导致编译报错或运行时栈溢出崩溃，尤其是在并发执行或复杂调用链场景下

建议精确计算这些 kernel 中所有 NRAM 数组的总内存占用，并在注释中标注。如果占用接近硬件上限，建议考虑复用 NRAM 缓冲区、分块处理或使用动态 NRAM 分配策略以降低峰值内存。

## R04 — 邓英旭 (dengyingxu1)｜2026/10/29 12:18:37｜xllm/core/kernels/mlu/bangc/src/kda/bangc_kda.mlu:2｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：原文未标 P 级严重性。
- 线程锚点：[`discussion_r3484077`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484077)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。
- 代码上下文：页面锚点仅显示文件开头的版权注释或空行，没有与意见对应的实现片段；未据评论补造代码。

### 评论原文

**3. 复用 — xllm/core/kernels/mlu/bangc/src/kda/bangc\_kda.mlu 代码中多处调用了**
位置：`xllm/core/kernels/mlu/bangc/src/kda/bangc_kda.mlu:2`

xllm/core/kernels/mlu/bangc/src/kda/bangc\_kda.mlu 代码中多处调用了 \_\_bang\_fusion(FUSION\_FAM, gate\_window, gate\_window, bias, -rate \* 1.4426950408889634f, gate\_count, kDim); 等复杂的内建函数。这类融合算子的参数顺序和语义（如 FUSION\_FAM 具体代表 A=A\*B+C 还是其他形式）并不直观，缺乏注释说明会导致后续维护者难以理解其计算逻辑，极易在重构或修改时引入错误

建议在调用 \_\_bang\_fusion 等复杂内建函数处添加注释，明确说明该融合操作的具体数学公式（如 gate\_window = gate\_window \* (-rate \* log2e) + bias）以及各参数的对应关系，提高代码可维护性。

## R05 — 邓英旭 (dengyingxu1)｜2026年10月29日 13:13:13｜MR 总体评论｜未见解决状态标记

- 类型：Review 小结；根评论，0 条回复。

### 评论原文

MR 633 评审结论（base 47c03115ecaaf15051b1d3186874cd7f6b07e668，head 87d2af311272bbb6fd6e0cbc6fd82060b3db09ef，merge base aab43e71c11576c114d42faf798f787e2a78f846）。已检查 25 个提交、59 个变更文件，重点覆盖 MLU BangC KDA/KPool kernel、torch wrapper、GLM5-Next/PCP caller 及相关测试。静态调用链检查未发现可确认的 P0-P3 缺陷，因此未发布 inline finding。`git diff --check` 通过；未执行 MLU 硬件、BangC 编译或推理运行，页面中的作者测试结果未作本地复现。未发现未解决的确认性缺陷。合并建议：有条件推荐合并，条件是合并前在目标 MLU/BangC 环境完成编译及相关 KDA/KPool 回归和推理验证。

## R06 — 冯艳 (fengyan.119)｜2026/10/29 13:33:38｜xllm/core/kernels/mlu/bangc/include/kpool_bangc.h:26｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：important / style。
- 线程锚点：[`discussion_r3484128`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484128)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+
+// Each function inspects metadata before launch and returns true when the
+// operation has been dispatched. False means unsupported inputs and no device
+// work or output mutation, so the caller may execute its existing fallback.
```

### 评论原文

#### 【🟠 important】【🎨 style】

#### torch\_api 返回 false 的契约注释与调用方 CHECK 行为不一致

---

**🔎 问题**

> kpool\_bangc.h:24-26 的契约注释承诺 'False means unsupported inputs and no device work or output mutation, so the caller may execute its existing fallback'，但 kpool.cpp 中 update\_kpool(kpool.cpp:140-156)、score\_kpool(kpool.cpp:251-262)、expand\_kpool(kpool.cpp:364-375) 三个调用方都是 CHECK(bangc\_kpool\_\*(...)) 直接崩溃，没有 fallback。注释描述的契约与实际调用方式矛盾：读者按注释新增调用方会以为 false 是可恢复分支，实际是进程终止。同一头文件的其他消费者(select\_kpool/select\_kpool\_live/kpool\_cache\_positions 有 torch fallback)又确实依赖该契约，两种语义混在同一份契约注释下。

---

**🔧 修改建议**

在 kpool\_bangc.h 注释中按函数区分契约：明确标注 update/score/expand 的调用方以 CHECK 视 false 为不可恢复错误，select/gather 系列的调用方以 false 走 fallback。本 diff 内最小修法是改注释，不改变运行语义。

## R07 — 冯艳 (fengyan.119)｜2026/10/29 13:33:38｜xllm/core/kernels/mlu/fused_sigmoid_gating_delta_rule_update.cpp:246｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：minor / style。
- 线程锚点：[`discussion_r3484129`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484129)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+  const bool use_bangc =
+      bangc_same_device && is_kda && kda_use_safe_gate &&
+      use_qk_l2norm_in_kernel && kda_gate_lower_bound == -5.0f &&
+      static_cast<float>(scale) == 0.08838834764831845f && bangc_capacity &&
```

### 评论原文

#### 【🔵 minor】【🎨 style】

#### use\_bangc 谓词用裸浮点字面量相等比较匹配 scale 与 gate 下界

---

**🔎 问题**

> fused\_sigmoid\_gating\_delta\_rule\_update.cpp:245-246 用 kda\_gate\_lower\_bound == -5.0f 与 static\_cast(scale) == 0.08838834764831845f 精确匹配才能进入 bangc 路径。0.08838834764831845 是 1/sqrt(128) 的 double 十进制表示，截断为 float 后与上游传入的 scale 只要有一位 ULP 差异即静默退回 triton 路径，无任何日志。裸魔法常量也无注释说明匹配的是 GLM5 Next 的固定配置。

---

**🔧 修改建议**

把两个常量提为文件级具名 constexpr(如 constexpr float kBangcKdaScale = 0.08838834764831845f; constexpr float kBangcKdaGateLowerBound = -5.0f;)并注释来源(scale=1/sqrt(128)、GLM5 Next 固定配置)。该改动不改变谓词语义，只消除裸字面量与静默 fallback 的可读性陷阱。

## R08 — 马晓龙 (maxiaolong.maxwell)｜2026/10/29 14:47:13｜xllm/core/kernels/mlu/bangc/include/kpool_bangc.h:96｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：P2。
- 线程锚点：[`discussion_r3484370`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484370)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+                                 const torch::Tensor& slot_mapping,
+                                 torch::Tensor& output);
+
+bool bangc_kpool_gather(const torch::Tensor& key,
```

### 评论原文

[P2] 死代码链：`bangc_kpool_gather` + 258 行 device kernel 零调用方仍编入 xllm\_mlu\_ops

**后果**：整条 gather 链（torch\_api 包装 `kpool_bangc.cpp:689-749` → extern "C" launcher `launchers.h:230` → `kpool_gather_wide_members` kernel 本体 `kpool_bangc.mlu:4758-5015`，258 行）被编进 `xllm_mlu_ops` 并导出符号，但生产与测试均无调用方，~330 行三层级死代码。后续读者需要判断它是否活着；kernel 源参与编译拖慢 MLU 构建。

**触发条件**：head（87d2af31）全树。`git grep bangc_kpool_gather 87d2af31` 仅命中声明（kpool\_bangc.h:96）与定义两处；`git grep kpool_gather_wide_members` 仅命中 kernel 定义、launcher 声明和 torch\_api 内部调用。

**证据**：

```cpp
// kpool_bangc.cpp:725（torch_api 内部调用 kernel 后直接返回 true）
::kpool_gather_wide_members(
    static_cast<void*>(torch_mlu::getCurMLUStream(device.index())),
    key.data_ptr(), ...);
return true;
```

**根因**（语义考古三代）：`c57f7c970`（MR 第一个 BangC commit）引入 `try_bangc_kpool_gather`，当时 `kpool.cpp:226` 有真实调用；`05ec3a56` 布局整合保留；`eec040ae`（restrict mlu kpool to glm5 next bangc kernels）删掉最后一个调用方并把函数改名为 `bangc_kpool_gather`，但改名后的函数与 kernel 本体被留下——"生于本 MR、死于本 MR"却没删干净。

**最小修复**：删除 `bangc_kpool_gather`（kpool\_bangc.h:96-107、kpool\_bangc.cpp:689-749）、`kpool_gather_wide_members`（launchers.h:230 起、kpool\_bangc.mlu:4758-5015）；或给出保留理由（如近期复用计划）。

**验证**：删除后全量编译 + `git grep bangc_kpool_gather` 归零。

## R09 — 马晓龙 (maxiaolong.maxwell)｜2026/10/29 14:49:28｜xllm/core/kernels/mlu/kpool.h:54｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：P2。
- 线程锚点：[`discussion_r3484372`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484372)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。
- 时间差异：reviewed 事件为 2026年10月29日 14:49:29，评论自身为 2026/10/29 14:49:28；本标题采用评论时间。

### 页面代码上下文（锚点附近节选）

```diff
+// Nonpositive state IDs represent missing/padding
+// states. All pooling reads must complete before this operation updates the
+// tail.
 void stash_kpool_from_source(const torch::Tensor& k,
```

### 评论原文

[P2] `stash_kpool_from_source` 生产调用方被本 MR 摘除，公共 kernel API 变为 test-only

**后果**：stash 语义已下沉进 `bangc_kpool_update`（torch\_api/kpool\_bangc.cpp:235-255 在 `!decode` 时追加 `kpool_stash_workspace` launch），公共入口 `stash_kpool_from_source`（kpool.h:54 / kpool.cpp:159-203）在生产路径不再被走到，但仍导出、仍被 update\_test.cpp 维护（7 处调用）——公共 API 面与真实数据流脱节，测试覆盖了一个生产不走的路径，给人已验证的错觉。

**触发条件**：`git grep stash_kpool_from_source 87d2af31 -- ':!tests'` 仅命中 kpool.h:54 与 kpool.cpp:159 声明/定义。

**证据**（kpool.cpp:196-202，生产入口只剩这条链，不再经过 stash\_kpool\_from\_source）：

```cpp
if (bangc_kpool_stash(
        raw, gates, tail, tail_ids, read_tail_ids, positions, starts)) {
  return;
}
CHECK(bangc_kpool_tail_generic(...)) << "Unsupported KPool tail inputs";
```

**根因**（语义考古）：base 上 stash 是 prefill 更新的独立第二步（base glm5\_next\_kpool\_indexer.cpp:267 调 `stash_kpool_from_source`）；`c57f7c970` 把 stash 折进 BangC update 的 `!decode` 分支并摘掉两个生产调用方；此后 24 个 commit 一直保留孤儿入口。语义没丢（下沉进了 update），丢的是 API 边界一致性。

**最小修复**：删除该公共 API（测试改为直接测 `bangc_kpool_update` 的 stash 语义），或在 kpool.h:48-53 注释里明确"仅测试使用/保留原因"。

**验证**：`git grep stash_kpool_from_source` 生产树归零；kpool 测试仍通过 update\_kpool 的 prefill 路径覆盖 stash 语义。

## R10 — 马晓龙 (maxiaolong.maxwell)｜2026/10/29 14:50:13｜xllm/core/kernels/mlu/chunk_kda.h:26｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：P2。
- 线程锚点：[`discussion_r3484374`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484374)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
 // XLLM_MLU_KDA_CHUNK_SIZE may select 16.
```

### 评论原文

[P2] `XLLM_MLU_KDA_CHUNK_SIZE` 的文档契约与实际作用域不符；chunk\_indices 成为"计算+校验但不消费"的悬空数据流

**后果**：运维按环境变量名调参会得出"改了没效果"的错误结论——env 只影响 PCP 分片（glm5\_next\_kda\_pcp.cpp:198）、PCP build\_context（glm5\_next.h:72）与 conv block 调度表粒度（glm5\_next.h:342），**不影响 KDA staged prefill kernel 的分块**（固定 32-token）。同时每个非 PCP prefill 都在为 chunk\_indices 付出 torch 链 + `.item<int32_t>()` 设备同步的成本，而 head 的两条 KDA prefill 路径都不消费它。

**触发条件**：任何非 PCP 的 GLM5-Next MLU prefill；设置 `XLLM_MLU_KDA_CHUNK_SIZE=16` 期望影响主 prefill 路径时。

**证据**：

```cpp
// chunk_kda.h:25-27（契约注释）
// Returns the chunk size used by the KDA prefill kernel. The default is 64;
// XLLM_MLU_KDA_CHUNK_SIZE may select 16.
int64_t kda_prefill_chunk_size(int64_t num_heads, bool use_qk_l2norm);

// chunk_kda.cpp:231（staged workspace 按 32-token 切 slot，与 env 无关）
const int64_t slots = (total_tokens + 31) / 32 + num_sequences;

// chunk_kda.cpp:240-257（staged native ABI：无 chunk_size / chunk_indices 参数）
CHECK_EQ(bangc_kda_prefill_staged(static_cast<void*>(queue),
             q_work.data_ptr(), ..., cu_work.data_ptr(),
             workspace.defined() ? workspace.data_ptr() : nullptr,
             static_cast<int32_t>(num_sequences), ...), 0);

// glm5_next_kda.cpp:640-641（调用方硬性要求 chunk_indices 存在——假契约）
CHECK(attn_metadata.chunk_indices.defined())
    << "GLM5-Next KDA prefill requires chunk indices";
```

**根因**：Triton→BangC 迁移时 chunk 语义从"kernel 分块参数"（base chunk\_kda.cpp:426 真实传给 Triton kernel）变成"调度/分片参数"，但头文件契约、CHECK 和 chunk\_indices 生成链没有同步收敛。

**最小修复**：① 更新 chunk\_kda.h:25-27 注释为"PCP 分片与 conv block 调度的 chunk 粒度；KDA staged prefill kernel 固定 32-token 分块"；② 评估 chunk\_indices 生成是否可从非 PCP prefill 路径移除（conv 只需要 batch/token\_block\_offset/tot）。

**验证**：设置 env=16/64 跑 prefill 层测试，确认输出不变（staged 路径下应完全不变）。

## R11 — 马晓龙 (maxiaolong.maxwell)｜2026/10/29 14:52:29｜xllm/core/kernels/mlu/kpool.h:81｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：P2。
- 线程锚点：[`discussion_r3484380`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484380)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。
- 时间差异：reviewed 事件为 2026年10月29日 14:52:30，评论自身为 2026/10/29 14:52:29；本标题采用评论时间。

### 页面代码上下文（锚点附近节选）

```diff
-// Native torch.topk tie ordering; nonfinite selections and missing columns -1.
+// The ID set torch.topk(count) selects per row, in unspecified order;
+// nonfinite selections and missing columns -1.
 torch::Tensor select_kpool(const torch::Tensor& scores, int64_t count);
```

### 评论原文

[P2] `glm5_next_kpool_indexer.h` 的函数契约注释仍写"Triton scoring / torch::topk tie 顺序"，与本 MR 改后的实现相反

（主现场在 `xllm/core/layers/mlu/glm5_next/glm5_next_kpool_indexer.h:46-47`，该文件不在本 MR diff 内，但本 MR 的改动使其陈述失效，故锚在本 MR 写下的新契约处。）

**后果**：头文件承诺两件已不成立的事——scoring 走 Triton（实际 BangC）、等分顺序遵循 torch::topk（实际顺序不指定）。按旧注释写的下游断言（如逐位置比较 selected ids）会间歇性失败且难以定位。

**触发条件**：任何读者/后续开发者按头文件注释理解 `glm5_next_kpool_select` 语义时；依赖 tie 顺序做跨版本对比测试时。

**证据**：

```cpp
// extra/head glm5_next_kpool_indexer.h:46-47（失效的旧注释）
// Select causal pools with paged Triton scoring and torch::topk. Query chunks
// reuse bounded scores. Equal-score ordering follows native torch::topk.

// 本 MR 改写后的新契约（kpool.h:79-81）
// The ID set torch.topk(count) selects per row, in unspecified order;
// nonfinite selections and missing columns -1.
```

**根因**：kpool.h 的契约改写与本 MR 改动的相邻头文件注释未联动。

**最小修复**：更新 glm5\_next\_kpool\_indexer.h:46-47 注释，与新契约一致（集合语义、顺序不指定、BangC scoring）。

**验证**：纯注释修改，无行为验证需求。

## R12 — 马晓龙 (maxiaolong.maxwell)｜2026/10/29 14:54:43｜xllm/core/layers/mlu/glm5_next/glm5_next_kda.cpp:693｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性 / 标签：P2。
- 线程锚点：[`discussion_r3484387`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/633#discussion_r3484387)；未见 OutDated 标记。
- 状态证据：仅见处理动作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
   output_gate = output_gate.view({-1, head_dim_});
-  torch::Tensor normalized = o_norm_->forward(core_output);
-  normalized = glm5_next_kda_apply_output_gate(normalized, output_gate);
+  auto fused_output = kernel::mlu::bangc_kda_norm_gate(
```

### 评论原文

[P2] 本 MR 引入的 `bangc_kda_norm_gate` 融合没有覆盖 PCP prefill 路径（forward\_pcp 仍走两段式）

**后果**：cp\_size\>1 的 GLM5-Next MLU prefill（PCP 路径）每层每步多走 `o_norm_->forward` + `glm5_next_kda_apply_output_gate` 两个 kernel，而单卡/非 PCP prefill 与 decode 都走单 kernel 融合——同一语义两条代码路径，PCP 路径损失一次 kernel launch 与中间张量往返。数值上无风险（未融合路径就是旧路径）。

**触发条件**：cp\_size\>1 的 GLM5-Next MLU prefill。

**证据**：

```cpp
// 本 MR 新增的融合接线（glm5_next_kda.cpp:693-701，forward() 尾部）
auto fused_output = kernel::mlu::bangc_kda_norm_gate(
    core_output, output_gate, o_norm_->weight(), norm_eps_);
torch::Tensor normalized;
if (fused_output.has_value()) {
  normalized = std::move(*fused_output);
} else {
  normalized = o_norm_->forward(core_output);
  normalized = glm5_next_kda_apply_output_gate(normalized, output_gate);
}

// PCP 路径未接线（glm5_next_kda_pcp.cpp:247-248，forward_pcp 尾部，本 MR 未改该文件）
torch::Tensor normalized = o_norm_->forward(core_output);
normalized = glm5_next_kda_apply_output_gate(normalized, output_gate);
```

两条路径输入同为 `core_output.view({-1, head_dim_})` 与 `output_gate`、同一份 weight 与 eps；`bangc_kda_norm_gate` 谓词（bangc\_kda\_norm\_gate.cpp:31-43）只要求 BF16/2D/128/contiguous，PCP 输入同样满足。

**根因**：融合接线只改了 `forward()` 所在文件；`forward_pcp` 在另一文件，接线时漏配。

**最小修复**：在 forward\_pcp.cpp:247 前插入与 glm5\_next\_kda.cpp:693-701 相同的融合+回退块（`norm_eps_` 成员可直接访问），或注明不融合的原因。

**验证**：PCP 路径已有测试（chunk\_kda\_pcp\_test / glm5\_next decoder-layer PCP 用例），接线后跑通即等价；可加一个融合 vs 未融合的 output 位级对比用例。
