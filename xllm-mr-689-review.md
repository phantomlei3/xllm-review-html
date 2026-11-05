# xLLM MR #689 — Review 意见

- 来源：[MR #689](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689)
- 标题：`feat: support mlu engram on shared memory among local ranks.`
- 分支：`feat/mlu-engram-tp-shared-memory` → `main`；5 commits、14 files changed。
- 页面状态：Open / Reviewing；有邓英旭的 approved changes 事件，不代表 MR 整体已批准。
- 当前 head：未从独立 head 字段核实；页面提交首项链接为 `a6e25fb955a9e4118f5f702df931dda67cfd046d`；评审总结报告 base=`79546340b`、head=`a6e25fb95`。
- 提取时间：2026-11-05T09:40:32+08:00；通过已登录 Chrome 刷新页面，并逐一切换全部 8 个 View Changes 评审事件。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评论和回复。
- 清单：7 条行级根评论、4 条行级回复；另有作者说明、LGTM 评审评论、冯艳自动审查总结、马晓龙评审总结各 1 条。共 11 条根评论、4 条回复。页面 Conversation 13 不作为评论总数。
- 线程状态：仅见 No Processing Required / Resolved 操作按钮，未核实实际解决状态。R02 的末条回复称“可标记为已解决”，这是评论者的判断，不等于已发生解决事件。
- 范围限制：已核对全部 8 个评审事件的已加载讨论，未发现讨论分页、加载更多、隐藏同类评论或过时线程控件；部分完整文件 diff 未加载，不能独立排除其他或过时讨论。不含草稿。
- 位置说明：标题使用评论前页面差异行的新侧行号，无新侧时注明旧侧；评论正文报告的位置照录。差异片段仅包含页面已有代码。
- 原文说明：技术判断均为评审者意见，未经导出者独立验证。保留各方相互冲突的判断；R06 的 `:unknow_emoji:` 等异常文本照录，不自行修复。

## 行动索引（导出者摘要）

| 评论 | 作者 | 主题 | 回复数 | 状态 |
| --- | --- | --- | --- | --- |
| R01 | 邓英旭 | allgather 元数据形状与 memmove；回复对前提提出反证 | 2 | 未核实 |
| R02 | 邓英旭 | boot_id 哈希身份判断；回复讨论失效边界与修复判断 | 2 | 未核实 |
| R03 | 马晓龙 | 每表重复收集 boot_id / node_ranks | 0 | 未核实 |
| R04 | 马晓龙 | 屏障后恒真 CHECK | 0 | 未核实 |
| R05 | 马晓龙 | memfd seals 防御加固 | 0 | 未核实 |
| R06 | 马晓龙 | 平台配置语法分裂与旧配置迁移 | 0 | 未核实 |
| R07 | 马晓龙 | DP1 门禁下 per_node 内存收益边界 | 0 | 未核实 |

## R01 — 邓英旭｜2026/11/4 19:14:32｜xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:1｜状态未核实

- 类型：行级根评论，2 条已发布回复。
- 线程：[discussion_r3495412](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495412)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

**1. 正确性 — xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp**
位置：`xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:1`

xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp 在 load\_shared\_engram\_tensor 函数中，通过 gather\_metadata 收集 SharedEngramDescriptor（包含 9 个 int64\_t 字段）。gather\_metadata 内部调用了 allgather\_base\_sync，该操作在 PyTorch 等框架中默认会将张量沿第 0 维拼接，因此对于形状为 {9} 的输入，返回的张量形状是 {world\_size \* 9} 的一维张量，而非 {world\_size, 9} 的二维张量。随后代码执行 std::memmove(&descriptor, metadata[owner].data\_ptr<int64\_t>(), sizeof(descriptor))，如果 metadata 是一维张量，metadata[owner] 将获取的是一个标量（0维张量），其 data\_ptr 指向第 owner 个元素。当 owner > 0 时，memmove 会从该位置读取 72 字节（sizeof(descriptor)），导致读取越界并获取错误的元数据

建议在 gather\_metadata 返回后，显式将结果重塑为 {world\_size, 9}，例如 metadata = metadata.view({group.world\_size(), 9});，或者直接使用 metadata.narrow(0, owner \* 9, 9).data\_ptr<int64\_t>() 来获取正确偏移量的数据指针。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
```

### 回复 1 — 马晓龙｜2026/11/4 20:38:31

- 评论 ID：`discussion_r3495556`；属于上述线程，页面未明确指定回复哪一条子评论。

补充一个对这条 finding 的源码验证结论：**前提不成立，这不是一个 bug**——`allgather_base_sync` 的输出形状由 wrapper 自己分配，是堆叠（stacked）而不是平坦拼接（flat concat）。

证据链（均在 head a6e25fb95 核实）：

1. `ProcessGroup::allgather_base_sync`（xllm/core/framework/parallel\_state/process\_group.cpp:263-271）自行分配输出：

```cpp
torch::Tensor output =
    torch::empty(get_gather_shape(world_size(), input), input.options());
allgather_base_async(input, output)->wait();
```

2. `get_gather_shape`（process\_group.cpp:54-63）把 world\_size **前置为第 0 维**：

```cpp
out_shape.push_back(world_size);
for (int64_t dim_size : input.sizes()) {
  out_shape.push_back(dim_size);
}
```

即 `{9}` 输入 → 输出形状 **`{world_size, 9}`** 二维张量，而非一维 `{ws*9}`。

3. `allgather_base_async`（process\_group.cpp:243-258）还显式校验输出就是堆叠形状：

```cpp
CHECK_EQ(output.sizes(), torch::IntArrayRef(out_shape))
    << "output shape mismatch for allgather_base_async";
```

因此在 `gather_metadata` 返回后，`metadata[owner]`（engram\_shared\_memory.cpp:141）取到的是第 owner 行的 `{9}` 视图，`data_ptr` = 基址 + owner\*72 字节；`sizeof(SharedEngramDescriptor)` 被 `static_assert` 钉死为 72 字节（engram\_shared\_memory.cpp:67），`std::memmove(&descriptor, ..., 72)` 恰好复制一整行。owner ∈ node\_ranks ⊂ [0, ws)，行索引合法，**无越界、无错位**。

c10d 文档里 `_allgather_base` 的「平坦写入」描述的是字节布局（堆叠与平坦在内存字节上确实相同），差别在张量逻辑形状——而 `metadata[owner]` 的语义由逻辑形状决定。8-rank 测试（EightRanksShareRawBytesAndMappingLifetime 等）真实走过这条路径；若 memmove 错位，sm.cpp:143-148 的 descriptor 四元组一致性 CHECK（bytes/rows/columns/name）几乎必然先失败。

另外一个小提醒：原建议的 `metadata.narrow(0, owner * 9, 9)` 在真实（堆叠）语义下反而是错的——对 `{ws,9}` 二维张量按 dim0 narrow 到 9「行」会把行数当元素数用。

可选的 low 级防御加固（非必须）：`gather_metadata` 返回后加一行 `CHECK_EQ(metadata.size(0), group.world_size())`（或 `metadata.view({group.world_size(), 9})`，当前语义下是 no-op），防止未来某个 `ProcessGroup` 子类改返回一维拼接时静默读错字节。

### 回复 2 — 邓英旭｜2026/11/4 20:39:35

- 评论 ID：`discussion_r3495558`；属于上述线程，页面未明确指定回复哪一条子评论。

针对你的疑问补充说明如下：
[correctness] xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp
位置：xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp:1
问题：xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp 在 load\_shared\_engram\_tensor 函数中，通过 gather\_metadata 收集 SharedEngramDescriptor（包含 9 个 int64\_t 字段）。gather\_metadata 内部调用了 allgather\_base\_sync，该操作在 PyTorch 等框架中默认会将张量沿第 0 维拼接，因此对于形状为 {9} 的输入，返回的张量形状是 {world\_size \* 9} 的一维张量，而非 {world\_size, 9} 的二维张量。随后代码执行 std::memmove(&descriptor, metadata[owner].data\_ptr<int64\_t>(), sizeof(descriptor))，如果 metadata 是一维张量，metadata[owner] 将获取的是一个标量（0维张量），其 data\_ptr 指向第 owner 个元素。当 owner > 0 时，memmove 会从该位置读取 72 字节（sizeof(descriptor)），导致读取越界并获取错误的元数据
建议：在 gather\_metadata 返回后，显式将结果重塑为 {world\_size, 9}，例如 metadata = metadata.view({group.world\_size(), 9});，或者直接使用 metadata.narrow(0, owner \* 9, 9).data\_ptr<int64\_t>() 来获取正确偏移量的数据指针。

如果此处确属必要写法或有特殊上下文，欢迎进一步说明，我们会重新评估该条建议。

## R02 — 邓英旭｜2026/11/4 19:14:32｜xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:4｜状态未核实

- 类型：行级根评论，2 条已发布回复。
- 线程：[discussion_r3495413](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495413)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

**2. 正确性 — xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp**
位置：`xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:4`

xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp 在 host\_identity 函数中，使用 std::hash<std::string>{}(identity) 将 /proc/sys/kernel/random/boot\_id 转换为 int64\_t。std::hash 的结果在不同编译器或标准库实现下可能不同，且存在极小概率的哈希冲突。如果同一节点上的不同容器或虚拟机由于某种原因共享了相同的 boot\_id（或者不同节点发生了哈希冲突），会导致错误的节点身份识别，进而使得不同物理机上的进程错误地共享同一块内存映射，引发严重的数据损坏

建议使用更可靠的节点标识符（如完整的 boot\_id 字符串直接比较，或使用定长哈希如 SHA256 截断），或者在 SharedEngramDescriptor 中直接传递并比较字符串，避免依赖 std::hash 的稳定性和唯一性。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
+
+Licensed under the Apache License, Version 2.0 (the "License");
+you may not use this file except in compliance with the License.
```

### 回复 1 — 马晓龙｜2026/11/4 20:42:23

- 评论 ID：`discussion_r3495565`；属于上述线程，页面未明确指定回复哪一条子评论。

对这条 finding 的源码验证结论：**前提部分成立（std::hash 确实是实现定义），但「严重的数据损坏」这一后果不成立**——身份误判的失效模式被两道闸全部转化为启动期 fail-loud 崩溃。

逐子命题裁决：

1. 「std::hash 跨编译器/标准库实现不同」——字面为真，但同一模型 world 组的全部 rank 运行**同一二进制**，一次部署内 `host_identity()` 必然一致；跨实现差异只影响哈希值跨部署可比，而本用途只要求同次运行内相等。
2. 「哈希冲突 → 不同物理机被认成同机」——64 位空间、集群规模 n（几~几百台）下碰撞概率 ~n²/2^65，可忽略。同机容器共享同一 boot\_id 不是异常而是必然（同一 kernel），且这正是「同机 = 可经 /proc 共享存储」想要的语义。
3. 「引发严重的数据损坏」——沿代码路径走完（engram\_shared\_memory.cpp:149-161）：若 node\_ranks 混入异机 rank，peer 执行 `open("/proc/<异机pid>/fd/<fd>")`——跨主机 PID 不可见，open 失败 → `PCHECK` 立即 abort；即便本机凑巧存在同号 PID，打开的是无关文件，fstat 的 `st_dev/st_ino/st_size` 三重比对必失败 → `CHECK` abort。不存在静默共享错表 → 数据损坏的路径：

```cpp
const int32_t fd = ::open(path.c_str(), O_RDWR | O_CLOEXEC);
PCHECK(fd >= 0) << "Cannot open shared Engram storage; ...";
struct stat info{};
PCHECK(::fstat(fd, &info) == 0);
CHECK(static_cast<int64_t>(info.st_dev) == descriptor.device &&
      static_cast<int64_t>(info.st_ino) == descriptor.inode &&
      info.st_size == bytes)
    << "Shared Engram descriptor identity mismatch";
```

可选的低优先级加固（同意原 finding 的方向）：gather 全量 boot\_id 字符串（36 字节/rank）或截断 SHA-256 代替 64 位折叠，可同时消除「实现定义哈希」的理论争议；descriptor 已是 9×int64 定长信封，加一个字段成本极小。但这是打磨项，不是正确性修复。

### 回复 2 — 邓英旭｜2026/11/4 20:47:28

- 评论 ID：`discussion_r3495574`；属于上述线程，页面未明确指定回复哪一条子评论。

已在当前提交确认相关改动：在 xllm/core/layers/mlu/deepseek\_v41/engram\_shared\_memory.cpp:4 附近发现新增行 [1, 2, 3, 4, 5, 6, 7]。该 note 可标记为已解决，感谢修复。

## R03 — 马晓龙｜2026/11/4 20:44:40｜xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:100｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3495570](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495570)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

[P2] 加载期 boot\_id / node\_ranks 推导每表重复执行：同一 host 拓扑被 allgather 2L 次

**后果**：`replication=per_node` 时，每个 engram 层对 `embed.weight` 与 `embed.scale` 各调一次 `load_shared_engram_tensor`（deepseek\_v41\_engram.cpp:125-131），每次都重新读 `/proc/sys/kernel/random/boot_id` + 在 world 组上做一次 {1} allgather（含 device 往返）+ 重新推导 node\_ranks。L 个 engram 层共 2L 次重复。payload 微小、加载一次性，性能上可忽略；主要是 maintainability 问题——进程级不变量（同机拓扑）的推导逻辑复制在每张表的加载路径里，每处都需保持一致。

**触发条件**：`replication=per_node` 且模型有 L 个 engram 层，每次模型加载。

**代码**（engram\_shared\_memory.cpp:100-117，每次调用都执行）：

```cpp
const int64_t host = host_identity();
const auto hosts =
    gather_metadata(torch::tensor({host}, torch::kInt64), group);
const int64_t* host_ids = hosts.data_ptr<int64_t>();
std::vector<int32_t> node_ranks;
// ... 按 host_ids 推导 node_ranks / local_rank / owner
```

**根因**：`load_shared_engram_tensor` 以「单表自包含集合操作」为边界设计，进程级不变量（同机拓扑）未抽出。

**最小修复**：把 `host_identity()` 结果与 node\_ranks 推导提为按 world 组缓存（进程级 `std::once` 或以 `ProcessGroup*` 为 key 的小缓存），`load_shared_engram_tensor` 接收 `{owner, local_rank, local_size}`；或接受现状并加一行注释说明「每表重复、量级可忽略」。不阻塞合入。

**验证**：现有 4 个多进程测试（EightRanks / TwoRanks / TwoDpFourTp / SeparateHosts）重跑即可，行为无变化。

### 页面差异片段

```diff
+ CHECK_LT(group.rank(), group.world_size());
+
+ const int64_t bytes = source.numel();
+ const int64_t host = host_identity();
```

## R04 — 马晓龙｜2026/11/4 20:46:56｜xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:179｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3495573](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495573)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

[P2] 屏障 allreduce 之后的 CHECK 是恒真算术自我验证

**后果**：maintainability（自我验证式防护，无法因业务错误失败）；每次共享表加载多一次 `ready.to(CPU).item()` 同步拷贝与一次恒真比较。

**触发条件**：无——该 CHECK 在所有可达路径下恒真。

**代码**（engram\_shared\_memory.cpp:174-180，`load_shared_engram_tensor` 尾部）：

```cpp
if (group.world_size() > 1) {
  auto ready = torch::ones(
      {1},
      torch::TensorOptions().dtype(torch::kInt64).device(group.device()));
  group.allreduce(ready);
  CHECK_EQ(ready.to(torch::kCPU).item<int64_t>(), group.world_size());
}
```

`ready` 由 `torch::ones` 初始化为 1，world 组 allreduce（求和）后数学上恒等于 world\_size——判据左端是本函数自己刚生成的数据经恒等运算的结果。allreduce 本身才是屏障（集合通信的到达语义已保证所有 rank 的 `table.slice(...).copy_(...)` 先于各自 allreduce 完成）；CHECK 只重复确认了算术。若集合通信栈本身损坏，allreduce 会挂起或崩溃，而非返回错误和。

**最小修复**：删除 `CHECK_EQ` 行，注释改为说明屏障目的（如 `// barrier: publish CPU writes before any rank queries`）；或保留并注明 rationale（例如防御某后端 allreduce 错误求和的场景）。最小 diff。

**验证**：现有 4 个多进程测试重跑（均经过此屏障，跨进程写可见性断言已覆盖屏障语义）。

### 页面差异片段

```diff
+        {1},
+        torch::TensorOptions().dtype(torch::kInt64).device(group.device()));
+    group.allreduce(ready);
+ CHECK_EQ(ready.to(torch::kCPU).item<int64_t>(), group.world_size());
```

## R05 — 马晓龙｜2026/11/4 20:49:15｜xllm/core/layers/mlu/deepseek_v41/engram_shared_memory.cpp:123｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3495579](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495579)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

[P2] memfd 未设置 seals：持有 fd 的任意同机进程可 ftruncate 共享表

**后果**：operability/robustness。owner 创建匿名 memfd 后经 `/proc/<pid>/fd/<fd>` 分发给同机 peers（engram\_shared\_memory.cpp:149-161），每个 peer 拿到指向同一文件的独立可写 fd。memfd 的 seal 机制未使用，因此任何持有该 fd 的进程都能在任意时刻 `ftruncate` 改变文件长度，使其他进程的 mmap 区间与文件大小脱钩（SIGBUS 或静默截断）。当前威胁模型下无 correctness 影响——全部 rank 运行同一二进制，代码库中不存在加载后 ftruncate 的调用路径；这是防御未来代码或第三方库误操作的加固项。

**触发条件**：同机、同 world 组内的某个 rank 在加载完成后对该表 fd 执行 ftruncate（当前代码库无此路径）。

**代码**（engram\_shared\_memory.cpp:123-126）：

```cpp
const int32_t fd = ::memfd_create("xllm-engram", MFD_CLOEXEC);
PCHECK(fd >= 0) << "Cannot create shared Engram storage";
PCHECK(::ftruncate(fd, bytes) == 0);
mapping = std::make_shared<SharedEngramMapping>(fd, bytes);
```

**根因**：memfd seal 是可选特性，默认关闭；共享内存的「长度不变」契约目前只存在于约定中。

**最小修复**：创建时带 `MFD_ALLOW_SEALING`，并在 sm.cpp:174 屏障后由 owner 追加 `fcntl(fd, F_ADD_SEALS, F_SEAL_SHRINK | F_SEAL_GROW)`（`F_SEAL_WRITE` 不可用——peers 加载期仍需写自己的行区间；查询期各 rank 只读，封 shrink/grow 不影响）。注意 `SharedEngramMapping` 需暴露 fd 或在 owner 分支内联调用。低优先级，非必须。

**验证**：可选新增断言「seal 后 ftruncate 返回 EPERM」的单测。

### 页面差异片段

```diff
+  std::shared_ptr<SharedEngramMapping> mapping;
+  SharedEngramDescriptor descriptor;
+ if (group.rank() == owner) {
+ const int32_t fd = ::memfd_create("xllm-engram", MFD_CLOEXEC);
```

## R06 — 马晓龙｜2026/11/4 20:51:48｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:70｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3495584](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495584)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

[P2] engram\_config 语法按平台分裂 + 旧语法（含顶层布尔）无过渡期直接拒绝

**后果**：compatibility / operability。MLU 原生侧本 MR 后只认 `placement`/`replication`（本行所在 CHECK），NPU 的 Python 侧 parser（xllm/python/models/engram\_config.py:38,104，本 MR 未改）仍只认 `cpu_offload`/`dp_shared_memory`，而同一 gflag 被 py\_causal\_lm.cpp:552-553 原样转发给两侧——两个 parser 的合法键集在 MR 后**互不相交**（此前双方都认 `cpu_offload`）。因此：① MLU 部署带旧配置 `--engram_config='{"cpu_offload":true}'` 或 `='true'` → 启动期 CHECK abort（fail-loud，测试已覆盖）；② NPU 部署带新语法 → Python `ValueError`；③ NPU/MLU 机群混用同一 gflags 文件时，除 `{}` 外任何显式 engram 配置都无法两平台通用。

**触发条件**：见上①②③。语义考古：`cpu_offload`（#659 出生）语义被 `placement` 完整继承（true:unknow\_emoji:"cpu"、false:unknow\_emoji:"gpu"），无契约丢失；MLU 侧 `dp_shared_memory` 出生即被拒（墓碑清理）；唯一被删除的「活语法」是顶层布尔（`--engram_config='true'`）。

**代码**（deepseek\_v41\_engram.cpp:68-74，MLU 原生侧只认 placement/replication）：

```cpp
CHECK(config.is_object()) << "engram_config must be a JSON object";
for (const auto& [key, value] : config.items()) {
  CHECK(key == "placement" || key == "replication")
      << "Unknown Engram configuration key: " << key;
  CHECK(value.is_string())
      << "Engram configuration " << key << " must be a string";
}
```

base（79546340b）同时接受顶层布尔与 `cpu_offload`：

```cpp
CHECK(config.is_object() || config.is_boolean()) << ...;
cpu_offload_ = config.is_boolean() ? config.get<bool>()
                                   : config.value("cpu_offload", true);
```

**根因**：同一 flag 承载两套平台方言，两侧 parser 从未共享键表；本 MR 把 MLU 侧单方面改写加深了分裂。

**最小修复**：fail-loud 本身是对的，不建议恢复旧键兼容（污染新语义）。建议在 en/zh 文档 engram\_host\_offload.md 增加「旧配置迁移」两行表（`cpu_offload:true → {"placement":"cpu"}` 等）；中期可考虑两侧 parser 共享一份键-方言映射或按平台命名空间拆 flag。

**验证**：文档改动无需测试；拒绝行为已有 EXPECT\_DEATH 矩阵覆盖（RejectsUnknownOrUnsupportedConfiguration）。

### 页面差异片段

```diff
-        << "MLU Engram does not support dp_shared_memory";
+ CHECK(config.is_object()) << "engram_config must be a JSON object";
+ for (const auto& [key, value] : config.items()) {
+ CHECK(key == "placement" || key == "replication")
```

## R07 — 马晓龙｜2026/11/4 20:54:18｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:407｜状态未核实

- 类型：行级根评论，0 条已发布回复。
- 线程：[discussion_r3495586](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/689#discussion_r3495586)
- 状态证据：仅见处理与解决操作按钮，实际状态未核实。

### 根评论原文

[Question] per\_node 的节点内存收益在整模型 DP1 门禁下当前不可达，何时解除？

**背景**：MR 动机是「避免不同 DP 组各自重复持有表数据」——内存收益的来源是同机多个 DP 组合一份表。但 MLU 原生 V4.1 整模型构造函数硬性 CHECK `dp_size()==1`（xllm/models/llm/mlu/deepseek\_v41.h:57，本 MR 未改），即当前生产拓扑下 world 组 = 同机 TP/CP/EP ranks、无 DP 复制。

**推导**：DP1 时默认分片模式（`replication=none`）在 lookup\_group\_（覆盖全部 world ranks）上切行，节点合计本就约等于一份完整表（doc 的 23.60 GiB/rank × 8 ≈ 188.83 GiB）；`per_node` 模式节点同样是一份完整表。**节点物理内存基本不变**。变化的是：

1. 每 rank RSS 记账上升（映射全表；doc 已提醒看 PSS 口径）；
2. 每层每 iteration 少一次 lookup\_group\_ AllReduce（本行所在代码，**这是当前唯一实际可达的收益**）：

```cpp
if (!node_shared_memory_) {
  rows = parallel_state::reduce(rows, lookup_group_);
}
```

**想请作者确认**：DP>1 支持的路线图与本特性内存收益的兑现时间点；在此之前，doc 的内存表述是否应补一句「DP1 部署下 per\_node 不省节点内存，当前收益为跳过查表 AllReduce」——避免用户为省内存开启 per\_node，实际只得到通信收益，反而增加每进程 RSS 记账。

**验证**：（可选）一次 TP8 单机 per\_node vs none 的节点物理内存对照，预期两者接近相等。功能本身正确、测试完备（DP2×TP4 在 Engram 层已验证共享与 eager/graph 一致性）。

### 页面差异片段

```diff
   rows = rows.view({hashes.size(0), cols, dim}) * owned.unsqueeze(-1);
   rows = rows.to(hidden.scalar_type());
- rows = parallel_state::reduce(rows, lookup_group_);
+ if (!node_shared_memory_) {
```

## A01 — 作者说明｜2026年11月4日 18:36:37｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

feat: 支持 MLU Engram 表在同机模型 ranks 间共享。

变更说明

为原生 MLU DeepSeek-V4.1 Engram 新增节点内共享存储模式，使同一模型实例的同机 ranks 跨 attention DP/TP 组共享一份完整 FP8 权重表及 scale，避免不同 DP 组重复持有表数据。

* 使用模型 world 进程组识别同机 ranks，通过匿名 memfd 和 mmap 共享存储；各 rank 分工加载 checkpoint 行，完成同步后开放查询，存储随最后一个引用释放自动回收。
* 保留原始 FP8 权重及 scale 字节，仅搬运当前命中行到 MLU 反量化；共享模式跳过查表结果的 AllReduce，并复用现有 Graph host callback。
* MLU 配置统一为 placement=cpu/gpu 和 replication=none/per\_node，默认 cpu/none。旧布尔配置及 cpu\_offload、dp\_shared\_memory 等旧键不再接受；同步更新中英文文档、参数帮助和相关测试。

启用方式：

```bash
--engram_config='{"placement":"cpu","replication":"per_node"}'
```

测试与验证

* 新增共享字节可见性、映射生命周期、模拟多主机隔离，以及 TP2、DP2 × TP4 下 eager 与 Graph replay 一致性测试。
* 提交记录注明已通过 MLU 编译闸口及 14 项 Engram 回归；本次生成描述未重新执行。

使用限制

共享 ranks 必须位于同一 PID namespace，并有权限访问 leader 的文件描述符。不同主机各保留一份完整表，独立服务实例之间不共享。DP2 × TP4 覆盖限于 Engram 层，原生 MLU V4.1 整模型当前仍要求 DP1。

## S01 — 邓英旭 (dengyingxu1) LGTM 评审评论｜2026年11月4日 18:44:01｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

LGTM，改动比较收敛，没什么问题

## S02 — 冯艳 (fengyan.119) 自动审查总结｜2026年11月4日 19:45:40｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

### 🤖 自动审查小结：**LGTM，这代码看着真舒服！**

本轮静态代码审查未发现需要修改代码的问题。

## S03 — 马晓龙 (maxiaolong.maxwell) 评审总结｜2026年11月4日 20:58:50｜全局评论

- 类型：独立根评论，非行级线程回复；未见明确解决状态。

### 独立深度审查小结（教学式 review 流水线产物）

对 79546340b...a6e25fb95（5 commits，14 文件，+795/-49）做了源码级独立审查，结论：**无 P0/P1，建议合入**。共享内存机制本身实现严谨（fstat 三重身份校验、descriptor 四元组一致性、from\_blob deleter 生命周期与 graph 地址稳定契约、行分工边界含不整除情形、真实多进程 fork 测试矩阵均到位）。

已发布评论（详见各 inline / 回复）：

* 对 note:3495412（allgather 形状）的验证回复：**前提不成立**——`allgather_base_sync` 按 `get_gather_shape` 分配 `{ws,9}` 堆叠输出，`metadata[owner]` 是整行视图，memmove 72B（static\_assert 钉死）无越界；原建议的 `narrow(0, owner*9, 9)` 在堆叠语义下反而是错的修法。
* 对 note:3495413（boot\_id 哈希）的验证回复：**前提部分成立、后果不成立**——同二进制内 hash 必一致；即便冲突，/proc open PCHECK + fstat 三重校验使失效模式为启动期 fail-loud，无数据损坏路径。
* [P2] 加载期 boot\_id/node\_ranks 每表重复 allgather（2L 次，量级可忽略，建议缓存或注释）。
* [P2] 屏障后 CHECK\_EQ(ones-allreduce, world\_size) 恒真自我验证（建议删或注明 rationale）。
* [P2] memfd 未设 F\_SEAL\_SHRINK/GROW（加固项，当前无触发路径）。
* [P2] engram\_config 语法按平台分裂（MLU 认 placement/replication、NPU Python 侧仍认 cpu\_offload/dp\_shared\_memory、键集互不相交）+ 顶层布尔旧语法断代拒绝，建议 doc 补迁移表。
* [Question] per\_node 的节点内存收益在整模型 DP1 门禁（deepseek\_v41.h:57）下当前不可达，当前实际收益只有跳过查表 AllReduce；请作者确认 DP 路线图或 doc 补一句收益边界。

建议处理顺序：恒真 CHECK（一行）→ 迁移表（两行 doc）→ 收益边界（一句话 doc）→（可选）host 缓存 / memfd seals。以上均不阻塞合入。
