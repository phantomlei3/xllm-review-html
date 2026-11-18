# xLLM MR #866 — Review 意见

- 来源：[MR #866](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/866)
- 标题：`perf: optimize mlu engram hashing, lookup, fusion and communication.`
- 分支：`perf/mlu-engram-fused-pipeline` → `main`；21 commits、47 files changed（7947 additions、295 deletions）。
- 页面状态：Open / Reviewing；要求 2 人批准；冯艳（fengyan.119）于 2026/11/17 20:17:57 留有 approved changes 事件，22:09:10 又发布本次评审意见。页面仍提示等待人工评审；状态检查为 2 items passed、1 item failed，必需的 xLLM 服务 CI 流水线 #9271455 显示 failed，合并条件尚未满足。上述事件无单独评论正文，不计入评论数量。
- 当前版本：页面未显示独立 head 字段，未核验 Git 引用；提交列表首项为 `3402460862781736e92a156c7643446fe563a09e`。评审者自述审查范围为 `920bdf8023668e03344f22d8f849dd20349ce635..3402460862781736e92a156c7643446fe563a09e`，此范围为评审原文，未独立验证。
- 提取时间：2026-11-18T09:39:06+08:00；通过已登录 Chrome 刷新并等待加载后读取，展开 MR 作者说明，检查 Conversation 和相关 Files Changed 线程；提交前再次刷新核对正文及数量。
- 筛选：用户未指定评审者、回复或解决状态过滤；收录全部已读取的评审评论及作者说明。
- 清单：1 条行级根评论、0 条已发布回复、0 条独立评审小结，与页面 Conversation 1 一致；另保留 MR 作者说明 1 条，共 2 条根内容及 0 条回复。Conversation 的 `discussion_r3522467` 与 Files Changed 的 `r3522467` 是同一评论，已去重；回复输入框不计为回复。
- 解决状态：R01 仅显示 No Processing Required / Resolved 操作按钮，未见明确解决事件，标为未核实。页面 AICR 的已处理(0)、未处理(0)不用于判断线程状态。
- 范围限制：未发现隐藏评论组、评论分页、load more 控件或 OutDated 标记。Files Changed 的部分大差异按需加载，本次检查了评审所在文件及已加载的讨论，未独立遍历全部历史差异版本和其余文件的所有大差异，不能排除未加载的历史内容；不含草稿。
- 原文说明：以下技术判断和修改建议均来自评审者或 MR 作者，未经导出者独立验证。作者明确说明 `placement=gpu` 改为 `placement=device` 且旧启动参数需调整；R01 要求兼容旧值，两种表述分别保留，不推断该兼容性取舍已经达成一致。作者默认配置原为带换行的普通段落，以下按原文值加 JSON 围栏；未改写正文。

## 行动索引（导出者摘要）

| 评论 | 作者 | 严重性 / 类别 | 主题 | 回复数 | 线程状态 |
| --- | --- | --- | --- | --- | --- |
| R01 | 冯艳（fengyan.119） | important / api | 建议将 placement=gpu 作为 device 的废弃别名接受，保留旧配置兼容性 | 0 | 未核实 |

## 作者说明 — ext.luolei15｜2026/11/17 19:57:36｜MR 描述

perf: optimize mlu engram hashing, lookup, fusion and communication.

### PR 描述

本 PR 优化 DeepSeek V4.1 在 MLU 上的 Engram 执行链路，覆盖 hash 生成、CPU 查表、Host/Device 数据传输、反量化、门控计算，以及 CP/TP 通信和投影，减少逐层同步、重复计算和中间张量搬运。

### Engram 优化项

1. CPU 查表并行化

   将串行 FP8 权重及 scale 行收集改为 torch::parallel_for，大 prefill 使用受 intra-op 线程数约束的并行查表，小 decode 保持在 callback 线程执行。为查表实现单独启用 OpenMP，确保 ATen 并行循环实际使用多线程。

2. 提前预取各层查表结果

   在 decoder stack 执行前启动各 Engram 层的输入准备，通过独立 MLU stream 执行 CPU hash、查表和上传，使后续层的查表工作有机会与前面的模型计算重叠。Graph 路径在 capture/replay 前准备输入，消费端通过 event 等待结果就绪。

3. 复用上游 Host token，减少逐层 D2H

   将输入构建阶段已有的 CPU token 快照传递给 Engram；快照有效时直接用于 CPU hash，避免每层从 MLU 回读 token。Eager 模式需要回读时，由协调器统一回读一次供各层使用；投机验证、DFlash proposal 和含负值占位符的输入会拒绝复用失效快照。

4. 合并权重与 scale 传输，复用 staging buffer

   将查表得到的 FP8 权重和 E8M0 scale 打包到同一个 pinned buffer，并通过一次 packed H2D 上传。复用 Host/Device staging，减少分散传输和重复分配；通过 allocation、upload、consumption event 约束跨 stream 的写入与复用，避免覆盖仍被上一轮使用的数据。

5. 共享 CPU 表按 TP hash head 分工查表和上传

   在已有 replication=per_node 模式下，传输量足够大时，各 TP rank 仅查询并上传自己负责的 hash heads，再在 TP 组内 AllGather 压缩后的 packed payload，减少重复 CPU 读取和 H2D 流量。反量化直接读取 rank-major 布局，避免额外重排；预计节省的 Host 传输量不足 512 KiB 时保留独立查表路径。

6. Device 驻留表跨层合并 hash 计算

   新增 BangC engram_hash，直接使用 device tokens、请求边界和 lookback history。表常驻 MLU 时，一次 kernel launch 生成全部 Engram 层的 hash，替代逐层 CPU hash 和 Host/Device 往返；各层共享一份 device hash 常量，减少重复存储。

   Kernel 对满足参数范围的输入使用向量化整数取模和请求边界扫描，其他参数范围使用通用整数路径，保持 hash 结果与 CPU 语义一致。

7. CPU 表的 eager prefill 默认使用 Device hash

   在保留 CPU 大表的同时，将普通 eager prefill 的 hash 计算迁移到 MLU。仅将 hash 索引下载至 pinned buffer 供 CPU 查表，device hash 直接保留给后续计算，避免再次上传。

   该优化要求已有 device token 快照；CPU 表的 decode、混合批次、投机验证及 graph replay 继续使用 CPU hash，可通过 prefill_hash=cpu 显式选择 CPU 路径。

8. 融合 FP8 反量化与 Device 查表

   CPU 表路径新增 packed 反量化 kernel，将 FP8 查表解码、E8M0 scale 展开、所属分片判断及 BF16 输出合并执行，替代多个 Torch 中间算子。

   Device 表路径新增 gather + dequantize kernel，直接读取命中的权重和 scale 并生成反量化结果，避免先物化 gather 张量。常用的每 32 个值共享 scale 的布局采用批量处理和异步搬运。

9. 融合门控计算并接入 MLU Matmul

   WKV 投影改用已有 MLU Matmul 接口，直接消费原始权重布局。新增 engram_gate，融合 hidden/key 归一化统计、加权点积、门控变换、sigmoid 和残差更新，减少 Torch kernel 启动及 FP32 中间张量读写。

   保留图像特殊 token 的 hidden 透传语义，并覆盖极端权重、非有限值和舍入顺序等数值边界。

10. 按负载自适应选择融合算子粒度

    反量化及 gather kernel 根据工作量调整每个 task 的行数，在短 decode 的并行度与长 prefill 的批量处理效率之间选择合适粒度。Gate 对短批次采用 token/head 任务，对适用的大批次采用跨 head 的 token kernel，其他形状使用通用实现。

11. CP 通信改用 ReduceScatter，延后 TP AllGather

    对启用 CP 的非共享表，将完整 embedding 的 AllReduce 改为 EP 组内 ReduceScatter，使各 rank 仅接收自己负责的 token 行。

    TP 大于 1 时，继续保持 token 分片执行 WKV 投影和 gate，最后才在 TP 组内 AllGather 最终 hidden，减少各 TP rank 对相同 token 的重复投影，并缩小 AllGather 的数据范围。显式配置 communication=allreduce 可使用原有通信策略。

12. 将 CP 重排合并到反量化输出

    根据 CP token 归属构建统一的目标行索引，由反量化/gather kernel 直接按 ReduceScatter 所需顺序写出，省去完整 embedding 的额外重排和复制。TP AllGather 后通过 view 合并维度并裁掉 padding，支持不均匀 CP 行数和部分 TP shard 仅含 padding 的情况。

### 配套调整

- 新增层级输入协调器 DeepseekV41EngramInputs，统一管理跨层 hash、eager 预取、graph staging 和 CP 布局，模型负责传入 tokens 并分发各层输入。

- 空闲 DP shard 跳过 Engram 准备和计算。

- 集中解析和校验 MLU Engram 配置，同步更新中英文文档。

- placement=gpu 改为 placement=device，已有启动参数需要同步调整。

当前默认配置为：

```json
{
"placement": "cpu",
"replication": "none",
"communication": "reduce_scatter_allgather",
"prefill_hash": "device"
}
```

其中，communication 的新路径仅用于启用 CP 的非共享表；prefill_hash 仅控制 CPU 表的适用 eager prefill 场景。

## R01 — 冯艳｜2026/11/17 22:09:10｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:99｜状态未核实

- 类型：行级根评论；账号 `fengyan.119`；important / api；0 条已发布回复。
- 线程：[discussion_r3522467](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/866#discussion_r3522467)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决记录或已发布回复。
- 位置说明：锚定新文件第 99 行；正文自述涉及新文件第 98–99 行及旧文件第 78 行。以下是 Conversation 线程预览显示的新文件第 96–99 行，保留新增标记，未补全预览外代码。

```diff
+  }
+  EngramOptions options;
+  options.cpu_offload =
+      engram_option(config, "placement", "cpu", {"cpu", "device"}) == "cpu";
```

### 根评论原文

#### 【🟠 important】【🔌 api】

#### engram_config 的 placement 不再接受 "gpu"，已部署配置升级后启动即 FATAL

---

**🔎 问题**

> parse_engram_options 的 placement 合法值从旧侧的 {"cpu","gpu"} 改为 {"cpu","device"}（新 98-99 行 engram_option(config, "placement", "cpu", {"cpu", "device"})）。旧版本接受 engram_config='{"placement":"gpu"}' 正常运行（旧文档 en/zh 均记载 gpu 为关闭 offload 的公开值，旧代码 CHECK(placement == "cpu" || placement == "gpu")）；升级到本版本后同一配置在启动时直接 LOG(FATAL)（engram_option 对非法 choice 崩溃），已部署的 MLU 放置配置服务无法拉起。这是对既有配置契约的破坏性收紧，本 diff 内没有任何兼容层。

---

**🔧 修改建议**

把 choice 集合改为 {"cpu", "device", "gpu"}，将 "gpu" 作为 "device" 的废弃别名接受：`options.cpu_offload = engram_option(config, "placement", "cpu", {"cpu", "device", "gpu"}) == "cpu";` 语义完全不变（cpu_offload 只按 == "cpu" 判定），同时保持旧配置可启动。帮助文本与文档已用 device，无需回改。

---

**🔬 复现步骤** `🔧 静态代码审查（diff base 920bdf8023668e03344f22d8f849dd20349ce635..head 3402460862781736e92a156c7643446fe563a09e），MLU 后端代码路径`

静态比对：旧侧 deepseek_v41_engram.cpp:78 为 CHECK(placement == "cpu" || placement == "gpu")；新侧同函数 choices 只剩 {"cpu", "device"}。用旧版本以 --engram_config='{"placement":"gpu"}' 启动的服务升级到本 HEAD 后同一命令在 engram_option 处 FATAL。
