# 第07章 上下文压缩：Agent 如何跨越 Token 边界的工程实现

> 本文是 claw-code 学习系列的第七篇，聚焦于 Agent 系统中最硬核的工程挑战——**上下文压缩（Context Compaction）**。
> 前几章我们已经看到 Session 如何存储消息、工具如何执行、会话如何分叉。但一个真正运行中的 Agent 还有一个无法回避的问题：**模型的上下文窗口是有限的，而用户的需求是无限的。**
> 当对话越来越长、工具输出越来越多，token 预算终将被耗尽。claw-code 的解法不是简单地丢弃旧消息，而是一套精心设计的**检测 → 摘要 → 压缩 → 恢复**流水线。
> **本文面向不懂 Rust 的读者**：每段代码后都有「读懂这段 Rust」小节，只解释理解上下文压缩所必需的语法。

读完本章，你应该能回答六件事：

1. 自动压缩的触发条件是什么？`100,000 token` 阈值从何而来？
2. `should_compact` 如何判断"够了，该压缩了"？
3. 摘要生成的算法是什么？信息如何被提炼为结构化文本？
4. `ToolUse / ToolResult` 边界安全机制如何避免 OpenAI 兼容层的 400 错误？
5. 二次压缩时，旧摘要和新摘要如何合并？
6. `SummaryCompressionBudget` 的四级优先级选择器是如何在预算内保留最重要信息的？

---

## 与前几章的关系

| 主题 | 第06章已覆盖 | 本章新增深入 |
|---|---|---|
| 上下文压缩 | 触发条件概述、compact_session 边界安全简介 | 完整四阶段流水线深度解析 |
| SessionCompaction | 结构体字段列表 | count/removed_message_count 的生命周期、与 JSONL 持久化的联动 |
| compact.rs | 函数签名 | `summarize_messages` 的完整算法、文件提取逻辑、待办推断 |
| summary_compression.rs | 未覆盖 | 全新内容：budget 驱动的行级选择器 |
| 健康探针 | 未覆盖 | 压缩后的 session 健康检查机制 |
| 二次压缩 | 未覆盖 | `merge_compact_summaries` 的合并策略 |

---

## Rust 语法速查（本章新增）

| 符号 / 写法 | 含义 | 本章出现的场景 |
|---|---|---|
| `&[T]` | 对切片（一段连续元素）的引用，比 `&Vec<T>` 更通用 | `summarize_messages(messages: &[ConversationMessage])` |
| `.flat_map(\|x\| ...)` | 先 map 再展平（类似 Python 的列表推导+chain） | 提取所有 ToolUse 名称 |
| `.filter_map(\|x\| ...)` | 同时过滤和映射——None 被丢弃，Some 被保留 | 提取文本内容 |
| `BTreeSet<T>` | 有序去重集合（类似 Python 的 sorted set） | `select_line_indexes` 中的行选择 |
| `matches!(expr, Pattern)` | 判断一个值是否匹配某个模式 | `matches!(b, ContentBlock::ToolResult { .. })` |
| `.saturating_sub(n)` | 减法溢出时停在 0 而不是 panic | `keep_from` 的安全递减 |
| `..Default::default()` | 其余字段使用默认值 | `CompactionConfig { max_estimated_tokens: 0, ..Default::default() }` |

---

## 第一部分：问题——为什么需要上下文压缩？

### 1.1 Token 窗口的硬约束

大语言模型都有一个上下文窗口上限。以 Claude 系列为例：

| 模型 | 上下文窗口 | 最大输出 |
|---|---|---|
| Claude Opus 4.6 | 200,000 tokens | 32,000 tokens |
| Claude Sonnet 4.6 | 200,000 tokens | 64,000 tokens |
| Grok 系列 | 131,072 tokens | 64,000 tokens |

200K 看起来很大，但 Agent 的对话是**指数增长**的：

1. 用户发送请求（~100 tokens）
2. 助手分析代码并调用工具（~500 tokens）
3. 工具返回结果——比如 `grep` 的输出可能就是数千 tokens
4. 助手基于结果再次调用工具（又是数百 tokens）
5. 工具又返回大量结果……

一轮 Agent Loop 可能消耗 5,000–20,000 tokens。10 轮下来，100K–200K tokens 就没了。而一个真实的编码会话可能持续几十轮甚至上百轮。

**如果不做压缩，对话要么因为超出窗口而被截断，要么不得不丢弃完整的历史。两种方案都会导致 Agent 丢失上下文，做出错误的决策。**

### 1.2 设计目标

claw-code 的上下文压缩追求四个目标：

| 目标 | 具体含义 |
|---|---|
| **自动触发** | 用户不需要手动管理，系统在 token 超限时自动压缩 |
| **信息保留** | 不是粗暴截断，而是生成结构化摘要保留关键信息 |
| **边界安全** | 压缩不会拆散 ToolUse/ToolResult 消息对，避免 API 报错 |
| **可组合** | 支持多次压缩——第二次压缩时，旧摘要和新上下文要能合并 |

---

## 第二部分：全局流程——从检测到恢复

上下文压缩不是一步完成的。它是一条**四阶段流水线**，嵌入在 Agent Loop 的生命周期中：

```
┌────────────────────────────────────────────────────────────────┐
│                    run_turn() 对话循环                          │
│                                                                │
│  ① turn 开始                                                    │
│     │                                                          │
│     ▼                                                          │
│  [健康探针] ← 如果 session 曾被压缩，先做健康检查                │
│     │                                                          │
│     ▼                                                          │
│  [Agent Loop] 用户输入 → API 调用 → 工具执行 → 结果聚合          │
│     │                                                          │
│     ▼                                                          │
│  ② maybe_auto_compact() ← turn 结束后检查是否需要压缩           │
│     │                                                          │
│     ├── 不需要 → 返回 TurnSummary                               │
│     │                                                          │
│     └── 需要 → ③ compact_session()                             │
│              │                                                 │
│              ├── should_compact()  检测                         │
│              ├── compact_session() 执行压缩                     │
│              │    ├── 边界安全保护                               │
│              │    ├── summarize_messages()  生成摘要             │
│              │    ├── merge_compact_summaries()  合并旧摘要      │
│              │    └── 构建续接 System 消息                       │
│              └── 替换 session                                   │
│                                                                │
│     ▼                                                          │
│  返回 TurnSummary（携带 auto_compaction 事件）                  │
│                                                                │
│  下次 turn 开始时 → 回到 ①，触发健康探针                        │
└────────────────────────────────────────────────────────────────┘
```

这四个阶段分别由不同模块负责：

| 阶段 | 模块 | 关键函数 |
|---|---|---|
| 检测（何时压缩） | `conversation.rs` | `maybe_auto_compact()` |
| 决策（能否压缩） | `compact.rs` | `should_compact()` |
| 执行（如何压缩） | `compact.rs` | `compact_session()` → `summarize_messages()` |
| 精炼（摘要压缩） | `summary_compression.rs` | `compress_summary()` |

让我们逐一深入。

---

## 第三部分：检测——何时触发压缩？

### 3.1 自动压缩的阈值

自动压缩由 `ConversationRuntime` 中的 `maybe_auto_compact()` 方法驱动。它在每次 turn 结束后被调用：

```rust:rust/crates/runtime/src/conversation.rs
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
const AUTO_COMPACTION_THRESHOLD_ENV_VAR: &str = "CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS";
```

**读懂这段 Rust**

- `const` 是编译时常量，类似于 C 的 `#define`。
- `u32` 是 32 位无符号整数，最大值约 42 亿，对 token 计数绰绰有余。

默认阈值是 **100,000 input tokens**。为什么不是 200K（模型窗口上限）？因为需要留空间给：
- 系统提示词（可能占 10K–30K tokens）
- 工具定义（每个工具的 JSON Schema）
- 最新一轮的回复（输出也需要在窗口内）

这个阈值可以通过环境变量覆盖：

```rust:rust/crates/runtime/src/conversation.rs
pub fn auto_compaction_threshold_from_env() -> u32 {
    parse_auto_compaction_threshold(
        std::env::var(AUTO_COMPACTION_THRESHOLD_ENV_VAR)
            .ok()
            .as_deref(),
    )
}

fn parse_auto_compaction_threshold(value: Option<&str>) -> u32 {
    value
        .and_then(|raw| raw.trim().parse::<u32>().ok())
        .filter(|threshold| *threshold > 0)
        .unwrap_or(DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD)
}
```

**读懂这段 Rust**

- `.ok()` 把 `Result` 变成 `Option`——把错误变成 `None`。
- `.and_then()` 对 `Option` 做"链式操作"——如果前面是 `None`，直接短路返回 `None`。
- `.filter()` 对 `Option` 的值做过滤——不满足条件就变 `None`。
- `.unwrap_or()` 最终兜底——如果一路下来还是 `None`，用默认值。

解析逻辑很简单：读环境变量 → 转成数字 → 必须 > 0 → 否则用默认值。如果用户设置了 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS=50000`，就会在 50K tokens 时提前触发压缩。

### 3.2 `maybe_auto_compact` 的实际流程

```rust:rust/crates/runtime/src/conversation.rs
fn maybe_auto_compact(&mut self) -> Option<AutoCompactionEvent> {
    // 1. 检查累计 input tokens 是否达到阈值
    if self.usage_tracker.cumulative_usage().input_tokens
        < self.auto_compaction_input_tokens_threshold
    {
        return None;  // 还没到，不压缩
    }

    // 2. 执行压缩（max_estimated_tokens 设为 0，意思是"只要有可压缩的消息就压缩"）
    let result = compact_session(
        &self.session,
        CompactionConfig {
            max_estimated_tokens: 0,       // 不设最低门槛，满足阈值就压缩
            ..CompactionConfig::default()   // preserve_recent_messages = 4
        },
    );

    // 3. 如果没实际移除消息，返回 None
    if result.removed_message_count == 0 {
        return None;
    }

    // 4. 替换 session 为压缩后的版本
    self.session = result.compacted_session;
    Some(AutoCompactionEvent {
        removed_message_count: result.removed_message_count,
    })
}
```

**读懂这段 Rust**

- `&mut self` 表示"可变借用 self"——这个方法会修改 `self` 的字段。
- `..CompactionConfig::default()` 是结构体更新语法——未指定的字段用默认值填充。
- `Some(AutoCompactionEvent { ... })` 构造一个 `Some` 值，表示"压缩确实发生了"。

注意一个微妙的设计：`max_estimated_tokens: 0`。在自动压缩模式下，阈值由 `maybe_auto_compact` 外部的 token 累计器控制（100K）。一旦进入 `compact_session`，不再要求"至少有多少 tokens 才压缩"——只要能压缩就压缩。这与手动 `/compact` 命令的行为不同（手动模式下 `max_estimated_tokens` 使用默认值 10,000）。

---

## 第四部分：决策——`should_compact` 的判断逻辑

`compact_session` 内部首先调用 `should_compact` 判断是否满足压缩条件：

```rust:rust/crates/runtime/src/compact.rs
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,   // 默认 4
    pub max_estimated_tokens: usize,        // 默认 10,000
}

pub fn should_compact(session: &Session, config: CompactionConfig) -> bool {
    // 跳过已有的摘要 System 消息
    let start = compacted_summary_prefix_len(session);
    let compactable = &session.messages[start..];

    // 条件 1：可压缩消息数 > 保留数量
    // 条件 2：可压缩消息的预估 token 数 >= 最低门槛
    compactable.len() > config.preserve_recent_messages
        && compactable
            .iter()
            .map(estimate_message_tokens)
            .sum::<usize>()
            >= config.max_estimated_tokens
}
```

**读懂这段 Rust**

- `&session.messages[start..]` 是切片语法——从 `start` 索引开始取到末尾。
- `.iter().map(f).sum()` 是经典的 map-reduce 模式。
- 两个条件用 `&&` 连接，意味着**必须同时满足**。

这里有两条规则：

| 规则 | 含义 |
|---|---|
| `compactable.len() > preserve_recent_messages` | 可压缩的消息必须比要保留的多——不能把所有消息都压缩了 |
| `总 tokens >= max_estimated_tokens` | 可压缩部分的 token 量必须达到门槛——不值得为几条消息做压缩 |

### 4.1 Token 估算——不精确但足够好的启发式

```rust:rust/crates/runtime/src/compact.rs
fn estimate_message_tokens(message: &ConversationMessage) -> usize {
    message
        .blocks
        .iter()
        .map(|block| match block {
            ContentBlock::Text { text } => text.len() / 4 + 1,
            ContentBlock::ToolUse { name, input, .. } => (name.len() + input.len()) / 4 + 1,
            ContentBlock::ToolResult { tool_name, output, .. } => {
                (tool_name.len() + output.len()) / 4 + 1
            }
        })
        .sum()
}
```

**读懂这段 Rust**

- `match` 是模式匹配——根据 `block` 的具体类型执行不同分支。
- `text.len()` 返回字节数（不是字符数），除以 4 是估算"4 字节约等于 1 token"。

这是一个**粗略启发式**：用字符长度除以 4 来估算 token 数。对于英文文本，1 token ≈ 4 个字符是比较准确的；对于中文或其他多字节语言会偏高。但对于"判断是否需要压缩"这个决策来说，不需要精确到个位数——只要方向对就行。

---

## 第五部分：执行——`compact_session` 的核心算法

这是整个系统最复杂的部分。让我们逐步拆解：

### 5.1 整体骨架

```rust:rust/crates/runtime/src/compact.rs
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    // Step 0: 不需要压缩？直接返回
    if !should_compact(session, config) {
        return CompactionResult {
            summary: String::new(),
            formatted_summary: String::new(),
            compacted_session: session.clone(),
            removed_message_count: 0,
        };
    }

    // Step 1: 提取已有的旧摘要（如果之前压缩过）
    let existing_summary = session.messages.first()
        .and_then(extract_existing_compacted_summary);
    let compacted_prefix_len = usize::from(existing_summary.is_some());

    // Step 2: 计算分割点——哪些消息要压缩，哪些要保留
    let raw_keep_from = session.messages.len()
        .saturating_sub(config.preserve_recent_messages);

    // Step 3: 边界安全保护——确保不拆散 ToolUse/ToolResult 对
    let keep_from = /* 边界修正逻辑 */;

    // Step 4: 分割消息
    let removed = &session.messages[compacted_prefix_len..keep_from];   // 要压缩的
    let preserved = session.messages[keep_from..].to_vec();             // 要保留的

    // Step 5: 生成摘要并合并旧摘要
    let summary = merge_compact_summaries(
        existing_summary.as_deref(),
        &summarize_messages(removed),
    );

    // Step 6: 构建压缩后的 session
    let formatted_summary = format_compact_summary(&summary);
    let continuation = get_compact_continuation_message(
        &summary, true, !preserved.is_empty()
    );

    let mut compacted_messages = vec![ConversationMessage {
        role: MessageRole::System,
        blocks: vec![ContentBlock::Text { text: continuation }],
        usage: None,
    }];
    compacted_messages.extend(preserved);

    let mut compacted_session = session.clone();
    compacted_session.messages = compacted_messages;
    compacted_session.record_compaction(summary.clone(), removed.len());

    CompactionResult { summary, formatted_summary, compacted_session, removed_message_count: removed.len() }
}
```

压缩后的 session 结构如下：

```
压缩前：
  [消息1] [消息2] [消息3] [消息4] [消息5] [消息6] [消息7] [消息8]
                    ↑                              ↑
               要压缩的                       要保留的(4条)

压缩后：
  [System: 摘要续接消息] [消息5] [消息6] [消息7] [消息8]
   ↑ 新生成的                                      ↑ 原封不动保留
```

### 5.2 边界安全——不拆散 ToolUse/ToolResult

这是整个系统中最精妙的防御性编程之一。问题场景：

```
消息4: Assistant —— 包含 ToolUse(name="Bash", input="cargo test")
消息5: Tool —— ToolResult(output="test passed")  ← 这条刚好在保留边界上
```

如果简单按位置切割，消息 5（ToolResult）被保留，但消息 4（包含配对的 ToolUse）被压缩了。在 OpenAI 兼容的 API 中，一个 `tool` 角色的消息**必须**紧跟在包含 `tool_calls` 的 `assistant` 消息之后。否则 API 返回 **400 Bad Request**。

解决方案是一个**回退循环**：

```rust:rust/crates/runtime/src/compact.rs
let keep_from = {
    let mut k = raw_keep_from;
    loop {
        if k == 0 || k <= compacted_prefix_len {
            break;
        }
        let first_preserved = &session.messages[k];
        let starts_with_tool_result = first_preserved
            .blocks
            .first()
            .is_some_and(|b| matches!(b, ContentBlock::ToolResult { .. }));
        if !starts_with_tool_result {
            break;
        }

        // 检查前一条消息是否有 ToolUse
        let preceding = &session.messages[k - 1];
        let preceding_has_tool_use = preceding
            .blocks
            .iter()
            .any(|b| matches!(b, ContentBlock::ToolUse { .. }));

        if preceding_has_tool_use {
            // 配对完整——再往前退一步，把 assistant 消息也保留
            k = k.saturating_sub(1);
            break;
        }
        // 前一条没有 ToolUse——已经是一个孤立的 ToolResult，继续回退
        k = k.saturating_sub(1);
    }
    k
};
```

**读懂这段 Rust**

- `loop { ... break; }` 是无限循环，靠 `break` 退出。
- `.is_some_and(|b| ...)` 检查 `Option` 是否为 `Some` 且值满足条件。
- `.any(|b| ...)` 检查迭代器中是否有任何一个元素满足条件。

回退逻辑总结：

| 场景 | 动作 |
|---|---|
| 保留边界上的第一条消息不是 ToolResult | 安全，直接切割 |
| 是 ToolResult，前一条有 ToolUse | 完整配对，退一步把 assistant 也保留 |
| 是 ToolResult，前一条没有 ToolUse | 孤立结果，继续往前找配对的 ToolUse |

这确保了压缩后的消息列表中，**永远不会出现一个没有前驱 ToolUse 的 ToolResult**。

### 5.3 摘要生成——`summarize_messages` 算法

这是压缩的核心：把一组消息提炼为结构化的摘要文本。

```rust:rust/crates/runtime/src/compact.rs
fn summarize_messages(messages: &[ConversationMessage]) -> String {
    // 1. 统计各角色的消息数
    let user_messages = messages.iter()
        .filter(|m| m.role == MessageRole::User).count();
    let assistant_messages = messages.iter()
        .filter(|m| m.role == MessageRole::Assistant).count();
    let tool_messages = messages.iter()
        .filter(|m| m.role == MessageRole::Tool).count();

    // 2. 提取所有被使用的工具名（去重）
    let mut tool_names = messages.iter()
        .flat_map(|m| m.blocks.iter())
        .filter_map(|block| match block {
            ContentBlock::ToolUse { name, .. } => Some(name.as_str()),
            ContentBlock::ToolResult { tool_name, .. } => Some(tool_name.as_str()),
            ContentBlock::Text { .. } => None,
        })
        .collect::<Vec<_>>();
    tool_names.sort_unstable();
    tool_names.dedup();

    // 3. 构建摘要（七段式结构）
    let mut lines = vec![
        "<summary>".to_string(),
        "Conversation summary:".to_string(),
        format!(
            "- Scope: {} earlier messages compacted (user={}, assistant={}, tool={}).",
            messages.len(), user_messages, assistant_messages, tool_messages
        ),
    ];

    if !tool_names.is_empty() {
        lines.push(format!("- Tools mentioned: {}.", tool_names.join(", ")));
    }

    // 4. 最近 3 条用户请求
    let recent_user_requests = collect_recent_role_summaries(messages, MessageRole::User, 3);
    if !recent_user_requests.is_empty() {
        lines.push("- Recent user requests:".to_string());
        lines.extend(recent_user_requests.into_iter()
            .map(|r| format!("  - {r}")));
    }

    // 5. 推断待办工作
    let pending_work = infer_pending_work(messages);
    if !pending_work.is_empty() {
        lines.push("- Pending work:".to_string());
        lines.extend(pending_work.into_iter().map(|item| format!("  - {item}")));
    }

    // 6. 提取关键文件
    let key_files = collect_key_files(messages);
    if !key_files.is_empty() {
        lines.push(format!("- Key files referenced: {}.", key_files.join(", ")));
    }

    // 7. 推断当前工作
    if let Some(current_work) = infer_current_work(messages) {
        lines.push(format!("- Current work: {current_work}"));
    }

    // 8. 时间线
    lines.push("- Key timeline:".to_string());
    for message in messages {
        let role = match message.role {
            MessageRole::System => "system",
            MessageRole::User => "user",
            MessageRole::Assistant => "assistant",
            MessageRole::Tool => "tool",
        };
        let content = message.blocks.iter()
            .map(summarize_block)
            .collect::<Vec<_>>()
            .join(" | ");
        lines.push(format!("  - {role}: {content}"));
    }
    lines.push("</summary>".to_string());
    lines.join("\n")
}
```

**读懂这段 Rust**

- `.flat_map(|m| m.blocks.iter())` 是两层嵌套展开——先遍历消息，再遍历每条消息的内容块。
- `.sort_unstable()` 是不保证稳定排序的快速排序——比稳定排序快，适合简单的字符串排序。
- `.dedup()` 去除相邻的重复元素——所以要先排序再去重。
- `if let Some(x) = expr` 是模式匹配的"只关心成功"写法。

生成的摘要有**七段式结构**：

```
<summary>
Conversation summary:
- Scope: 12 earlier messages compacted (user=4, assistant=4, tool=4).
- Tools mentioned: Bash, Read, Edit.
- Recent user requests:
  - Fix the compilation error in runtime module
  - Add tests for compact_session
  - Run cargo clippy
- Pending work:
  - Next: add integration tests and follow up on remaining CLI polish.
- Key files referenced: rust/crates/runtime/src/compact.rs, rust/crates/runtime/src/session.rs.
- Current work: Working on regression coverage for compaction.
- Key timeline:
  - user: Fix the compilation error in runtime module
  - assistant: I will inspect the compact flow. | tool_use Read({"file_path":"..."})
  - tool: tool_result Read: ...
  - assistant: The issue is on line 42...
  ...
</summary>
```

每一段都有独立的信息价值：

| 段落 | 信息价值 | 如何提取 |
|---|---|---|
| **Scope** | 被压缩消息的规模和组成 | 直接统计角色计数 |
| **Tools mentioned** | 使用了哪些能力 | 遍历所有 ToolUse/ToolResult 块 |
| **Recent user requests** | 用户最近的意图 | 取最后 3 条 User 消息 |
| **Pending work** | 未完成的任务 | 关键词匹配（todo/next/pending/follow up/remaining） |
| **Key files** | 涉及的关键文件路径 | 正则提取带路径扩展名的文件引用 |
| **Current work** | 当前正在做什么 | 取最后一条非空文本 |
| **Key timeline** | 逐条消息的时间线 | 每条消息浓缩为一行（截断到 160 字符） |

### 5.3.1 端到端示例：摘要是如何一步步生成的

> **关键认知：整个 `summarize_messages` 没有调用 LLM。**
> 它是纯函数——输入一组消息，输出一段拼接好的文本。七个字段全部由**计数 + 取值 + 关键词匹配 + 截断 + 字符串拼接**生成，没有任何一步需要"理解"对话内容。

假设被压缩的 8 条消息如下：

```text
messages[0]: User        —— "Fix the compilation error in rust/crates/runtime/src/compact.rs"
messages[1]: Assistant   —— "I will inspect the compact flow." + ToolUse(Read, {file_path: "compact.rs"})
messages[2]: Tool        —— ToolResult(Read, "Line 42: missing semicolon...")
messages[3]: Assistant   —— "Fixed on line 42. Next: add tests and follow up on remaining CLI polish."
messages[4]: User        —— "Please add regression tests for compaction."
messages[5]: Assistant   —— "Working on regression coverage now."
messages[6]: User        —— "Run cargo clippy"
messages[7]: Assistant   —— "Clippy check passed."
```

下面逐字段追踪代码是如何把它们变成摘要的。

#### 字段 1：Scope——直接数数

```rust
let user_messages = messages.iter()
    .filter(|m| m.role == MessageRole::User).count();       // → 3
let assistant_messages = messages.iter()
    .filter(|m| m.role == MessageRole::Assistant).count();  // → 3
let tool_messages = messages.iter()
    .filter(|m| m.role == MessageRole::Tool).count();       // → 1
// 注：messages[2] 的 role 是 Tool，所以 tool=1 而不是 3
```

输出：
```
- Scope: 8 earlier messages compacted (user=3, assistant=3, tool=1).
```

> 就是数数，没有理解语义。

#### 字段 2：Tools mentioned——遍历所有 block 提取 name 字段

```rust
let mut tool_names = messages.iter()
    .flat_map(|m| m.blocks.iter())        // 展开所有消息的所有内容块
    .filter_map(|block| match block {
        ContentBlock::ToolUse { name, .. } => Some(name.as_str()),
        ContentBlock::ToolResult { tool_name, .. } => Some(tool_name.as_str()),
        ContentBlock::Text { .. } => None,  // 文本块跳过
    })
    .collect::<Vec<_>>();                 // ["Read", "Read"]
tool_names.sort_unstable();
tool_names.dedup();                       // ["Read"]
```

输出：
```
- Tools mentioned: Read.
```

> 就是从数据结构的 `name` 字段取值，去重。

#### 字段 3：Recent user requests——取最后 3 条 User 消息的文本

`collect_recent_role_summaries` 做的事：从后往前遍历 → 找到 User 角色 → 取第一个 Text block → 截断到 160 字符 → 收集 3 条 → 恢复时间顺序。

对于上面的例子，3 条 User 文本分别是：
1. `"Fix the compilation error in rust/crates/runtime/src/compact.rs"`
2. `"Please add regression tests for compaction."`
3. `"Run cargo clippy"`

输出：
```
- Recent user requests:
  - Fix the compilation error in rust/crates/runtime/src/compact.rs
  - Please add regression tests for compaction.
  - Run cargo clippy
```

> 就是倒序取文本，截断。

#### 字段 4：Pending work——关键词匹配

```rust
fn infer_pending_work(messages: &[ConversationMessage]) -> Vec<String> {
    messages.iter().rev()
        .filter_map(first_text_block)
        .filter(|text| {
            let lowered = text.to_ascii_lowercase();
            lowered.contains("todo")
                || lowered.contains("next")
                || lowered.contains("pending")
                || lowered.contains("follow up")
                || lowered.contains("remaining")
        })
        .take(3)
        // ...
}
```

遍历所有消息，如果文本包含 `todo`、`next`、`pending`、`follow up`、`remaining` 中的任意一个，就认为这条消息包含"待办信息"。

上例中 messages[3] 的文本是 `"Fixed on line 42. Next: add tests and follow up on remaining CLI polish."`——同时匹配 `next`、`follow up`、`remaining` 三个关键词。

输出：
```
- Pending work:
  - Fixed on line 42. Next: add tests and follow up on remaining CLI polish.
```

> 就是字符串 `contains` 检查，没有任何语义理解。

#### 字段 5：Key files——分词 + 后缀检查

`extract_file_candidates` 的逻辑：把所有消息的所有 block 文本按空白拆词 → 去标点 → 保留同时包含 `/` 且扩展名是 `rs/ts/tsx/js/json/md` 之一的 token。

messages[0] 中有 `rust/crates/runtime/src/compact.rs`——包含 `/`，扩展名 `.rs`——命中。messages[1] 的 ToolUse input 中也有同样的路径——但去重后只出现一次。

输出：
```
- Key files referenced: rust/crates/runtime/src/compact.rs.
```

> 就是分词 + 后缀检查。

#### 字段 6：Current work——取最后一条非空文本

```rust
fn infer_current_work(messages: &[ConversationMessage]) -> Option<String> {
    messages.iter().rev()
        .filter_map(first_text_block)
        .find(|text| !text.trim().is_empty())
        .map(|text| truncate_summary(text, 200))
}
```

从后往前找第一条有文本内容的消息，截断到 200 字符。messages[7] 的文本是 `"Clippy check passed."` → 这就是 "current work"。

输出：
```
- Current work: Clippy check passed.
```

> 甚至连"推断"都算不上，就是 `last_non_empty_text`。

#### 字段 7：Timeline——逐条拼接，每条截断到 160 字符

```rust
for message in messages {
    let role = /* "user" / "assistant" / "tool" */;
    let content = message.blocks.iter()
        .map(summarize_block)    // 每个块变成文本 或 "tool_use Read({input})" 或 "tool_result Read: output"
        .collect::<Vec<_>>()
        .join(" | ");           // 多个块用 | 连接
    lines.push(format!("  - {role}: {content}"));
}
```

每条消息被浓缩为一行，多条 block 用 ` | ` 分隔：

```
- Key timeline:
  - user: Fix the compilation error in rust/crates/runtime/src/compact.rs
  - assistant: I will inspect the compact flow. | tool_use Read({"file_path":"compact.rs"})
  - tool: tool_result Read: Line 42: missing semicolon...
  - assistant: Fixed on line 42. Next: add tests and follow up on remaining CLI polish.
  - user: Please add regression tests for compaction.
  - assistant: Working on regression coverage now.
  - user: Run cargo clippy
  - assistant: Clippy check passed.
```

> 就是格式化 + 截断。

#### 最终拼接

所有字段拼在一起，外面套上 `<summary>...</summary>` 标签：

```
<summary>
Conversation summary:
- Scope: 8 earlier messages compacted (user=3, assistant=3, tool=1).
- Tools mentioned: Read.
- Recent user requests:
  - Fix the compilation error in rust/crates/runtime/src/compact.rs
  - Please add regression tests for compaction.
  - Run cargo clippy
- Pending work:
  - Fixed on line 42. Next: add tests and follow up on remaining CLI polish.
- Key files referenced: rust/crates/runtime/src/compact.rs.
- Current work: Clippy check passed.
- Key timeline:
  - user: Fix the compilation error in rust/crates/runtime/src/compact.rs
  - assistant: I will inspect the compact flow. | tool_use Read({"file_path":"compact.rs"})
  - tool: tool_result Read: Line 42: missing semicolon...
  - assistant: Fixed on line 42. Next: add tests and follow up on remaining CLI polish.
  - user: Please add regression tests for compaction.
  - assistant: Working on regression coverage now.
  - user: Run cargo clippy
  - assistant: Clippy check passed.
</summary>
```

#### 汇总：每个字段背后的操作

| 摘要字段 | 生成方式 | 涉及 LLM？ |
|---|---|---|
| Scope | 数数（按 role 计数） | ❌ |
| Tools mentioned | 从结构体 `name` 字段取值 + 去重 | ❌ |
| Recent user requests | 倒序取 User 文本 + 截断 160 字符 | ❌ |
| Pending work | `contains("todo"/"next"/...)` 关键词匹配 | ❌ |
| Key files | 分词 + 检查是否包含 `/` 且后缀在白名单中 | ❌ |
| Current work | 取最后一条非空文本 + 截断 200 字符 | ❌ |
| Timeline | 逐条 `format!` + `summarize_block` 截断 160 字符 | ❌ |

它是工程化的**模板填充**，而不是 LLM 的语义压缩。这意味着：
- **零延迟**——不需要等一次 API 往返，压缩是毫秒级完成
- **零成本**——不额外消耗 token，不会在预算紧张时"越压缩越花钱"
- **确定性**——同样的输入永远产出同样的摘要，不会出现 LLM 幻觉

LLM 第一次"看到"压缩结果，是在**下一个 turn**——它从 System 消息中读到续接摘要，然后继续工作。LLM 只是压缩结果的**消费者**，不是压缩过程的**参与者**。

### 5.4 关键文件提取——一个精巧的启发式

```rust:rust/crates/runtime/src/compact.rs
fn extract_file_candidates(content: &str) -> Vec<String> {
    content
        .split_whitespace()
        .filter_map(|token| {
            let candidate = token.trim_matches(|c: char| {
                matches!(c, ',' | '.' | ':' | ';' | ')' | '(' | '"' | '\'' | '`')
            });
            if candidate.contains('/') && has_interesting_extension(candidate) {
                Some(candidate.to_string())
            } else {
                None
            }
        })
        .collect()
}

fn has_interesting_extension(candidate: &str) -> bool {
    std::path::Path::new(candidate)
        .extension()
        .and_then(|ext| ext.to_str())
        .is_some_and(|ext| {
            ["rs", "ts", "tsx", "js", "json", "md"]
                .iter()
                .any(|expected| ext.eq_ignore_ascii_case(expected))
        })
}
```

这个启发式的逻辑是：
1. 把内容按空白字符拆成 token
2. 去掉标点符号
3. 保留同时满足两个条件的 token：
   - 包含 `/`（有路径分隔符，说明是个路径）
   - 扩展名是 `.rs`、`.ts`、`.tsx`、`.js`、`.json`、`.md` 之一

最终去重后最多保留 8 个文件路径。这是一个典型的"宁可漏掉一些，也不要误报"的设计——只关注最常见的代码文件类型。

### 5.5 待办推断——关键词匹配

```rust:rust/crates/runtime/src/compact.rs
fn infer_pending_work(messages: &[ConversationMessage]) -> Vec<String> {
    messages
        .iter()
        .rev()                    // 从最新消息开始看
        .filter_map(first_text_block)
        .filter(|text| {
            let lowered = text.to_ascii_lowercase();
            lowered.contains("todo")
                || lowered.contains("next")
                || lowered.contains("pending")
                || lowered.contains("follow up")
                || lowered.contains("remaining")
        })
        .take(3)                  // 最多 3 条
        .map(|text| truncate_summary(text, 160))
        .collect::<Vec<_>>()
        .into_iter()
        .rev()                    // 恢复时间顺序
        .collect()
}
```

从最新到最旧遍历消息，找出包含"待办"关键词的文本块，最多取 3 条，然后恢复时间顺序。这些信息被保留在摘要中，让 Agent 在压缩后仍然"记得"自己还有什么没做完。

### 5.6 每条消息的浓缩——`summarize_block`

时间线中的每条消息被浓缩为一行，每个内容块截断到 160 字符：

```rust:rust/crates/runtime/src/compact.rs
fn summarize_block(block: &ContentBlock) -> String {
    let raw = match block {
        ContentBlock::Text { text } => text.clone(),
        ContentBlock::ToolUse { name, input, .. } => format!("tool_use {name}({input})"),
        ContentBlock::ToolResult { tool_name, output, is_error, .. } => format!(
            "tool_result {tool_name}: {}{output}",
            if *is_error { "error " } else { "" }
        ),
    };
    truncate_summary(&raw, 160)
}
```

`truncate_summary` 是一个简单的截断函数：如果超过 `max_chars`，截断并追加省略号 `…`。

---

## 第六部分：续接——如何让模型"无缝接续"？

压缩后，session 的第一条消息变成一条特殊的 System 消息，它告诉模型"你是从之前对话延续下来的"：

```rust:rust/crates/runtime/src/compact.rs
const COMPACT_CONTINUATION_PREAMBLE: &str =
    "This session is being continued from a previous conversation that ran out of context. \
     The summary below covers the earlier portion of the conversation.\n\n";
const COMPACT_RECENT_MESSAGES_NOTE: &str =
    "Recent messages are preserved verbatim.";
const COMPACT_DIRECT_RESUME_INSTRUCTION: &str =
    "Continue the conversation from where it left off without asking the user any further questions. \
     Resume directly — do not acknowledge the summary, do not recap what was happening, \
     and do not preface with continuation text.";
```

```rust:rust/crates/runtime/src/compact.rs
pub fn get_compact_continuation_message(
    summary: &str,
    suppress_follow_up_questions: bool,
    recent_messages_preserved: bool,
) -> String {
    let mut base = format!(
        "{COMPACT_CONTINUATION_PREAMBLE}{}",
        format_compact_summary(summary)
    );

    if recent_messages_preserved {
        base.push_str("\n\n");
        base.push_str(COMPACT_RECENT_MESSAGES_NOTE);
    }

    if suppress_follow_up_questions {
        base.push('\n');
        base.push_str(COMPACT_DIRECT_RESUME_INSTRUCTION);
    }

    base
}
```

这条 System 消息有三层指令：

| 层级 | 内容 | 目的 |
|---|---|---|
| **情境说明** | "This session is being continued..." | 告诉模型发生了压缩 |
| **上下文补充** | "Recent messages are preserved verbatim." | 提醒模型近期消息是完整的 |
| **行为约束** | "Resume directly — do not acknowledge the summary..." | 防止模型说"好的，我了解了之前的对话..."这类废话 |

第三层特别重要：没有它，模型在压缩后的第一个回复很可能是"根据之前的对话摘要，我了解到..."这样的确认性废话。对于 Agent 来说，这是浪费 token 且打断工作流。

---

## 第七部分：合并——二次压缩的策略

Agent 的对话可能持续很长时间，触发不止一次压缩。第二次压缩时，session 中已经有一条摘要 System 消息了。`merge_compact_summaries` 负责把旧摘要和新摘要合并：

```rust:rust/crates/runtime/src/compact.rs
fn merge_compact_summaries(existing_summary: Option<&str>, new_summary: &str) -> String {
    let Some(existing_summary) = existing_summary else {
        return new_summary.to_string();  // 第一次压缩，直接用新摘要
    };

    // 从旧摘要中提取高亮信息（去掉时间线）
    let previous_highlights = extract_summary_highlights(existing_summary);
    let new_formatted_summary = format_compact_summary(new_summary);
    let new_highlights = extract_summary_highlights(&new_formatted_summary);
    let new_timeline = extract_summary_timeline(&new_formatted_summary);

    let mut lines = vec![
        "<summary>".to_string(),
        "Conversation summary:".to_string(),
    ];

    // 旧摘要的高亮 → "Previously compacted context"
    if !previous_highlights.is_empty() {
        lines.push("- Previously compacted context:".to_string());
        lines.extend(previous_highlights.into_iter().map(|line| format!("  {line}")));
    }

    // 新摘要的高亮 → "Newly compacted context"
    if !new_highlights.is_empty() {
        lines.push("- Newly compacted context:".to_string());
        lines.extend(new_highlights.into_iter().map(|line| format!("  {line}")));
    }

    // 新摘要的时间线 → 保留（旧时间线被丢弃，只保留最新的）
    if !new_timeline.is_empty() {
        lines.push("- Key timeline:".to_string());
        lines.extend(new_timeline.into_iter().map(|line| format!("  {line}")));
    }

    lines.push("</summary>".to_string());
    lines.join("\n")
}
```

合并策略的关键决策：

| 决策 | 理由 |
|---|---|
| 旧摘要的高亮被标记为 "Previously compacted context" | 模型能区分历史和近期 |
| 新摘要的高亮被标记为 "Newly compacted context" | 最新的上下文优先级更高 |
| **旧时间线被丢弃** | 时间线是逐条消息的，多次压缩后体积会膨胀。只保留最新一轮的时间线 |
| 旧摘要中的 Scope/Tools/Files 被保留在 "highlights" 中 | 这些是高频引用的信息，丢失代价高 |

一个二次压缩后的摘要看起来像这样：

```
<summary>
Conversation summary:
- Previously compacted context:
  - Scope: 6 earlier messages compacted (user=2, assistant=2, tool=2).
  - Current work: Investigate compact flow.
  - Key files referenced: rust/crates/runtime/src/compact.rs.
- Newly compacted context:
  - Scope: 4 earlier messages compacted (user=2, assistant=2).
  - Current work: Working on regression coverage.
  - Key files referenced: rust/crates/runtime/src/session.rs.
- Key timeline:
  - user: Please add regression tests for compaction.
  - assistant: Working on regression coverage now.
</summary>
```

---

## 第八部分：精炼——`SummaryCompressionBudget` 的行级选择器

到目前为止，我们看到的是"把消息压缩成摘要"。但摘要本身也可能很长——特别是时间线部分。`summary_compression.rs` 提供了**二级压缩**：在给定的预算内，保留最重要的行。

### 8.1 Budget 配置

```rust:rust/crates/runtime/src/summary_compression.rs
const DEFAULT_MAX_CHARS: usize = 1_200;
const DEFAULT_MAX_LINES: usize = 24;
const DEFAULT_MAX_LINE_CHARS: usize = 160;

pub struct SummaryCompressionBudget {
    pub max_chars: usize,       // 总字符数上限
    pub max_lines: usize,       // 行数上限
    pub max_line_chars: usize,  // 单行字符数上限
}
```

三个维度共同约束：

| 维度 | 默认值 | 含义 |
|---|---|---|
| `max_chars` | 1,200 | 摘要总长度不超过 1,200 字符 |
| `max_lines` | 24 | 摘要不超过 24 行 |
| `max_line_chars` | 160 | 每行不超过 160 字符（超出截断+`…`） |

### 8.2 四级优先级选择器

核心算法是 `select_line_indexes`——一个按优先级逐步选择行的贪心算法：

```rust:rust/crates/runtime/src/summary_compression.rs
fn select_line_indexes(lines: &[String], budget: SummaryCompressionBudget) -> Vec<usize> {
    let mut selected = BTreeSet::<usize>::new();

    for priority in 0..=3 {  // 从最高优先级开始
        for (index, line) in lines.iter().enumerate() {
            if selected.contains(&index) || line_priority(line) != priority {
                continue;
            }

            // 试探：加入这一行后，是否还满足预算？
            let candidate = selected.iter()
                .map(|i| lines[*i].as_str())
                .chain(std::iter::once(line.as_str()))
                .collect::<Vec<_>>();

            if candidate.len() > budget.max_lines { continue; }
            if joined_char_count(&candidate) > budget.max_chars { continue; }

            selected.insert(index);
        }
    }

    selected.into_iter().collect()
}
```

优先级定义：

```rust:rust/crates/runtime/src/summary_compression.rs
fn line_priority(line: &str) -> usize {
    if line == "Summary:" || line == "Conversation summary:" || is_core_detail(line) {
        0    // 最高优先级：核心信息
    } else if is_section_header(line) {
        1    // 次高：段落标题
    } else if line.starts_with("- ") || line.starts_with("  - ") {
        2    // 普通：列表项
    } else {
        3    // 最低：其他内容
    }
}

fn is_core_detail(line: &str) -> bool {
    ["- Scope:", "- Current work:", "- Pending work:",
     "- Key files referenced:", "- Tools mentioned:",
     "- Recent user requests:", "- Previously compacted context:",
     "- Newly compacted context:"]
        .iter()
        .any(|prefix| line.starts_with(prefix))
}
```

**四级优先级的工作方式**：

```
第一轮（Priority 0）：只选核心行
  ✓ "Conversation summary:"
  ✓ "- Scope: 12 earlier messages compacted..."
  ✓ "- Current work: Working on..."
  ✓ "- Key files referenced: ..."

第二轮（Priority 1）：选段落标题
  ✓ "- Key timeline:"
  ✓ "- Recent user requests:"

第三轮（Priority 2）：选列表项（直到预算耗尽）
  ✓ "  - user: asked for..."
  ✓ "  - assistant: analyzed..."
  ✗ "  - tool: executed..."  ← 预算不够了，跳过

第四轮（Priority 3）：其他内容
  （通常已经被前面的轮次占满了）
```

### 8.3 去重和规范化

在选择之前，摘要文本会先经过规范化：

```rust:rust/crates/runtime/src/summary_compression.rs
fn normalize_lines(summary: &str, max_line_chars: usize) -> NormalizedSummary {
    let mut seen = BTreeSet::new();
    let mut lines = Vec::new();
    let mut removed_duplicate_lines = 0;

    for raw_line in summary.lines() {
        let normalized = collapse_inline_whitespace(raw_line);
        if normalized.is_empty() { continue; }

        let truncated = truncate_line(&normalized, max_line_chars);
        let dedupe_key = dedupe_key(&truncated);

        // 大小写不敏感去重
        if !seen.insert(dedupe_key) {
            removed_duplicate_lines += 1;
            continue;
        }
        lines.push(truncated);
    }

    NormalizedSummary { lines, removed_duplicate_lines }
}
```

规范化做了三件事：

| 步骤 | 操作 | 效果 |
|---|---|---|
| 折叠空白 | `"  hello   world  "` → `"hello world"` | 消除格式差异 |
| 截断长行 | 超过 `max_line_chars` 的行截断+`…` | 控制单行长度 |
| 去重 | 大小写不敏感比较 | 去掉重复信息 |

### 8.4 省略提示

当有行被省略时，压缩结果会添加一条省略提示：

```rust:rust/crates/runtime/src/summary_compression.rs
fn omission_notice(omitted_lines: usize) -> String {
    format!("- … {omitted_lines} additional line(s) omitted.")
}
```

这让模型知道信息被压缩了——它不会误以为时间线是完整的。

---

## 第九部分：健康探针——压缩后的安全网

压缩修改了 session 的消息列表，这可能引入不一致。为了捕获问题，claw-code 在每次 turn 开始时检查 session 是否曾被压缩，如果是，先运行一个**健康探针**：

```rust:rust/crates/runtime/src/conversation.rs
// run_turn() 开头
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

```rust:rust/crates/runtime/src/conversation.rs
fn run_session_health_probe(&mut self) -> Result<(), String> {
    // 基本完整性检查
    if self.session.messages.is_empty() && self.session.compaction.is_some() {
        return Ok(());  // 刚压缩完，消息列表为空是正常的
    }

    // 非破坏性探测：搜索一个不会匹配任何内容的模式
    let probe_input = r#"{"pattern": "*.health-check-probe-"}"#;
    match self.tool_executor.execute("glob_search", probe_input) {
        Ok(_) => Ok(()),
        Err(e) => Err(format!("Tool executor probe failed: {e}")),
    }
}
```

**读懂这段 Rust**

- `r#"..."#` 是原始字符串——反斜杠不需要转义。
- `Ok(())` 中的 `()` 是单元类型，类似于 Python 的 `None`，表示"没有返回值，但成功了"。

健康探针做了两件事：

| 检查 | 意义 |
|---|---|
| 空消息 + 有压缩记录 | 正常：刚压缩完，摘要 System 消息可能被后续操作移除 |
| 工具执行器可用性 | 用一个无害的 glob 搜索确认工具系统没有被压缩破坏 |

如果探针失败，返回一个清晰的错误消息，建议用户用 `/session new` 重新开始。

---

## 第十部分：SessionCompaction 的生命周期

最后，让我们追踪压缩元数据在 session 中的完整生命周期：

```rust:rust/crates/runtime/src/session.rs
pub struct SessionCompaction {
    pub count: u32,                  // 被压缩了多少次
    pub removed_message_count: usize, // 最近一次移除了多少条消息
    pub summary: String,             // 最新的摘要文本
}
```

```rust:rust/crates/runtime/src/session.rs
pub fn record_compaction(&mut self, summary: impl Into<String>, removed_message_count: usize) {
    self.touch();
    let count = self.compaction.as_ref().map_or(1, |value| value.count + 1);
    self.compaction = Some(SessionCompaction {
        count,
        removed_message_count,
        summary: summary.into(),
    });
}
```

**读懂这段 Rust**

- `impl Into<String>` 表示"任何能转成 String 的类型"——`&str`、`String` 都行。
- `.map_or(1, |value| value.count + 1)` 如果 `compaction` 已存在，计数加 1；否则从 1 开始。

`SessionCompaction` 在 session 中的角色：

```
Session 创建
  → compaction: None

第一次压缩
  → compaction: Some { count: 1, removed_message_count: 8, summary: "..." }
  → messages[0] 是 System 消息（摘要续接）

第二次压缩
  → compaction: Some { count: 2, removed_message_count: 4, summary: "合并摘要..." }
  → messages[0] 仍然是 System 消息（合并后的摘要续接）

Session 持久化到 JSONL
  → compaction 字段被序列化
  → 下次加载时恢复压缩状态

Turn 开始 → run_session_health_probe()
  → 检查 compaction.is_some() → 是 → 运行探针
```

---

## 总结：上下文压缩的设计哲学

回到本章开头的四个设计目标，看看 claw-code 是如何实现的：

| 目标 | 实现方式 |
|---|---|
| **自动触发** | `maybe_auto_compact` 在每次 turn 结束后检查累计 token；阈值可通过环境变量配置 |
| **信息保留** | 七段式摘要结构：Scope → Tools → Requests → Pending → Files → Current → Timeline |
| **边界安全** | ToolUse/ToolResult 回退循环确保不产生孤立消息；健康探针验证压缩后的一致性 |
| **可组合** | `merge_compact_summaries` 区分 "Previously compacted" 和 "Newly compacted"；旧时间线被丢弃防止膨胀 |

更深层的设计哲学：

1. **渐进退化而非突然失败**：不是"超限就崩溃"，而是"超限就压缩"。用户体验是平滑的。
2. **优先级驱动的信息保留**：从核心信息（Scope、Current work）到细节（Timeline 中的每条消息），按重要性逐步填充。
3. **防御性编程**：边界安全、健康探针、省略提示——每一步都假设上游可能出错，并优雅地处理。
4. **可观测性**：`CompactionResult` 携带丰富的元数据（removed_message_count、formatted_summary），上层代码可以向用户展示"刚刚压缩了 N 条消息"。

这套机制让 Agent 能够在一个有限的 token 窗口中"无限"地工作下去——每一次压缩都是对历史的提炼，而不是对记忆的删除。
