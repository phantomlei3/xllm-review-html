# xLLM MR #659 — Review 意见

- 来源：[MR #659](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659)
- 标题：`feat: add deepseek v4.1 model support on mlu.`
- 分支：`feat/deepseek-v41-mlu-model` → `main`；5 commits、41 files changed。
- 页面状态：Open / Reviewing；要求 2 人批准；状态检查 3 项通过，因人工评审未通过而阻止合并。
- 当前 head：页面未独立核实；可见提交首项为 `fba39ae`。评审总结报告 head=`fba39aefeb7043d401b7ca327dd54b5ee328853a`、base 与 merge-base 均为 `22fd5ae1032cb69dc7dfc25d1209d8af6940a8b0`，仅作为评审者记录保留。
- 提取时间：2026-11-03T14:32:41+08:00；通过已登录 Chrome 刷新后读取，并切换全部 16 个 View Changes 评审事件，读取其已加载的线程内容。
- 筛选：未指定作者或解决状态过滤，收录全部已读取评论。
- 清单：16 条行级根评论（曾玉婷 3 条、邓英旭 6 条、冯艳 7 条）、1 条邓英旭评审总结、0 条已发布回复；另有 1 条作者说明。页面 Conversation 16 不作为评论总数。
- 线程状态：各行级线程仅见 No Processing Required / Resolved 操作按钮，未见明确解决事件，均标为“状态未核实”。回复输入框不计作已发布回复。
- 范围限制：已读取全部 16 个评审事件中的线程，未发现分页、加载更多或 OutDated 控件；未逐一加载全部 41 个文件的完整 diff，不能独立排除其他或过时讨论。不含草稿。
- 位置说明：标题中的行号是页面显示的新侧差异片段范围，不能据此推定精确行锚点；原文中评审者报告的位置另外原样保留。
- 原文说明：技术判断均为评审者意见，未经独立验证；重复主题按独立评论保留；R12 建议中的版权注释仅为单行开头，按原文保留，不补全。总结仅报告 1 个 P1，但页面还存在其他评审意见，二者范围不同，不据总结替代线程清单。

## 行动索引（导出者摘要）

| 评论 | 作者 | 主题 | 状态 |
| --- | --- | --- | --- |
| R01 | 曾玉婷 | block_size=128 约束 | 状态未核实 |
| R02 | 邓英旭 | Rust FFI panic | 状态未核实 |
| R03 | 邓英旭 | 异步 kernel 错误检查 | 状态未核实 |
| R04 | 邓英旭 | 哈希有符号溢出 | 状态未核实 |
| R05 | 邓英旭 | 非法 JSON 处理 | 状态未核实 |
| R06 | 邓英旭 | 特殊 token 魔法数字 | 状态未核实 |
| R07 | 邓英旭 | Tensor 连续性断言 | 状态未核实 |
| R08 | 曾玉婷 | Engram pad 下标与配置检查 | 状态未核实 |
| R09 | 曾玉婷 | indices 尾部与 selected_lengths | 状态未核实 |
| R10 | 冯艳 | Engram pad 下标检查 | 状态未核实 |
| R11 | 冯艳 | block_size=128 约束 | 状态未核实 |
| R12 | 冯艳 | 测试文件版权头 | 状态未核实 |
| R13 | 冯艳 | 图片 token 常量 | 状态未核实 |
| R14 | 冯艳 | 页表块宽常量 | 状态未核实 |
| R15 | 冯艳 | 模型类型 helper | 状态未核实 |
| R16 | 冯艳 | checkpoint 解析与后端耦合 | 状态未核实 |

## R01 — 曾玉婷 (zengyuting12)｜2026/11/3 10:02:40｜xllm/xllm.cpp:478-480｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491147](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491147)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**校验只是"允许 128"，没有"要求 128"，但新代码把 128 写死了。**

这条改动的措辞是 `MLU requires block_size 1 or 16; DeepSeek V4.1 requires 128.`，但条件表达的意思是：

```cpp
const bool uses_v41_blocks = model_args.model_type() == "deepseek_v41" &&
                             kv_cache_config.block_size() == 128;
if (kv_cache_config.block_size() != 16 && kv_cache_config.block_size() != 1 &&
    model_config.backend() != "dit" && !uses_v41_blocks) { LOG(FATAL) ... }
```

`--block_size=16` 时第一个子句就是 false，整条条件短路，**直接放行**。而本 PR 新增的 MLU V4.1 代码把 128 当常量用：

- `deepseek_v41_attention.cpp:300` 和 `:444`：`const int64_t block = 128;`
- `deepseek_v41_indexer.cpp:117`：`floor_divide(safe, 128) * 128 + safe.remainder(128)`（逻辑 token → 物理 cache 页的换算）

也就是说 `block_size=16` 会通过校验，然后 attention/indexer 仍按 128 算页号和页内偏移，**不报错、结果错**。默认值恰好是 128（`kv_cache_config.h:60`），所以默认跑没问题，问题出在沿用别的模型配置（MLU 上 16 很常见）时静默坏掉。

建议二选一：

1. 把"允许"改成"要求"：`if (model_type == "deepseek_v41" && block_size != 128) LOG(FATAL) << "... got " << block_size;`
2. 或者让新增代码从 `kv_cache_config.block_size()` 取值，别用字面量 128（顺带能把 `attention.cpp` 两处和 `indexer.cpp` 一处收成一个共享常量，以后再改块大小不会漏）。

我倾向第 1 个：既然注释都写了 "128-token contract"，就把它变成硬约束，比让代码适配任意块大小更符合当前实现。

### 页面差异片段

```diff
- "support other block sizes in the future.";
+      model_config.backend() != "dit" && !uses_v41_blocks) {
+ LOG(FATAL)
+        << "MLU requires block_size 1 or 16; DeepSeek V4.1 requires 128.";
```

## R02 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/framework/tokenizer/tokenizers/src/lib.rs:6-8｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491160](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491160)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**1. 健壮性 — xllm/core/framework/tokenizer/tokenizers/src/lib.rs 在**
位置：`xllm/core/framework/tokenizer/tokenizers/src/lib.rs:8`

xllm/core/framework/tokenizer/tokenizers/src/lib.rs 在 tokenizers\_normalize 函数中使用了 std::str::from\_utf8(...).unwrap() 和 normalizer.normalize(&mut normalized).unwrap()。当 C++ 侧传入非合法 UTF-8 数据或 normalizer 执行失败时，Rust 侧会触发 panic。跨越 FFI 边界触发 panic 属于未定义行为，可能导致宿主进程直接崩溃

建议移除 unwrap()，改为返回错误码（如 0 长度或空指针）或使用 std::panic::catch\_unwind 捕获异常，确保 FFI 边界绝对安全。

### 页面差异片段

```diff
 use std::ffi::{c_char, CStr};
 use std::io;
-use tokenizers::tokenizer::Tokenizer;
+use tokenizers::tokenizer::{Tokenizer, NormalizedString, Normalizer};
```

## R03 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/kernels/mlu/bangc/torch_api/deepseek_v41.cpp:1｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491161](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491161)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**2. 正确性 — xllm/core/kernels/mlu/bangc/torch\_api/deepseek\_v41.cpp 在**
位置：`xllm/core/kernels/mlu/bangc/torch_api/deepseek_v41.cpp:1`

xllm/core/kernels/mlu/bangc/torch\_api/deepseek\_v41.cpp 在 kernel 启动后调用了 CHECK\_EQ(cnrtGetLastError(), cnrtSuccess);。由于 MLU kernel 启动是异步操作，cnrtGetLastError() 通常只能捕获到启动时的同步错误，无法捕获 kernel 在设备端执行期间发生的错误（如越界访问）。这会导致执行期错误被静默忽略，难以排查

建议如果需要捕获执行错误，应在检查前调用 cnrtQueueSync 进行流同步；若仅用于检查启动错误，建议添加注释说明意图。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
```

## R04 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:1｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491162](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491162)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**3. 正确性 — xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:1`

xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在 compute\_hashes 中执行 rolling ^= value \* mul[shift]; 时，value 和 mul[shift] 均为 int64\_t 类型。当词表较大或哈希计算累积时，两者的乘积极易发生有符号整数溢出。在 C++ 中，有符号整数溢出属于未定义行为，可能导致哈希值计算错误或程序崩溃

建议将相关变量改为 uint64\_t 以利用良定义的无符号回绕语义，或在确保不溢出的前提下添加断言与注释。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
```

## R05 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_hash.cpp:1｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491163](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491163)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**4. 健壮性 — xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_hash.cpp 在构建**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_hash.cpp:1`

xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_hash.cpp 在构建 token\_map 时使用 nlohmann::json::parse(input) 解析配置文件。如果文件存在但内容为非法 JSON 格式，该函数会抛出异常。若调用栈上层未捕获，将导致进程意外终止

建议使用 nlohmann::json::parse(input, nullptr, false) 并检查返回值是否 is\_discarded()，或使用 try-catch 块包裹并转化为带有明确错误信息的 CHECK 断言。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
```

## R06 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:1-2｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491164](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491164)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**5. 健壮性 — xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:2`

xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在 compute\_hashes 中硬编码了 token == 129264 || token == 129265 这样的魔法数字。这些数字可能对应特定的特殊 token（如 BOS/EOS），但硬编码不仅降低可读性，一旦模型词表变更或配置不同，会导致哈希逻辑错误且极难排查

建议将这些 ID 提取为具名常量，或从模型配置（如 args\_）中动态读取对应的特殊 token ID。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
+
```

## R07 — 邓英旭 (dengyingxu1)｜2026/11/3 10:11:38｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:1-3｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491165](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491165)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

**6. 健壮性 — xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在**
位置：`xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:3`

xllm/core/layers/mlu/deepseek\_v41/deepseek\_v41\_engram.cpp 在 lookup\_host\_rows 中直接使用 std::memset 和 std::memcpy 操作 Tensor 的 `data_ptr<uint8_t>()`。这隐式假设了 Tensor 在内存中是连续的，如果传入的 Tensor 不满足连续性条件，会导致内存越界写

建议在执行内存操作前，添加 CHECK(weight\_rows.is\_contiguous()) 等断言以确保内存布局符合预期。

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors.
+
+Licensed under the Apache License, Version 2.0 (the "License");
```

## R08 — 曾玉婷 (zengyuting12)｜2026/11/3 10:21:59｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:146-149｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491177](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491177)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

[P2] `engram_pad_token_id` 没有区间校验，这里会 `map[-1]` 越界读

`map` 是 `token_map_` 的 int64 data\_ptr（长度 = `vocab_size`），这里直接用 `args_.engram_pad_token_id()` 当下标。

**触发**：`text_config.engram_pad_token_id` 缺省时，该字段停在 `model_args.h:257` 的 `-1`（`models/llm/deepseek_v41.h:358` 的 `LOAD_ARG_OR` 默认值也是 -1）。一旦 `blocked` 为真就执行 `map[-1]`，读到 `token_map_` 缓冲区**之前** 8 字节。而 `blocked` 在每个序列的**第一个 token** 上必然为真 —— `batch_input_builder.cpp:911` 把序列起点之前的 lookback 写成 `-1`；decode 图里被打 padding 的行 history 也全是 `-1`。所以这不是边角路径，是每个请求都会走。

**后果**：读到的是分配器元数据，这个垃圾值参与 `rolling ^= value * mul[shift]` 和取模，engram 注入行算错 → logits 悄悄偏移。不崩、不打日志。

**为什么算这个 MR 的问题**：`model_args.h:255-259` 那段注释写着 "*This args loader only feeds the Python impl … C++ never indexes a tensor with this value*" —— 本 MR 是第一个用这个值索引张量的 C++ 实现，这条不变量需要在新消费点补上。Python 侧两处都做了保护，C++ 都没有：

- `deepseek_v41_engram.py:341-343`：`if not 0 <= pad_token_id < vocab_size: raise ValueError(...)`
- `_lookback_source`：`token_safe.clamp(0, numel-1)`

**建议**（两处都补最省事）：

1. 构造函数里 fail fast，和 Python 对齐：

    ```cpp
    CHECK_GE(args_.engram_pad_token_id(), 0)
       << "engram enabled but engram_pad_token_id is unset (-1)";
    CHECK_LT(args_.engram_pad_token_id(), args_.vocab_size())
       << "engram_pad_token_id out of vocab range";
    ```
2. `compute_hashes` 里对 token 做 `std::clamp(token, 0, map_numel - 1)`（现在只挡了 `< 0` 和 129264/129265 两个 image sentinel，`token >= vocab_size` 同样无保护）。

顺带一条同源的：上方第 54 行 `args_.engram_num_embeddings().at(engram_id_)`，当 `engram_layer_ids.size() != engram_num_embeddings.size()` 时抛 `std::out_of_range`，而且是在构造期、早于 `deepseek_v41_hash.cpp` 里那句 `CHECK_EQ(layers, engram_num_embeddings.size())`，所以运维看到的是 terminate 而不是可读信息。Python `EngramLayout.__init__` 是有这条长度校验的，建议一起挪到构造函数里 CHECK。

### 页面差异片段

```diff
+                                  : history[seq * depth - relative - 1];
+        blocked = blocked || token < 0 || token == 129264 || token == 129265;
+ const int64_t value =
+            blocked ? map[args_.engram_pad_token_id()] : map[token];
```

## R09 — 曾玉婷 (zengyuting12)｜2026/11/3 10:22:01｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_indexer.cpp:166-169｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491178](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491178)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

[Question] `indices` 没预填 -1、`selected_lengths` 又被丢掉：尾部可能是未初始化值

`indices` 是 `torch::empty({1,1,index_topk})`，这里按 `k = min(length, index_topk)` 切片后直接当 logical KV 索引用（`forward_cached_attention` 把它交给 `deepseek_v41_decode_gather`）；而 `selected_lengths` 只被 kernel 写、没有一处读。

对照同目录的 v4 实现 `deepseek_v4/deepseek_v4_indexer.cpp:268-269`：

```cpp
new_block_tables.fill_(-1);
new_context_lens.zero_();
```

先整体填 -1 再交给 kernel，`-1` 就是"无有效项"的哨兵。v41 这两步都没有。

**风险**：只要 kernel 实际写出的条数 < `k`，尾部就是 `torch::empty` 的残留，被当成 logical index 往下传。而 `gather.mlu` 对负值没有跳过（`comp_table[logical / 128] * 128 + logical % 128`，`-1` 会算成 block0 的 -1 行），`lengths` 又是按 `win + count` 记的 —— 等于把未初始化的行算进 attention。prefill 那条路径的 `compute_attention` 有 `topk.ge(0)` 掩码，只有 graph decode 这条没有。

**想确认**：kernel 是否保证"一定写满 `index_topk`、空位补 -1"？

- 若保证：`selected_lengths` 可以删掉，或者至少在注释里把这个契约写清楚；同时建议给 `gather.mlu` 加一句 `if (logical < 0) continue;`，让 decode 和 prefill 两条路径对 `topk` 的假设一致。
- 若不保证：先 `indices.fill_(-1)`，并把 `k` 收成 `min(k, selected_lengths[0])`（需要一次 D2H，放在 eager callback 里即可，不影响 captured segment）。

### 页面差异片段

```diff
+ true,
+ 1,
+      std::nullopt);
+  shared.topk.at(seq) = indices.view({1, -1}).slice(1, 0, k).to(torch::kInt64);
```

## R10 — 冯艳 (fengyan.119)｜2026/11/3 12:29:14｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:146-149｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491521](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491521)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🟠 important】【✅ correctness】

##### compute\_hashes 用 engram\_pad\_token\_id 直接索引 token\_map，未设置的 -1 会越界读

---

**🔎 问题**

> `blocked ? map[args_.engram_pad_token_id()] : map[token]` 把 engram\_pad\_token\_id 当作 token\_map 的下标。该参数在 xllm/models/llm/deepseek\_v41.h:358 的默认值是 -1（LOAD\_ARG\_OR(engram\_pad\_token\_id, "text\_config.engram\_pad\_token\_id", -1)），xllm/core/framework/model/model\_args.h:256 也把 -1 定义为 unset 哨兵。同一处加载器注释明确写着「C++ never indexes a tensor with this value」，但本文件是新增的 C++ 消费方，确实拿它做了下标：配置里给了 engram\_layer\_ids 而漏写 engram\_pad\_token\_id 时，只要有任一 token 被判定 blocked，map[-1] 就会读到 token\_map\_ 缓冲区前 8 字节，hash 结果随机且随分配器状态漂移，最终静默污染整层 Engram 召回。Python 参考实现 xllm/python/models/deepseek\_v41.py:2616 对这种配置直接抛错，C++ 侧没有任何等价校验（deepseek\_v41\_engram.cpp 的构造与 verify\_loaded\_weights 都只检查权重与 token\_map 是否存在）。

---

**🔧 修改建议**

在 DeepseekV41EngramImpl 构造（deepseek\_v41\_engram.cpp:43 附近的 CHECK 段）与 verify\_loaded\_weights 中补上取值校验，与 Python 的 guard 对齐：

```cpp
CHECK_GE(args_.engram_pad_token_id(), 0)
    << "Engram is enabled but engram_pad_token_id is unset";
CHECK_LT(args_.engram_pad_token_id(), args_.engram_vocab_size())
    << "engram_pad_token_id out of token_map range";
```

或者把 compute\_hashes 里的 pad 下标在构造期解析成成员（pad\_map\_index\_），使热路径不再依赖该哨兵值。

### 页面差异片段

```diff
+                                  : history[seq * depth - relative - 1];
+        blocked = blocked || token < 0 || token == 129264 || token == 129265;
+ const int64_t value =
+            blocked ? map[args_.engram_pad_token_id()] : map[token];
```

## R11 — 冯艳 (fengyan.119)｜2026/11/3 12:29:15｜xllm/xllm.cpp:474-477｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491522](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491522)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🟠 important】【✅ correctness】

##### deepseek\_v41 的 block\_size 校验只是放行 128，没有强制 128

---

**🔎 问题**

> 新增的 uses\_v41\_blocks 只在 block\_size() == 128 时把 V4.1 从 FATAL 里豁免，条件本身仍允许 block\_size 为 16 或 1 时通过（kv\_cache\_config.block\_size() != 16 && != 1 为假即跳过报错）。但 V4.1 的 MLU 实现把 block 宽度硬编码成 128：deepseek\_v41\_attention.cpp:300 与 :444 的 `const int64_t block = 128;`，以及 deepseek\_v41\_indexer.cpp:117-119 的 `torch::floor_divide(safe, 128) * 128 + safe.remainder(128)`。block\_size 的默认值本身是 128（kv\_cache\_config.cpp:23-25），所以用户显式传 --block\_size=16 时校验通过、启动正常，随后 slots()/indexer 用 128 折算页号与页内偏移，而实际 KV cache 页宽是 16，读写落到错误的 slot，输出静默错误。这条新增校验的语义应当是「V4.1 必须 128」，而不是「V4.1 也可以是 128」。

---

**🔧 修改建议**

把豁免改成对 V4.1 的强制要求，与层内常量保持一致：

```cpp
  const bool is_v41 = model_args.model_type() == "deepseek_v41";
  if (is_v41) {
    LOG_IF(FATAL, kv_cache_config.block_size() != 128)
        << "DeepSeek V4.1 requires block_size 128, got "
        << kv_cache_config.block_size();
  } else if (kv_cache_config.block_size() != 16 &&
             kv_cache_config.block_size() != 1 &&
             model_config.backend() != "dit") {
    LOG(FATAL) << "MLU requires block_size 1 or 16;";
  }
```

### 页面差异片段

```diff
+ // V4.1's grouped SWA/C2/C1 caches use the model's 128-token contract.
+ const bool uses_v41_blocks = model_args.model_type() == "deepseek_v41" &&
+                               kv_cache_config.block_size() == 128;
  if (kv_cache_config.block_size() != 16 && kv_cache_config.block_size() != 1 &&
```

## R12 — 冯艳 (fengyan.119)｜2026/11/3 12:29:15｜tests/models/deepseek_v41_mlu_model_test.cpp:1｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491523](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491523)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🔵 minor】【📄 license】

##### 新增测试文件版权头包含 "All Rights Reserved."，与项目模板不符

---

**🔎 问题**

> 本 MR 新增文件的版权头应只写 "Copyright 2026 The xLLM Authors."，不带 "All Rights Reserved."。本文件第 1 行是 `/* Copyright 2026 The xLLM Authors. All Rights Reserved.`，而同批新增的 deepseek\_v41\_attention.cpp 等文件均正确省略了该句。

---

**🔧 修改建议**

删除第 1 行版权头中的 " All Rights Reserved."，改为：

```cpp
/* Copyright 2026 The xLLM Authors.
```

### 页面差异片段

```diff
+/* Copyright 2026 The xLLM Authors. All Rights Reserved.
```

## R13 — 冯艳 (fengyan.119)｜2026/11/3 12:29:15｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_engram.cpp:144-147｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491524](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491524)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🔵 minor】【🎨 style】

##### Engram hash 屏蔽分支使用裸魔法数字 129264/129265，已有 model\_args.image\_token\_id() 可用

---

**🔎 问题**

> 第 147 行 `blocked = blocked || token < 0 || token == 129264 || token == 129265;` 裸写图片哨兵与图片 pad 的 token id。仓库已有该数据的单一来源：model\_args.image\_token\_id()（deepseek\_v41.h:399，V4.1 加载器从 image\_token\_id 读入），且 constants.h:68-76 对 V4.1 图片 pad（image\_token\_id + 1）已有 LOCKED PAIR 注释，明确要求单一定义保持同步。裸数字使这两个 id 的语义不可读，且与既有 constants/model\_args 约定脱钩，改配置或复用常量时此处会被遗漏。项目 custom-code-style 明确禁止未命名魔法数字。

---

**🔧 修改建议**

改为引用 args 与命名常量（推荐用 ModelArgs 字段，保留 +1 与 constants.h 注释一致）：

```cpp
const int64_t image_sentinel = args_.image_token_id();
const int64_t image_pad = image_sentinel + 1;
...
blocked = blocked || token < 0 || token == image_sentinel || token == image_pad;
```

### 页面差异片段

```diff
+ const int64_t token = relative >= 0
+                                  ? tokens[start + relative]
+                                  : history[seq * depth - relative - 1];
+        blocked = blocked || token < 0 || token == 129264 || token == 129265;
```

## R14 — 冯艳 (fengyan.119)｜2026/11/3 12:29:15｜xllm/core/layers/mlu/deepseek_v41/deepseek_v41_indexer.cpp:114-117｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491525](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491525)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🔵 minor】【🎨 style】

##### select\_candidate\_indices 的页表换算裸写 128，与同文件 attention.cpp 命名常量不一致

---

**🔎 问题**

> 第 117-119 行 `torch::floor_divide(safe, 128) * 128 + safe.remainder(128)` 三处裸写 KV block\_size 128。同一目录的 deepseek\_v41\_attention.cpp:300/444 对同一语义已用命名局部常量 `const int64_t block = 128;`；本文件第 111 行的候选块宽也已命名为 args\_.candidate\_block\_size()。同一 block 语义一处命名一处裸写，可读性和一致性都差。

---

**🔧 修改建议**

在函数内与 attention.cpp 对齐，使用命名局部常量：

```cpp
const int64_t block = 128;  // KV block size.
auto physical =
    table.to(torch::kInt64).index_select(0, torch::floor_divide(safe, block)) *
        block +
    safe.remainder(block);
```

### 页面差异片段

```diff
+                     .flatten();
+ auto safe = logical.clamp_max(length - 1);
+ auto physical =
+      table.to(torch::kInt64).index_select(0, torch::floor_divide(safe, 128)) *
```

## R15 — 冯艳 (fengyan.119)｜2026/11/3 12:29:15｜xllm/xllm.cpp:473-475｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491526](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491526)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🔵 minor】【🎨 style】

##### validate\_config 裸比较 "deepseek\_v41" 字符串，应复用 util::is\_deepseek\_v41\_model\_type

---

**🔎 问题**

> 第 475 行 `model_args.model_type() == "deepseek_v41"` 在编排层硬编码模型名字符串。仓库已有唯一 owner：util::is\_deepseek\_v41\_model\_type（utils.h:206，基于 kDeepseekV41ModelType 常量），且 utils.h:213 的 is\_deepseek\_v41\_family\_model\_type 覆盖 DSpark 拼写；utils.h:210-216 注释明确要求 V4.1 家族豁免覆盖两种拼写，裸 `== "deepseek_v41"` 只覆盖一种。项目规则：通用/编排代码不写模型名字符串，走 registry/util helper。

---

**🔧 修改建议**

复用既有 helper：

```cpp
#include "core/util/utils.h"
...
const bool uses_v41_blocks =
    util::is_deepseek_v41_model_type(model_args.model_type()) &&
    kv_cache_config.block_size() == 128;
```

### 页面差异片段

```diff
    }
- // TODO: support other block sizes in the future
+ // V4.1's grouped SWA/C2/C1 caches use the model's 128-token contract.
+ const bool uses_v41_blocks = model_args.model_type() == "deepseek_v41" &&
```

## R16 — 冯艳 (fengyan.119)｜2026/11/3 12:29:16｜xllm/core/framework/hf_model_loader.cpp:171-174｜状态未核实

- 类型：行级根评论，0 条回复。
- 线程：[discussion_r3491527](http://xingyun.jd.com/codingRoot/xLLM_AI/xllm/merges/659#discussion_r3491527)
- 状态证据：仅见处理及解决操作按钮，实际解决状态未核实。

### 评论原文

##### 【🔵 minor】【🏛️ architecture】

##### checkpoint 格式解析里引入 Platform::is\_mlu()，同一权重文件在不同后端解析结果不同

---

**🔎 问题**

> load\_ct\_int\_config 是 checkpoint 侧（QuantArgs）解析，`mlu_smooth_w8a8 = Platform::is_mlu() && scheme.bits == 8 && scheme.preserve_smooth && format == "fakequant"` 让「这份 compressed-tensors 配置是否被接受」取决于当前编译/运行后端。同一份 quantization\_config 在 MLU 构建上被接受并生成量化组，在非 MLU 构建上走 LOG(ERROR) 并返回 false，进而改变上游是走量化路径还是回退。hf\_model\_loader.cpp 属于 framework 层的通用加载器，后端能力本应由执行侧决定，而不是让加载器读出两套结论。

---

**🔧 修改建议**

让解析器只描述 checkpoint 事实，把 fakequant-W8A8 的可用性判断留给执行侧：这里对 format == "fakequant" && scheme.bits == 8 && scheme.preserve\_smooth 直接接受并在 scheme 上打标记（例如 scheme.smooth\_w8a8\_fakequant = true），由消费该 QuantArgs 的后端初始化处决定是否报不支持。相应地把第 541 行注释改回解析器语义。

### 页面差异片段

```diff
+                                   : config.value("format", "int-quantized");
+ // MLU SmoothQuant exporters may retain the fakequant label for W8A8.
+ // W4A8 still requires an explicit LE-packed format; its layout differs.
+ const bool mlu_smooth_w8a8 = Platform::is_mlu() && scheme.bits == 8 &&
```

## A01 — 作者 ext.luolei15 说明｜2026年11月3日 09:53:09｜全局评论｜无明确解决状态

- 类型：独立根评论，非行级线程回复。

本 PR 为 DeepSeek-V4.1 新增 MLU 原生推理支持，接入模型加载、注意力、Indexer、Engram、Decoder 和分段 Graph 执行。

主要改动：

- 实现 SWA 与 CSA2 注意力、压缩 KV 缓存和稀疏索引，补充 BangC decode gather 算子。
- 适配原始量化 checkpoint 的权重命名与分片加载，兼容 MLU SmoothQuant W8A8 导出格式，并加强权重完整性校验。
- 根据 tokenizer 构建 Engram hash 状态；默认将 FP8 表及 scale 存放在主机内存，支持通过 engram\_config.cpu\_offload=false 切换为显存常驻。
- 接入 decode 分段 Graph，支持动态页表、token 历史更新和 Engram 命中行刷新；跨层复用 gather 缓冲，减少 Graph 常驻显存。
- 补充中英文 Engram 卸载文档。

测试覆盖：

- 新增或扩展模型加载、量化配置解析、Engram 主机与设备结果一致性、Graph replay、动态页表及共享缓冲生命周期测试。
- 本次仅整理 PR 描述，未执行编译或测试。

当前限制：要求 CP=1、DP=1，暂不支持投机解码及 Engram dp\_shared\_memory=true；MLU DeepSeek-V4.1 使用 block\_size=128

## S01 — 邓英旭 (dengyingxu1) 评审总结｜2026年11月3日 10:20:09｜全局评论｜无明确解决状态

- 类型：独立根评论，非行级线程回复。

MR #659 评审总结｜base=22fd5ae1032cb69dc7dfc25d1209d8af6940a8b0｜head=fba39aefeb7043d401b7ca327dd54b5ee328853a｜merge-base=22fd5ae1032cb69dc7dfc25d1209d8af6940a8b0

审查范围：完整 5 个提交、41 个变更文件；追踪 DeepSeek V4.1 MLU KV block 配置与 attention/indexer 页映射，并检查当前及过期讨论。

检查结果：git diff --check 通过。未运行 MLU 测试（本地缺少 PyTorch），未进行 MLU 硬件或推理验证。

未解决：1 个 P1，已有等价行内讨论：xllm/xllm.cpp:478 对 block\_size=16 放行，但 V4.1 attention/indexer 固定使用 128 计算页号和偏移。该配置会造成 KV cache 物理页映射错误并可能产生错误输出；应显式要求 V4.1 block\_size=128，或实现与配置一致的页映射。本次未重复该行内评论。

合并建议：不推荐合并
