# xLLM MR #833 — Review 意见

- 来源：[MR #833](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/833)
- 标题：`bugfix: align draft cache sizing with MLU parallel layout.`
- 分支：`bugfix/fix-draft-cache` → `main`；2 commits、4 files changed。
- 页面状态：Open / Reviewing；要求 2 人批准，等待人工评审；2 items passed、1 item failed；必需的 xLLM 服务 CI 流水线 #9271455 显示 failed。
- 当前 head：页面未显示独立 head 字段，未核实；提交列表展示 `7d5ba4f934ea8d2dfa697e0d8fadf4f3802f88a4` 和 `5ef0508df161466334f3ae6fa83130aa9f8f0e5c`，不据此推断 head。
- 提取时间：2026-11-16T18:13:11+08:00；通过已登录 Chrome 刷新并等待加载后读取；写入后再次刷新核对，作者说明及全部评论正文一致。
- 筛选：未指定作者或解决状态过滤；收录全部已读取评论及作者说明。
- 清单：2 条行级根评论、0 条已发布回复、0 条独立评审小结；另有 MR 作者说明 1 条，共 3 条根内容及 0 条回复。页面 Conversation 为 2。Conversation 与 Files Changed 中相同 ID 的评论已去重，回复输入框不计为回复。
- 解决状态：2 个行级线程均只见 No Processing Required / Resolved 操作按钮，未见明确解决事件，逐线程标为未核实。页面 AICR 汇总显示已处理(0)、未处理(0)，不能据此确定这 2 个线程的状态。
- 范围限制：已检查加载的 Conversation 和 Files Changed DOM；未发现隐藏评论组、评论分页、load more 控件或 OutDated 标记。未独立遍历全部历史差异版本，不能排除未加载的历史内容；不含草稿。
- 原文说明：以下技术判断及测试结果属于评审者或 MR 作者，未经导出者独立验证。作者说明中的测试通过记录与页面当前 CI failed 状态分别保留，二者未核实为同一次运行。R02 原文类型片段显示为 `std::vectorstd::shared_ptr get_worker_clients()`，其中 `std::shared_ptr` 被平台渲染为异常链接；保留可见文字，不推断补全模板参数。R01 建议注释的换行取自页面 DOM，并加代码围栏。

## 行动索引（导出者摘要）

| 评论 | 作者 | 严重性 / 类别 | 主题 | 回复数 | 线程状态 |
| --- | --- | --- | --- | --- | --- |
| R01 | 冯艳（fengyan.119） | minor / style | 为公开函数声明补充 Draft 并行布局调整的语义注释 | 0 | 未核实 |
| R02 | 冯艳（fengyan.119） | minor / perf | 增加只读计数访问器，避免仅取长度时复制 worker client 容器 | 0 | 未核实 |

## 作者说明 — ext.luolei15｜2026/11/16 14:52:24｜MR 描述

### 问题描述

MLU 使用 Block Diffusion 投机解码且目标模型启用 CP 时，Draft Worker 实际采用独立于 CP 的 TP 布局，但 Draft Engine 仍沿用目标模型的并行配置，导致每个 rank 的 KV Cache 容量估算与实际分配布局不一致。

### 主要修改

- 提取 build_draft_engine_options，统一构建 Draft Engine 配置。
- 在控制节点依据实际 Worker 数量，将 Draft 配置调整为 CP=1、TP=world_size/DP，保留 DP 划分，使容量估算与 Worker 布局一致。
- 非控制节点保留原有配置，避免因本地 WorkerClient 列表为空而触发检查失败，影响多进程启动。

### 测试覆盖

新增回归测试，覆盖 CP 场景的缓存预算与形状、CP/DP 混合布局、Decode 与 MTP 配置保持，以及非控制节点无 WorkerClient 时的启动路径。

根据分支提交记录，MLU 编译通过，Engine 测试 25/25、KV Cache 估算测试 39/39 通过，已执行 clang-format 20.1.6；本次确认分支差异的 git diff --check 通过。

## R01 — 冯艳｜2026/11/16 16:02:59｜xllm/core/distributed_runtime/engine_utils.h:31｜状态未核实

- 类型：行级根评论；账号 `fengyan.119`；minor / style；0 条已发布回复。
- 线程：[discussion_r3516294](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/833#discussion_r3516294)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：锚定新文件第 31 行。以下是页面显示的新增行片段，声明后续部分未包含在该线程预览中。

```diff
+runtime::Options build_draft_engine_options(const runtime::Options& options,
```

### 根评论原文

【🔵 minor】【🎨 style】

#### 新公开函数 build_draft_engine_options 声明处缺语义注释

---

**🔎 问题**

> 本函数携带 node_rank==0 && cp_size>1 && uses_cp_independent_tp_for_block_diffusion_draft && is_block_diffusion_algorithm 四重条件，命中时把 draft 的 world_size/cp_size/tp_size 重塑为 CP 并入 TP 的布局，语义非自明。同文件相邻声明 build_decode_graph_warmup_config（engine_utils.h:34-35）与 configure_dsv4_block_manager_options（engine_utils.h:39-49）均在声明处有注释，本函数没有；解释只分散在 engine_utils.cpp 实现内（engine_utils.cpp:44、53），头文件使用者无从得知何时触发 reshape。项目规则要求 non-obvious semantics 的新函数在声明处有简短注释（[custom-code-style-cpp.md](http://custom-code-style-cpp.md) 第 519-520 行）。

---

**🔧 修改建议**

在 engine_utils.h:31 声明前补注释，例如：

```cpp
// Builds the draft engine's options from the target's. On the controller
// (node_rank == 0), CP x TP block-diffusion drafts merge CP into the draft
// TP domain (cp_size = 1, tp_size = world_size / dp_size) so cache sizing
// follows the draft worker's parallel layout.
```

## R02 — 冯艳｜2026/11/16 16:02:59｜xllm/core/distributed_runtime/speculative_engine.cpp:70｜状态未核实

- 类型：行级根评论；账号 `fengyan.119`；minor / perf；0 条已发布回复。
- 线程：[discussion_r3516295](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/833#discussion_r3516295)
- 状态证据：所在讨论仅显示处理操作按钮，未见明确解决事件。
- 位置说明：锚定新文件第 70 行；以下为页面显示的新文件第 68–70 行片段。
- 原文异常：下文 `std::vectorstd::shared_ptr get_worker_clients()` 按页面可见文字保留，未修复异常类型片段。

```cpp
    runtime::Options draft_engine_options = build_draft_engine_options(
        options_,
        static_cast<int64_t>(dist_manager_->get_worker_clients().size()));
```

### 根评论原文

【🔵 minor】【⚡ perf】

#### 为读取 worker 数量而按值复制整个 worker client 向量

---

**🔎 问题**

> DistManager::get_worker_clients() 按值返回整个容器（dist_manager.h:31，std::vectorstd::shared_ptr get_worker_clients()），本行只用它取 .size()，却因此拷贝 world_size 个 shared_ptr 并逐个做引用计数增减。另外四个调用点（dit_engine.cpp:75、llm_engine.cpp:1348、rec_engine.cpp:298、vlm_engine.cpp:718）都要把容器本身赋给 worker_clients_，说明该访问器不是为计数设计的，本行是唯一只取长度的用法。这次多余拷贝发生在 DistManager 已建好全部 client、正在构造 draft 引擎的启动路径上，world_size 越大代价越大。

---

**🔧 修改建议**

给 DistManager 增加只读计数访问器，本行改用它。dist_manager.h 中新增：

```cpp
size_t num_worker_clients() const { return worker_clients_.size(); }
```

本行改为：

```cpp
    runtime::Options draft_engine_options = build_draft_engine_options(
        options_, static_cast<int64_t>(dist_manager_->num_worker_clients()));
```

其余四个需要容器本身的调用点保持不动。
