# xLLM MR #662 — Review 意见

- 来源：[MR #662](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662)
- 标题：`test: optimize mlu unit test and correct bugs.`
- 分支：`bugfix/fix-mlu-unit-test` → `main`；6 commits、46 files changed。
- 页面状态：Open / Approved，要求 2 人批准；状态检查 2 项通过、1 项失败，CI #9271455 failed，合并被阻止。
- 当前 head：未核实；会话可见提交首项为 `59a0bb0`，不据此推定 head。
- 提取时间：2026-11-03T14:15:08+08:00；通过已登录 Chrome 刷新后读取，并打开评审事件的文件差异。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评审意见。
- 清单：9 条评审根评论、0 条已发布回复；邓英旭 5 条，冯艳 4 条。另有 1 条作者说明，单列于后。批准及提交事件不计作评论。
- 线程状态：各线程仅见 No Processing Required / Resolved 操作控件，未见明确解决事件，均标为“状态未核实”。输入回复区域不计作已发布回复。
- 范围限制：已读取评审事件展开的 9 条行级线程，与页面 Conversation 9 对应；未逐一加载全部 46 个文件的完整 diff，不能独立排除其他或过时讨论。不含草稿；未见已读取线程的 OutDated 标记。
- 原文说明：下述技术判断为评审者意见，未经独立验证。评论中的位置与实际行锚点可能不同，分别保留；建议中不完整的 `TIMEOUT )` 为原文。

## 行动索引（导出者摘要）

| 评论 | 主题 | 状态 |
| --- | --- | --- |
| R01 | 移除 MLU 宏后的返回路径 | 状态未核实 |
| R02 | fork 失败后的子进程回收 | 状态未核实 |
| R03 | 测试结果目录保留策略 | 状态未核实 |
| R04 | query=1 边界覆盖 | 状态未核实 |
| R05 | subreaper 环境兼容性 | 状态未核实 |
| R06 | 套件超时 600 秒 | 状态未核实 |
| R07 | 重复设备同步 | 状态未核实 |
| R08 | rank/port 等重复赋值 | 状态未核实 |
| R09 | PLAN_MISMATCH 重复判断 | 状态未核实 |

## R01 — 邓英旭  (dengyingxu1)｜2026/11/3 11:41:30｜xllm/core/kernels/ops_api.cpp:1313｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491376](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491376)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

1. 正确性 — 移除 MLU 宏导致逻辑缺失
位置：xllm/core/kernels/ops_api.cpp:1313
原代码为 #if defined(USE_MLU) ... #elif defined(USE_NPU)。修改后变为 #if defined(USE_NPU)。如果编译时定义了 USE_MLU 且未定义 USE_NPU，fused_gdn_gating 函数将没有返回值，导致未定义行为（UB）。 恢复 #elif defined(USE_NPU) 结构，或者如果 MLU 确实移除了该算子，请确保有适当的 #else 抛出编译错误或返回默认值，避免 UB。


## R02 — 邓英旭  (dengyingxu1)｜2026/11/3 11:41:30｜tests/core/forked_rank_session.h:0｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491377](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491377)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

2. 正确性 — fork 失败时存在孤儿进程泄漏
位置：tests/core/forked_rank_session.h:233-244
在 start() 函数中，如果在 fork() 循环中途失败（pid < 0），代码会 break 跳出循环。但此时已经成功 fork 的子进程仍在运行（在 child_loop 中阻塞等待命令）。父进程随后清理了 socket，但没有向这些已启动的子进程发送退出信号或 kill 它们，导致孤儿进程泄漏。 在 fork 失败的 break 之后，应遍历已加入 workers_ 的进程，向它们发送终止信号并回收（类似 reap(true) 的逻辑），然后再清理 socket。


## R03 — 邓英旭  (dengyingxu1)｜2026/11/3 11:41:30｜scripts/build_support/grouped_gtest.py:60｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491378](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491378)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

3. 测试缺口 — 临时目录未清理
位置：scripts/build_support/grouped_gtest.py:60
attempt = Path(tempfile.mkdtemp(prefix=f"{suite}-", dir=output_root)) 创建了临时目录用于存放 XML 和 JSON 结果，但在测试运行结束后（无论成功或失败）都没有调用 shutil.rmtree 或类似方法清理这些临时目录。长期运行大量测试会导致磁盘空间浪费。 如果不需要保留历史运行记录，建议在 _run_suite 结束时（例如 finally 块中）清理 attempt 目录。如果需要保留用于调试，建议在测试成功时清理，失败时保留并记录路径。


## R04 — 邓英旭  (dengyingxu1)｜2026/11/3 11:41:30｜tests/core/kernels/mlu/dcp/dcp_indexer_test.cpp:347｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491379](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491379)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

4. 测试缺口 — 移除边界测试降低覆盖率
位置：tests/core/kernels/mlu/dcp/dcp_indexer_test.cpp:347
MtpPackedExpandedAndDraft 测试中，将 for (const int32_t query : {1, 4}) 修改为 for (const int32_t query : {4})，移除了 query=1 的边界测试。虽然减少了测试时间，但丢失了对单 query 打包解码路径的覆盖。 确认 query=1 的路径是否在其他测试用例中被覆盖。如果没有，建议保留 query=1 的测试，或者添加注释说明为何移除。


## R05 — 邓英旭  (dengyingxu1)｜2026/11/3 11:41:30｜tests/core/forked_rank_session_test.cpp:2000｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491380](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491380)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

5. 测试缺口 — 环境兼容性风险
位置：tests/core/forked_rank_session_test.cpp:147
ReapsRanksAfterParentExit 测试使用了 prctl(PR_SET_CHILD_SUBREAPER, 1)。在某些受限的容器环境（如未开启特定权限的 Docker）中，设置 subreaper 可能会失败，导致测试不稳定。 检查 prctl 的返回值，如果设置失败则 GTEST_SKIP() 跳过该测试，而不是直接 ASSERT_EQ 导致测试失败。


## R06 — 冯艳  (fengyan.119)｜2026/11/3 12:26:54｜cmake/cc_test.cmake:139｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491505](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491505)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

【🟠 important】【✅ correctness】
GTEST_SUITE 分支把超时硬编码成 600 秒，重型 MLU 套件比原逐用例注册更容易超时

🔎 问题

新增的 GTEST_SUITE 分支用 set_tests_properties(... TIMEOUT 600) 把整组用例超时固定为 600 秒，而 else 分支的 gtest_add_tests 走 CTest 默认 1500 秒。本 MR 同时把 mlu_graph_executor_test、deepseek_v32 注意力、glm5_attention_dcp_test 等大量原本分散在多条 CTest 用例里的 MLU 用例合并成单条套件用例，单条耗时本就是原来若干条之和，再叠加 600 秒上限，比改动前每条用例各自 1500 秒预算严格收紧。冷启动、首次加载 MLU kernel 或慢卡时套件会被 CTest 判超时，表现为整组用例失败而非逐条失败。


🔧 修改建议
把 TIMEOUT 600 换成与 else 分支一致的行为，即不设置 TIMEOUT 让 CTest 用默认 1500 秒，或显式写 TIMEOUT 1500；若确有需要更短上限的套件，在对应 tests/**/CMakeLists.txt 里对该目标单独 set_tests_properties(${suite} PROPERTIES TIMEOUT )，不要把这个上限写进通用宏。


## R07 — 冯艳  (fengyan.119)｜2026/11/3 12:26:54｜tests/core/layers/mlu/fused_moe_all2all_test.cpp:487｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491506](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491506)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

【🔵 minor】【⚡ perf】
7 处新增 torch_mlu::synchronize() 被 DeepEpTestScope 析构完全覆盖，全部冗余

🔎 问题

本次 diff 在 fused_moe_all2all_test.cpp 新增 7 处 torch_mlu::synchronize()（行 383 在 run_all2all_smoothquant_test_child 内；行 487/507/538/570/600/627 在各 case lambda 尾部）。但每个被调用的 child 函数内部都已声明局部 ::xllm::test::DeepEpTestScope（run_all2all_basic_test_child 行 295、run_all2all_selected_route_test_child 行 418），其析构函数（tests/core/layers/mlu/deep_ep_test_scope.h:32-35）在函数返回前已执行 torch_mlu::synchronize() + DeepEPManager::destroy()。child 返回后 case lambda 再同步一次是重复的全设备同步；run_all2all_smoothquant_test_child 包装层同理。child 提前返回的路径（设备不足 return EXIT_CODE_SKIP 等）发生在 DeepEpTestScope 声明之前，没有排队的 GPU 工作，不需要同步。


🔧 修改建议
删除这 7 处 torch_mlu::synchronize()：行 383 处恢复为直接 return run_all2all_basic_test_child(params);，行 487/507/538/570/600/627 处恢复为直接 return run_*_test_child(params);，使 DeepEpTestScope 析构保持唯一同步点。


## R08 — 冯艳  (fengyan.119)｜2026/11/3 12:26:54｜tests/core/layers/mlu/fused_moe_all2all_test.cpp:2｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491507](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491507)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

【🔵 minor】【🧹 dead-code】
all2all_cases() 里 IIFE 内设置的 rank/world_size/port/host 紧接着被外层覆盖，属于死赋值

🔎 问题

四个 case 都写成先由 IIFE 设一组硬编码值、再在 IIFE 之外用参数逐字段覆盖：IIFE 里写 params.rank = 0; params.world_size = 2; params.port = 29503; params.host = "127.0.0.1";，IIFE 之后立刻 params.rank = rank; params.world_size = 2; params.port = port; params.host = "127.0.0.1";。IIFE 里这四个字段的写入一次都读不到，其中 port = 29503/29504/29505/29506 还带着 "避免端口冲突" 的注释，会让人误以为端口由这里决定，而实际端口来自调用方传入的 port。同一模式在另外三个 case 重复出现。


🔧 修改建议
删掉 IIFE 中对 rank、world_size、port、host 的赋值，只保留真正需要按 case 区分的字段（如 dtype、tokens、hidden、topk 等），端口与 rank 交由外层统一赋值；若 IIFE 里已无剩余差异字段，直接把构造收敛成一个 make_case(...) 辅助函数，四个 case 各调用一次。


## R09 — 冯艳  (fengyan.119)｜2026/11/3 12:26:54｜tests/core/runtime/mlu_graph_dp_collective_test.cpp:433｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491508](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/662#discussion_r3491508)
- 状态证据：仅见处理及解决操作控件，实际解决状态未核实。

### 评论原文

【🔵 minor】【🎨 style】
PLAN_MISMATCH 场景里重复判断 scenario == Scenario::PLAN_MISMATCH

🔎 问题

新增代码写成 if (scenario == Scenario::PLAN_MISMATCH) { if (scenario == Scenario::PLAN_MISMATCH && rank == 1) { ... } }。外层条件已经保证 scenario == Scenario::PLAN_MISMATCH，内层再判一次是恒真的冗余条件，读者需要多推一层才能确认这段只在 rank 1 执行。


🔧 修改建议
内层只保留 rank 判断：
```cpp
if (scenario == Scenario::PLAN_MISMATCH && rank == 1) {
  const auto ints = torch::TensorOptions().dtype(torch::kInt32).device(device);
  input.params.multi_block_tables.emplace_back(torch::zeros({3, 2}, ints));
}
```
若外层还有其它分支需要保留，则把内层简化为 if (rank == 1)。


## 作者说明

- 作者：ext.luolei15；页面显示时间：11:30（未取得完整时间戳）。
- 类型：MR 说明，无行级锚点；不适用行级线程解决状态。

修复 MLU 单元测试并缩短回归测试耗时

当前部分 MLU 测试的输入配置、并行拓扑和输出断言与实际接口约定不一致，多卡测试还存在重复初始化和设备资源竞争。本 PR 修正相关测试，并通过复用进程与合并测试场景降低回归耗时。

主要变更：

- 修正 Attention、DCP、采样、缓存管理及模型测试配置，按 TP/CP 输出布局和 KV 分片规则校验结果。
- 为多卡测试引入可复用的 rank 会话，减少通信组和模型重复初始化；隔离设备断言场景，完善异常、超时及资源清理处理。
- 支持按 GTest 测试套件注册 CTest，保留用例级 XML、每次执行结果和单用例重跑能力；串行测试组显式使用单 worker，并记录整体耗时。
- 将重复的形状遍历收敛为代表性边界场景，保留关键分派边界、精度、内存访问和独立参考结果校验。
- 移除仅供测试使用的 MLU 独立 GDN gating、通用 recurrent 包装实现及其专属测试。

验证结果（依据分支提交记录，本次未重新执行）：

- MLU 编译闸口通过，相关失败用例及相邻回归测试通过。
- DCP 专项 CTest 条目由 80 个合并为 52 个，两轮串行运行均通过，墙钟耗时分别为 152.15 秒和 149.71 秒。
- 最终以并行度 16 执行完整构建和测试通过；原有 42 个串行用例全部通过，串行测试组总耗时为 82.02 秒
