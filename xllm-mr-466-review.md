# xLLM MR #466 — Review 意见与修复交接

- 来源：[MR #466](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/466)
- 标题：perf: batch l3 cache store reads.
- 仓库：xLLM_AI/xllm（ID 958063）
- 分支：`feat/batch-store-opt` → `main`
- 页面快照：Open；Conversation 显示 11；2 commits；5 files changed；Reviewing，要求 2 人批准；状态检查显示 4 项通过。
- 页面显示的提交：`1187a3a`（perf: batch l3 cache store reads.）、`2dd7297`（perf: avoid cache read buffer copies and share get logic.）。审查总结注明完整 head `2dd7297756ce0f98753db9521cd95a72fd4a9679`，base `d01a46cce2ad975aa2343251e5c567ba59c73ddc`。
- 导出时间：2026-10-10（Asia/Shanghai）
- 提取范围：刷新已登录 Chrome 页面，展开“同类信息已隐藏”，记录当时已加载的作者说明、行级讨论、回复与审查总结。未应用 reviewer 或 resolution 筛选；保留所有展开后评论。
- 状态说明：部分行级讨论旁显示 “No Processing Required” 和 “Resolved”。页面提取到的是动作/状态控件文本，无法确认其是否代表已完成的线程状态，因此不据此过滤；按原样记录。
- 范围限制：评论中的代码片段、锚点和页面可见时间均保留；本导出未独立验证评论中的代码结论。页面没有提供可见的日期，评论时间按页面显示原样记录。页面对话数显示 11，但展开后存在多个嵌套回复及汇总，故不将该数字当作评论条数。

## 给修复 agent 的交接索引（摘要）

| 主题 | 讨论位置 | 关注点 |
| --- | --- | --- |
| backend_ / transport_ 双状态 | `kv_cache_store.h:179`；`kv_cache_store.cpp:448/455, 506/517` | tier 查询与 `batch_exist` 依赖裸指针，注入 transport 时可能静默跳过；回复还指出已初始化实例替换 transport 后可能悬垂。 |
| 析构清理顺序 | `kv_cache_store.cpp:113-116` | 先清空 `backend_` 再 reset transport 的顺序建议调整或删除多余清理。 |
| 单 key 与批量成功判定 | `mooncake_store_backend.cpp:167-180` | 特殊分支与通用循环判定重复。 |
| 批量失败隔离 | `mooncake_store_backend.cpp:142-145, 162-185` | 单个校验错误或异常会让整个批次 miss；增量回复对比了基线逐对象异常隔离。 |
| 未使用的单 key API | `mooncake_store_backend.h:84`；`.cpp:189-193` | 建议删除不再被生产代码调用的 `get()` 包装。 |
| 日志缺少 key 上下文 | `mooncake_store_backend.cpp:129, 144, 152, 169, 182, 184` | 批量改造后错误日志没有 key 或批次大小。 |
| transport 返回契约 | 接口 `mooncake_store_backend.h:60-69`；消费端 `kv_cache_store.cpp:483-487` | 短结果向量被静默当作 miss；建议明确长度/顺序契约并检查。 |
| 行为分支测试覆盖 | `mooncake_store_backend.cpp:143-154, 171-174` | 评论认为前置校验整批失败与单 key 特判没有直接测试覆盖。 |
| 审查结论 | MR 总结及增量总结 | 一份总结有条件推荐合并并要求目标环境测试；另一份增量总结列出 P2×5、建议优先处理批量失败隔离与日志。 |

## 页面评论原文（按展开后的顺序）

ext.luolei15
 commented 16:19
收益

在磁盘为主的 64K 回放上有小幅收益：每个磁盘对象的读取耗时从约 1.65 ms 降到约 1.42 ms，prefetch省 0.3–0.6 s（6–13%）。

文件	改动
kv_cache_store.cpp	batch_get_with_status 先收集所有有效对象的 key 和缓冲区，再调一次 transport_->batch_get
mooncake_store_backend.h/.cpp	新增接口 KVCacheStoreTransport（batch_put、batch_get）；MooncakeStoreBackend::batch_get 把多个 key 一次交给 batch_get_into_multi_buffers
kv_cache_store.h	成员从 MooncakeStoreBackend backend_ 换成 KVCacheStoreTransport transport_，单测可以注入假后端
kv_cache_store_test.cpp	覆盖批量状态映射、多 component、部分失败

行为差别：

	基线	批量版
每次 Mooncake Get 带的 key 数	1	一个预取 batch 的全部对象，默认 8
64K 请求每个 rank 的 Get 调用次数	4095	512

命中判定没变：返回字节数必须等于期望字节数才算命中，多个 component 仍是全部命中才算逻辑命中。

收益从哪来，为什么只有这么多

省掉的是控制面往返。基线每个对象要一次 master BatchQuery，磁盘对象另有“取落盘对象”和“释放 buffer”两次 RPC。批量后这些都按 batch 做一次。
磁盘读本身没变快。服务端 BucketStorageBackend::BatchLoad（third_party/Mooncake/mooncake-store/src/storage_backend.cpp:1842）对一个 RPC 里的多个 key 是单线程逐个同步读，每个 rank 仍然只有一路读盘。
上限受盘限制。磁盘部分当前约 8 GiB/s，裸读上限约 10.2 GiB/s，剩余空间本来就不大。
要明显缩短预取时间，得让单个 rank 的读盘并行，或者消除 8 个 rank 重复读同一份数据。

Collapse
ext.luolei15
  added 2 commits  
16:19
P
 
perf: avoid cache read buffer copies and share get logic.
 
 
2dd7297
BY
 
perf: batch l3 cache store reads.
   
1187a3a
 
dengyingxu1
 
reviewed    16:21
  View Changes
3
xllm/core/framework/kv_cache_transfer/kv_cache_store.h
View file
...
...
@@ -176,7 +176,8 @@
177
177

  std::map<BlockType, std::vector<StoreEntry>> store_index_;
178
178

  size_t max_entries_per_type_ = 0;
179
-
  std::unique_ptr<MooncakeStoreBackend> backend_;
179
+
  std::unique_ptr<KVCacheStoreTransport> transport_;
邓英旭  6 小时前

1. 健壮性 — 冗余的裸指针 backend_ 位置：xllm/core/framework/kv_cache_transfer/kv_cache_store.h:179

引入了 KVCacheStoreTransport 接口并将 transport_ 作为智能指针管理生命周期，但保留了 MooncakeStoreBackend* backend_ 裸指针。在 kv_cache_store.cpp 的读写逻辑中已全部使用 transport_，backend_ 似乎不再被使用。 如果 backend_ 确实不再使用，请删除它以简化状态管理；如果仍需调用 MooncakeStoreBackend 特有方法，建议将这些方法下沉到 KVCacheStoreTransport 接口中，避免破坏抽象层。

No Processing Required
Resolved
 
dengyingxu1
 
reviewed    16:21
  View Changes
58
xllm/core/framework/kv_cache_transfer/kv_cache_store.cpp
View file
...
...
@@ -110,7 +111,10 @@
110
111

  return config.rdma_devices;
111
112

}
112
113

邓英旭  6 小时前

2. 正确性 — 析构函数中的指针清理顺序 位置：xllm/core/framework/kv_cache_transfer/kv_cache_store.cpp:113-116

析构函数中先执行 backend_ = nullptr; 再执行 transport_.reset();。虽然不会导致崩溃，但逻辑上应先释放资源再置空指针，或者由于对象即将销毁，直接移除 backend_ = nullptr;。 调整顺序为先 transport_.reset(); 再 backend_ = nullptr;，或直接移除对 backend_ 的置空操作。

No Processing Required
Resolved
 
dengyingxu1
 
reviewed    16:21
  View Changes
81
xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp
View file
...
...
@@ -121,48 +121,75 @@
152
164

        client_ptr_->batch_get_into_multi_buffers(keys,
153
165

                                                  all_buffers,
154
166

                                                  all_sizes,
155
167

                                                  /*prefer_same_node=*/false);
邓英旭  6 小时前

3. 复用 — batch_get 中单次与批量逻辑不一致 位置：xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp:167-174

当 keys.size() == 1 时，使用 get_succeeded 判断成功；当 keys.size() > 1 时，使用内联表达式 results[index] >= 0 && results[index] == expected_bytes[index] 判断。这种不一致容易导致后续维护时行为分叉。 统一判断逻辑，提取一个内联函数或 lambda 复用，或者直接在所有情况下使用相同的内联逻辑（如果 get_succeeded 逻辑等价）。

No Processing Required
Resolved
 
dengyingxu1
 
reviewed    16:21
  View Changes
81
xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp
View file
...
...
@@ -121,48 +121,75 @@
139
+
  all_buffers.reserve(buffers.size());
140
+
  all_sizes.reserve(buffers.size());
141
+
  expected_bytes.reserve(buffers.size());
142
+
  for (const MooncakeMultiBuffer& buffer : buffers) {
邓英旭  6 小时前

4. 健壮性 — 批量请求中单个 buffer 校验失败导致整批失败 位置：xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp:142-145

在 batch_get 中，如果发现任何一个 buffer.addresses.size() != buffer.sizes.size()，直接返回全 0 的 statuses，导致整批请求失败。 确认这是否为预期行为。如果是快速失败机制，建议添加注释说明；如果希望跳过错误项继续处理，需修改为仅将该项结果置 0 并记录日志。

马晓龙  3 小时前

补充一个增量证据，回答「整批失败是否为批量机制的固有语义」：不是固有的，是 xllm 侧自身引入的。

1. head 的 catch 范围扩大到整批（mooncake_store_backend.cpp:162-185）：MooncakeStoreBackend::batch_get 的执行段用单个 try/catch 包住全部 key 的 client_ptr_->batch_get_into_multi_buffers 调用，任一 key 抛异常 → statuses 保持入口初始化的全 0 返回（:127），整 chunk 全 miss。基线 get()（base mooncake_store_backend.cpp:147-165）的 try/catch 只包一个对象，且它在 KVCacheStore::batch_get_with_status 的逐对象循环（base kv_cache_store.cpp:460-467）里被调用——对象 i 异常只失败 i，循环继续尝试 i+1。

2. Mooncake client 内部本就是逐 key 隔离的（third_party/Mooncake real_client.cpp 的 batch_get_into_multi_buffers_internal）：逐 key 循环里单 key 查询失败/副本无效/缓冲区过小只写该 key 的负错误码，不中断其他 key；计时包装 execute_timed_operation（client_metric.h:92-100）不吞异常。也就是说：负错误码路径 head 已逐 key 映射（通用循环 :175-180 与基线隔离度等价），只有异常路径被 catch-all 放大为整批失败。

3. 影响面：网络抖动等瞬态异常从「损失 1 块」放大为「损失整个预取 chunk（默认 2 块，作者部署 8 块）」。方向保守（miss → 重算），无数据损坏，但 prefetch 命中率对瞬态错误更脆弱。

建议（二选一）：

若整批快速失败是有意设计：在前置校验（:143-146/:151-154）和 catch 处加注释声明，并至少在 LOG 中带上 keys.size()（另开了新评论：错误日志同时丢了 key 上下文）；
若要恢复基线隔离度：catch 保留全 0 但注释承认行为差异；前置校验改为「标记该对象 status=0 并 continue」，与 Mooncake 内部负错误码语义对齐。

验证方式：给 MemoryStoreTransport 加 throw_on_get_ 开关注入异常，断言其余对象仍命中（现有 RestoresBatchBlocksWithIndependentMisses 只覆盖负错误码路径，未覆盖异常路径）。

（证据基线：head 2dd729775，xllm 侧结构差异由双版本源码逐行对照，Mooncake 逐 key 隔离由 third_party 源码证实。）

邓英旭  3 小时前

针对你的疑问补充说明如下： [robustness] 批量请求中单个 buffer 校验失败导致整批失败 位置：xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp:142-145 问题：在 batch_get 中，如果发现任何一个 buffer.addresses.size() != buffer.sizes.size()，直接返回全 0 的 statuses，导致整批请求失败。 确认这是否为预期行为。如果是快速失败机制，建议添加注释说明；如果希望跳过错误项继续处理，需修改为仅将该项结果置 0 并记录日志。

如果此处确属必要写法或有特殊上下文，欢迎进一步说明，我们会重新评估该条建议。

No Processing Required
Resolved
 
fengyan.119
 
reviewed    16:31
  View Changes
3
xllm/core/framework/kv_cache_transfer/kv_cache_store.h
View file
...
...
@@ -176,7 +176,8 @@
178
178

  size_t max_entries_per_type_ = 0;
179
-
  std::unique_ptr<MooncakeStoreBackend> backend_;
179
+
  std::unique_ptr<KVCacheStoreTransport> transport_;
180
+
  MooncakeStoreBackend* backend_ = nullptr;
冯艳  6 小时前
【🟠 important】【🏛️ architecture】
backend_ 与 transport_ 是同一对象的双重状态，失配后 tier 查询静默降级

🔎 问题

本次改动把所有权从 backend_ 搬到 transport_，却在 init() 额外留下 backend_ = backend.get()（kv_cache_store.cpp:100），使 backend_ 成为 transport_ 的非拥有裸指针别名。tier 查询（kv_cache_store.cpp:448/455）与 batch_exist（506/517）用 backend_ 判空，batch_get_with_status 主路径（480）用 transport_ 判空。当注入非 MooncakeStoreBackend 的 transport（本 MR 新增的 MemoryStoreTransport 测试已如此，attach_transport 只设 transport_），backend_ 恒为 nullptr：batch_get 正常读写，而 tier 查询被跳过、batch_exist 返回全 0，命中率与缓存统计静默变差且无日志。

🔧 修改建议

删除 MooncakeStoreBackend* backend_ 成员，让 KVCacheStore 只持有 transport_ 唯一所有权；把 batch_query_tiers 提升为 KVCacheStoreTransport 纯虚方法（virtual std::vector batch_query_tiers(const std::vectorstd::string&) = 0;），并将 kv_cache_store.cpp:448/455/506/517 四处 backend_ 改为 transport_。析构函数随之可去掉手工 backend_=nullptr / transport_.reset()。

马晓龙  3 小时前

为该 important 结论补一条新实证证据（test-only，但一步之遥就是真 UAF）：

测试 peer 的注入点在已 init 的 store 上构成悬垂指针（tests/core/framework/kv_cache_transfer/kv_cache_store_test.cpp:40-45）：

static void attach_transport(
    KVCacheStore* store,
    std::unique_ptr<KVCacheStoreTransport> transport) {
  store->transport_ = std::move(transport);
  store->is_initialized_ = true;
}


std::move 赋值会析构旧 transport 对象，而 backend_（kv_cache_store.h:179-180，生产 init() 在 kv_cache_store.cpp:100 设为 backend.get()）是非拥有别名、不会被清掉。对本 PR 之前已 init() 成功的 store 调用 attach_transport（换假后端重跑场景），随后 batch_exist（kv_cache_store.cpp:506/517）或带 stats 的 batch_get_with_status 走 tier 查询（:448/:455）即 use-after-free。现有测试只对未 init 的 store 注入（test.cpp:1252-1258），未踩中。

这印证了「双重状态是上膛的枪」：连本 PR 自己写的测试工具都差一步踩中，生产代码演进一步（如在 reload/重配路径复用注入思路）就会变成真缺陷。同意原建议（删 backend_、batch_query_tiers 提升为接口纯虚方法、四处判空改 transport_）；若短期保留双状态，attach_transport 至少应同时 store->backend_ = nullptr;。

（证据基线：head 2dd729775，UAF 路径由 move 赋值语义 + 别名不变的源码结构证明，未实际运行复现。）

No Processing Required
Resolved
 
fengyan.119
 
reviewed    16:31
  View Changes
20
xllm/core/framework/kv_cache_transfer/mooncake_store_backend.h
View file
...
...
@@ -57,16 +57,30 @@
81
+
  std::vector<uint8_t> batch_get(
82
+
      const std::vector<std::string>& keys,
83
+
      const std::vector<MooncakeMultiBuffer>& buffers) override;
70
84

  bool get(const std::string& key, const MooncakeMultiBuffer& buffer);
冯艳  6 小时前
【🔵 minor】【🧹 dead-code】
MooncakeStoreBackend::get 已无任何调用者，应删除

🔎 问题

本次改动把 KVCacheStore::batch_get_with_status 中的逐 key backend_->get(...) 替换为 transport_->batch_get(...) 后，MooncakeStoreBackend::get 不再有生产调用者，只剩 batch_get({key},{buffer}) 的薄包装定义（mooncake_store_backend.cpp:189-193），并在头文件继续公开声明（mooncake_store_backend.h:84）。保留该 public 方法使单键读取有两个入口，且绕过 KVCacheStore 侧 key 去重/统计，易被误用。

🔧 修改建议

删除 mooncake_store_backend.h:84 的 bool get(const std::string& key, const MooncakeMultiBuffer& buffer); 声明和 mooncake_store_backend.cpp:189-193 的定义；单 key 语义已由 batch_get 的 keys.size()==1 路径完整覆盖。

No Processing Required
Resolved
 
fengyan.119
 
reviewed    16:31
  View Changes
81
xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp
View file
...
...
@@ -121,48 +121,75 @@
169
+
      LOG(ERROR) << "Mooncake Get returned extra result items.";
158
170

    }
159
-
    return get_succeeded(expected_bytes, results);
171
+
    if (keys.size() == 1) {
冯艳  6 小时前
【🔵 minor】【🔧 refactor】
keys.size()==1 特判重复通用循环的成功判定，应删除

🔎 问题

batch_get 在 keys.size()==1 时经 get_succeeded(expected_bytes.front(), results) 判定（cpp:171-174），而通用循环（175-180）对 index 0 已执行完全相同谓词 results[index]>=0 && results[index]==expected_bytes[index]。前置校验（147-156）已保证每个 expected_bytes 落在 [0, INT32_MAX]，results 为空时 result_count=0 使状态保持 0，与 get_succeeded 分支等价。同一成功不变量被两处维护。

🔧 修改建议

删除 mooncake_store_backend.cpp:171-174 的 if (keys.size() == 1) { ... return statuses; } 特判，单 key 请求直接落入通用循环；get_succeeded 仍作为 private 静态谓词供 get() 包装与单测复用。

No Processing Required
Resolved
dengyingxu1
 commented 17:39

MR #466 评审结论：基于 base d01a46cce2ad975aa2343251e5c567ba59c73ddc、head 2dd7297756ce0f98753db9521cd95a72fd4a9679，检查 2 个提交及 19 个改动文件，重点覆盖 Mooncake KV cache store 的批量读取、transport 生命周期、结果聚合与相关单测，以及同步删除的 Kimi K3/NPU 自定义算子代码。静态检查未发现可确认的 P0-P3 缺陷，因此未发布 inline finding。已执行 git diff --check（通过）和 python3 -m compileall -q xllm/python/kernels_npu/_custom_op.py（通过）；未执行 C++ 构建/单测，未执行 Mooncake、NPU、ATB、ACL Graph、HCCL、CUDA 或推理运行。合并建议：有条件推荐合并，条件是完成目标环境 C++/Mooncake 批量 KV 读写测试及相关运行时冒烟验证。

maxiaolong.maxwell
 commented 18:57
【🔵 P2】【operability】commit2 重构丢失错误日志的 key 上下文——单 key 路径并非与基线「逐字节等价」的可观察行为

位置：xllm/core/framework/kv_cache_transfer/mooncake_store_backend.cpp:129,144,152,169,182,184

head batch_get 的所有错误出口全部不带 key、不带 keys.size()：

LOG(ERROR) << "Invalid Mooncake multi-buffer batch Get request.";   // :129
LOG(ERROR) << "Mismatched Mooncake Get buffer addresses and sizes."; // :144
LOG(ERROR) << "Mooncake Get exceeds int32 result range.";            // :152
LOG(ERROR) << "Mooncake batch Get failed: " << error.what();         // :182


基线（base mooncake_store_backend.cpp:128,137,143,161）每条都带 key：

LOG(ERROR) << "Mooncake Get failed for key=" << key << ": " << error.what();


commit2 把 get() 收敛为 batch_get({key},{buffer}) 薄包装时，错误文案随旧实现一起被替换。MR 描述称 get 逻辑共享后单 key 路径与基线等价——对返回值成立，对错误输出不成立。叠加整批 catch 语义后，线上排障表现为「一个 chunk 全 miss + 一行不知道谁引起的 ERROR」。

建议（最小修法）：错误文案带上 keys.size()，前置校验两个分支带上触发 index 与 key，例如 LOG(ERROR) << "Mismatched Mooncake Get buffer addresses and sizes at index=" << index << " key=" << keys[index];。不改结果正确性，纯排障成本问题。

（证据基线：head 2dd729775，双版本源码逐行对照；无运行时日志样本。）

Collapse
maxiaolong.maxwell
 commented 18:58
【🔵 P2】【architecture】新接口 KVCacheStoreTransport 未定义返回值契约，消费端静默容忍短结果向量

位置：mooncake_store_backend.h:60-69（接口）；kv_cache_store.cpp:483-487（消费端）

接口只定义了签名，没有契约注释；消费端散射循环用 index < results.size() 做防御：

const std::vector<uint8_t> results =
    transport_->batch_get(get_keys, get_buffers);
for (size_t index = 0;
     index < results.size() && index < get_request_indices.size();
     ++index) {
  const size_t request_index = get_request_indices[index];
  physical_results[request_index] = results[index] != 0;


未来任一 transport 实现（本 PR 的核心目的就是引入注入点）返回空/短 vector 时，短出的对象全部静默 miss 且无日志；返回顺序与 keys 不对齐则产生错误的命中图。两个现有实现都返回 keys.size() 长度（mooncake_store_backend.cpp:127、kv_cache_store_test.cpp:194），但接口本身不强制。

建议（最小修法）：

接口注释写明契约：「返回值长度必须等于 keys.size()，第 i 项对应 keys[i]，非 0 = 成功」；
散射循环前加 CHECK_EQ(results.size(), get_keys.size())（两个现有实现都满足，不会破坏测试）；
batch_put 的对应循环（:397-399 同样模式）一并处理。

（证据基线：head 2dd729775；两个现有实现守约已证明，「未来实现违约」是工程推断而非已发生缺陷，故定 P2。）

Collapse
maxiaolong.maxwell
 commented 18:58
【🔵 P2】【test coverage】本 PR 引入的两个行为差异分支（整批前置校验失败、单 key 特判）在测试集零覆盖

位置：mooncake_store_backend.cpp:143-146,151-154,171-174（被测缺失分支）；tests/core/framework/kv_cache_transfer/kv_cache_store_test.cpp:130-150（MooncakeStoreBackendTestPeer 现有暴露面）

RestoresRoleBytesAndRequiresEveryRegisteredShard（test.cpp:1239-1300）用 MemoryStoreTransport 验证了状态散射/多 component/部分失败，但假后端没有前置校验逻辑；
RestoresBatchBlocksWithIndependentMisses（:590-622）走真 Mooncake 的部分失败（负错误码）路径。

两条都不触达 MooncakeStoreBackend::batch_get 自己的前置校验整批失败分支（即 note:3457280 讨论的对象）和 keys.size()==1 特判（note:3457279/3457307 讨论的对象）。事实上该函数在单测中唯一可达路径是 client_ptr_ == nullptr 早退——MooncakeStoreBackendTestPeer 只暴露 static 工具函数（unique_ranges/get_succeeded/put_succeeded/replica_tier），无法注入不合法输入。MR 描述称「覆盖批量状态映射、多 component、部分失败」对假后端层成立，对 Mooncake 后端层不成立。

建议（最小修法）：把前置校验提取为纯函数 validate_batch_request(keys, buffers)（返回 expected_bytes 或 nullopt），对纯函数加表驱动单测（段不匹配、int32 溢出、空 batch、单 key）；这同时是失败隔离修法的前置。

（证据基线：head 2dd729775；「CI 通过不代表分支被覆盖」由测试集源码遍历证明——无其他 TEST 触达 batch_get。）

maxiaolong.maxwell
 commented 18:58
MR #466 独立评审结论（增量 findings）

基于 head 2dd729775、merge-base 6be006116（2 commits / 5 files / +298 -49），对批量 L3 store 读取、KVCacheStoreTransport 抽象、结果聚合与单测做了独立审查，并与本 MR 已有 8 条讨论严格去重（backend_/transport_ 双状态、析构顺序、单 key 特判、get() 死代码等已有讨论不重复发布）。结论：With fixes（可合入）——P0=0，P1=0，P2×5：

[回复 note:3457280] 批量 Get 失败隔离粒度从单对象变整批：整批失败并非批量机制固有——Mooncake client 内部逐 key 出结果（负错误码路径 head 已等价映射），只有 xllm 侧包住全部 key 的 catch-all 把异常放大为整 chunk 全 miss；前置校验两分支从生产调用方看不可达。
[本条新评论] commit2 错误日志丢失 key 上下文：base 每条 ERROR 带 key，head 全部通用文案——单 key 路径对返回值等价、对可观察日志不等价。
[本条新评论] KVCacheStoreTransport 未定义返回值契约：散射循环 index < results.size() 静默容忍短结果，未来注入实现违约即静默降级；建议 CHECK_EQ + 接口注释。
[回复 note:3457305] attach_transport 测试注入点构成悬垂实证：move 赋值析构旧 transport 而 backend_ 别名不清——已 init 的 store 上注入即 UAF，是双状态问题的最直接实证。
[本条新评论] 两个行为差异分支测试零覆盖：整批前置校验失败与单 key 特判在 head 测试集不可达（MooncakeStoreBackendTestPeer 无法注入不合法输入）。

建议合入前处理 1+2（同一函数内小改动），3-5 可作 follow-up。另核对通过项：状态散射索引对齐、空 batch 双层防护、commit2 返回值逐分支等价（除日志外）、CMake 测试目标接线、并发模型与基线无差异。另有 5 项证据不足的疑点（生产异常频率、性能数字、CI 实跑记录等）未发布，可按需提供。

 收起同类信息
Reviewing

Review rules: 2 are required to approve

Status check

Pipelines will affect the merge status。
