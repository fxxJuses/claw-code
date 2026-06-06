# 第06章 会话管理：Agent 的状态机与生命周期

> 本文是 claw-code 学习系列的第六篇，聚焦于 Agent 系统中最容易被低估、却最关键的工程问题——**会话管理（Session Management）**。
> 前几章我们讲过 prompt 如何驱动 Agent、工具如何执行、流式事件如何聚合成消息。但这一切都建立在一个前提之上：**Agent 有状态，状态需要被可靠地存储、恢复、压缩、分叉。**
> **本文面向不懂 Rust 的读者**：每段代码后都有「读懂这段 Rust」小节，只解释理解会话管理所必需的语法。

读完本章，你应该能回答六件事：

1. Session 的 12 个字段各自承担什么职责？为什么 `messages` 是"唯一真相源"？
2. JSONL 持久化的追加写入 + 原子写 + 日志轮转是如何协同工作的？
3. SessionStore 如何通过 workspace fingerprint 实现多工作区隔离？
4. 上下文压缩的三阶段（触发 → 摘要 → 边界安全）分别做了什么？
5. 会话分叉的不可变语义如何保证原会话不被污染？
6. UsageTracker 如何从 session 恢复，使得跨重启的用量统计不丢失？

---

## 与前几章的关系

| 主题 | 第05章扩展阅读已覆盖 | 本章新增深入 |
|---|---|---|
| Session 结构体 | 字段列表、messages 是真相源 | 逐字段生命周期分析、12 字段分类 |
| JSONL 持久化 | 格式概述、追加 vs 全量 | 四种 record type 详解、原子写实现、日志轮转策略 |
| SessionStore | 未覆盖 | workspace fingerprint、多工作区隔离、引用别名系统 |
| 上下文压缩 | 触发条件、compact_session 边界安全 | 摘要生成算法、summary compression budget、ToolUse/ToolResult 边界修复 |
| 会话分叉 | fork 不可变语义 | fork 在 SessionStore 中的持久化、lineage 追踪 |
| UsageTracker | 基本结构 | 从 session 恢复的完整机制、四维度计费模型 |
| CLI 命令 | 未覆盖 | `/session`、`/compact`、`/resume`、`/clear` 等命令的会话管理语义 |
| 测试 | 21 个 conversation 测试 | session 模块 14 个测试 + session_control 模块 11 个测试 |

---

## Rust 语法速查（本章新增）

| 符号 / 写法 | 含义 | 本章出现的场景 |
|---|---|---|
| `AtomicU64` | 原子 64 位整数（线程安全的计数器） | `SESSION_ID_COUNTER`、`LAST_TIMESTAMP_MS` |
| `Ordering::SeqCst` | 最强的内存排序保证 | `compare_exchange` 中的原子操作 |
| `PathBuf` | 拥有所有权的路径类型（类似 Python 的 pathlib.Path） | `workspace_root`、`persistence.path` |
| `impl AsRef<Path>` | "任何能被当作 Path 引用的类型" | `from_cwd(cwd: impl AsRef<Path>)` |
| `fs::rename(a, b)` | 原子重命名（在 POSIX 上是原子操作） | `write_atomic` |
| `saturating_add(n)` | 加法溢出时停在最大值而不是 panic | `current_time_millis` |
| `fetch_add(n, ordering)` | 原子地"取当前值，然后加 n" | `generate_session_id` |

---

## 第一部分：Session——Agent 的"唯一真相源"

### 1.1 Session 的 12 个字段——三层分类

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
    pub prompt_history: Vec<SessionPromptEntry>,
    pub last_health_check_ms: Option<u64>,
    pub model: Option<String>,
    persistence: Option<SessionPersistence>,
}
```

**读懂这段 Rust**

- `u32` 是 32 位无符号整数，`u64` 是 64 位无符号整数。时间戳用 `u64` 是因为毫秒级的 Unix 时间戳已经超过了 32 位的范围。
- `Vec<T>` 是动态数组（类似 Python 的 list）。
- `Option<T>` 表示"可能有、可能没有"——`None` 就是空，`Some(value)` 就是有值。
- `pub` 表示公开字段，`persistence` 没有 `pub` 是私有字段——外部不能直接访问。

这 12 个字段可以分为三层：

**第一层：核心对话数据（Agent 的记忆）**

| 字段 | 类型 | 角色 |
|---|---|---|
| `messages` | `Vec<ConversationMessage>` | **唯一真相源**：所有对话消息，包括 user、assistant、tool、system |
| `compaction` | `Option<SessionCompaction>` | 上次压缩的元数据——压缩了几次、移除了多少条、摘要是什么 |
| `prompt_history` | `Vec<SessionPromptEntry>` | 用户输入历史（带时间戳），独立于 messages 以便快速检索 |

**第二层：身份与定位（Agent 的身份证和住址）**

| 字段 | 类型 | 角色 |
|---|---|---|
| `session_id` | `String` | 格式 `"session-{millis}-{counter}"`，全局唯一 |
| `version` | `u32` | 数据格式版本号，当前固定为 1，为未来格式升级预留 |
| `workspace_root` | `Option<PathBuf>` | 绑定到哪个工作目录，防止跨工作区写入冲突 |
| `model` | `Option<String>` | 使用的模型名，恢复时可以知道"上次用的什么模型" |

**第三层：运维与持久化（Agent 的基础设施）**

| 字段 | 类型 | 角色 |
|---|---|---|
| `created_at_ms` / `updated_at_ms` | `u64` | 创建和最后更新时间戳，用于排序和显示 |
| `fork` | `Option<SessionFork>` | 分叉来源：parent_session_id + branch_name |
| `last_health_check_ms` | `Option<u64>` | 上次健康检查时间戳 |
| `persistence` | `Option<SessionPersistence>` | 私有字段：持久化文件路径 |

**为什么 messages 是"唯一真相源"？**

在 claw-code 的设计中，**Agent 的所有状态都在 `session.messages` 里**。没有隐式变量、没有全局状态。想知道 Agent 当前在哪一步、调了哪些工具、哪些被拒绝了——全部都在 messages 里。

这种设计带来了三个巨大的好处：

1. **可调试**：想看 Agent 状态，只看 `session.messages` 就够了。
2. **可恢复**：把 messages 持久化到文件，下次启动从文件恢复，Agent 就能"接着上次的对话继续"。
3. **可重放**：按时间顺序回放 messages，就能复现 Agent 的每一步行为。

**Agent 视角要点**

- 12 个字段不是随意设计的，而是三层层级：**对话数据 > 身份定位 > 运维基础设施**。
- `Option<T>` 的广泛使用意味着"每个字段都有'不存在'的合法状态"——这是一种**防御性设计**，让 Session 在不同阶段（刚创建、恢复中、压缩后）都能保持合法状态。
- `persistence` 是唯一私有字段——外部不需要知道文件存在哪里，只需要通过方法来读写。

---

### 1.2 Session ID——单调递增的时间戳 + 原子计数器

```1057:1061:rust/crates/runtime/src/session.rs
fn generate_session_id() -> String {
    let millis = current_time_millis();
    let counter = SESSION_ID_COUNTER.fetch_add(1, Ordering::Relaxed);
    format!("session-{millis}-{counter}")
}
```

```1033:1055:rust/crates/runtime/src/session.rs
fn current_time_millis() -> u64 {
    let wall_clock = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map(|duration| u64::try_from(duration.as_millis()).unwrap_or(u64::MAX))
        .unwrap_or_default();

    let mut candidate = wall_clock;
    loop {
        let previous = LAST_TIMESTAMP_MS.load(Ordering::Relaxed);
        if candidate <= previous {
            candidate = previous.saturating_add(1);
        }
        match LAST_TIMESTAMP_MS.compare_exchange(
            previous,
            candidate,
            Ordering::SeqCst,
            Ordering::Relaxed,
        ) {
            Ok(_) => return candidate,
            Err(actual) => candidate = actual.saturating_add(1),
        }
    }
}
```

**读懂这段 Rust**

- `AtomicU64` 是一个**原子操作**的 64 位整数——多线程同时读写也不会出错。
- `compare_exchange(expected, new, success_ordering, failure_ordering)` 是一个**CAS（Compare-And-Swap）**操作：如果当前值等于 expected，就替换为 new 并返回 Ok；否则返回 Err(actual)。
- `fetch_add(1, Ordering::Relaxed)`：原子地给计数器加 1 并返回旧值。
- `loop { ... match ... { Ok(_) => return, Err(actual) => ... } }` 是一个**自旋锁**模式——不断重试直到成功。

**这段代码解决了什么问题？**

一个问题：**如果在同一毫秒内创建两个 session，ID 会重复吗？**

答案是**不会**。`current_time_millis()` 通过两层保护保证了单调递增：

1. **CAS 自旋**：用 `LAST_TIMESTAMP_MS` 这个全局原子变量记住"上一次返回的时间戳"。如果当前时间 ≤ 上次，就 `saturating_add(1)` 往前推一毫秒。
2. **原子计数器**：`SESSION_ID_COUNTER` 是第二个保险——即使时间戳被推到了同一个值，计数器也保证不同。

生产环境里这很重要：Agent 可能在一次会话中快速 fork 出多个子会话，或者在测试中紧密循环创建 session。没有单调保证，`latest` 别名就会搞乱"哪个是最新"的判断。

测试 `session_timestamps_are_monotonic_under_tight_loops` 验证了这一点：

```rust
let first = current_time_millis();
let second = current_time_millis();
let third = current_time_millis();
assert!(first < second);
assert!(second < third);
```

三个连续调用，严格递增——不是"大概率递增"，而是**保证**递增。

**Agent 视角要点**

- Session ID 的设计哲学是：**在并发和快速创建场景下仍然保持唯一和有序**。
- `AtomicU64` + `compare_exchange` 是无锁编程的经典模式——比 `Mutex` 更轻量，但需要更仔细地推理。
- 时间戳的单调递增不仅影响 ID，还影响 `updated_at_ms`——这直接决定了 `latest` 别名解析的准确性。

---

### 1.3 push_message——追加写入与回滚

```229:243:rust/crates/runtime/src/session.rs
pub fn push_message(&mut self, message: ConversationMessage) -> Result<(), SessionError> {
    self.touch();
    self.messages.push(message);
    let persist_result = {
        let message_ref = self.messages.last().ok_or_else(|| {
            SessionError::Format("message was just pushed but missing".to_string())
        })?;
        self.append_persisted_message(message_ref)
    };
    if let Err(error) = persist_result {
        self.messages.pop();
        return Err(error);
    }
    Ok(())
}
```

**读懂这段 Rust**

- `self.touch()`：更新 `updated_at_ms` 为当前时间。
- `self.messages.push(message)`：先往内存里加。
- `self.append_persisted_message(message_ref)`：尝试写入磁盘。
- `if let Err(error) = persist_result { self.messages.pop(); ... }`：**如果写盘失败，把刚加的消息撤回来**。

**这段代码的精妙之处：先加后写、失败回滚**

这是一个经典的**乐观更新**模式：

1. **乐观假设**：先把消息加到内存（`self.messages.push`），然后写磁盘。
2. **回滚**：如果写磁盘失败（磁盘满、权限不够），用 `self.messages.pop()` 把刚加的消息撤回。
3. **一致性**：最终结果是——要么内存和磁盘都成功了，要么内存和磁盘都保持原样。

为什么不用"先写后加"（先写磁盘，成功了再加到内存）？因为"先写后加"有一个问题：如果写磁盘成功但加到内存时 panic 了（虽然不太可能），磁盘上就多了一条内存里没有的消息。**乐观更新**让"内存是主、磁盘是从"的语义更清晰。

---

## 第二部分：JSONL 持久化——追加写入的艺术

### 2.1 两种格式：JSONL vs JSON

```213:227:rust/crates/runtime/src/session.rs
pub fn load_from_path(path: impl AsRef<Path>) -> Result<Self, SessionError> {
    let path = path.as_ref();
    let contents = fs::read_to_string(path)?;
    let session = match JsonValue::parse(&contents) {
        Ok(value)
            if value
                .as_object()
                .is_some_and(|object| object.contains_key("messages")) =>
        {
            Self::from_json(&value)?
        }
        Err(_) | Ok(_) => Self::from_jsonl(&contents)?,
    };
    Ok(session.with_persistence_path(path.to_path_buf()))
}
```

**格式自动检测的逻辑：**

1. 尝试解析为 JSON 对象。
2. 如果是对象且包含 `"messages"` 键 → 走 legacy JSON 路径。
3. 否则（解析失败或不是带 messages 的对象）→ 走 JSONL 路径。

这意味着 **claw-code 同时支持两种格式，且加载时自动识别**——这是向后兼容的经典做法。

### 2.2 JSONL 的四种 record type

```435:486:rust/crates/runtime/src/session.rs
match object.get("type").and_then(JsonValue::as_str) ... {
    "session_meta" => { ... },
    "message" => { ... },
    "compaction" => { ... },
    "prompt_history" => { ... },
}
```

一个 JSONL 文件长这样：

```jsonl
{"type":"session_meta","version":1,"session_id":"session-1749123456789-0","created_at_ms":1749123456789,"updated_at_ms":1749123456789,"workspace_root":"/Users/fan/project","model":"claude-sonnet-4-20250514"}
{"type":"prompt_history","timestamp_ms":1749123456800,"text":"fix the bug in main.rs"}
{"type":"message","message":{"role":"user","blocks":[{"type":"text","text":"fix the bug in main.rs"}]}}
{"type":"message","message":{"role":"assistant","blocks":[{"type":"text","text":"Let me read the file."},{"type":"tool_use","id":"tool-1","name":"read_file","input":"{\"path\":\"src/main.rs\"}"}],"usage":{"input_tokens":150,"output_tokens":30,"cache_creation_input_tokens":0,"cache_read_input_tokens":120}}}
{"type":"message","message":{"role":"tool","blocks":[{"type":"tool_result","tool_use_id":"tool-1","tool_name":"read_file","output":"fn main() { ... }","is_error":false}]}}
{"type":"compaction","count":1,"removed_message_count":4,"summary":"..."}
```

四种 record type 各自的职责：

| Record Type | 何时写入 | 包含什么 |
|---|---|---|
| `session_meta` | 全量快照时写入一次 | version、session_id、时间戳、workspace_root、model、fork |
| `message` | 每次 `push_message` 追加 | 一条完整的 ConversationMessage（role + blocks + usage） |
| `compaction` | 压缩后写入 | count（第几次压缩）、removed_message_count、summary |
| `prompt_history` | 每次 `push_prompt_entry` 追加 | timestamp_ms + text |

**为什么需要 `session_meta` 而不是只在文件名里存信息？**

因为文件名可以被重命名、移动。`session_meta` 保证了即使文件名变了，session 的身份信息仍然可以从内容中恢复。

### 2.3 追加写入——不是每次都全量重写

```541:555:rust/crates/runtime/src/session.rs
fn append_persisted_message(&self, message: &ConversationMessage) -> Result<(), SessionError> {
    let Some(path) = self.persistence_path() else {
        return Ok(());
    };

    let needs_bootstrap = !path.exists() || fs::metadata(path)?.len() == 0;
    if needs_bootstrap {
        self.save_to_path(path)?;
        return Ok(());
    }

    let mut file = OpenOptions::new().append(true).open(path)?;
    writeln!(file, "{}", message_record(message).render())?;
    Ok(())
}
```

**这段代码的分支逻辑：**

1. **没有 persistence path** → 直接返回 Ok（纯内存 session，不写文件）。
2. **文件不存在或为空** → 全量快照（`save_to_path`）：先写 meta，再写所有消息。
3. **文件已存在且非空** → 追加一行（`append + writeln`）。

**为什么区分"全量快照"和"追加"？**

全量快照是一个安全网：当文件为空（刚创建、或者被手动清空）时，需要从头写入完整状态。追加则是在正常操作中只写增量——效率高、写入量小。

### 2.4 原子写入——write_atomic

```1063:1071:rust/crates/runtime/src/session.rs
fn write_atomic(path: &Path, contents: &str) -> Result<(), SessionError> {
    if let Some(parent) = path.parent() {
        fs::create_dir_all(parent)?;
    }
    let temp_path = temporary_path_for(path);
    fs::write(&temp_path, contents)?;
    fs::rename(temp_path, path)?;
    Ok(())
}
```

**三步原子写：**

1. `create_dir_all`：确保目录存在。
2. `fs::write(temp_path)`：先写到一个临时文件（`session-xxx.tmp-12345-0.jsonl`）。
3. `fs::rename(temp_path, path)`：把临时文件重命名为目标文件。

**为什么这是"原子"的？**

在 POSIX 系统（Linux、macOS）上，`rename` 是原子操作——它要么完全成功（新文件替换旧文件），要么完全失败（旧文件不受影响）。这意味着：

- 如果 `fs::write` 成功了但 `rename` 前进程崩溃 → 旧文件完好无损。
- 如果 `rename` 成功了 → 新文件完全替换旧文件，不存在"写了一半"的状态。

### 2.5 日志轮转——256KB 阈值 + 最多保留 3 个历史文件

```13:14:rust/crates/runtime/src/session.rs
const ROTATE_AFTER_BYTES: u64 = 256 * 1024;
const MAX_ROTATED_FILES: usize = 3;
```

```1085:1095:rust/crates/runtime/src/session.rs
fn rotate_session_file_if_needed(path: &Path) -> Result<(), SessionError> {
    let Ok(metadata) = fs::metadata(path) else {
        return Ok(());
    };
    if metadata.len() < ROTATE_AFTER_BYTES {
        return Ok(());
    }
    let rotated_path = rotated_log_path(path);
    fs::rename(path, rotated_path)?;
    Ok(())
}
```

**轮转策略：**

1. 每次全量快照前检查文件大小。
2. 超过 256KB → 把当前文件重命名为 `session-xxx.rot-1749123456789.jsonl`。
3. 然后正常写入新文件。
4. `cleanup_rotated_logs` 保留最多 3 个历史文件，删除更早的。

**为什么不直接删除旧文件而是轮转？**

轮转保留历史文件，这样如果新文件损坏，还能从轮转文件恢复。这是**数据安全**的又一个保险。

**Agent 视角要点**

- JSONL + 追加写入 + 原子写 + 日志轮转，这四个机制协同工作，构成了一个**生产级的持久化系统**。
- 每个机制都只解决一个问题：JSONL 解决追加、原子写解决崩溃安全、轮转解决文件过大、四种 record type 解决内容组织。
- 测试 `persists_and_restores_session_jsonl` 验证了完整的"创建 → 追加 → 保存 → 恢复"流程。

---

## 第三部分：SessionStore——多工作区隔离

### 3.1 问题：为什么需要多工作区隔离？

想象你同时在两个项目里开着 Agent：

```
终端 1：cd ~/project-alpha && claw    → agent 在 alpha 项目里工作
终端 2：cd ~/project-beta && claw     → agent 在 beta 项目里工作
```

如果两个 Agent 的 session 文件混在同一个目录里，就会产生两个问题：

1. **`latest` 别名混淆**：`latest` 到底是 alpha 的还是 beta 的？
2. **文件名冲突**：如果两个 session 恰好生成了相同的 ID（虽然不太可能），文件会互相覆盖。

### 3.2 解决方案：workspace fingerprint

```20:26:rust/crates/runtime/src/session_control.rs
pub struct SessionStore {
    sessions_root: PathBuf,     // e.g. /home/user/project/.claw/sessions/a1b2c3d4e5f60718/
    workspace_root: PathBuf,    // e.g. /home/user/project
}
```

```32:43:rust/crates/runtime/src/session_control.rs
pub fn from_cwd(cwd: impl AsRef<Path>) -> Result<Self, SessionControlError> {
    let cwd = cwd.as_ref();
    let sessions_root = cwd
        .join(".claw")
        .join("sessions")
        .join(workspace_fingerprint(cwd));
    fs::create_dir_all(&sessions_root)?;
    Ok(Self {
        sessions_root,
        workspace_root: cwd.to_path_buf(),
    })
}
```

**磁盘上的布局：**

```
~/project-alpha/.claw/sessions/a1b2c3d4e5f60718/
├── session-1749123456789-0.jsonl
├── session-1749123460000-1.jsonl
└── session-1749123460000-1.rot-1749123500000.jsonl

~/project-beta/.claw/sessions/f7e6d5c4b3a29180/
├── session-1749123458000-2.jsonl
└── session-1749123458000-2.rot-1749123490000.jsonl
```

`workspace_fingerprint` 用 FNV-1a 哈希把路径映射成 16 字符的十六进制字符串：

```296:304:rust/crates/runtime/src/session_control.rs
pub fn workspace_fingerprint(workspace_root: &Path) -> String {
    let input = workspace_root.to_string_lossy();
    let mut hash = 0xcbf2_9ce4_8422_2325_u64;
    for byte in input.as_bytes() {
        hash ^= u64::from(*byte);
        hash = hash.wrapping_mul(0x0100_0000_01b3);
    }
    format!("{hash:016x}")
}
```

**读懂这段 Rust**

- `0xcbf2_9ce4_8422_2325` 是 FNV-1a 的初始哈希值（"offset basis"）。
- `0x0100_0000_01b3` 是 FNV-1a 的乘法常数（"FNV prime"）。
- `hash ^= byte; hash = hash.wrapping_mul(prime)` 就是 FNV-1a 的核心循环——对每个字节做异或再乘以常数。
- `format!("{hash:016x}")` 把 64 位哈希格式化为 16 个十六进制字符。

**为什么用哈希而不是直接用路径做目录名？**

因为路径可能包含特殊字符（空格、中文、符号），做目录名会有各种问题。哈希把任意路径映射到一个固定的、安全的字符串。

测试 `workspace_fingerprint_is_deterministic_and_differs_per_path` 验证了两个关键性质：

```rust
assert_eq!(fp_a1, fp_a2, "same path must produce the same fingerprint");
assert_ne!(fp_a1, fp_b, "different paths must produce different fingerprints");
```

### 3.3 引用别名——`latest` / `last` / `recent`

```306:310:rust/crates/runtime/src/session_control.rs
pub const LATEST_SESSION_REFERENCE: &str = "latest";
const SESSION_REFERENCE_ALIASES: &[&str] = &[LATEST_SESSION_REFERENCE, "last", "recent"];
```

```86:116:rust/crates/runtime/src/session_control.rs
pub fn resolve_reference(&self, reference: &str) -> Result<SessionHandle, SessionControlError> {
    if is_session_reference_alias(reference) {
        let latest = self.latest_session()?;
        return Ok(SessionHandle {
            id: latest.id,
            path: latest.path,
        });
    }
    // ... otherwise, try as a file path or session ID
}
```

三个别名 `"latest"`、`"last"`、`"recent"` 全部指向同一个东西：**最近更新的 session**。

**"最近更新"是怎么判断的？**

```329:337:rust/crates/runtime/src/session_control.rs
fn sort_managed_sessions(sessions: &mut [ManagedSessionSummary]) {
    sessions.sort_by(|left, right| {
        right
            .updated_at_ms
            .cmp(&left.updated_at_ms)
            .then_with(|| right.modified_epoch_millis.cmp(&left.modified_epoch_millis))
            .then_with(|| right.id.cmp(&left.id))
    });
}
```

三级排序：**语义 updated_at_ms > 文件系统 mtime > session_id**。

为什么用 `updated_at_ms` 而不是文件的修改时间？因为文件复制、`touch` 命令、甚至备份恢复都可能改变 mtime。`updated_at_ms` 是 session 自己维护的语义时间戳，更可靠。

测试 `latest_session_prefers_semantic_updated_at_over_file_mtime` 验证了这一点：一个文件的 mtime 更新但 session 的 `updated_at_ms` 更旧的，排在后面。

### 3.4 工作区校验——防止跨工作区加载

```205:225:rust/crates/runtime/src/session_control.rs
fn validate_loaded_session(
    &self,
    session_path: &Path,
    session: &Session,
) -> Result<(), SessionControlError> {
    let Some(actual) = session.workspace_root() else {
        // legacy session without workspace binding
        if path_is_within_workspace(session_path, &self.workspace_root) {
            return Ok(());
        }
        return Err(...);
    };
    if workspace_roots_match(actual, &self.workspace_root) {
        return Ok(());
    }
    Err(SessionControlError::WorkspaceMismatch {
        expected: self.workspace_root.clone(),
        actual: actual.to_path_buf(),
    })
}
```

**校验逻辑：**

1. 如果 session 没有 `workspace_root`（旧格式）→ 检查文件路径是否在工作区目录内 → 放行。
2. 如果 session 有 `workspace_root` → 必须和当前工作区匹配 → 否则拒绝。

测试 `session_store_rejects_legacy_session_from_other_workspace` 验证了跨工作区加载会被拒绝，并返回 `WorkspaceMismatch` 错误。

**Agent 视角要点**

- 多工作区隔离是**生产环境的基本要求**——不是可选的优化。
- FNV-1a 哈希的确定性和区分性保证了"同一个路径永远同一个命名空间、不同路径不同命名空间"。
- 工作区校验是**最后一道防线**——即使文件名对了，如果 workspace_root 不匹配也拒绝加载。

---

## 第四部分：上下文压缩——Agent 的"选择性遗忘"

### 4.1 为什么要压缩？

LLM 有上下文窗口限制。一个持续运行的 Agent 会在 messages 里积累大量历史对话。如果不压缩，最终会超出上下文窗口，API 调用失败。

claw-code 的压缩策略是：**把旧消息替换成摘要，保留最近的消息不变**。

### 4.2 压缩的触发条件

```41:51:rust/crates/runtime/src/compact.rs
pub fn should_compact(session: &Session, config: CompactionConfig) -> bool {
    let start = compacted_summary_prefix_len(session);
    let compactable = &session.messages[start..];

    compactable.len() > config.preserve_recent_messages
        && compactable
            .iter()
            .map(estimate_message_tokens)
            .sum::<usize>()
            >= config.max_estimated_tokens
}
```

两个条件**同时满足**才触发：

1. **可压缩消息数 > 保留数**：确保压缩后真的减少了消息。
2. **可压缩消息的估算 token ≥ 阈值**：确保压缩有意义。

```9:13:rust/crates/runtime/src/compact.rs
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,     // 默认 4
    pub max_estimated_tokens: usize,         // 默认 10,000
}
```

**手动压缩 vs 自动压缩**

手动压缩（`/compact`）使用默认配置：保留最近 4 条、阈值为 10,000 token。

自动压缩使用不同的阈值——基于累计 input token：

```18:19:rust/crates/runtime/src/conversation.rs
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
const AUTO_COMPACTION_THRESHOLD_ENV_VAR: &str = "CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS";
```

当累计 input token 超过 100,000（可通过环境变量覆盖），自动压缩触发，且使用 `max_estimated_tokens: 0`——这意味着**尽可能多地压缩**。

### 4.3 压缩算法：三段切割 + 边界安全

```96:183:rust/crates/runtime/src/compact.rs
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    // 1. 如果不满足压缩条件，直接返回
    if !should_compact(session, config) { return no_change_result; }

    // 2. 识别已有的压缩摘要（如果之前压缩过）
    let existing_summary = session.messages.first().and_then(extract_existing_compacted_summary);
    let compacted_prefix_len = usize::from(existing_summary.is_some());

    // 3. 计算切割点
    let raw_keep_from = session.messages.len().saturating_sub(config.preserve_recent_messages);

    // 4. 边界安全：不在 ToolUse/ToolResult 之间切割
    let keep_from = { /* ... 边界修复逻辑 ... */ };

    // 5. 三段切割
    let removed = &session.messages[compacted_prefix_len..keep_from];
    let preserved = session.messages[keep_from..].to_vec();

    // 6. 生成摘要
    let summary = merge_compact_summaries(existing_summary, &summarize_messages(removed));

    // 7. 构建新 session
    let continuation = get_compact_continuation_message(&summary, true, !preserved.is_empty());
    let mut compacted_messages = vec![ConversationMessage { role: System, blocks: ... }];
    compacted_messages.extend(preserved);
}
```

**三段切割示意：**

```
压缩前：
[messages: 旧消息1, 旧消息2, ..., 旧消息N, 最近1, 最近2, 最近3, 最近4]

切割后：
+--- 已有摘要 ---+--- 被压缩的中间段 ---+--- 保留的尾部 ---+
|  System 摘要   |  将被移除的消息      |  最近 4 条消息   |
+----------------+---------------------+------------------+

压缩后：
[System 摘要（合并版）, 最近1, 最近2, 最近3, 最近4]
```

### 4.4 ToolUse/ToolResult 边界修复——防止孤儿消息

这是压缩算法中最精细的部分。考虑这个场景：

```
messages[3]: Assistant → ToolUse { name: "read_file", id: "tool-1" }
messages[4]: Tool → ToolResult { tool_use_id: "tool-1", output: "..." }
messages[5]: Assistant → Text { "Done." }
```

如果 `preserve_recent_messages = 1`，切割点在 index 5（只保留最后一条）。那 messages[4]（ToolResult）就会被压缩掉，但 messages[3]（ToolUse 对应的 Assistant）也被压缩掉了——结果就是一个**孤儿 ToolResult**。

在 OpenAI 兼容的 API 路径上，这种孤儿消息会导致 400 错误："tool message must follow assistant with tool_calls"。

边界修复的代码：

```121:158:rust/crates/runtime/src/compact.rs
let keep_from = {
    let mut k = raw_keep_from;
    loop {
        if k == 0 || k <= compacted_prefix_len { break; }
        let first_preserved = &session.messages[k];
        let starts_with_tool_result = first_preserved
            .blocks
            .first()
            .is_some_and(|b| matches!(b, ContentBlock::ToolResult { .. }));
        if !starts_with_tool_result { break; }
        // 如果切割点落在 ToolResult 上，往前回退
        let preceding = &session.messages[k - 1];
        let preceding_has_tool_use = preceding
            .blocks
            .iter()
            .any(|b| matches!(b, ContentBlock::ToolUse { .. }));
        if preceding_has_tool_use {
            k = k.saturating_sub(1);  // 回退到包含 ToolUse 的 Assistant 消息
            break;
        }
        k = k.saturating_sub(1);      // 继续往前找
    }
    k
};
```

**边界修复的逻辑：**

1. 检查切割点的第一条消息是否以 ToolResult 开头。
2. 如果是，往前找它对应的 ToolUse（在 preceding Assistant 消息里）。
3. 把切割点回退到那条 Assistant 消息——确保 ToolUse 和 ToolResult 总是一起保留或一起移除。

测试 `compaction_does_not_split_tool_use_tool_result_pair` 专门验证了这个修复。

### 4.5 摘要生成——summarize_messages

```195:280:rust/crates/runtime/src/compact.rs
fn summarize_messages(messages: &[ConversationMessage]) -> String {
    // 1. 统计消息类型数量
    let user_messages = ...;
    let assistant_messages = ...;
    let tool_messages = ...;

    // 2. 提取使用的工具名列表
    let mut tool_names = ...;  // 去重排序后

    // 3. 收集最近 3 条用户请求
    let recent_user_requests = collect_recent_role_summaries(messages, User, 3);

    // 4. 推断待完成的工作
    let pending_work = infer_pending_work(messages);

    // 5. 提取关键文件
    let key_files = collect_key_files(messages);

    // 6. 推断当前正在进行的工作
    let current_work = infer_current_work(messages);

    // 7. 生成时间线
    for message in messages {
        lines.push(format!("  - {role}: {content}"));
    }
}
```

生成的摘要长这样：

```xml
<summary>
Conversation summary:
- Scope: 8 earlier messages compacted (user=4, assistant=3, tool=1).
- Tools mentioned: read_file, bash, glob_search.
- Recent user requests:
  - Fix the bug in main.rs
  - Run the tests
  - Commit the changes
- Pending work:
  - TODO: add regression tests for compaction
  - Next: update the documentation
- Key files referenced: src/main.rs, src/lib.rs, tests/integration.rs.
- Current work: Working on session persistence
- Key timeline:
  - user: Fix the bug in main.rs
  - assistant: I will inspect the main.rs file. | tool_use read_file({"path":"src/main.rs"})
  - tool: tool_result read_file: fn main() { ... }
  - assistant: The bug is on line 42. I'll fix it. | tool_use bash({"command":"sed -i 's/old/new/' src/main.rs"})
</summary>
```

**七个维度的信息，优先级从高到低：**

1. **Scope**：压缩了多少消息，什么类型。
2. **Tools mentioned**：用了哪些工具。
3. **Recent user requests**：用户最近在问什么。
4. **Pending work**：还有什么没做完的（通过关键词 todo/next/pending/follow up/remaining 检测）。
5. **Key files**：涉及哪些文件（通过路径 + 扩展名检测）。
6. **Current work**：当前正在做什么（最后一条非空文本）。
7. **Key timeline**：完整的时间线（每条消息一行摘要）。

### 4.6 摘要压缩——budget 感知的二次压缩

如果摘要去掉了 `<analysis>` 标签后还是太长，`summary_compression.rs` 提供了二次压缩：

```rust
pub struct SummaryCompressionBudget {
    pub max_chars: usize,          // 默认 1,200 字符
    pub max_lines: usize,          // 默认 24 行
    pub max_line_chars: usize,     // 默认 160 字符/行
}
```

压缩策略：

1. **去重**：去掉内容相同但空白不同的行。
2. **按优先级选择**：核心细节（Scope、Current work）> 章节标题 > 列表项 > 其他。
3. **截断**：如果还是太长，在 budget 内尽可能保留高优先级的行。
4. **省略提示**：如果省略了行，添加 `"… N additional line(s) omitted."`。

### 4.7 合并摘要——多次压缩的累积

当 session 被多次压缩时，`merge_compact_summaries` 保证之前的摘要不会丢失：

```xml
<summary>
Conversation summary:
- Previously compacted context:
  - Scope: 8 earlier messages compacted.
  - Key files referenced: src/main.rs.
- Newly compacted context:
  - Scope: 4 earlier messages compacted.
  - Key files referenced: tests/integration.rs.
- Key timeline:
  - user: add regression tests
  - assistant: working on it
</summary>
```

"之前压缩的上下文"和"新压缩的上下文"被分别标注——Agent 能清楚地区分历史和最近。

**Agent 视角要点**

- 压缩不是"删掉旧消息"这么简单——它需要保证**语义完整性**（摘要包含关键信息）和**结构安全性**（不切割 ToolUse/ToolResult 对）。
- 七个维度的摘要设计，让 Agent 在压缩后仍然能恢复足够的上下文来继续工作。
- 二次压缩（summary compression budget）保证了摘要本身不会变成新的"上下文窗口杀手"。
- 多次压缩的累积设计，让 Agent 可以在长时间运行中反复压缩而不丢失早期关键信息。

---

## 第五部分：会话分叉——不修改原会话的不可变操作

### 5.1 fork 的语义

```259:279:rust/crates/runtime/src/session.rs
pub fn fork(&self, branch_name: Option<String>) -> Self {
    let now = current_time_millis();
    Self {
        version: self.version,
        session_id: generate_session_id(),     // 新 ID
        created_at_ms: now,                     // 新创建时间
        updated_at_ms: now,
        messages: self.messages.clone(),         // 克隆所有消息
        compaction: self.compaction.clone(),     // 克隆压缩元数据
        fork: Some(SessionFork {                 // 记录分叉来源
            parent_session_id: self.session_id.clone(),
            branch_name: normalize_optional_string(branch_name),
        }),
        workspace_root: self.workspace_root.clone(),
        prompt_history: self.prompt_history.clone(),
        last_health_check_ms: self.last_health_check_ms,
        model: self.model.clone(),
        persistence: None,                       // 新 session 没有绑定持久化路径
    }
}
```

**fork 做了三件事：**

1. **克隆数据**：messages、compaction、prompt_history 全部深拷贝。原 session 不受任何影响。
2. **生成新 ID**：新的 session_id，独立于原 session。
3. **记录血统**：`fork` 字段记录 parent_session_id 和 branch_name。

**关键设计：persistence 被设为 None**

fork 出来的新 session 没有绑定文件路径。它需要由 `SessionStore.fork_session` 来绑定路径并保存：

```174:196:rust/crates/runtime/src/session_control.rs
pub fn fork_session(
    &self,
    session: &Session,
    branch_name: Option<String>,
) -> Result<ForkedManagedSession, SessionControlError> {
    let forked = session
        .fork(branch_name)
        .with_workspace_root(self.workspace_root.clone());
    let handle = self.create_handle(&forked.session_id);
    let forked = forked.with_persistence_path(handle.path.clone());
    forked.save_to_path(&handle.path)?;
    Ok(ForkedManagedSession {
        parent_session_id: session.session_id.clone(),
        handle,
        session: forked,
        branch_name,
    })
}
```

**为什么 fork 的持久化不在 `Session.fork()` 里做？**

因为 `Session` 是纯数据结构——它不应该知道文件存在哪里。持久化路径由 `SessionStore` 管理，这是**关注点分离**的设计。

### 5.2 血统追踪——ManagedSessionSummary

```318:327:rust/crates/runtime/src/session_control.rs
pub struct ManagedSessionSummary {
    pub id: String,
    pub path: PathBuf,
    pub updated_at_ms: u64,
    pub modified_epoch_millis: u128,
    pub message_count: usize,
    pub parent_session_id: Option<String>,
    pub branch_name: Option<String>,
}
```

`parent_session_id` 和 `branch_name` 让你可以在 CLI 里追踪分叉关系：

```
session-1749123456789-0  (原始会话)
  ├── session-1749123460000-1  (branch: "bugfix", parent: session-1749123456789-0)
  └── session-1749123462000-2  (branch: "experiment", parent: session-1749123456789-0)
```

测试 `forks_session_into_managed_storage_with_lineage` 验证了分叉后的 lineage 信息完整：

```rust
assert_eq!(forked.parent_session_id, source.session_id);
assert_eq!(forked.branch_name.as_deref(), Some("incident-review"));
assert_eq!(summary.parent_session_id.as_deref(), Some(source.session_id.as_str()));
```

**Agent 视角要点**

- **会话分叉是不可变操作**——原会话不会被修改，新会话从快照开始独立演进。这和 git branch 的语义一致。
- 分叉后的 session 继承了所有对话历史和压缩元数据，但有了新的 ID 和独立的持久化路径。
- 血统追踪让你可以在 UI 里展示"这个会话从哪个会话分出来的、叫什么分支"。

---

## 第六部分：UsageTracker——跨重启的用量统计

### 6.1 TokenUsage——四维计费模型

```30:36:rust/crates/runtime/src/usage.rs
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_creation_input_tokens: u32,
    pub cache_read_input_tokens: u32,
}
```

四个维度直接对应 LLM API 的计费字段：

| 维度 | 含义 | 单价倍率 |
|---|---|---|
| `input_tokens` | 发给模型的 token 数 | 1x |
| `output_tokens` | 模型生成的 token 数 | 5x（Sonnet） |
| `cache_creation_input_tokens` | 写入缓存的 token 数 | 1.25x |
| `cache_read_input_tokens` | 从缓存读取的 token 数 | 0.1x |

缓存读取的单价只有 input 的 1/10——这就是为什么 PromptCacheEvent（第05章扩展阅读讲过）是"一等公民"：**缓存命中 = 省钱**。

### 6.2 UsageTracker——双层累计

```168:214:rust/crates/runtime/src/usage.rs
pub struct UsageTracker {
    latest_turn: TokenUsage,    // 最近一次 API 调用的 usage
    cumulative: TokenUsage,     // 所有调用的累计 usage
    turns: u32,                 // 总共调了几次 API
}
```

**双层累计的意义：**

- `latest_turn`：当前这轮 API 调用花了多少——用于显示"本轮费用"。
- `cumulative`：整个会话累计花了多少——用于触发自动压缩（`cumulative.input_tokens >= threshold`）和显示总费用。
- `turns`：总共调了几次 API——用于诊断"为什么这次对话这么贵"。

### 6.3 从 Session 恢复——UsageTracker::from_session

```182:190:rust/crates/runtime/src/usage.rs
pub fn from_session(session: &Session) -> Self {
    let mut tracker = Self::new();
    for message in &session.messages {
        if let Some(usage) = message.usage {
            tracker.record(usage);
        }
    }
    tracker
}
```

**这段代码在做什么？**

遍历 session 里的所有消息，找到带 `usage: Some(...)` 的消息（只有 assistant 消息才会有 usage），把它们的 usage 累加起来。

**为什么需要这个？**

当用户重启 CLI、恢复一个之前的 session 时，`UsageTracker` 是新的（从零开始）。如果不从 session 恢复，`cumulative.input_tokens` 就会从 0 开始计算，导致自动压缩的触发判断失效——明明之前已经用了 80,000 token 了，系统以为只用了 0。

`from_session` 保证了**跨重启的用量统计不丢失**。

测试 `reconstructs_usage_tracker_from_restored_session` 验证了恢复后的 tracker 数据正确：

```rust
let tracker = UsageTracker::from_session(&session);
assert_eq!(tracker.turns(), 1);
assert_eq!(tracker.cumulative_usage().total_tokens(), 8);
```

**Agent 视角要点**

- UsageTracker 的准确性直接影响两个关键决策：**自动压缩何时触发**和**费用显示是否准确**。
- `from_session` 是一个容易被忽略但至关重要的设计——没有它，跨重启的 Agent 会错误地认为"token 还很充裕"。
- 每条 assistant 消息都携带 usage 元数据，这是**数据自包含**的设计——session 文件本身就包含了重建用量统计所需的全部信息。

---

## 第七部分：Session 的完整生命周期

### 7.1 创建

```
Session::new()
  ├── generate_session_id()     → "session-1749123456789-0"
  ├── current_time_millis()     → created_at_ms, updated_at_ms
  ├── messages: Vec::new()      → 空
  ├── compaction: None          → 未压缩
  ├── fork: None                → 非分叉
  └── persistence: None         → 未绑定文件
```

创建后通过 Builder 方法链配置：

```rust
let session = Session::new()
    .with_workspace_root("/path/to/project")
    .with_persistence_path("/path/to/.claw/sessions/abc/session-xxx.jsonl");
```

### 7.2 对话循环

```
run_turn("fix the bug")
  ├── push_user_text("fix the bug")
  │   └── append_persisted_message → 写一行 JSONL
  ├── api_client.stream(request)
  ├── build_assistant_message(events)
  ├── push_message(assistant_message)
  │   └── append_persisted_message → 写一行 JSONL
  ├── for tool_use in pending_tool_uses:
  │   ├── execute tool
  │   └── push_message(tool_result)
  │       └── append_persisted_message → 写一行 JSONL
  └── maybe_auto_compact()
      └── 如果触发 → compact_session → save_to_path → 全量快照
```

### 7.3 压缩

```
maybe_auto_compact()
  ├── cumulative.input_tokens >= 100,000 ?
  ├── compact_session(session, config)
  │   ├── should_compact → true?
  │   ├── 边界安全修复
  │   ├── summarize_messages(removed)
  │   ├── merge_compact_summaries (如果有旧摘要)
  │   └── 构建 continuation message
  └── self.session = result.compacted_session → 替换整个 session
```

### 7.4 恢复

```
Session::load_from_path(path)
  ├── fs::read_to_string(path)
  ├── 自动检测格式（JSON vs JSONL）
  ├── from_jsonl(contents) 或 from_json(value)
  │   ├── 解析 session_meta → version, session_id, ...
  │   ├── 逐行解析 message → messages: Vec
  │   ├── 解析 compaction → Option<SessionCompaction>
  │   └── 解析 prompt_history → Vec<SessionPromptEntry>
  └── with_persistence_path(path) → 绑定文件路径

ConversationRuntime::new_with_features(session, ...)
  └── UsageTracker::from_session(&session) → 恢复用量统计
```

### 7.5 分叉

```
SessionStore::fork_session(session, branch_name)
  ├── session.fork(branch_name) → 克隆数据、新 ID、记录血统
  ├── create_handle(forked.session_id) → 生成文件路径
  ├── forked.with_persistence_path(handle.path)
  ├── forked.save_to_path(handle.path) → 写入新文件
  └── Ok(ForkedManagedSession { parent_id, handle, forked, branch })
```

### 7.6 生命周期全图

```
                        ┌────────────┐
                        │ Session::new │
                        └─────┬──────┘
                              │
                    ┌─────────▼─────────┐
                    │  with_workspace_root │
                    │  with_persistence_path│
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   save_to_path      │ ← 全量快照（JSONL）
                    └─────────┬──────────┘
                              │
              ┌───────────────▼───────────────┐
              │        对话循环                  │
              │  push_message × N               │ ← 追加写入
              │  (user → assistant → tool ...)   │
              └───────────────┬───────────────┘
                              │
                  ┌───────────▼────────────┐
              ┌───┤  token 超过阈值？        │
              │   └───────────┬────────────┘
              │               │ Yes
              │   ┌───────────▼────────────┐
              │   │  compact_session         │ ← 三段切割 + 摘要
              │   │  self.session = compacted │
              │   └───────────┬────────────┘
              │               │
              │   ┌───────────▼────────────┐
              │   │  save_to_path            │ ← 全量快照（压缩后）
              │   └───────────┬────────────┘
              │               │
              │               └──── 继续对话 ────┐
              │                                   │
              │              ┌────────────────────┘
              │              │
              │   ┌──────────▼──────────┐
              │   │  load_from_path       │ ← 恢复（JSONL 或 JSON）
              │   │  UsageTracker::       │
              │   │    from_session       │ ← 用量重建
              │   └──────────┬──────────┘
              │              │
              │   ┌──────────▼──────────┐
              │   │  fork(branch_name)    │ ← 分叉
              │   │  新 ID + 继承数据      │
              │   │  save_to_path         │ ← 写入新文件
              │   └──────────┬──────────┘
              │              │
              │   ┌──────────▼──────────┐
              │   │  两个 session 独立演进  │
              └───► (原 session 不受影响)   │
                  └─────────────────────┘
```

---

## 第八部分：CLI 中的会话管理命令

### 8.1 六个核心命令

| 命令 | 作用 | 会话操作 |
|---|---|---|
| `/compact` | 手动压缩上下文 | 调用 `compact_session`，替换当前 session |
| `/clear [--confirm]` | 开始新的空会话 | 创建新的 `Session::new()`，旧会话保留在磁盘 |
| `/resume <ref>` | 恢复之前的会话 | 调用 `SessionStore.load_session(reference)` |
| `/session list` | 列出所有会话 | 调用 `SessionStore.list_sessions()` |
| `/session fork [branch]` | 分叉当前会话 | 调用 `SessionStore.fork_session(session, branch)` |
| `/session delete <id>` | 删除指定会话 | 删除 JSONL 文件 |

### 8.2 `--resume` 标志——非交互模式恢复

```bash
# 恢复最近的会话，执行一个 prompt
claw --resume latest "继续修复那个 bug"

# 恢复特定会话，先压缩再执行
claw --resume session-1749123456789-0 --compact "继续"
```

`--resume` 的解析流程：

1. `parse_resume_args()` 解析 session 引用和可选的 slash 命令。
2. `SessionStore.load_session(reference)` 加载 session。
3. `UsageTracker::from_session(&session)` 恢复用量。
4. 如果指定了 `--compact`，先压缩再执行。
5. 执行 prompt 或 slash 命令。

### 8.3 Session 引用解析的优先级

```86:116:rust/crates/runtime/src/session_control.rs
pub fn resolve_reference(&self, reference: &str) -> Result<SessionHandle, SessionControlError> {
    // 1. 检查是否是别名（latest/last/recent）
    if is_session_reference_alias(reference) {
        return self.latest_session()?.into();
    }
    // 2. 检查是否是文件路径（绝对路径或相对路径）
    let direct = PathBuf::from(reference);
    if candidate.exists() {
        return Ok(SessionHandle { id: ..., path: candidate });
    }
    // 3. 检查是否是 managed session ID
    self.resolve_managed_path(reference)
}
```

**三种引用方式：**

1. **别名**：`latest`、`last`、`recent` → 解析为最近更新的 session。
2. **文件路径**：`/path/to/session.jsonl` → 直接加载。
3. **Session ID**：`session-1749123456789-0` → 在 managed sessions 目录里查找。

**Agent 视角要点**

- CLI 命令是会话管理的用户接口——每个命令背后都对应着 `Session` 或 `SessionStore` 的一个核心操作。
- 引用解析的三级降级策略让用户可以用最方便的方式引用 session。
- `--resume` 的非交互模式让 Agent 可以在 CI/CD 中恢复之前的会话继续工作。

---

## 第九部分：测试覆盖——Session 模块的 25+ 测试

### 9.1 session.rs 的 14 个测试

| 测试 | 验证什么 | 行号 |
|---|---|---|
| `session_timestamps_are_monotonic_under_tight_loops` | 时间戳严格递增 | 1156-1163 |
| `persists_and_restores_session_jsonl` | JSONL 完整写读一致 | 1166-1209 |
| `loads_legacy_session_json_object` | JSON 格式向后兼容 | 1212-1236 |
| `appends_messages_to_persisted_jsonl_session` | 追加写入正确 | 1239-1259 |
| `persists_compaction_metadata` | 压缩元数据持久化 | 1262-1278 |
| `forks_sessions_with_branch_metadata_and_persists_it` | 分叉元数据持久化 | 1281-1307 |
| `rotates_and_cleans_up_large_session_logs` | 日志轮转和清理 | 1310-1337 |
| `rejects_jsonl_record_without_type` | 拒绝无 type 的 JSONL | 1340-1354 |
| `rejects_jsonl_message_record_without_message_payload` | 拒绝无 payload 的消息 | 1357-1368 |
| `rejects_jsonl_record_with_unknown_type` | 拒绝未知的 record type | 1371-1382 |
| `rejects_legacy_session_json_without_messages` | 拒绝无 messages 的 JSON | 1385-1399 |
| `normalizes_blank_fork_branch_name_to_none` | 空白分支名规范化 | 1402-1411 |
| `rejects_unknown_content_block_type` | 拒绝未知的 block 类型 | 1414-1428 |
| `persists_workspace_root_round_trip_and_forks_inherit_it` | workspace_root 持久化 + fork 继承 | 1431-1451 |

### 9.2 session_control.rs 的 11 个测试

| 测试 | 验证什么 | 行号 |
|---|---|---|
| `latest_session_prefers_semantic_updated_at_over_file_mtime` | 语义时间戳优先于文件时间 | 610-636 |
| `creates_and_lists_managed_sessions` | 创建和列出 session | 639-656 |
| `resolves_latest_alias_and_loads_session_from_workspace_root` | 别名解析和加载 | 659-680 |
| `forks_session_into_managed_storage_with_lineage` | 分叉 + lineage | 683-708 |
| `workspace_fingerprint_is_deterministic_and_differs_per_path` | 指纹确定性和区分性 | 728-745 |
| `session_store_from_cwd_isolates_sessions_by_workspace` | 多工作区隔离 | 748-775 |
| `session_store_from_data_dir_namespaces_by_workspace` | data-dir 命名空间 | 778-806 |
| `session_store_create_and_load_round_trip` | 创建-加载往返 | 809-825 |
| `session_store_rejects_legacy_session_from_other_workspace` | 跨工作区拒绝 | 828-861 |
| `session_store_loads_safe_legacy_session_from_same_workspace` | 同工作区 legacy 加载 | 864-889 |
| `session_store_fork_stays_in_same_namespace` | 分叉保持在同命名空间 | 939-965 |

### 9.3 compact.rs 的 8 个测试

| 测试 | 验证什么 | 行号 |
|---|---|---|
| `formats_compact_summary_like_upstream` | 摘要格式化 | 563-566 |
| `leaves_small_sessions_unchanged` | 小 session 不压缩 | 569-578 |
| `compacts_older_messages_into_a_system_summary` | 压缩为 System 摘要 | 581-639 |
| `keeps_previous_compacted_context_when_compacting_again` | 多次压缩累积 | 642-694 |
| `ignores_existing_compacted_summary_when_deciding_to_recompact` | 摘要不计入再压缩判断 | 697-721 |
| `truncates_long_blocks_in_summary` | 长内容截断 | 724-730 |
| `extracts_key_files_from_message_content` | 关键文件提取 | 733-739 |
| `compaction_does_not_split_tool_use_tool_result_pair` | ToolUse/ToolResult 边界安全 | 746-812 |

**Agent 视角要点**

- 25+ 个测试覆盖了 session 管理的所有关键路径：创建、持久化、恢复、压缩、分叉、隔离、校验。
- 每个测试都验证了一个**具体的不变量**——比如"时间戳严格递增"、"fork 后原 session 不变"、"跨工作区加载被拒绝"。
- 测试的命名遵循了 `does_X_when_Y` 或 `verifies_X` 的模式——测试名本身就是一个文档。

---

## 第十部分：小结与延伸练习

### 10.1 七条带走的结论

1. **Session 是 Agent 的唯一真相源**：所有状态都在 `messages` 里，没有隐式变量。12 个字段分为三层：对话数据、身份定位、运维基础设施。
2. **JSONL 追加写入是最优的持久化策略**：追加（不是全量重写）+ 原子写（temp + rename）+ 日志轮转（256KB 阈值）三者协同，构成了崩溃安全的持久化系统。
3. **多工作区隔离是生产环境的基本要求**：FNV-1a fingerprint + workspace_root 校验，保证了不同项目的 session 永远不会混淆。
4. **上下文压缩是常态而非异常**：Token 超过阈值就自动压缩。压缩不只是"删掉旧消息"，而是生成七维度摘要 + 保证 ToolUse/ToolResult 边界安全。
5. **会话分叉是不可变操作**：克隆数据 + 新 ID + 记录血统，原 session 完全不受影响——和 git branch 的语义一致。
6. **用量统计必须可恢复**：`UsageTracker::from_session` 扫描所有带 usage 的消息重建累计值，保证跨重启的压缩触发和费用显示不丢失。
7. **测试驱动了设计**：25+ 个测试验证了每一个不变量——"时间戳严格递增"、"fork 不变"、"边界不切割"等。这些不变量组成了 session 管理的契约。

### 10.2 设计模式回顾

| 模式 | 在 session 管理中的体现 |
|---|---|
| 单一真相源 | `session.messages` 是 Agent 状态的唯一来源 |
| Builder 模式 | `Session::new().with_workspace_root(...).with_persistence_path(...)` |
| 乐观更新 + 回滚 | `push_message` 先加后写、失败 pop |
| 原子写入 | `write_atomic`：temp + rename |
| 日志轮转 | 256KB 阈值 + 最多 3 个历史文件 |
| 不可变分叉 | `fork` 不修改原 session，创建独立副本 |
| 命名空间隔离 | workspace fingerprint 分区 |
| 从数据重建 | `UsageTracker::from_session` 从消息重建用量 |

### 10.3 可以深入的方向

1. **实现 session 导出**：给 Session 加一个 `export_markdown()` 方法，把 JSONL 转成人类可读的 Markdown 格式。思考哪些字段需要导出、哪些不需要。
2. **给 SessionStore 加搜索功能**：实现 `search_sessions(query: &str)` 方法，根据用户输入文本或文件名搜索历史会话。思考搜索的粒度。
3. **实现压缩的可视化**：在 CLI 里展示"压缩前 N 条消息 → 压缩后 1 条摘要"的对比，让用户理解 Agent 丢掉了什么。
4. **给 fork 加合并**：设计一个 `merge_session(source, target)` 方法，把分叉后的新工作合并回原会话。思考冲突如何处理。
5. **实现 session 加密**：给 `write_atomic` 加一层 AES-256 加密，保护敏感的对话内容。思考密钥管理和性能影响。

### 10.4 对应论文和资料

1. **The Twelve-Factor App（日志篇）**：JSONL 追加写入的思想来源——事件流式记录、不可变追加。
2. **Martin Kleppmann —— Designing Data-Intensive Applications（Chapter 5: Replication）**：日志轮转和压缩的思想来源。
3. **git 的 branch 语义**：会话分叉的不可变语义直接借鉴了 git branch 的设计。
4. **FNV-1a Hash**：workspace fingerprint 使用的哈希算法——简单、快速、分布均匀。
5. **Write-Ahead Logging (WAL)**：追加写入 + 原子重命名的模式，是数据库 WAL 的简化版本。
