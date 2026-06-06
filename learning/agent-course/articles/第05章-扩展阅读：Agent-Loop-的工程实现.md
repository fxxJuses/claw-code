# 第05章 扩展阅读：Agent Loop 的工程实现

> 本文是第05章《工具系统》的深度扩展。
> 第04章扩展阅读从 `prompt.rs` 走到 Agent 闭环，给出了 `conversation.rs` 的全景概览。本文在此基础上，**深入到每一条焊缝**——流式事件如何聚合成结构化消息、工具执行的每个分支如何处理、权限与 Hook 如何穿插在执行管线里、会话如何持久化与恢复、自动压缩如何触发，以及整套系统如何测试。
> **本文面向不懂 Rust 的读者**：每段代码后都有「读懂这段 Rust」小节，只解释理解 agent 逻辑所必需的语法。Rust 只是这里的表达工具，Agent 的思想本身与语言无关。

读完本章，你应该能回答五件事：

1. `AssistantEvent` 流是怎样一步步被 `build_assistant_message()` 聚合成结构化消息的？
2. 一个 ToolUse 从被模型产出到最终 ToolResult 写回 session，中间每一步发生了什么？有多少种失败路径？
3. PermissionPolicy + Hook 的三层拦截架构是如何设计的？为什么 Hook 的结果能影响权限决策？
4. 会话数据是如何用 JSONL 持久化的？自动压缩何时触发、怎么验证安全？
5. `StaticToolExecutor` 和 `ScriptedApiClient` 是怎样让 1800 行的 agent 循环变得完全可测试的？

---

## 与第04章扩展阅读的关系

第04章扩展阅读已经覆盖了 `conversation.rs` 的**骨架**：`ApiRequest` 结构、`ConversationRuntime` 字段拆解、`run_turn` 核心循环概述、工具执行六步流水线、数据流图、设计范式（CoT、ReAct、Tool Use）。

本文**不再重复骨架**，而是深入到骨架内部的每条分支、每个边界条件、每个测试用例。下表标注了哪些主题在哪篇文章：

| 主题 | 第04章扩展阅读已覆盖 | 本章新增深入 |
|---|---|---|
| ApiRequest 结构 | 字段拆解、为什么 Vec\<String\> | — |
| ConversationRuntime 字段 | 逐字段拆解 | Builder 模式、with_\* 方法链 |
| run_turn 核心循环 | 骨架概述、数据流图 | 健康探针、遥测记录、iterations 计数、usage 实时累计 |
| 工具执行六步流水线 | 概述 + 全流程图 | PreHook 三重判断、PermissionPolicy 四路决策、PostHook vs failure hook、merge_hook_feedback |
| AssistantEvent 流 | 提到但没有展开 | 五个变体详解、build_assistant_message 逐步拆解、flush_text_block、防御性检查 |
| PromptCacheEvent | — | 缓存遥测为什么是一等公民 |
| Session 持久化 | — | JSONL 格式、追加写入、write_atomic、日志轮转 |
| 会话分叉 | — | fork 的不可变语义 |
| 自动压缩 | 一段提及 | 触发条件、环境变量、compact_session 边界安全 |
| 健康探针 | — | 探针策略、两种测试 |
| 测试模式 | — | StaticToolExecutor、ScriptedApiClient、21 个测试覆盖表 |
| 错误处理 | "结构化负反馈"一段 | RuntimeError vs ToolError、三种错误统一处理、"没有异常"世界观 |
| 可观测性 | — | SessionTracer 六个事件、UsageTracker 四维度累计 |

---

## Rust 语法速查（本章补充版）

第04章扩展阅读已经有完整的速查表，这里只补充本章新出现的语法：

| 符号 / 写法 | 含义 | 本章出现的场景 |
|---|---|---|
| `BTreeMap<K, V>` | 有序键值映射（类似 SortedDict） | `StaticToolExecutor` 的 handler 注册表 |
| `Box<dyn FnMut(...)>` | 堆上的可变闭包 | 工具 handler 的类型签名 |
| `type Name = ...` | 类型别名（给复杂类型取个短名） | `ToolHandler` |
| `.map_or_else(\|\| a, \|x\| b)` | 有值就用它，没值就用默认 | Hook 改写 input |
| `std::mem::take(x)` | 拿走值，留下默认空值 | `flush_text_block` 清空 text 累加器 |
| `let Some(x) = expr else { return }` | "如果不是 Some 就提前退出" | telemetry 方法的提前返回 |
| `unreachable!("msg")` | 标记"这行代码不应该被执行" | 测试中的 `ScriptedApiClient` |

---

## 第一部分：从流式事件到结构化消息

第04章扩展阅读提到 `api_client.stream()` 返回 `AssistantEvent` 流，但没有展开这个流是什么样子、怎么变成 `ConversationMessage` 的。本部分填补这个空白。

### 1.1 AssistantEvent 枚举——流式 API 的"最小事件集"

```29:40:rust/crates/runtime/src/conversation.rs
pub enum AssistantEvent {
    TextDelta(String),
    ToolUse {
        id: String,
        name: String,
        input: String,
    },
    Usage(TokenUsage),
    PromptCache(PromptCacheEvent),
    MessageStop,
}
```

**读懂这段 Rust**

- `enum` 可以携带数据：`TextDelta(String)` 是一个"带一个 String 的变体"，`ToolUse { id, name, input }` 是一个"带三个命名字段的变体"。
- `Usage(TokenUsage)` 和 `PromptCache(PromptCacheEvent)` 把相关类型包裹在变体里——拆包时可以用 `AssistantEvent::Usage(usage)` 匹配。
- `MessageStop` 不带数据——它是一个"纯信号"，出现就表示"这条消息结束了"。

**这段代码到底在做什么？**

一句话概括：**定义了模型 API 返回流的五种最小事件类型，任何一条模型回复都可以被拆解成这五种事件的有序序列。**

逐个看每个变体的角色：

**① `TextDelta(String)` —— 文本增量**

为什么叫"Delta"而不是直接返回完整文本？因为模型是**流式**输出的——它一个 token 一个 token 地生成文本，不是一次性吐出整段话。每个 `TextDelta` 就是一小段新增文本。

想象你在手机上打字：每敲一个字，屏幕上就多一个字。`TextDelta` 就是那个"多一个字"的信号。这样做有三个工程好处：

1. **UI 实时显示**：用户能立刻看到"agent 正在想..."，不用等模型全部生成完。
2. **内存友好**：不需要把整段回复存在内存里再一次性返回。
3. **延迟低**：第一个 token 出来就能开始渲染，而不是等最后一个 token。

**② `ToolUse { id, name, input }` —— 工具调用意图**

模型决定调工具时，会产生一个 `ToolUse` 事件，包含三个信息：

| 字段 | 含义 | 示例 |
|---|---|---|
| `id` | 本次工具调用的唯一标识 | `"tool-1"` |
| `name` | 要调的工具名字 | `"read_file"` |
| `input` | 工具的输入参数（JSON 字符串） | `'{"path": "src/main.rs"}'` |

注意 `input` 的类型是 `String`，不是 JSON 对象。这是有意为之的：**序列化/反序列化的边界在 `ApiClient` 层**，runtime 只负责搬运字符串。这样 runtime 不需要关心具体的 JSON 库，降低了耦合。

**③ `Usage(TokenUsage)` —— token 计费信号**

每次 API 调用都会返回 token 用量。`Usage` 事件把这个数据流式传回来，让 runtime 可以实时累计。`TokenUsage` 的四个维度（`input_tokens`、`output_tokens`、`cache_creation_input_tokens`、`cache_read_input_tokens`）直接对应 API 的计费字段。

**④ `PromptCache(PromptCacheEvent)` —— 缓存遥测**

如果 prompt caching 出了问题（比如"我预期命中缓存但没有"），`PromptCache` 事件会报告具体情况。详见下一节 1.3。

**⑤ `MessageStop` —— 结束信号**

一个纯信号，表示"这条消息生成完毕"。为什么需要它？因为 `TextDelta` 可以出现 0 次或 N 次，`ToolUse` 也可以出现 0 次或 M 次——**没有 `MessageStop`，接收方不知道消息何时结束**。后面 `build_assistant_message` 会检查它——缺失等于"流异常截断"。

**为什么不直接用非流式 API？**

非流式 API 一次性返回完整 JSON，代码更简单。但对 agent 来说，流式几乎是必须的：

- 用户在终端前等待时，实时看到输出 → 体验好
- 长回复时可以提前开始处理 ToolUse → 延迟低
- 可以在流中间捕获 Usage → 用量追踪更精确

**Agent 视角要点**

- 五个变体构成了 agent 的"**脉搏**"——文本（推理）、工具调用（行动）、用量（计费）、缓存（省钱）、结束（终止）。
- `MessageStop` 是心跳的"最后一下"——没有它，整个消息就是"不完整"的。
- 流式不是性能优化，而是 **agent 交互模型的基本需求**。

---

### 1.2 build_assistant_message()——事件聚合器

```706:753:rust/crates/runtime/src/conversation.rs
fn build_assistant_message(
    events: Vec<AssistantEvent>,
) -> Result<
    (
        ConversationMessage,
        Option<TokenUsage>,
        Vec<PromptCacheEvent>,
    ),
    RuntimeError,
> {
    let mut text = String::new();
    let mut blocks = Vec::new();
    let mut prompt_cache_events = Vec::new();
    let mut finished = false;
    let mut usage = None;

    for event in events {
        match event {
            AssistantEvent::TextDelta(delta) => text.push_str(&delta),
            AssistantEvent::ToolUse { id, name, input } => {
                flush_text_block(&mut text, &mut blocks);
                blocks.push(ContentBlock::ToolUse { id, name, input });
            }
            AssistantEvent::Usage(value) => usage = Some(value),
            AssistantEvent::PromptCache(event) => prompt_cache_events.push(event),
            AssistantEvent::MessageStop => {
                finished = true;
            }
        }
    }

    flush_text_block(&mut text, &mut blocks);

    if !finished {
        return Err(RuntimeError::new(
            "assistant stream ended without a message stop event",
        ));
    }
    if blocks.is_empty() {
        return Err(RuntimeError::new("assistant stream produced no content"));
    }

    Ok((
        ConversationMessage::assistant_with_usage(blocks, usage),
        usage,
        prompt_cache_events,
    ))
}
```

**读懂这段 Rust**

- `Result<(A, B, C), RuntimeError>`：返回值是一个**三元组**包在 Result 里。调用方用 `let (msg, usage, cache_events) = build_assistant_message(events)?` 解构。
- `let mut text = String::new()`：创建一个空字符串，用 `push_str` 逐段拼接。
- `let mut finished = false`：一个布尔标记，用来追踪"有没有收到 MessageStop"。
- `match event { ... }`：对五种事件分别处理。
- `Ok((...))`：最后的返回值，三样东西包在 Ok 里。

**这个函数到底在做什么？（逐步拆解）**

一句话概括：**把一堆杂乱的事件流，整理成一个结构化的消息对象、一个可选的 token 用量、一组缓存事件。**

分六步看：

**① 准备五个容器**

```rust
let mut text = String::new();              // 文本累加器
let mut blocks = Vec::new();               // 内容块列表
let mut prompt_cache_events = Vec::new();  // 缓存事件列表
let mut finished = false;                  // 是否收到 MessageStop
let mut usage = None;                      // token 用量（可能有、可能没有）
```

`text` 是关键——它是一个"正在累积中的文本缓冲区"，当遇到 ToolUse 时会被刷清（flush）成一个 Text block。

**② 遍历事件，分类处理**

```rust
for event in events {
    match event {
        TextDelta(delta) => text.push_str(&delta),       // 追加到文本缓冲区
        ToolUse { .. } => {
            flush_text_block(&mut text, &mut blocks);    // 先把缓冲区刷成 Text block
            blocks.push(ContentBlock::ToolUse { .. });   // 再追加 ToolUse block
        }
        Usage(value) => usage = Some(value),             // 记录用量
        PromptCache(event) => prompt_cache_events.push(event), // 记录缓存事件
        MessageStop => finished = true,                   // 标记结束
    }
}
```

最精妙的是 `flush_text_block` 的调用时机——**只在遇到 ToolUse 时才触发**。这意味着：

- 如果事件序列是 `[TextDelta("hello"), TextDelta(" world"), MessageStop]`，text 累加成 `"hello world"`，最后在循环外 `flush_text_block` 把它变成一个 Text block。
- 如果事件序列是 `[TextDelta("Let me "), ToolUse{...}, TextDelta("Done"), MessageStop]`，第一次 flush 把 `"Let me "` 变成 Text block，ToolUse 作为独立 block 追加，循环外第二次 flush 把 `"Done"` 变成另一个 Text block。最终 blocks = `[Text("Let me "), ToolUse{...}, Text("Done")]`。

这保证了 blocks 里 Text 和 ToolUse **交替正确**，不会出现"文本粘在工具调用后面"的问题。

**③ flush_text_block 的实现**

```755:761:rust/crates/runtime/src/conversation.rs
fn flush_text_block(text: &mut String, blocks: &mut Vec<ContentBlock>) {
    if !text.is_empty() {
        blocks.push(ContentBlock::Text {
            text: std::mem::take(text),
        });
    }
}
```

只有两行，但有一个关键细节：`std::mem::take(text)` 的意思是"**拿走 text 的内容，留下一个空字符串**"。效果是：

- text 里的内容被移到 `ContentBlock::Text { text: "..." }` 里。
- text 变成空字符串，准备接收下一轮 `TextDelta`。

这比 `text.clone()` + `text.clear()` 更高效——`std::mem::take` 是零拷贝移动。

**④ 两个防御性检查**

```rust
if !finished {
    return Err(RuntimeError::new(
        "assistant stream ended without a message stop event",
    ));
}
if blocks.is_empty() {
    return Err(RuntimeError::new("assistant stream produced no content"));
}
```

- 第一个检查：**流被截断了**。可能是网络中断、API 超时、服务端错误。模型的消息不完整，不能使用。
- 第二个检查：**模型什么都没说**。既没有文本也没有工具调用——这是异常情况。

两个检查对应两个测试：

- `build_assistant_message_requires_message_stop_event`（lines 1697-1709）：只发 TextDelta 不发 MessageStop → 报错。
- `build_assistant_message_requires_content`（lines 1711-1724）：只发 MessageStop 不发内容 → 报错。

**⑤ 返回三元组**

```rust
Ok((
    ConversationMessage::assistant_with_usage(blocks, usage),
    usage,
    prompt_cache_events,
))
```

为什么返回一个三元组而不是一个 struct？因为这个函数是**内部函数**（`fn` 不是 `pub fn`），调用点只有 `run_turn` 里一处，直接解构比定义一个新 struct 更轻量。

**⑥ ConversationMessage 的结构**

```47:53:rust/crates/runtime/src/session.rs
pub struct ConversationMessage {
    pub role: MessageRole,
    pub blocks: Vec<ContentBlock>,
    pub usage: Option<TokenUsage>,
}
```

```28:44:rust/crates/runtime/src/session.rs
pub enum ContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: String },
    ToolResult { tool_use_id: String, tool_name: String, output: String, is_error: bool },
    Thinking { text: String },
}
```

`ConversationMessage` 是会话中的"一条消息"。它包含：

- `role`：这条消息是谁说的？（User / Assistant / Tool / System）
- `blocks`：消息的内容块列表（可以混合文本和工具调用）
- `usage`：可选的 token 用量（只有 Assistant 消息才有）

`ContentBlock` 是消息的"原子内容单元"，有四种形态：文本、工具调用意图、工具执行结果、思考过程。

**为什么 blocks 是 `Vec<ContentBlock>` 而不是"要么文本要么工具"？**

因为模型的一条回复可能同时包含文本和工具调用。比如：

```
[Text("让我读一下这个文件。"), ToolUse { name: "read_file", ... }]
```

模型先说了一段话（解释要做什么），然后发起了工具调用。这种"混合内容"是 agent 场景的常态，不是特例。

**Agent 视角要点**

- `build_assistant_message` 是**流式世界和结构化世界之间的桥梁**——把"一堆 Delta"翻译成"一条有结构的消息"。
- `flush_text_block` 在 ToolUse 出现时触发，保证了 Text 和 ToolUse 的正确交替。
- 两个防御性检查（缺 MessageStop、空内容）对应了两种真实的生产事故：网络截断和模型空回复。
- 返回三元组而不是 struct —— 内部函数的轻量设计，不需要额外类型定义。

---

### 1.3 PromptCacheEvent——缓存遥测为什么是一等公民

```43:50:rust/crates/runtime/src/conversation.rs
pub struct PromptCacheEvent {
    pub unexpected: bool,
    pub reason: String,
    pub previous_cache_read_input_tokens: u32,
    pub current_cache_read_input_tokens: u32,
    pub token_drop: u32,
}
```

**读懂这段 Rust**

- 所有字段都是 `pub`——公开的，外部可以读取。
- `u32`：32 位无符号整数。token 数量不可能为负，用无符号类型表达这个约束。

**这段代码到底在做什么？**

一句话概括：**记录一次 prompt caching 异常事件的完整诊断信息。**

prompt caching 是现代 LLM API 的省钱利器——以 Anthropic 为例，`cache_read` 的单价只有 `input` 的 1/10。但缓存有时会"失灵"：明明应该命中，实际没有。

`PromptCacheEvent` 的五个字段各司其职：

| 字段 | 含义 | 生产价值 |
|---|---|---|
| `unexpected: bool` | 这次缓存失灵是否"出乎意料" | 运维告警的触发条件 |
| `reason: String` | 失灵原因的文本描述 | 排查问题的第一手信息 |
| `previous_cache_read_input_tokens` | 上一轮的缓存命中 token 数 | 对比基准 |
| `current_cache_read_input_tokens` | 本轮的缓存命中 token 数 | 当前实际值 |
| `token_drop` | 两轮之间的缓存 token 下降量 | 直接量化"损失了多少钱" |

**为什么它是一等公民（不是日志里的一行文字）？**

三个理由：

1. **可度量**：`token_drop` 直接告诉你"丢了多少 token"。运维可以设阈值（比如 `token_drop > 5000` 就告警），而不是靠人肉读日志。
2. **可聚合**：每个 `TurnSummary` 里收集 `Vec<PromptCacheEvent>`，上层 CLI 可以计算"本轮缓存命中率"，展示给用户。
3. **可回溯**：异常事件被持久化在 session 里，事后可以分析"哪一轮开始缓存失灵的"。

测试 `runs_user_to_tool_to_result_loop_end_to_end_and_tracks_usage`（lines 914-966）里，`ScriptedApiClient` 的第二次调用故意返回了一个 `PromptCacheEvent { unexpected: true, token_drop: 5000, ... }`，验证它最终出现在 `summary.prompt_cache_events` 里。

**Agent 视角要点**

- **可观测性不能是后加的，要从数据结构设计时就考虑**。`PromptCacheEvent` 就是一个例子——它不是调试时打的一行 `println!`，而是一等公民数据结构。
- 这个事件从流里采集、在 `TurnSummary` 里汇总、给上层消费——形成了一个完整的可观测性管道。
- `unexpected: bool` 字段暗示了一个更深层的设计：**agent 不只关心"做了什么"，还关心"有没有什么不对劲"**。这是生产级 agent 和 demo 的区别。

---

## 第二部分：一个 Turn 的完整生命周期

第04章扩展阅读给出了 `run_turn` 的核心循环骨架，但只关注"提示词如何驱动每一步"。本部分从"一个用户输入进去，到最终 TurnSummary 出来"的全流程出发，关注第04章没展开的细节。

### 2.1 预检阶段——健康探针

```321:330:rust/crates/runtime/src/conversation.rs
if self.session.compaction.is_some() {
    if let Err(error) = self.run_session_health_probe() {
        return Err(RuntimeError::new(format!(
            "Session health probe failed after compaction: {error}. \
             The session may be in an inconsistent state. \
             Consider starting a fresh session with /session new."
        )));
    }
}
```

**读懂这段 Rust**

- `if self.session.compaction.is_some()`：如果 session 上次被压缩过（`compaction` 字段有值），才跑探针。
- `if let Err(error) = ...`：只关心失败情况（`Err`），成功时什么都不做。
- `format!("... {error} ...")`：把底层错误信息嵌入到更大的错误消息里。

**这段代码到底在做什么？**

一句话概括：**如果上次对话结束前做了上下文压缩，那么本轮对话开始前先验证 runtime 还活着。**

为什么需要这个检查？因为压缩改变了 session 的结构——早期消息被替换成了摘要。如果压缩出了问题（比如摘要丢失、工具执行器挂了），继续跑下去会产生不可预测的行为。

探针的实现：

```297:311:rust/crates/runtime/src/conversation.rs
fn run_session_health_probe(&mut self) -> Result<(), String> {
    if self.session.messages.is_empty() && self.session.compaction.is_some() {
        return Ok(());
    }
    let probe_input = r#"{"pattern": "*.health-check-probe-"}"#;
    match self.tool_executor.execute("glob_search", probe_input) {
        Ok(_) => Ok(()),
        Err(e) => Err(format!("Tool executor probe failed: {e}")),
    }
}
```

探针策略很简单：

1. 如果 session 空了但刚压缩过（`messages.is_empty() && compaction.is_some()`）——这是正常情况，直接放行。
2. 否则，用 `glob_search` 配一个不可能匹配的模式 `*.health-check-probe-`，验证 tool executor 还能正常工作。
3. 如果 tool executor 报错 → 整个 turn 被拒绝，返回一条详细的错误消息。

错误消息里甚至给了用户建议："Consider starting a fresh session with /session new."——不只是报错，还告诉你怎么修。

两个测试验证了这个机制：

- `compaction_health_probe_blocks_turn_when_tool_executor_is_broken`（lines 1615-1656）：tool executor 挂了 → 整个 turn 被拒绝。
- `compaction_health_probe_skips_empty_compacted_session`（lines 1658-1694）：空 session → 不跑探针，直接放行。

**Agent 视角要点**

- **压缩改变了上下文，必须验证 agent 还能正常工作**——这不是可选的，是必须的。
- 探针的策略是"用无害的操作验证基础设施"——`glob_search` 配不可能匹配的模式，不会产生副作用。
- 错误消息不只是"出错了"，还给了具体建议——这是生产级错误处理的标配。

---

### 2.2 用户消息入队

```332:335:rust/crates/runtime/src/conversation.rs
self.record_turn_started(&user_input);
self.session
    .push_user_text(user_input)
    .map_err(|error| RuntimeError::new(error.to_string()))?;
```

两行做了两件事：

1. **遥测记录**：`record_turn_started` 把 `user_input` 记录到 telemetry 系统（如果配置了的话）。
2. **消息入队**：`push_user_text` 把用户输入包装成 `ConversationMessage::user_text(...)` 追加到 `session.messages`。

`.map_err(|error| RuntimeError::new(error.to_string()))?` 的作用：session 层可能因为持久化失败而报错（比如磁盘满了写不了 JSONL），这个错误被转换为 `RuntimeError` 后向上传播。

此时 `session.messages` 末尾增加了一条 `User` 角色消息。下一轮 API 调用时，这条消息会作为对话历史的一部分发给模型。

---

### 2.3 循环体——每一次迭代的精确语义

```342:500:rust/crates/runtime/src/conversation.rs
loop {
    iterations += 1;
    if iterations > self.max_iterations { ... return Err(...); }

    let request = ApiRequest {
        system_prompt: self.system_prompt.clone(),
        messages: self.session.messages.clone(),
    };
    let events = match self.api_client.stream(request) { ... };
    let (assistant_message, usage, turn_prompt_cache_events) =
        match build_assistant_message(events) { ... };
    if let Some(usage) = usage {
        self.usage_tracker.record(usage);
    }
    prompt_cache_events.extend(turn_prompt_cache_events);
    let pending_tool_uses = assistant_message.blocks.iter().filter_map(...).collect::<Vec<_>>();
    self.record_assistant_iteration(iterations, &assistant_message, pending_tool_uses.len());

    self.session.push_message(assistant_message.clone())...;
    assistant_messages.push(assistant_message);

    if pending_tool_uses.is_empty() {
        break;
    }

    for (tool_use_id, tool_name, input) in pending_tool_uses {
        // 工具执行管线（第三部分展开）
    }
}
```

第04章已经讲过循环的骨架，这里只补充第04章没展开的细节：

**① iterations 计数器**

从 0 开始、每轮 +1。有两个用途：
- 做循环上限保护（`iterations > self.max_iterations`）。
- 写进遥测事件（`record_assistant_iteration(iterations, ...)`、`record_tool_started(iterations, ...)`），让运维知道"这一轮跑了几次迭代"。

**② 失败先记录遥测，再返回错误**

```rust
let error = RuntimeError::new("...");
self.record_turn_failed(iterations, &error);
return Err(error);
```

即使 turn 失败了，遥测事件也会被记录。这让运维能从 telemetry 数据里看到"多少次 turn 失败了、在第几次迭代失败的"——而不是只在日志里看到一条错误。

**③ usage 实时累计**

```rust
if let Some(usage) = usage {
    self.usage_tracker.record(usage);
}
```

每次 API 调用返回后，如果有 usage 数据就立刻累计。这意味着：

- 自动压缩的触发判断（`cumulative_usage().input_tokens >= threshold`）用的是**实时累计值**。
- 如果一次 turn 里调了 3 次 API，usage_tracker 已经累计了 3 次的用量。

**④ 为什么 assistant_message clone 两次？**

```rust
self.session.push_message(assistant_message.clone())...;
assistant_messages.push(assistant_message);
```

- `self.session.push_message(assistant_message.clone())`：一份进入 session，作为持久化 + 下轮 API 调用的对话历史。
- `assistant_messages.push(assistant_message)`：一份留在 `TurnSummary` 里，给上层调用者（CLI/IDE）看。

两份的原因：**session 是"真相源"，TurnSummary 是"收据"**。上层不需要翻 session 就能知道"本轮发生了什么"。

---

### 2.4 TurnSummary——一次 Turn 的收据

```109:117:rust/crates/runtime/src/conversation.rs
pub struct TurnSummary {
    pub assistant_messages: Vec<ConversationMessage>,
    pub tool_results: Vec<ConversationMessage>,
    pub prompt_cache_events: Vec<PromptCacheEvent>,
    pub iterations: usize,
    pub usage: TokenUsage,
    pub auto_compaction: Option<AutoCompactionEvent>,
}
```

**读懂这段 Rust**

- `pub auto_compaction: Option<AutoCompactionEvent>`：`Option` 表示"可能有、也可能没有"。`None` = 没触发压缩；`Some(...)` = 压缩了，里面告诉你移除了多少条消息。

六个字段各司其职：

| 字段 | 语义 | 上层用途 |
|---|---|---|
| `assistant_messages` | 本轮所有 assistant 消息 | 展示模型回复、提取文本 |
| `tool_results` | 本轮所有 tool result | 展示工具执行结果 |
| `prompt_cache_events` | 本轮所有缓存异常 | 展示缓存命中率、告警 |
| `iterations` | 循环执行了几次 | 监控 agent 行为 |
| `usage` | 累计 token 用量 | 展示成本 |
| `auto_compaction` | 是否触发了自动压缩 | 提示用户"上下文被压缩了" |

为什么是 `run_turn` 的返回值而不是存在 session 上？因为 **TurnSummary 是"本轮"的临时视图**——session 只存持久化数据（消息列表），不存"本轮跑了几个迭代、触发了几次压缩"这种一次性的统计信息。

**Agent 视角要点**

- `TurnSummary` 是一次 turn 的完整收据——**你能从它重建"这轮发生了什么"的全部信息**。
- `auto_compaction: Option` 的设计让上层不需要额外查询就能知道"压缩了没、压缩了多少"。
- 这是"**数据驱动 UI**"的基础——CLI/IDE 不需要解析日志，只需要读 TurnSummary。

---

## 第三部分：工具执行管线的每一条分支

第04章扩展阅读给出了六步流水线的概述和全流程图，但没有展开代码中的每一个分支条件。本部分深入到 `for (tool_use_id, tool_name, input) in pending_tool_uses` 循环内部的每一条路径。

### 3.1 PreHook：执行前的三重拦截

```401:445:rust/crates/runtime/src/conversation.rs
let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
let effective_input = pre_hook_result
    .updated_input()
    .map_or_else(|| input.clone(), ToOwned::to_owned);
let permission_context = PermissionContext::new(
    pre_hook_result.permission_override(),
    pre_hook_result.permission_reason().map(ToOwned::to_owned),
);

let permission_outcome = if pre_hook_result.is_cancelled() {
    PermissionOutcome::Deny { reason: ... }
} else if pre_hook_result.is_failed() {
    PermissionOutcome::Deny { reason: ... }
} else if pre_hook_result.is_denied() {
    PermissionOutcome::Deny { reason: ... }
} else if let Some(prompt) = prompter.as_mut() {
    self.permission_policy.authorize_with_context(..., Some(*prompt))
} else {
    self.permission_policy.authorize_with_context(..., None)
};
```

**读懂这段 Rust**

- `pre_hook_result.updated_input()` 返回 `Option<&str>`（可能有改写、可能没有）。
- `.map_or_else(|| input.clone(), ToOwned::to_owned)`：有改写就用改写后的，没有就用原始 input 的克隆。
- `if ... else if ... else if ...` 的优先级链——从上到下，匹配第一个就短路。

**这段代码到底在做什么？（逐步拆解）**

一句话概括：**PreHook 是工具执行的"海关"——它可以拦截、改写、否决工具调用，它的决定会直接影响后续的权限决策。**

分四步看：

**① 跑 PreHook**

```rust
let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
```

Hook 是用户/企业预先注册的脚本。`pre_hook_result` 可能返回 5 种状态：

| 状态 | 含义 | 后续影响 |
|---|---|---|
| 正常通过 | Hook 没有异议 | 继续走 PermissionPolicy |
| 通过 + 改写 input | Hook 修改了工具参数 | 用改写后的 input 执行 |
| denied | Hook 明确拒绝 | 直接 Deny，不问 PermissionPolicy |
| failed | Hook 脚本报错 | 直接 Deny（安全默认） |
| cancelled | Hook 被中断信号取消 | 直接 Deny（安全默认） |

**② 计算 effective_input**

```rust
let effective_input = pre_hook_result
    .updated_input()
    .map_or_else(|| input.clone(), ToOwned::to_owned);
```

如果 Hook 改写了 input（比如给 `Bash` 的命令前面自动加 `timeout 30s`），就用改写后的版本。否则用原始 input。这个"改写能力"是 Hook 最强大的功能之一——**它能在不改工具代码的情况下改变工具行为**。

**③ 构建 PermissionContext**

```rust
let permission_context = PermissionContext::new(
    pre_hook_result.permission_override(),
    pre_hook_result.permission_reason().map(ToOwned::to_owned),
);
```

`PermissionContext` 携带 Hook 的权限覆盖信息——Hook 可以直接说"这个工具允许/拒绝/要问用户"。这个覆盖会影响后续 `PermissionPolicy` 的决策。

**④ 判断优先级链**

三个 `if` 判断的优先级：`is_cancelled` > `is_failed` > `is_denied`。任何一个命中就直接 Deny，**完全绕过 PermissionPolicy**。

只有当 Hook "正常通过"时，才进入 PermissionPolicy 的授权逻辑。

测试 `denies_tool_use_when_pre_tool_hook_blocks`（lines 1055-1115）验证了 PreHook 拒绝（退出码 2）时工具不会被执行。测试 `denies_tool_use_when_pre_tool_hook_fails`（lines 1117-1180）验证了 Hook 脚本报错（退出码 1）时也会被拒绝。

**Agent 视角要点**

- **PreHook 可以绕过 PermissionPolicy 直接 Deny**——它的优先级比权限策略更高。
- **Hook 能改写 input**——这给了企业/团队一个"不改工具代码就能加安全策略"的能力。
- **三个 `if` 的优先级不是随意的**：cancelled（用户主动取消）> failed（Hook 坏了）> denied（Hook 主动拒绝）。这体现了"**安全优先于可用性**"的原则——Hook 出问题时宁可拒绝也不要冒险。

---

### 3.2 PermissionPolicy：规则引擎的四路决策

```175:292:rust/crates/runtime/src/permissions.rs
pub fn authorize_with_context(
    &self,
    tool_name: &str,
    input: &str,
    context: &PermissionContext,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> PermissionOutcome {
    // ...
}
```

**这段代码的决策路径（简化版）：**

```
1. Hook 说了 Deny？  → Deny
2. deny_rules 命中？  → Deny
3. Hook 说了 Allow？ → Allow
4. ask_rules 命中？  → 问用户或 Deny
5. allow_rules 命中？ → Allow
6. PermissionMode 满足？ → Allow
7. 都不满足？       → 问用户或 Deny
```

**PermissionMode 的层级关系**（lines 9-15）：

```rust
pub enum PermissionMode {
    ReadOnly,           // 只能读文件，不能写
    WorkspaceWrite,     // 可以写项目文件
    DangerFullAccess,   // 可以执行任意操作（包括 shell）
    Prompt,             // 每次都问
    Allow,              // 全部允许
}
```

注意 `#[derive(PartialOrd, Ord)]`——这意味着 PermissionMode 可以比较大小：`ReadOnly < WorkspaceWrite < DangerFullAccess`。每个工具有一个"最低权限要求"，只有当前模式 >= 要求时才能执行。

**prompt_or_deny 函数**（lines 294-324）：

```rust
fn prompt_or_deny(
    prompter: Option<&mut dyn PermissionPrompter>,
    request: PermissionRequest,
) -> PermissionPromptDecision {
    match prompter {
        Some(p) => p.decide(&request),
        None => PermissionPromptDecision::Deny {
            reason: "no prompter available".to_string(),
        },
    }
}
```

关键设计：**有 prompter 时弹交互式确认，没有时直接 Deny**。这意味着在非交互模式（`-p`、CI）下，如果没有预先配置 allow 规则，所有"需要确认"的工具都会被自动拒绝——安全默认值。

**Agent 视角要点**

- **权限不是"yes/no"，而是"deny → ask → allow"的渐变光谱**。
- deny 规则优先级最高——**显式拒绝永远优先于显式允许**。
- 非交互模式下，没有 prompter → 自动 Deny——**宁可拒绝也不要冒险**。

---

### 3.3 Allow 分支：执行 + PostHook + 反馈合并

```447:486:rust/crates/runtime/src/conversation.rs
PermissionOutcome::Allow => {
    self.record_tool_started(iterations, &tool_name);
    let (mut output, mut is_error) =
        match self.tool_executor.execute(&tool_name, &effective_input) {
            Ok(output) => (output, false),
            Err(error) => (error.to_string(), true),
        };
    output = merge_hook_feedback(pre_hook_result.messages(), output, false);

    let post_hook_result = if is_error {
        self.run_post_tool_use_failure_hook(...)
    } else {
        self.run_post_tool_use_hook(...)
    };
    // PostHook 能把 is_error 从 false 改成 true
    if post_hook_result.is_denied() || post_hook_result.is_failed() || post_hook_result.is_cancelled() {
        is_error = true;
    }
    output = merge_hook_feedback(post_hook_result.messages(), output, ...);

    ConversationMessage::tool_result(tool_use_id, tool_name, output, is_error)
}
```

**逐步拆解 Allow 分支：**

**① 遥测记录**

```rust
self.record_tool_started(iterations, &tool_name);
```

先记"开始执行"，后面执行完再记"执行结束"——形成一对事件，可以算工具执行耗时。

**② 执行工具——错误不抛异常**

```rust
let (mut output, mut is_error) =
    match self.tool_executor.execute(&tool_name, &effective_input) {
        Ok(output) => (output, false),
        Err(error) => (error.to_string(), true),
    };
```

**这是整个 agent 循环最关键的设计之一**：工具执行失败时，错误信息被转成字符串，`is_error` 设为 `true`，但**循环不会中断**。这个 `(output, is_error)` 对会被包装成 ToolResult 写回 session，模型下轮能看到"这个工具报错了"，从而换策略。

**③ PostHook 分两种：成功的 vs 失败的**

```rust
let post_hook_result = if is_error {
    self.run_post_tool_use_failure_hook(...)
} else {
    self.run_post_tool_use_hook(...)
};
```

**失败时不会跑正常的 PostHook，而是跑专门的 failure hook**。这三种 hook 是不同的：

| Hook 类型 | 何时跑 | 典型用途 |
|---|---|---|
| PreToolUse | 工具执行前 | 拦截、改写参数 |
| PostToolUse | 工具成功执行后 | 审计、脱敏、附加反馈 |
| PostToolUseFailure | 工具执行失败后 | 错误上报、降级建议 |

**④ PostHook 能把成功变成失败**

```rust
if post_hook_result.is_denied() || post_hook_result.is_failed() || post_hook_result.is_cancelled() {
    is_error = true;
}
```

即使工具执行成功了，如果 PostHook 拒绝了（比如检测到输出里有敏感信息），`is_error` 会被设为 `true`。这意味着**模型会看到一个"成功但被 Hook 标记为错误"的 ToolResult**。

测试 `appends_post_tool_hook_feedback_to_tool_result`（lines 1182-1255）验证了成功路径的完整流程。测试 `appends_post_tool_use_failure_hook_feedback_to_tool_result`（lines 1257-1334）验证了失败时只跑 failure hook 不跑正常 post hook。

---

### 3.4 Deny 分支：结构化负反馈

```487:492:rust/crates/runtime/src/conversation.rs
PermissionOutcome::Deny { reason } => ConversationMessage::tool_result(
    tool_use_id,
    tool_name,
    merge_hook_feedback(pre_hook_result.messages(), reason, true),
    true,
),
```

被拒的工具调用也生成 `is_error: true` 的 ToolResult，写回 session。模型下轮看到这条消息就知道"这个工具被拒了，原因是..."。

对比两种处理方式：

- ❌ **静默拒绝**：被拒后什么都不告诉模型 → 模型不知道发生了什么，可能反复尝试同一个被拒的动作 → 撞 max_iterations。
- ✅ **结构化负反馈**：被拒原因写回 session → 模型能看到"为什么被拒" → 有机会换策略。

测试 `records_denied_tool_results_when_prompt_rejects`（lines 1002-1053）展示了完整的 Deny 流程：模型调了 `blocked` 工具 → prompter 拒绝（"not now"）→ ToolResult 的 output 就是 "not now" → 模型下一轮说"I could not use the tool."。

---

### 3.5 merge_hook_feedback——反馈合并机制

```771:787:rust/crates/runtime/src/conversation.rs
fn merge_hook_feedback(messages: &[String], output: String, is_error: bool) -> String {
    if messages.is_empty() {
        return output;
    }
    let mut sections = Vec::new();
    if !output.trim().is_empty() {
        sections.push(output);
    }
    let label = if is_error { "Hook feedback (error)" } else { "Hook feedback" };
    sections.push(format!("{label}:\n{}", messages.join("\n")));
    sections.join("\n\n")
}
```

**这段代码的合并逻辑：**

1. 没有消息 → 直接返回原 output。
2. 有消息 → 原 output 在上、Hook 反馈在下，用 `\n\n` 分隔。
3. `label` 区分 "Hook feedback" 和 "Hook feedback (error)"——让模型和日志读者能区分正常反馈和错误反馈。

最终 ToolResult 的 output 可能长这样：

```
4

Hook feedback:
pre hook ran

Hook feedback:
post hook ran
```

工具结果（"4"）在最上面，PreHook 和 PostHook 的反馈分别在下面，都用明确的标签标识。

**Agent 视角要点**

- 整条管线的三种失败模型——Hook 失败、权限拒绝、工具执行错误——**全部统一为 `(output, is_error=true)` 的 ToolResult**。
- 没有异常、没有 panic、没有隐式跳过。每一条路径都有明确的输出。
- `merge_hook_feedback` 的标签设计让多来源反馈**可追溯**——日志里能清楚看到哪段是工具输出、哪段是 Hook 反馈。

---

## 第四部分：会话持久化与恢复

### 4.1 Session——Agent 的"唯一真相源"

```89:106:rust/crates/runtime/src/session.rs
pub struct Session {
    pub version: u32,
    pub session_id: String,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub fork: Option<SessionFork>,
    pub workspace_root: Option<PathBuf>,
    pub prompt_history: Vec<String>,
    pub model: Option<String>,
    persistence: Option<SessionPersistence>,
}
```

**逐字段拆解：**

| 字段 | 含义 | 什么时候变 |
|---|---|---|
| `version` | 数据格式版本号 | 初始化时设为 1 |
| `session_id` | 唯一标识符 | 初始化时生成 |
| `created_at_ms` / `updated_at_ms` | 创建/最后更新时间戳 | 每次写消息时更新 |
| `messages` | **核心**：所有对话消息 | 每轮 push |
| `compaction` | 上次压缩的元数据 | 压缩后设置 |
| `fork` | 分叉来源 | 从另一个 session fork 时设置 |
| `workspace_root` | 工作目录 | 绑定工作区时设置 |
| `prompt_history` | 用户输入历史 | 每次 push_user_text 时追加 |
| `model` | 使用的模型名 | 初始化时设置 |
| `persistence` | 持久化配置（私有） | 通过 `with_persistence_path` 设置 |

**为什么 `messages` 是"唯一真相源"？**

在 claw-code 的设计中，**agent 的所有状态都在 `session.messages` 里**。没有隐式变量、没有全局状态。想知道 agent 当前在哪一步、调了哪些工具、哪些被拒绝了——全部都在 messages 里。

这种设计带来了巨大的好处：

- **可调试**：想看 agent 状态，只看 `session.messages` 就够了。
- **可恢复**：把 messages 持久化到文件，下次启动从文件恢复，agent 就能"接着上次的对话继续"。
- **可重放**：按时间顺序回放 messages，就能复现 agent 的每一步行为。

### 4.2 JSONL 持久化——追加写入而非全量覆盖

Session 的持久化采用 **JSONL（JSON Lines）格式**——每行一个 JSON 对象。

**全量快照**（`render_jsonl_snapshot`，lines 521-540）：

```
{"type":"meta","version":1,"session_id":"abc123",...}
{"type":"compaction","summary":"...","removed_count":4}
{"type":"prompt","content":"fix the bug"}
{"type":"message","role":"user","blocks":[...]}
{"type":"message","role":"assistant","blocks":[...]}
{"type":"message","role":"tool","blocks":[...]}
```

**追加写入**（`append_persisted_message`，lines 541-556）：

如果文件已存在且非空，只 append 新消息的一行：

```
{"type":"message","role":"user","blocks":[{"Text":{"text":"next question"}}]}
```

**为什么用 JSONL 而不是 JSON？**

| 对比维度 | JSON 数组 | JSONL |
|---|---|---|
| 追加写入 | 需要修改 `[` 和 `]` | 直接 `writeln!` 一行 |
| 崩溃恢复 | 写到一半的数据可能损坏 | 每行独立，最后一行坏了不影响前面的 |
| 解析方式 | 需要整体解析 | 可以逐行解析 |
| 文件大小 | 需要全量重写 | 可以只 append |

**原子写入**（`write_atomic`）：先写临时文件再 rename，避免写到一半崩溃导致数据损坏。

**日志轮转**（`rotate_session_file_if_needed`）：文件超过 256KB 时做轮转，最多保留 3 个历史文件。

测试 `persists_conversation_turn_messages_to_jsonl_session`（lines 1420-1456）验证了完整流程：创建 session → run_turn → 持久化 → 从文件恢复 → 验证消息数量和内容一致。

### 4.3 Session fork——不修改原会话的分叉

```259:279:rust/crates/runtime/src/session.rs
pub fn fork(&self, branch_name: Option<String>) -> Session {
    let fork = SessionFork {
        parent_session_id: self.session_id.clone(),
        branch_name,
    };
    Session {
        session_id: generate_session_id(),
        messages: self.messages.clone(),
        compaction: self.compaction.clone(),
        fork: Some(fork),
        ..self.clone_base()
    }
}
```

fork 做了三件事：

1. 克隆当前 messages 和 compaction——**原 session 不受影响**。
2. 生成新的 session_id——新会话是独立的。
3. 记录 parent_session_id 和 branch_name——新会话知道自己从哪来。

测试 `forks_runtime_session_without_mutating_original`（lines 1458-1485）验证了 fork 后原 session 的 session_id、messages、fork 字段都没有变化。

**设计启示**：**会话分叉是不可变操作**——原会话不会被修改，新会话从快照开始独立演进。这和 git branch 的语义一致：创建分支不会修改当前分支。

---

## 第五部分：自动压缩机制

### 5.1 触发条件

```555:578:rust/crates/runtime/src/conversation.rs
fn maybe_auto_compact(&mut self) -> Option<AutoCompactionEvent> {
    if self.usage_tracker.cumulative_usage().input_tokens
        < self.auto_compaction_input_tokens_threshold
    {
        return None;
    }
    let result = compact_session(&self.session, CompactionConfig { ... });
    if result.removed_message_count == 0 {
        return None;
    }
    self.session = result.compacted_session;
    Some(AutoCompactionEvent {
        removed_message_count: result.removed_message_count,
    })
}
```

三个关键点：

**① 触发条件**

```rust
self.usage_tracker.cumulative_usage().input_tokens
    < self.auto_compaction_input_tokens_threshold
```

累计 input token 超过阈值就触发。默认阈值 100,000，可以通过环境变量 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` 覆盖。

**② 直接替换 session**

```rust
self.session = result.compacted_session;
```

这是 runtime 里少数几个**直接替换整个 session** 的地方。压缩后的 session 完全替代原来的。

**③ 压缩发生在 turn 的最末尾**

```rust
let auto_compaction = self.maybe_auto_compact();
// ...然后返回 TurnSummary
```

压缩在 `run_turn` 的最后一行调用，在 TurnSummary 返回之前。这意味着模型已经完成了所有工具调用和回复，压缩不会中断正在进行的工作。

### 5.2 compact_session 的边界安全

`compact_session`（compact.rs lines 95-183）的核心逻辑：

1. 从 messages 开头开始，把"旧消息"替换成一条摘要。
2. **不在 ToolUse 和 ToolResult 之间切割**——如果切割点恰好落在 ToolUse 后面，就回退到 ToolUse 之前。这保证了"每个 ToolUse 都有对应的 ToolResult"。
3. 压缩后的 session 第一条是 System 角色的 continuation message：`"这是一个从之前对话继续的会话..."` + 摘要文本。

### 5.3 压缩后验证

压缩发生在 turn 末尾，验证发生在**下一次** turn 开始时（健康探针）。测试 `auto_compacts_when_cumulative_input_threshold_is_crossed`（lines 1505-1558）验证了触发流程。测试 `skips_auto_compaction_below_threshold`（lines 1560-1595）验证了不触发的情况。

**Agent 视角要点**

- **压缩是常态而非异常**——不是"出了问题才压缩"，而是"token 超了就压缩"。
- **压缩后必须验证**——健康探针确保 agent 还能正常工作。
- **边界安全**——不在 ToolUse/ToolResult 之间切割，保证每条工具调用都有配对的结果。

---

## 第六部分：可观测性

### 6.1 SessionTracer 的六个事件

```580:685:rust/crates/runtime/src/conversation.rs
```

runtime 在 `run_turn` 的每个关键节点都记录遥测事件：

| 方法 | 触发时机 | 记录内容 |
|---|---|---|
| `record_turn_started` | 用户消息入队时 | user_input |
| `record_assistant_iteration` | 每轮 API 调用返回后 | iteration、blocks 数、pending tool use 数 |
| `record_tool_started` | 工具开始执行时 | iteration + tool_name |
| `record_tool_finished` | 工具执行完成后 | iteration + tool_name + is_error |
| `record_turn_completed` | turn 正常结束时 | iterations、messages 数、cache events 数 |
| `record_turn_failed` | turn 异常终止时 | iteration + error |

每个方法都以这段代码开头：

```rust
let Some(session_tracer) = &self.session_tracer else { return; };
```

**没有配置 tracer 就直接返回**——遥测是完全可选的，不影响核心逻辑。

测试 `records_runtime_session_trace_events`（lines 968-999）验证了完整的事件序列：`turn_started` → `assistant_iteration_completed` → `tool_execution_started` → `tool_execution_finished` → `turn_completed`。

### 6.2 UsageTracker——Token 计费的眼睛

```168:214:rust/crates/runtime/src/usage.rs
pub struct UsageTracker {
    latest_turn: TokenUsage,
    cumulative: TokenUsage,
}
```

`UsageTracker` 维护两个字段：

- `latest_turn`：最近一次 API 调用的 usage（每次 `record` 都覆盖）。
- `cumulative`：所有调用的累计 usage（每次 `record` 都累加）。

**从 session 恢复**（lines 182-190）：

```rust
pub fn from_session(session: &Session) -> Self {
    // 扫描所有带 usage 的 assistant 消息，累加成 cumulative
}
```

这是 session 恢复后 usage 不丢失的秘密——从 messages 里扫描所有 `usage: Some(...)` 的 assistant 消息，重建累计值。

测试 `reconstructs_usage_tracker_from_restored_session`（lines 1337-1376）验证了恢复后 usage 正确。

**Agent 视角要点**

- **如果无法度量，就无法改进**——从 turn 级别到 tool 级别都有遥测事件。
- UsageTracker 的准确性直接影响自动压缩的触发决策——所以它必须精确。
- **从 session 恢复**是一个容易被忽略但至关重要的设计——用户重启 CLI 后，usage 不会从零开始。

---

## 第七部分：测试一个 Agent Loop——工程模式

### 7.1 StaticToolExecutor——工具的测试替身

```789:820:rust/crates/runtime/src/conversation.rs
type ToolHandler = Box<dyn FnMut(&str) -> Result<String, ToolError>>;

pub struct StaticToolExecutor {
    handlers: BTreeMap<String, ToolHandler>,
}
```

**读懂这段 Rust**

- `type ToolHandler = Box<dyn FnMut(&str) -> Result<String, ToolError>>`：类型别名——给这个复杂的类型取个短名。
  - `Box<dyn ...>`：堆分配的 trait object（类似 Java 的接口引用、Python 的 Callable）。
  - `FnMut(&str) -> Result<String, ToolError>`：一个可变闭包，接收 `&str` 返回 `Result`。
- `BTreeMap<String, ToolHandler>`：有序的键值映射，key 是工具名，value 是 handler。

**StaticToolExecutor 的设计精髓：**

```rust
StaticToolExecutor::new()
    .register("add", |input| {
        let total = input.split(',')
            .map(|part| part.parse::<i32>().unwrap())
            .sum::<i32>();
        Ok(total.to_string())
    })
```

链式调用注册工具 handler——每个 handler 就是一个闭包。在测试中：

- 可以注册正常工作的 handler（返回 `Ok(...)`）。
- 可以注册总是失败的 handler（返回 `Err(...)`）。
- 可以注册 `panic!` 的 handler 来验证"这段代码确实不会被执行"。

比如测试 `denies_tool_use_when_pre_tool_hook_blocks` 里：

```rust
StaticToolExecutor::new().register("blocked", |_input| {
    panic!("tool should not execute when hook denies")
})
```

如果 PreHook 拒绝了但工具还是被执行了，这个 panic 会立刻暴露问题。

### 7.2 ScriptedApiClient——模型的测试替身

```845:903:rust/crates/runtime/src/conversation.rs
struct ScriptedApiClient {
    call_count: usize,
}

impl ApiClient for ScriptedApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError> {
        self.call_count += 1;
        match self.call_count {
            1 => {
                // 第一次调用：返回 ToolUse + Text + Usage + MessageStop
            }
            2 => {
                // 第二次调用：检查 tool result 存在，返回纯文本 + MessageStop
            }
            _ => unreachable!("extra API call"),
        }
    }
}
```

**脚本化 mock 的设计精髓：**

- `call_count` 追踪 API 被调了几次——用 `match self.call_count` 返回不同的预设回复。
- 第一次调用模拟"模型想调工具"，第二次模拟"模型拿到结果给最终答案"。
- `_ => unreachable!("extra API call")`——如果调了第三次，直接 panic，帮助发现逻辑错误。

**为什么不用 mockall 这种通用 mock 框架？**

1. 手写更清晰——失败信息更好读，能直接看到"第几次调用返回了什么"。
2. 不引入额外依赖。
3. 每个 mock 都是一个文档——读测试就能理解"API 客户端应该怎么被调用"。

### 7.3 测试用例全景

21 个测试覆盖了所有分支：

| 测试 | 场景 | 行号 |
|---|---|---|
| `runs_user_to_tool_to_result_loop_end_to_end_and_tracks_usage` | 完整 E2E | 914-966 |
| `records_runtime_session_trace_events` | 遥测完整性 | 968-999 |
| `records_denied_tool_results_when_prompt_rejects` | 用户拒绝权限 | 1002-1053 |
| `denies_tool_use_when_pre_tool_hook_blocks` | PreHook 拦截（exit 2） | 1055-1115 |
| `denies_tool_use_when_pre_tool_hook_fails` | PreHook 失败（exit 1） | 1117-1180 |
| `appends_post_tool_hook_feedback_to_tool_result` | PostHook 反馈合并 | 1182-1255 |
| `appends_post_tool_use_failure_hook_feedback_to_tool_result` | 失败走 failure hook | 1257-1334 |
| `reconstructs_usage_tracker_from_restored_session` | Session 恢复后 usage | 1336-1376 |
| `compacts_session_after_turns` | 手动压缩 | 1378-1418 |
| `persists_conversation_turn_messages_to_jsonl_session` | JSONL 持久化 | 1420-1456 |
| `forks_runtime_session_without_mutating_original` | 会话分叉 | 1458-1485 |
| `auto_compacts_when_cumulative_input_threshold_is_crossed` | 自动压缩触发 | 1505-1558 |
| `skips_auto_compaction_below_threshold` | 不触发压缩 | 1560-1595 |
| `auto_compaction_threshold_defaults_and_parses_values` | 环境变量解析 | 1597-1612 |
| `compaction_health_probe_blocks_turn_when_tool_executor_is_broken` | 探针失败 → 拒绝 | 1614-1656 |
| `compaction_health_probe_skips_empty_compacted_session` | 空 session 不探针 | 1658-1694 |
| `build_assistant_message_requires_message_stop_event` | 缺 MessageStop | 1696-1709 |
| `build_assistant_message_requires_content` | 空输出 | 1711-1724 |
| `static_tool_executor_rejects_unknown_tools` | 未注册工具 | 1726-1738 |
| `run_turn_errors_when_max_iterations_is_exceeded` | 超迭代上限 | 1740-1779 |
| `run_turn_propagates_api_errors` | API 失败传播 | 1781-1811 |

### 7.4 实战演练——写一个三步测试

试试写一个测试：模型第一次调 `multiply(3, 4)` 返回 12，第二次调 `add(12, 5)` 返回 17，第三次给最终答案。验证 `summary.iterations == 3`。

框架代码（填空）：

```rust
struct ThreeStepApiClient {
    call_count: usize,
}

impl ApiClient for ThreeStepApiClient {
    fn stream(&mut self, _request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError> {
        self.call_count += 1;
        match self.call_count {
            // TODO: 第一次返回 ToolUse { name: "multiply", input: "3,4" }
            // TODO: 第二次返回 ToolUse { name: "add", input: "12,5" }
            // TODO: 第三次返回纯文本 "The answer is 17."
            _ => unreachable!("too many API calls"),
        }
    }
}

#[test]
fn three_step_tool_chain() {
    let tool_executor = StaticToolExecutor::new()
        // TODO: 注册 multiply handler
        // TODO: 注册 add handler
    ;
    let mut runtime = ConversationRuntime::new(
        Session::new(),
        ThreeStepApiClient { call_count: 0 },
        tool_executor,
        PermissionPolicy::new(PermissionMode::DangerFullAccess),
        vec!["system".to_string()],
    );

    let summary = runtime.run_turn("calculate", None).expect("should succeed");

    // TODO: assert iterations == 3
    // TODO: assert tool_results.len() == 2
    // TODO: assert session.messages.len() == ?
}
```

提示：每次 API 调用后 session.messages 会增加 2 条（1 条 assistant + 1 条 tool result），最后一次只增加 1 条（assistant 纯文本）。加上最初的 1 条 user 消息，总共应该是 `1 + 2*2 + 1 = 6` 条。

---

## 第八部分：错误处理哲学——结构化负反馈

### 8.1 三种错误，统一处理

`conversation.rs` 里有三种错误来源，处理方式截然不同：

| 错误来源 | 代码位置 | 处理方式 | 是否终止循环 |
|---|---|---|---|
| API 层（`api_client.stream()` 返回 Err） | line 356-362 | `return Err(error)` | ✅ 终止整个 turn |
| 工具执行（`tool_executor.execute()` 返回 Err） | line 451-453 | `(error.to_string(), true)` | ❌ 不终止，生成 ToolResult |
| 权限拒绝 / Hook 拒绝 | line 410-445, 487-492 | `is_error: true` 的 ToolResult | ❌ 不终止 |

**设计区分**：

- **API 错误是"我不活了"**（致命）——模型服务挂了，整个 runtime 不可用，继续跑也没意义。
- **工具/权限错误是"我告诉模型换条路"**（非致命）——这一步走不通，但模型可能换个工具、换个参数、甚至直接回答。

### 8.2 RuntimeError vs ToolError

```86:106:rust/crates/runtime/src/conversation.rs
pub struct RuntimeError {
    message: String,
}

pub struct ToolError {
    message: String,
}
```

两者的字段完全一样，但语义不同：

- `RuntimeError`：运行时级别的致命错误 → `return Err(...)` → 终止 turn → 上层处理。
- `ToolError`：工具级别的非致命错误 → 转成 ToolResult → 模型看到 → 自我修正。

两者都实现了 `Display` 和 `std::error::Error`（Rust 的错误标准协议），都有 `#[must_use]` 标注。

### 8.3 "没有异常"的世界观

对比三种语言的错误处理：

| 语言 | 机制 | agent runtime 的影响 |
|---|---|---|
| Python | `try/except` | 异常可以冒泡到任何地方，容易被遗漏 |
| Java | checked exception | 编译器强制处理，但代码冗长 |
| Rust | `Result<T, E>` + `?` | 每个可能失败的调用都在类型签名里，**不可能忘记处理** |

在 agent runtime 中，`Result<T, E>` 带来的好处：

1. **所有可能失败的调用都在签名里标明**——你不可能"忘记处理"一个错误。
2. **`?` 操作符让错误传播简洁但不隐式**——读代码就能看到"这行可能失败"。
3. **没有异常冒泡**——调用栈上每一层都知道自己处理了什么。

**对 agent 设计的启示**：agent 永远不应该因为"意外异常"崩溃。所有可预见的失败（工具错误、权限拒绝、Hook 拦截）都应该通过消息回流，而不是通过异常栈。

---

## 第九部分：Builder 模式与可组合性

### 9.1 构造器链

```191:222:rust/crates/runtime/src/conversation.rs
pub fn with_max_iterations(mut self, max_iterations: usize) -> Self { ... }
pub fn with_auto_compaction_input_tokens_threshold(mut self, threshold: u32) -> Self { ... }
pub fn with_hook_abort_signal(mut self, hook_abort_signal: HookAbortSignal) -> Self { ... }
pub fn with_hook_progress_reporter(mut self, reporter: Box<dyn HookProgressReporter>) -> Self { ... }
pub fn with_session_tracer(mut self, session_tracer: SessionTracer) -> Self { ... }
```

四个 `with_*` 方法都遵循 Builder 模式：

| 方法 | 默认值 | 用途 |
|---|---|---|
| `with_max_iterations` | `usize::MAX`（不限） | 防止死循环 |
| `with_auto_compaction_input_tokens_threshold` | 环境变量或 100,000 | 压缩触发时机 |
| `with_hook_abort_signal` | 默认信号 | 用户按 Ctrl+C 时中止 hook |
| `with_hook_progress_reporter` | None | UI 显示"正在跑 hook..." |
| `with_session_tracer` | None | 遥测收集 |

**默认安全**：不设 max_iterations 也能跑（默认 `usize::MAX`），不配 tracer 也不影响功能——但生产环境一定要设。

### 9.2 泛型的代价——一个 struct，多种 runtime

```126:139:rust/crates/runtime/src/conversation.rs
pub struct ConversationRuntime<C, T> {
    api_client: C,
    tool_executor: T,
    ...
}
```

三个核心 trait 的接口：

```rust
trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}

trait PermissionPrompter {
    fn decide(&mut self, request: &PermissionRequest) -> PermissionPromptDecision;
}
```

**每个 trait 都只有一个方法**——这是"最小接口原则"。

**为什么 PermissionPrompter 不是泛型参数？**

`PermissionPrompter` 是 `run_turn` 的方法参数（`Option<&mut dyn PermissionPrompter>`），而不是 `ConversationRuntime` 的泛型参数。因为权限提示是**每次调用可能不同的**——同一次 runtime 交互里，不同工具可能需要不同的提示行为。如果它是泛型参数，整个 runtime 的类型签名会变成 `ConversationRuntime<C, T, P>`，每换一种 prompter 就要重建 runtime。

同一个 `ConversationRuntime` 代码，可以实例化出完全不同的 agent：

```rust
// 真实生产 agent
ConversationRuntime<AnthropicApiClient, FileSystemToolExecutor>

// 测试用的 mock agent
ConversationRuntime<ScriptedApiClient, StaticToolExecutor>

// 本地离线 agent
ConversationRuntime<OllamaApiClient, FileSystemToolExecutor>
```

**Agent 视角要点**

- 泛型 + trait 是**可测试性**的技术基础——没有它，mock 和脚本化测试都不可能。
- 三个单方法 trait 遵循"最小接口原则"——trait 越小，实现越容易，mock 越简单。
- `PermissionPrompter` 作为方法参数而非泛型参数，让同一次 runtime 能在不同调用中切换提示行为。

---

## 第十部分：小结与延伸练习

### 10.1 六条带走的结论

1. **流式事件是 Agent 的脉搏**：TextDelta + ToolUse + Usage + MessageStop 四种事件构成最小心跳集，`build_assistant_message` 把它们聚合成结构化消息。
2. **工具执行的每条路径都生成 ToolResult**：成功、失败、被拒、Hook 拦截——终点都是同一种数据结构，通过 `is_error` 标记区分。这让模型有统一的"反馈接收口"。
3. **三层拦截构成纵深防御**：PreHook → PermissionPolicy → PostHook，任何一层都能否决或改写操作，且否决信息一定回流到模型。
4. **会话是唯一真相源**：所有状态都在 `session.messages` 里，没有隐式变量。持久化用 JSONL 追加写入，恢复时从文件重建。
5. **压缩是常态而非异常**：token 超过阈值就自动压缩，压缩后必须验证健康，验证不过就拒绝继续。
6. **可测试性是设计出来的**：泛型 + trait 让 API 客户端和工具执行器都可以被脚本化 mock 替换，21 个测试覆盖了所有分支。

### 10.2 Rust 语言留给 Agent 开发者的启示

- `Result<T, E>` 比 try/catch 更适合 agent runtime：每一个可能失败的调用都在签名里标明。
- `Option<T>` 比 null 更安全：不存在"忘了检查 null"的问题。
- 泛型 + trait 是"可测试性"的技术基础——没有它，mock 和脚本化测试都不可能。
- `#[must_use]` 让你不可能忘记处理返回值。
- `std::mem::take` 在文本累加器场景下的优雅用法——零拷贝清空。

### 10.3 可以深入的方向

1. 给 `ConversationRuntime` 加一个新 trait `ToolRegistry`，让工具可以在运行时动态注册/注销，观察对测试的影响。
2. 实现 `build_assistant_message` 的真正的流式版本——不是先收集 `Vec<AssistantEvent>` 再聚合，而是边接收边处理。思考需要改变什么。
3. 写一个测试：模型调了工具，工具成功执行，但 PostHook 把结果改成了 "permission denied by post-hook"——验证模型下轮能看到这个"假拒绝"。
4. 在 `run_turn` 的循环里加一个 `println!` 打印每轮的 `iterations` 和 `pending_tool_uses.len()`，跑一个真实的 agent 会话，观察循环执行了几次。
5. 读 `hooks.rs` 的 `run_commands` 函数，理解 shell 命令是如何被构建和执行的。

### 10.4 对应论文和资料

1. **Yao et al., 2022 —— ReAct**: 本章的循环结构就是 ReAct 的工程实现。
2. **Shinn et al., 2023 —— Reflexion**: 结构化负反馈让模型自我修正。
3. **Anthropic Tool Use 文档**: ToolResult 的 `is_error` 设计来源于此。
4. **The Twelve-Factor App（日志篇）**: JSONL 追加写入的思想来源。
5. **Michael Nygard —— Release It!（Circuit Breaker 模式）**: 健康探针的设计灵感。

---

## 本章练习（选做，与语言无关）

1. 写一个测试场景：模型第一次调 `multiply(3, 4)`，工具返回 12；第二次调 `add(12, 5)`，工具返回 17；第三次给出最终答案。验证 `summary.iterations == 3`、`summary.tool_results.len() == 2`、`session.messages.len() == 6`。

2. 给 `StaticToolExecutor` 加一个 `register_failing` 方法，注册一个总是返回 `Err(ToolError::new("boom"))` 的 handler。用它测试"模型看到工具错误后会不会在下一轮改变策略"——你的 ScriptedApiClient 的第二次调用应该返回纯文本而不是再次调工具。

3. 手动构造一个 `Session`（包含 5 轮对话），设置 `auto_compaction_input_tokens_threshold` 为很低的值，调用 `run_turn`，观察 `auto_compaction` 是否为 Some，以及压缩后 session 的第一条消息是什么角色。

4. 读 `permissions.rs` 的 `PermissionRule::parse`（lines 349-375），理解 `"Bash(git commit*)"` 这样的规则字符串是怎么被解析的。写一条规则让 `Write(*)` 类的工具调用自动拒绝。

5. 用你熟悉的语言（Python / TypeScript / Go），实现一个简化版的 `build_assistant_message`：输入一组模拟的事件（包含多个 TextDelta 和一个 ToolUse），输出一个结构化的消息对象。验证 TextDelta 被正确合并、ToolUse 被正确提取、缺少 MessageStop 时报错。
