# xLLM MR #908 — Review 意见

- 来源：[MR #908](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/908)
- 标题：`perf: fuse deepseek v4.1 dsa projections on mlu.`
- 分支：`feat/perf-dsv41-indexer-qk` → `main`。页面显示 10 个提交、42 个变更文件、8402 行新增和 250 行删除。
- 当前版本：提交列表首项为 `4e225c9a1892d27dee8c249fe3dd793aaa64e6b7`。页面未单独标记 head，未核验 Git 引用。
- 页面状态：Open / Reviewing，需 2 人批准。状态检查显示 3 项通过，合并仍等待人工评审。
- 提取时间：2026-11-19T20:59:02+08:00。通过已登录 Chrome 刷新读取 Conversation，使用 Files Changed 确认第三条评论的完整路径。
- 筛选：收录全部可见评论，未按作者或解决状态筛选。
- 清单：页面 Conversation 计数为 3，对应 3 条行级根评论、0 条已发布回复、0 条独立评审小结。另保留 1 条 MR 作者说明。
- 范围：Conversation 未见分组隐藏、评论分页、加载更多评论或过期差异标记。Files Changed 按需加载，本次未逐一检查全部 42 个文件及历史差异。三条线程均显示状态操作按钮，未见明确解决记录，解决状态均标为未核实。
- 以下技术判断和验证数据来自评审者或 MR 作者，未经独立验证。

## 行动索引（导出者摘要）

| 评论 | 严重性 / 类别 | 建议 |
| --- | --- | --- |
| R01 | minor / architecture | 将模型配置中的 RMSNorm eps 传给 fused projection，移除重复硬编码及精确等值断言。 |
| R02 | minor / reuse | 将 fused DSA query projection 中的局部对齐 lambda 替换为已有的 `output_projection::align256`。 |
| R03 | minor / license | 为四个输出投影文件使用项目标准 Apache-2.0 notice 和 LICENSE 链接。 |

## 作者说明 — ext.luolei15｜2026/11/19 17:15:33｜MR 描述

在 MLU590 上为 DeepSeek V4.1 主模型和 DSpark 草稿模型接入三组 BangC 融合算子，减少 DSA 注意力投影中的算子启动和中间张量搬运。

- 输入投影　融合 wq_a、wkv、RMSNorm 和 KV RoPE，并按层合入压缩器投影。
- 查询投影　融合 wq_b 和 RoPE。支持符合 TP 分片宽度要求的 BF16 和带 smooth 的动态 W8A8 权重，其余配置沿用原线性层和 RoPE 路径。
- 输出投影　融合逆 RoPE、wo_a 分组 W8A8 投影和 wo_b 输入的逐 token 量化。wo_b 复用 scaled_matmul，多卡 TP 下归约一次。主模型和 DSpark 的 wo_a、wo_b 均要求 INT8 权重及对应的 weight_scale 和 smooth。
- 权重加载　DSpark 根据 checkpoint 分片中的实际权重格式初始化 decoder，兼容 BF16、动态 W8A8 和打包 W4A8 专家权重。Engram wkv 支持 BF16 和动态 W8A8。

### 验证

- MLU 编译通过。输出投影及相关模型、Graph、CP、DP 回归共 76/76 项通过。
- TP8 + DSpark（K=5）+ Graph 服务在 MMLU-Pro computer science 上答对 59/64 题（92.19%）。64 条请求的 DSpark 接受率均超过 15%，最低为 22.52%。

## R01 — 冯艳（fengyan.119）｜2026/11/19 19:27:40｜`deepseek_v41_attention.cpp:80`｜解决状态未核实

- 位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_attention.cpp:80`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/908#discussion_r3526973)

【🔵 minor】【🏛️ architecture】

### eps 在 attention 与 fused projection 中重复固定，配置没有单向传入 kernel

**🔎 问题**

> 构造函数新增 CHECK_EQ(args_.rms_norm_eps(), 1e-20f)，而新 fused projection 的 projection::init_projection 又独立执行 n.eps = 1e-20f。ProjectionArgs 不携带 eps，导致同一模型参数由两处硬编码共同维持；其他 decoder/attention 路径仍从 args_.rms_norm_eps() 读取。后续支持不同 eps 时必须同步修改多处，当前断言只是拒绝不一致配置，并没有建立正确的数据流。

**🔧 修改建议**

给 projection::ProjectionArgs 增加 float eps，并让 init_projection 使用 args.eps；在 fused_dsa_projection 的接口中接收调用方 eps 并填入 ProjectionArgs，attention 调用时传 args_.rms_norm_eps()，然后删除这里的精确等值 CHECK。现有 1e-20f 配置结果不变，同时消除第二处常量。

页面差异摘录

```diff
+  CHECK_EQ(args_.q_lora_rank(), 1280);
+  CHECK_EQ(args_.head_dim(), 512);
+  CHECK_EQ(args_.qk_rope_head_dim(), 64);
+  CHECK_EQ(args_.rms_norm_eps(), 1e-20f);
```

## R02 — 冯艳（fengyan.119）｜2026/11/19 19:27:41｜`deepseek_v41.cpp:360`｜解决状态未核实

- 位置：`xllm/core/kernels/mlu/bangc/torch_api/deepseek_v41.cpp:360`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/908#discussion_r3526974)

【🔵 minor】【♻️ reuse】

### fused_dsa_qb 重复实现同文件已有的 align256 helper

**🔎 问题**

> large INT8 分支新增局部 align256 lambda，而本文件已经 include output_projection.h，其中 output_projection::align256 提供完全相同的 256 字节向上对齐。两份本 MR 新增实现分别使用除法和位运算，后续调整 workspace 对齐规则时可能只改一处。

**🔧 修改建议**

删除第 360-362 行的局部 lambda，将 token_offset、weight_offset 和 workspace 总大小三处调用改为 output_projection::align256(...)。该 helper 已经可见，替换后的偏移和分配大小保持一致。

页面差异摘录

```diff
+  } else {
+    // The large GEMM consumes a 32-lane BF16 encoding of each scale.
+    constexpr int64_t kScaleBytes = 32 * sizeof(uint16_t);
+    const auto align256 = [](int64_t bytes) {
```

## R03 — 冯艳（fengyan.119）｜2026/11/19 19:27:41｜`output_projection.h:1`｜解决状态未核实

- 位置：`xllm/core/kernels/mlu/bangc/src/deepseek_v41/fused_dsa_output_projection/output_projection.h:1`。[源讨论](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/908#discussion_r3526975)

【🔵 minor】【📄 license】

### output projection 四个新文件的 Apache-2.0 notice 缺少项目 LICENSE 链接

**🔎 问题**

> output_projection.h、output_projection_device.h、output_projection_device.mlu 和 output_projection_host.mlu 都使用一行式版权头，没有项目规范要求的 https://github.com/xLLM-AI/xllm/blob/main/LICENSE 链接。

**🔧 修改建议**

将这四个文件的一行式首行替换为项目标准多行 Apache-2.0 notice，并在 You may obtain a copy of the License at 下加入 <https://github.com/xLLM-AI/xllm/blob/main/LICENSE>；形式与同 MR 的 torch_api/deepseek_v41.cpp 文件头一致。

页面自动链接将建议段落中的分号和后续汉字吞进链接地址。此处保留评审原文，将链接显示范围限定为 LICENSE URL。

页面差异摘录

```diff
+// Copyright 2026 The xLLM Authors. Licensed under Apache-2.0.
```
