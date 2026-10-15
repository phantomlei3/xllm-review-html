# xLLM MR #495 — Review 意见与修复交接

- 来源：[MR #495](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/495)
- 标题：`bugfix: batch-restore preempted decode requests and reserve admission headroom.`
- 分支：`bugfix/opt-dcp-preemption` → `main`
- Head：页面提交列表首项 `69e5397`；两位 reviewer 均引用 `69e5397a25f26ad12a521d0bffd8dc532b19c61b`。Base 完整 SHA 见评审总结，未另查代码仓库。
- 页面状态：Open / Reviewing；4 commits、23 files changed；要求 2 人批准，4 项状态检查通过，尚未满足合并要求。
- 提取时间：2026/10/15 16:11:11（Asia/Shanghai）；已登录 Chrome 刷新后读取，逐组展开同类隐藏信息及 Expand。
- 筛选：不按作者、时间或解决状态过滤；包括作者说明、行级意见、独立总体意见和评审总结；不计提交或 reviewed 事件本身。
- 清单：10 条根评论（1 条作者说明、1 条行级意见、6 条独立总体意见、2 条评审总结），0 条回复。马晓龙的“回应”以独立总体评论发表，并非行级线程回复。页面 Conversation 1 不作为评论总数。
- 状态：行级线程只见处理动作按钮，未见明确解决事件，状态未核实；其余总体评论未见解决状态标记。总结内“未解决”属于 reviewer 表述。
- 范围：已展开全部同类评论和正文；未见分页、加载更多或过期线程入口。仅导出已发布内容，不包含草稿。行级 diff 锚点展示旧行 580 / 新行 790，reviewer 引用 head 的 581–583；两种位置均保留，未自行判定对应关系。
- 原文说明：以下技术判断与验证声明均来自作者或 reviewer，未由导出者独立验证。保留原文中的 `:unknow_emoji:`、`used0`、`load.num_requests0` 等异常文本。

## 行动索引（导出者摘要）

| 评论 | 类型 / 主题 |
| --- | --- |
| R01 | 作者说明：背景、实现与实验数据 |
| R02 | 邓英旭总结：P2 prefill 增长预留口径 |
| R03 | P2 行级意见：仅对 decode 计算增长预留 |
| R04 | 补充验证：保留在途恢复预留，仅排除非恢复 prefill；与 R03 建议边界不同 |
| R05 | P3：MR 描述与 head 代码漂移 |
| R06 | P3：cached_tokens 截断缺观测 |
| R07 | P3：准入持锁性能；reviewer 建议可记录技术债 |
| R08 | Question：恢复等待时长观测 |
| R09 | Question：prefill 预留及跨线程测试缺口 |
| R10 | 马晓龙总体审查结论与合入建议 |

## R01 — 罗磊（ext.luolei15）｜2026年10月15日 10:00:01｜MR 总体评论｜未见解决状态标记

- 类型：作者说明；根评论，无回复。

### 评论原文

#### bugfix: 成批恢复被抢占的 PD decode 请求，并为准入预留增长空间

##### 背景

PD 分离部署中，decode 节点的 KV cache 不够时会抢占正在 decode 的请求。开启 host cache（`host_blocks_factor > 1`、开启 prefix cache、关闭 schedule overlap）的情况下，被抢占请求的 KV 会卸载到 host，之后再从 host 读回恢复，不需要重新计算。

main 上这条恢复路径有两个问题：

1. **恢复会被新请求饿死**：新请求的准入由 brpc 线程调用 `try_allocate` 完成，和调度线程互相独立。显存一旦释放，新请求往往先拿到，等待恢复的请求一直分不到 block，60 秒后报 `Decode request could not reacquire device KV cache within 60 seconds` 失败。
2. **准入时不考虑已准入请求后续还要增长**：decode 准入只看利用率阈值，然后按 prompt 分配 block，不为已准入请求后续生成的 token 预留空间。负载上来后抢占会很频繁，进而加重第 1 个问题。

##### 方案

###### 参考了什么

动手前对比了 vLLM 和 SGLang main 分支（2026-10-14）上 decode 侧的抢占和恢复实现：

|  | vLLM | SGLang | 本 PR |
| --- | --- | --- | --- |
| 被抢占请求的 KV | 释放后重算（GPU prefix cache 能命中的部分不用重算） | 默认拷到 host，恢复时拷回 | 卸载到 host cache，恢复时读回（沿用 xLLM 已有能力） |
| 恢复期间的新请求 | 发生抢占的那一步不放新请求 | 恢复队列非空时不放新请求（`process_decode_queue` 提前返回） | 恢复队列非空时暂停 decode 准入，prefill 收到 404 后重试 |
| 正在跑的请求 | 照常跑 | 照常跑 | 照常跑，不排空节点 |
| 恢复方式 | 和普通请求一样排队 | `resume_retracted_reqs` 按 FIFO 恢复，余量够几个就恢复几个，和其他请求一起组 batch | 每个 DP rank 内按 FIFO，一步内余量够几个就恢复几个，和 decode 混合组 batch |
| 预防抢占 | `watermark` 默认关闭 | `num_reserved_decode_tokens=512`：为每个活跃请求预留 512 token，等待恢复的请求也计入 | `decode_reserved_tokens_per_request=512`，用于准入和恢复 |
| 超时 | 无 | 无 | 无 |

本 PR 的主体沿用 SGLang 的思路：

- 只关新请求的入口，不排空节点；
- 恢复按 FIFO、按余量成批进行；
- 准入时为已准入请求预留增长空间。

"空闲出来的 block 先给被卡住的 decode"这一条取自 vLLM：发生抢占的那一步不做恢复。

###### 不需要超时的理由

同时满足以下三点，就能保证恢复一定会推进，不会卡死：

- 恢复队列非空时，新请求进不来；
- 正在跑的请求不断结束、释放 block，可用余量只会变多；
- 已准入但 KV 还在传输中的请求，block 已经计入"已占用"，恢复时不需要等它们完成。

所以队头请求只有两种结局：等到足够的余量后恢复，或者在本 rank 已完全空闲时仍放不下而直接失败。原来的 60 秒超时因此去掉了。

##### 主要改动

###### P0：成批恢复（`scheduler_policy.{h,cpp}`）

- **重写 `schedule_decode_restore`**：
  - 每一步按 rank 遍历恢复队列，保持 FIFO；某个 rank 的队头放不下时，只阻塞这个 rank，不影响其他 rank。
  - 恢复的前提是：可用余量 ≥ 本 rank 已承诺的占用 + 该请求的完整 footprint + 该请求自己的预留。只有满足这一条才分配 block，避免恢复到一半又被挤掉。
  - 恢复出来的请求进入本步 batch，和正在跑的 decode 混合执行。
- **被卡住的 decode 优先**：记录上一次 decode 调度里没拿到 block 的请求。只要它还在 decode 队列里等，或者本步发生过抢占，本步就不做恢复。
- **在途恢复被抢占时回到队首**：恢复到一半（仍在 chunked prefill）的请求如果再次被抢占，不再落进普通 prefill 队列，而是回到恢复队列的队首，保持顺序。续跑时 block 不够也同样回到队首等待，不报错。
- **准入开关**：有请求进入恢复队列时暂停准入；恢复队列为空、在途恢复都进入 decode 阶段、没有未完成的异步释放时，重新开放。暂停和重新开放都会打 INFO 日志，带暂停时长。
- **失败条件**：本 rank 没有任何请求占用 block、也没有调度中的请求，恢复仍然放不下时，返回 `RESOURCE_EXHAUSTED`。
- **`PrefillFirstPolicy`**：先让 `chunk_queue` 里的在途恢复续跑，再开始新的恢复。
- **删除 60 秒恢复超时**。

###### P1：准入预留

- **新参数 `decode_reserved_tokens_per_request`**（int32，默认 512，只对 PD DECODE 生效，设为 0 关闭）：每个已准入的 decode 请求预留 `min(R, 剩余可生成 token)` 对应的 block。
- **预留量的计算与发布**：调度线程每一步按 rank 汇总预留量，写进原子变量。汇总范围是运行中和排队中的请求，加上已准入但 KV 还在传输的请求（`received_request_map_`）。
- **准入检查**：`DisaggPDScheduler::try_allocate` 在 brpc 线程上读取这个预留量。预留不足时回滚已分配的 block，并通过 `retry_later` 返回 404，让 prefill 重试，不会被当成 413 处理。准入过程串行化，避免并发的 RPC 重复占用同一份余量。
- **余量的计算口径**：`decode_headroom_blocks` 统一按"可用 block 数 − 已占用 block 数"计算，扣除保留的 0 号 block；准入和恢复共用这一个函数。

###### 顺带修复：恢复后的 `cached_tokens` 超过 prompt 长度

- **问题**：被抢占的请求恢复时，命中的 prefix cache 范围也包括它之前生成的 token，所以 `Sequence::num_prefix_cache_tokens()` 可能超过 prompt 长度（实测 4096 对 3072）。原代码只有 `DCHECK` 约束这一点，release 构建里不生效。service 校验到 `cached_tokens > prompt_tokens` 后会丢弃整个结果，客户端一直挂起直到超时。
- **修复**：把这个值截断到 prompt 长度。它只用于 usage 和指标，不影响调度。

##### 测试

###### 单元测试

| 目标 | 结果 | 新增或调整的重点 |
| --- | --- | --- |
| `scheduler_test` | 135 个通过 | 成批恢复与 decode 混合组 batch；各 rank 的 FIFO 互不阻塞；恢复遵守 decode 预留；在途恢复被抢占后回到队首；PrefillFirst 先续跑在途恢复；`compute_decode_reserved_blocks` 的计算。另外保留了 main 原有的"被卡住的 decode 先拿 block"测试，并恢复了 `DecodeRestoreRetrySizesChunkAfterHostMatch` |
| `pd_decode_restore_test` | 19 个通过 | 用真实的 `BlockManagerPool` 加上 DP=2。覆盖：恢复不等其他 rank 的 reservation；rank 被其他请求占用时等待、不报错；rank 空闲但放不下时失败，后面的请求继续 FIFO；在途恢复续跑受阻时回到队首；取消请求；准入暂停与重开；MTP/kv\_split 下最后一个 chunk 后重新开放准入；P1 预留拦截准入（404 路径，block 已回滚）；已准入但 KV 尚在传输的请求计入预留 |
| `config_json_test` | 22 个通过 | 新参数的 flag、JSON 和默认值 |
| `request_prefix_cache_tokens_test` | 6 个通过 | `RestoredSequenceReportsAtMostPromptTokens` |

###### 端到端实验

**部署**

| 项目 | 配置 |
| --- | --- |
| 机器 | 单机 8 × MLU590 |
| 拓扑 | 1P1D：P 用卡 0,1，D 用卡 2,3，各自 TP2/EP2 |
| 模型 | Qwen3.5-35B-A3B（含 linear attention） |
| 脚本 | 基于 `qwen3.5-35b-pd` 部署脚本，PUSH 方式传输 KV |
| decode | `--host_blocks_factor=2`；`--max_cache_size=10.4e9`（约 12779 个 KV block，约为本负载峰值需求的 50%，保证一定发生抢占）；prefix cache 和 chunked prefill 开启，overlap 关闭，DecodeFirst 策略 |
| prefill | `--max_tokens_per_batch=65535`，用于绕开一个与本 PR 无关的 MLU 多线程 batch 构建问题（见文末"已知问题"） |

**负载**

- 192 个请求，64 并发，输入 3072 token、输出 3072 token，`ignore_eos`，temperature=0，流式输出。
- 正确性探针：压测开始 60 秒后串行发 2 轮，每轮 20 道加法题（greedy，关闭 thinking）。检查答案是否正确，并和空载时的输出逐字比对。

**结果**

|  | main | 本 PR，R=0 | **本 PR，R=512（默认）** |
| --- | --- | --- | --- |
| 成功请求 | 185/192 | 192/192 | **192/192** |
| 失败原因 | 7 个 60 秒恢复超时 | – | – |
| 总耗时 | 468 s | 500 s | **474 s** |
| 输出吞吐 | 1214 tok/s | 1180 tok/s | **1245 tok/s** |
| TPOT p50 / p99 | 35.3 / 83.8 ms | 36.6 / 67.1 ms | **33.1 / 60.5 ms** |
| TTFT p99 | 104.5 s | 116.1 s | **110.4 s** |
| 最长单次输出间隔 | 209.7 s | 69.2 s | **85.4 s** |
| 抢占次数 | – | 142 | **49** |
| 恢复批次（一次恢复 ≥2 个的批次 / 总批次，单批最多个数） | – | 31 / 97，最多 4 | 11 / 25，最多 9 |
| 准入暂停 / 重新开放 | – | 8 / 8 | 3 / 3 |
| 正确性探针 | 40/40 | 40/40 | 40/40 |
| 探针输出与空载一致 | 40/40 | 40/40 | 40/40 |

**结论**

- **不再有恢复超时**：main 上 7 个请求因 60 秒超时失败；本 PR 两组都是 192/192 成功，恢复全部完成，没有出现恢复失败。
- **吞吐持平或更高**：R=0 时吞吐为 main 的 97%（main 失败的 7 个请求没有生成 token，相当于少做了约 4% 的工作）；默认的 R=512 时吞吐为 main 的 102.5%。
- **尾延迟改善**：TPOT p99 从 83.8 ms 降到 60.5 ms（-28%）；最长单次输出间隔从 210 s 降到 85 s。
- **预留有效**：R=512 相比 R=0，抢占次数减少 65%（142 → 49），准入暂停次数从 8 次降到 3 次，吞吐提高 5.5%。默认值因此定为 512，与 SGLang 一致。
- **恢复确实成批进行**：多数恢复和正在跑的 decode 在同一个 batch 里完成，单批最多恢复 9 个请求。
- **准入不会卡在关闭状态**：每次暂停都会重新开放，压测结束后所有进程存活。
- **输出不受影响**：抢占和恢复没有改变 greedy 输出，压测期间探针的输出与空载时逐字一致。

##### 兼容性与限制

- **行为变化**：PD DECODE 默认开启 512 token 的准入预留。高并发时 decode 一次接纳的请求会变少，抢占也会变少。设 `--decode_reserved_tokens_per_request=0` 可以恢复原来的准入方式。
- **`try_allocate` 接口**：出参从 `admission_paused` 改名为 `retry_later`，含义扩展为"暂时性拒绝，prefill 可以重试"。只有一个参数的调用不受影响。
- **预留是近似值**：不计入 linear-state 和 MTP embedding slot；预留量每一步才发布一次。它只用来引导准入，最终兜底仍然是抢占。
- **PrefillFirst 下的停顿**：在 MTP/CP 等走 PrefillFirst 的场景，恢复 chunk 所在的步不跑 decode，这和 main 的行为一致。本次 e2e 走的是 DecodeFirst，没有覆盖这条路径。
- **UnifiedPolicy**：仍然不处理恢复队列，和 main 相同。

##### 补充测试

- MTP与Mooncake Store补充测试：  
  MTP（K=3）：192/192 成功，探针 40/40，恢复失败 0，所有进程存活。PrefillFirst 下的成批恢复和准入暂停都工作正常。
- Mooncake Store（P、D 都开，host\_blocks\_factor=2）：三组共 576 个请求全部成功，探针全对。decode 上传的 KV 全部写入成功，没有 store 报错。性能与不开 store 相比在单轮波动范围内（吞吐低 2–4%）。

## R02 — 邓英旭（dengyingxu1）｜2026年10月15日 10:22:26（同次 reviewed 事件时间）｜MR 总体评论｜未见解决状态标记

- 类型：评审总结；根评论，无回复。

### 评论原文

[P2 未解决] base `7a51718fb689543c72d47fc498b3d8c0edeb0879`，head `69e5397a25f26ad12a521d0bffd8dc532b19c61b`，merge-base 同 base。已检查 4 个提交、23 个变更文件，重点覆盖 `decode_admission`、`scheduler_policy`、PD decode admission/restore、相关调用链及 scheduler/config 测试；已运行 `git diff --check`（通过）。发现 1 个 P2 inline finding：`scheduler_policy.cpp:581-583` 将 prefill 序列按 decode 增长预留计入容量，可能导致合法 decode 请求持续 `RETRY_LATER`，需按 inline 建议修复。未运行 NPU、ATB、ACL Graph、HCCL、CUDA 或实际推理验证；本次结论为静态分析和 diff 检查。合并建议：修复后再合并。

## R03 — 邓英旭（dengyingxu1）｜2026/10/15 10:16:56｜xllm/core/scheduler/scheduler_policy.cpp（旧行 580 / 新行 790 后）｜状态未核实

- 类型：行级根评论；P2；根评论，无回复。
- 原讨论：[discussion_r3462522](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/495#discussion_r3462522)。线程的 Resolved / No Processing Required 为动作按钮，不能据此确定状态。

### 评论原文

[P2] 仅对 decode 阶段序列计算增长预留，避免 prefill 被错误计入 decode 容量

`compute_decode_rank_load` 遍历 `chunk_queue` 和当前运行批次时，对仍处于 `is_prefill_stage()` 的序列也调用 `decode_reserve_blocks`，默认配置会为每个 prefill 额外预留 512 token 对应的 KV block；这些 block 并非 decode 请求的增长承诺，却会被 `publish_decode_load` 纳入 `DecodeAdmission::admit` 的 headroom 判断，导致 prefill 较多时合法的 PD decode 请求被返回 `RETRY_LATER`，容量下降甚至出现饥饿。请仅对 decode 阶段计算 `decode_reserve_blocks`，prefill/restore 只保留 `missing_footprint_blocks`。

### 页面可见代码摘录（非完整上下文）

```cpp
  }

 size_t actual_tokens = 0;
 if (!allocate_for_prefill(sequence, num_tokens, &actual_tokens, state)) {
```

## R04 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:28:55｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【补充验证 · 回应 dengyingxu1 的 P2 inline finding：prefill 序列被计入 decode 增长预留】

对 note:3462522 指出的问题做了独立验证：**机制成立**，但对修复边界有一个重要补充，供作者修的时候参考。

- 位置：`xllm/core/scheduler/scheduler_policy.cpp:577-584`（compute\_decode\_rank\_load 的 add\_request lambda，遍历 running\_requests:586 / decode\_queue:589 / chunk\_queue:593；restore waiter 只进 requests 集合，598-600）
- 代码与上下文：

  ```cpp
      ++load.num_requests[dp_rank];
      for (const auto& sequence : request->sequences()) {
        if (sequence->finished()) { continue; }
        load.reserved_blocks[dp_rank] +=
            decode_reserve_blocks(*sequence, reserved_tokens, block_size) +
            missing_footprint_blocks(*sequence, block_size);
      }
  ```

  `decode_reserve_blocks`（`xllm/core/scheduler/decode_admission.cpp:48-64`）对 reserved\_tokens 折算 `ceil(min(R, 剩余可生成)/block_size)`，**不检查 is\_prefill\_stage()**；同文件 97-104 的 `missing_footprint_blocks` 才有该检查。发布与消费链：`publish_decode_load`（scheduler\_policy.cpp:604-611）每步写入 `DecodeAdmission::reserved_blocks_`（decode\_admission.cpp:167-178），brpc 线程 `admit()` 用它做准入判据（decode\_admission.cpp:122-123）。
- 独立验证结论：
  1. 机制成立：三个队列里每个未结束序列无条件贡献 decode\_reserve，默认 R=512、block\_size=16 时每个 prefill 阶段序列额外占 32 block 的「decode 增长预留」，prefill 多时准入/恢复判据收紧、合法 decode 可能 RETRY\_LATER——与原 finding 一致。
  2. 但「这些 block 并非 decode 请求的增长承诺」对**在途恢复**并不准确：作者在恢复成功时提交的正是同一份预留——`try_restore`（scheduler\_policy.cpp:801-803）`load.reserved_blocks[dp_rank] += missing_footprint_blocks(...) + own_reserve;`，且测试 `ComputesDecodeRankLoad`（tests/core/scheduler/scheduler\_policy\_test.cpp:2624-2633）**显式 pin 住了「在途恢复计入 decode\_reserve」**（注释 "plus min(20, 7) reserved tokens in 1 block"，期望 rank1 = 5 = 3 missing + 1 reserve + 1 另一 decode）。
  3. 因此按「仅对 decode 阶段计算」一刀切修复会：(a) 打破上述测试 pin 的意图行为；(b) 让在途恢复在 chunked 阶段失去增长保护，更容易再次被抢占回队首、加剧抖动——与本 MR 目标相悖。
  4. 真正无争议的超计对象只有**非恢复的 prefill 计算序列**（如 PUSH 部分前缀需本侧补算的序列）。在 e2e 配置（DecodeFirst + PUSH 全量传输）下此类序列很少，实际容量影响有限。
- 建议（最小修复）：收窄而非照抄——对 `is_inflight_restore(request)` 的序列保留 decode\_reserve（与 801-803 的提交一致），仅对非恢复的 prefill 阶段序列跳过；或者接受当前保守口径并在 compute\_decode\_rank\_load 注释里写明「在途恢复按 decode 计是故意的」。同步更新 ComputesDecodeRankLoad 增加一个非恢复 prefill 序列的用例。
- 验证：新增单测——chunk\_queue 放一个非恢复 chunked prefill，断言其对 reserved\_blocks 的贡献为 missing\_footprint only。
- 证据边界：机制与测试 pin 已静态证明（file:line + 代码原文见上）；「prefill 较多时导致饥饿」的定量影响未验证（依赖部署形态）。

## R05 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:29:01｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【P3 · 文档-代码漂移】MR 描述三处滞后于 head 实现（原子变量 / received\_request\_map\_ / try\_allocate 兼容性），误导后续读者与 reviewer

- 位置：MR 描述「准入预留」节与「兼容性与限制」节 :unknow\_emoji: `xllm/core/scheduler/decode_admission.cpp:97/167-178`、`xllm/core/scheduler/decode_admission.h:112-116`、`xllm/core/scheduler/disagg_pd_scheduler.h:139`
- 代码与上下文（head 实际机制）：

  ```cpp
  // decode_admission.cpp:167-178 —— 是 mutex 下整体覆盖，不是原子变量
  void DecodeAdmission::publish(const DecodeRankLoad& load) {
    CHECK_EQ(load.reserved_blocks.size(), reserved_blocks_.size());
    ...
    std::lock_guard<std::mutex> lock(mutex_);
    prune_pending_locked(load);
    reserved_blocks_ = load.reserved_blocks;
    num_requests_ = load.num_requests;
    for (const PendingAdmission& pending : pending_) { ... }
  }
  ```

  ```cpp
  // decode_admission.h:112-116 —— 传输期请求靠 pending_ 弱引用表，不是遍历 received_request_map_
  struct PendingAdmission {
    std::weak_ptr<Request> request;
    int32_t dp_rank = 0;
    size_t reserve_blocks = 0;
  };
  ```

  ```cpp
  // disagg_pd_scheduler.h:139 —— 签名级重构，不是出参改名
  DecodeAdmissionResult try_admit(const std::shared_ptr<Request>& request);
  ```

  描述原文称：「调度线程每一步按 rank 汇总预留量，**写进原子变量**。汇总范围是……加上已准入但 KV 还在传输的请求（**received\_request\_map\_**）」；「try\_allocate 接口：出参从 admission\_paused 改名 retry\_later……只有一个参数的调用不受影响」。
- 触发条件：任何人按描述理解 head 代码（本次审查中两位独立 reviewer 都需要额外成本对齐这套差异）。
- 当前行为：描述写的是中间 commit a6bce0d0 的版本；69e5397a2 重构后语义等价但机制不同（mutex+publish 覆盖、pending\_ 弱引用表、try\_admit 枚举）。另：描述称「暂停和重新开放都会打 INFO 日志，带暂停时长」，实际 pause 只打 "Decode admission paused"（decode\_admission.cpp:143），带 `paused_seconds=` 的只有重开日志（:152-153）。
- 期望行为：描述与 head 一致。
- 影响：maintainability（文档-代码漂移是本次审查中被重复踩的坑）。
- 根因：描述未随最后一个重构 commit 更新。
- 建议（最小修复）：合入前修订 MR 描述四处（原子变量→mutex 串行化；received\_request\_map\_→已准入未接管弱引用表；try\_allocate 兼容性段落改为 try\_admit 枚举说明；pause 日志措辞）。
- 验证：人工比对描述与 decode\_admission.{h,cpp}。
- 证据边界：已完全证明（两侧代码/文本均有原文）。

## R06 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:34:54｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【P3 · 可观测性】num\_prefix\_cache\_tokens 截断静默生效，无日志/计数，丢掉了原本暴露 invariant 破坏的信号

- 位置：`xllm/core/framework/request/sequence.cpp:927-933`（base 同文件 927-932）
- 代码与上下文（head）：

  ```cpp
  size_t Sequence::num_prefix_cache_tokens() const {
    const size_t cached_tokens = std::max(kv_state_.shared_tokens_num(),
                                          host_kv_state_.shared_tokens_num());
    // A restored preempted sequence also matches its generated tokens; usage
    // only reports cached prompt tokens.
    return std::min(cached_tokens, num_prompt_tokens_);
  }
  ```

  base 原文是 `DCHECK_LE(cached_tokens, num_prompt_tokens_); return cached_tokens;`
- 语义考古（三代）：该函数出生于 9c290a191（2026-06-03，"expose cached token usage in responses"），为 usage 上报而生，当时唯一来源是 prefix cache 命中，invariant 天然成立；host cache restore 上线后被抢占序列恢复时 host 匹配范围含已生成 token（MR 实测 4096 vs 3072），invariant 被合法打破但无人同步；本 MR 把 DCHECK 换成 min() 截断——**判断方向正确**（这是合法打破而非破坏，且测试 RestoredSequenceReportsAtMostPromptTokens 补上了覆盖）。
- 触发条件：任何 restored 序列（每次恢复必触发，非罕见路径）。
- 当前行为：debug 与 release 一律静默截断；cached>prompt 的事实不再留任何痕迹。
- 期望行为：截断保留（语义正确：usage 只该报 prompt 命中），但截断发生时应可观测——原 bug（service 丢弃结果、客户端挂起）正是靠这个差值被发现的，静默后同类回归只能靠客户端超时发现。
- 影响：operability。
- 根因：修复只处理了症状出口，没保留诊断信号。
- 建议（最小修复）：截断分支加 `VLOG(1)`/计数器（如 `COUNTER_ADD(num_prefix_cache_tokens_clamped, 1)`），仅在实际发生截断（cached\_tokens > num\_prompt\_tokens\_）时上报。
- 验证：单测构造 restored 序列断言计数器递增。
- 证据边界：截断语义正确性与观测缺失均已静态证明；无功能回归。

## R07 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:35:04｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【P3 · 性能】admit() 全程持锁，临界区包含 prefix-cache 匹配与 block 分配，PD 准入吞吐被串行化

- 位置：`xllm/core/scheduler/decode_admission.cpp:92-134`
- 代码与上下文：

  ```cpp
  DecodeAdmissionResult DecodeAdmission::admit(
      const std::shared_ptr<Request>& request) {
    ...
    std::lock_guard<std::mutex> lock(mutex_);        // 97
    if (paused_) { return DecodeAdmissionResult::RETRY_LATER; }
    if (block_manager_pool_->kv_cache_utilization() >= ...) { return REJECTED; }
    if (!block_manager_pool_->try_allocate(sequence)) {  // 107：含前缀匹配+分配
      return DecodeAdmissionResult::REJECTED;
    }
    ...
    if (decode_headroom_blocks(*block_manager_pool_, dp_rank) <
        reserved_blocks_[dp_rank] + reserve_blocks) {
      block_manager_pool_->deallocate_without_cache(sequence);
      return DecodeAdmissionResult::RETRY_LATER;
    }
  ```

  对比 base：brpc 线程直接调 `kv_cache_manager_->try_allocate(sequence)`（base disagg\_pd\_scheduler.cpp:1348-1358），无锁、可并发。
- 触发条件：PD 高并发（prefill 侧大量并发 decode\_recv\_new\_requests RPC 打到同一 decode 实例）。
- 当前行为：所有准入 RPC 在同一把 mutex\_ 后排队，每个请求的 try\_allocate（对 prompt 逐 block 哈希匹配 prefix cache）与回滚都在临界区内。
- 期望行为：串行化预留账本是必要的（防并发 RPC 重复消费同一份 headroom），但 block 分配/匹配本身可移出临界区：先无锁 try\_allocate，再持锁做 reserve 检查，不过则回滚——block manager 的分配本身是原子的，不会重复发块。
- 影响：performance（长 prompt + 深前缀缓存时准入延迟放大；e2e 64 并发未见瓶颈[作者实验数据]，4096+ prompt x 更高并发未知）。
- 根因：正确性修复（串行化）顺手把慢操作也圈进了临界区。
- 建议：若后续压测发现准入排队，把 try\_allocate 移到锁外、reserve 检查与登记留在锁内（失败回滚）。当前吞吐下可不动，仅记录技术债。
- 验证：压测 512 并发 prefill->decode 准入 RPC 的 p99 排队时间，锁内/锁外两版对比。
- 证据边界：机制已静态证明；实际吞吐影响未测量（无 benchmark）。

## R08 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:35:13｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【Question】删除 60s 超时后恢复等待无服务端上界——描述的三点论证逐条验证**成立**，但缺等待时长观测

对 MR 描述「不需要超时的理由」逐条独立验证，全部成立：

1. 「恢复队列非空时新请求进不来」——`enqueue_decode_restore` 先 `pause()`（scheduler\_policy.cpp:1107）再 `deallocate`（:1110），顺序保证释放出的块不会被并发 admit 抢走（admit 在 98-100 检查 paused\_，与 pause 同一把 mutex）。
2. 「正在跑的请求不断结束」——decode 只按 max\_tokens/EOS 推进；被抢占者经 requeue\_preempted（:650-655）回恢复队列并释放块；无新增竞争者（准入已关）。
3. 「在途传输请求 block 已计入占用」——admit 的 try\_allocate（decode\_admission.cpp:107）分配 prompt 块并计入 num\_used\_blocks\_，decode\_headroom\_blocks（:36-46）按 used 计算，恢复判据已含它们。

终止性另验：孤立 waiter 在 rank 空闲（used0 且 load.num\_requests0，scheduler\_policy.cpp:737-738）且仍放不下时走 `fail_decode_restore`（:657-669，RESOURCE\_EXHAUSTED），不会无限等；waiter 不计入 num\_requests（:598-600），所以孤立 waiter 能触发 idle 判定。

- 遗留问题：等待时长本身无上界（取决于同 rank 在跑请求的 max\_tokens；SGLang 同样无超时，可接受）。但 `fail_decode_restore`（:659-661）日志不含 waited 秒数（drain 路径有 waiting\_seconds，continuous\_scheduler.cpp:318-321，fail 路径没有），pause 总时长只在 reopen 日志（decode\_admission.cpp:152-153）。
- 建议：fail 日志补 waited 秒数 + 增加 waiting 秒数直方图指标，便于运维告警。
- 需要什么证据才能关闭：作者确认无服务端上界是预期（或补指标）。

## R09 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:35:22｜MR 总体评论｜未见解决状态标记

- 类型：独立总体评论；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 【Question】测试缺口：非恢复 prefill 序列的预留贡献、以及 brpcx调度线程并发交错均无测试

- 位置：`tests/core/scheduler/scheduler_policy_test.cpp:2615-2638`、`tests/core/scheduler/pd_decode_restore_test.cpp`
- 证据：ComputesDecodeRankLoad 只构造了 decode 阶段序列 + 在途恢复（:2621-2630），没有「非恢复 chunked prefill 序列」的 reserved\_blocks 贡献断言——恰是当前 P2 inline finding（note:3462522）争议点无测试锚定。全部恢复/准入测试为单线程 harness（RestoreHarness 直接调 policy->schedule），`admit()` 与 `publish()`/`pause()` 的跨线程交错（含 pending\_ 修剪窗口）无并发测试。
- 期望行为：P2 修复方向确定后补上述两类用例；并发窗口至少加一个 admit 与 publish 交错的 stress 测试（或 TSAN 下跑现有 harness）。
- 证据边界：缺口已证明存在；是否掩盖真实 bug 未知。

## R10 — 马晓龙（maxiaolong.maxwell）｜2026年10月15日 12:35:31｜MR 总体评论｜未见解决状态标记

- 类型：评审总结；根评论，无回复。

### 评论原文

> 自动化源码审查 · MR495（head 69e5397a2）· 一切行号以该 head 物化源码为准

##### 审查总评：With fixes（建议处理 P2 容量口径后合入）

- **审查范围**：4 个提交、23 个变更文件（+2063/-128），重点覆盖 decode\_admission、scheduler\_policy（成批恢复 + 预留汇总）、PD decode admission/restore 调用链、continuous\_scheduler / prefill\_first\_policy 接线、sequence.cpp 截断修复与三组测试；对照 base 7a51718fb 做了语义考古（60s 超时、DCHECK invariant）与重构行为改变计数（SchedulerState 回调 / 虚钩子为 MR 内 churn，base->head 净 diff 无行为损失；try\_admit 调用方迁移完整）。未运行 NPU/ATB/HCCL/实际推理，结论为静态源码级。
- **做得好的地方**：
  - `enqueue_decode_restore` 先 pause 后 deallocate（scheduler\_policy.cpp:1106-1110），从时序上根除「释放块被新请求抢走」的窗口——P0 修复的正确顺序。
  - `pending_` 弱引用表（decode\_admission.h:112-116 + prune\_pending\_locked:180-192）补上「已准入未接管请求」的预留欠计，admit->首次调度无缝，有 AdmittedRequestKeepsReserveUntilScheduled 测试背书（pd\_decode\_restore\_test.cpp:724）。
  - 恢复判据显式检查「本 rank 已承诺 + 完整 footprint + 自身预留」（try\_restore，scheduler\_policy.cpp:779-781），恢复到一半被挤的场景被显式设计并有测试（PreemptedInflightRestoreReturnsToQueueFront:2513）。
  - 测试矩阵覆盖 FIFO per-rank、idle-fail、取消、准入暂停/重开、MTP/kv\_split 重开、404 回滚等 19+7 用例，用真实 BlockManagerPool + DP=2。
- **发现问题**（各条已单独评论）：1 条 P2 与 dengyingxu1 已有 inline finding 同点（本审查独立验证成立，并给出更精确的修复边界——见补充评论）；3 条 P3（MR 描述滞后 head 三处、cached\_tokens 截断静默无观测、admit 持锁串行化）；2 条 Question（60s 超时删除论证验证成立但缺等待时长日志、测试缺口）。
- **合并建议**：处理 P2 容量口径（建议收窄到「非恢复 prefill 序列」而非原 inline 的一刀切，理由见补充评论）+ 修订 MR 描述后合入；F3/Q1/Q2 可后续跟进。

