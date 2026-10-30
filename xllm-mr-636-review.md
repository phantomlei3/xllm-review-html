# xLLM MR #636 — zengyuting12 Review 意见

- 来源：[MR #636](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636)
- 标题：`bugfix: reject oversized vlm inputs before worker dispatch.`
- 分支：`bugfix/vlm_oversize` → `main`
- 页面提交列表首项：`0b0d9465eee64196242d1d74d4ec247282f6f446`，已打开 Commits 页核对；未另查仓库确认 head。
- 页面状态：Open / Reviewing；5 commits、27 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。
- 提取时间：2026-10-30 19:54:46（Asia/Shanghai）；通过已登录 Chrome 刷新页面，加载 show more 并展开折叠讨论后读取。提交前于 2026-10-30 19:55:16 再次刷新、加载并展开讨论，核对所选两条评论的正文、时间、锚点和讨论控件未变。
- 筛选：仅收录曾玉婷（`zengyuting12`）发布的两条行级意见；删除此前导出的全部其他评论，不含其他作者回复、作者说明或评审小结。不另按时间或解决状态过滤。
- 清单：2 条根评论（P2、P3 各 1 条），0 条回复。已加载会话正文共 30 条根评论、0 条回复，其中仅这两条属于 zengyuting12；页面 Conversation 29 不作为根评论总数。
- 状态说明：两条线程仅见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件，逐条标为“状态未核实”。
- 范围：会话记录已加载至末尾，折叠讨论已展开，未见剩余 show more、隐藏同类评论或其他分页入口；所选线程未见 OutDated 标记。未逐一打开全部文件 diff，无法独立排除会话外讨论；不含草稿。
- 原文说明：以下严重性、技术判断、代码示例和建议均为评审者原文，未经导出者独立验证。R01 建议代码中的 `fail(part 的请求)` 为原文示意写法，未改写为可编译实现。

## 行动索引（导出者摘要）

以下仅概括评论主题，不代表导出者确认问题成立。

| 评论 | 原文严重性 | 主题 | 状态 |
| --- | --- | --- | --- |
| R01 | P2 | overlap 模式中，一个超限请求导致同一 DP rank 整批失败；建议按请求预检或修正错误提示 | 状态未核实 |
| R02 | P3 | 原子分支无法拆分时，将整批超限归因到每个请求；建议区分归因或调整批级错误提示 | 状态未核实 |

## R01 — 曾玉婷 (zengyuting12)｜2026/10/30 18:32:12｜xllm/core/distributed_runtime/vlm_engine.cpp:466｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性：P2（评审者原文）。
- 线程锚点：[`discussion_r3487341`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487341)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+        continue;
+      }
+      for (Sequence* sequence : batch[dp_rank].get_sequences()) {
+        sequence->fail(Status(
```

### 评论原文

[P2] overlap 模式下，一个超限请求会让同一 DP rank 上**整批**请求一起失败

开启 `--enable_schedule_overlap` 时，超限分支按 DP rank 整体处理：

```cpp
if (sizes[dp_rank] <= limits[dp_rank]) { continue; }
for (Sequence* sequence : batch[dp_rank].get_sequences()) {
  sequence->fail(Status(StatusCode::RESOURCE_EXHAUSTED, "...reduce multimodal input size or disable schedule overlap..."));
}
batch[dp_rank] = Batch();
```

`sizes[dp_rank]` 是整个 rank 的聚合大小，只要其中**一个**请求的图片/视频超限，同一 rank 上本来完全合法的请求会一起收到
`RESOURCE_EXHAUSTED`，错误信息还让它们「减小输入或关掉 overlap」——对它们来说是误导，客户端可能拿同样的输入重试。

注释里「overlap 只有一个 outstanding result slot，不能一个 scheduler step 发多次 forward」是对的，
但这条约束**只禁止多次 forward**，不要求把整批都打死：完全可以按请求预检（`split_by_request()` 已经有现成的
拆分粒度），把超限的那几个请求 `fail` 掉、从 batch 里摘掉，重新 `prepare_inputs` **一次**，再发**一次** forward。
下面的非 overlap 分支就是这么做的（`rejected` 之后 `sub_inputs = prepare_inputs(sub_batch)`），两条路径的容错度不必不一致。

建议改成：

```cpp
// 先按请求拆分做 per-request 预检（不提交状态）
auto parts = batch[dp_rank].split_by_request();
std::vector<Batch> keep;                     // 或直接把合法的序列搬回 batch
for (auto& part : parts) {
  const auto s = forward_input_sizes(prepare_inputs({part})).front();
  if (s > limits[dp_rank]) { fail(part 的请求); continue; }
  keep.push_back(std::move(part));
}
// 合并 keep（同一 rank 只能发一次），重新 prepare_inputs 后单次 execute_batch
```

如果确实出于复杂度考虑先保留现状，至少把错误信息改成「本批含超限请求，请重试」这种不指导客户端改输入的措辞，
并单独计数（现在这条路径没有任何 metric，线上只能靠日志猜）。

## R02 — 曾玉婷 (zengyuting12)｜2026/10/30 18:32:12｜xllm/core/framework/batch/batch.cpp:552｜状态未核实

- 类型：行级意见；根评论，0 条回复。
- 严重性：P3（评审者原文）。
- 线程锚点：[`discussion_r3487342`](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/636#discussion_r3487342)；未见 OutDated 标记。
- 状态证据：只见 “No Processing Required” 和 “Resolved” 操作按钮，未见明确解决事件；解决状态未核实。

### 页面代码上下文（锚点附近节选）

```diff
+std::vector<Batch> Batch::split_by_request() const {
+  // Block-copy operations may depend on sources belonging to another request.
+  // Such a batch must remain atomic rather than replay its copies per split.
+  if (requires_host_restore_ || !swap_block_transfer_infos_.empty()) {
```

### 评论原文

[P3] 原子分支拆分失败时，把「整批超限」错报成每个请求自身超限

`split_by_request()` 在两种情况下会放弃拆分、把整批原样返回：

```cpp
if (requires_host_restore_ || !swap_block_transfer_infos_.empty()) {
  return {*this};
}
```

于是 `parts[dp_rank].size() == 1`，拆分循环只跑一轮；一旦这一整批超过容量，下面的代码会对**批内每个序列**
`sequence->fail(INVALID_ARGUMENT, "Serialized inference input exceeds transport capacity: N bytes required, M bytes available. Reduce image resolution/count or video frames.")`。

问题在于 N 是**整批**的大小，M 是该 rank 的容量。这带来两个后果：

1. 批里没超限的小请求也被判失败，而且报错把整批的字节数归因到它们自己的输入；

2. 客户端按提示去「降分辨率/减帧」，即使降到很小，只要同批里还有那个大请求，就会**一直失败**，用户侧看起来像随机失败。

`requires_host_restore_` / `swap_block_transfer_infos_` 出现在 PD、KV offload 这些常态配置里，不是罕见路径。

建议：原子分支里改成逐个请求判断能否单独发（`prepare_inputs({single_request_batch})` 算大小，只 fail 真正超限的那个），
或者在无法归因时把错误文案换成「本批超出传输容量」并附上 rank/批内请求数，别让客户端以为是自己输入的问题。
另外这两个条件把整批变原子的原因（copy 依赖跨请求的 source）值得在注释里写一句，否则后面很容易被当作多余判断删掉。
