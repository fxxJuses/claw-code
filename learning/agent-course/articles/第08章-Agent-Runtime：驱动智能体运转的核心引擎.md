# 第08章 Agent Runtime：驱动智能体运转的核心引擎

> 本文是 claw-code 学习系列的第八篇，聚焦于 Agent 系统中最核心、也最容易被误解的概念——**Agent Runtime（运行时）**。
> 前几章我们已经理解了会话如何管理、上下文如何压缩、工具如何执行。但所有这些能力不是孤立存在的——它们需要一个**协调中心**来把它们串起来：从用户输入开始，到模型推理、工具调度、权限校验、Hook 拦截、用量追踪，直到最终结果返回。
> **这个协调中心就是 Agent Runtime。** 它不是一个单独的文件，而是一组精心设计的模块，像一个微型的"操作系统内核"，管理着 Agent 的完整生命周期。
> **本文面向不懂 Rust 的读者**：每段代码后都有「读懂这段 Rust」小节，只解释理解运行时所必需的语法。

读完本章，你应该能回答八件事：

1. Agent Runtime 的整体架构是什么样的？`ConversationRuntime` 如何把 Session、ApiClient、ToolExecutor、PermissionPolicy 串成一条管线？
2. 一次 Turn（对话轮次）的完整生命周期是什么？从用户输入到最终响应，数据如何流转？
3. `ApiClient` 和 `ToolExecutor` 这两个 trait 如何让 Runtime 与具体的模型和工具解耦？
4. Hook 系统如何在不侵入核心逻辑的前提下拦截和修改工具调用？
5. 权限策略如何在 Hook 和 Policy 两个层面保护工具执行？
6. SystemPromptBuilder 如何从零组装出一条完整的系统提示词？
7. WorkerBoot 状态机如何管理 Agent 进程的启动和信任建立？
8. 错误恢复系统（Recovery Recipes）如何让 Agent 从故障中自愈？

---

## 与前几章的关系

| 主题 | 前几章已覆盖 | 本章新增深入 |
|---|---|---|
| ConversationRuntime | 结构体字段概述 | 完整生命周期：构造 → Turn 执行 → 压缩 → 恢复 |
| ToolExecutor | 工具执行的基本流程 | trait 抽象、StaticToolExecutor 测试替身、Hook 集成 |
| ApiClient | 未覆盖 | trait 抽象、流式事件聚合（AssistantEvent → ConversationMessage） |
| Hook 系统 | 未覆盖 | 完整的三阶段生命周期（Pre/Post/Failure）、JSON 协议、abort signal |
| 权限系统 | 基本概念 | PermissionPolicy 的五级模式、PermissionContext 的 Hook 联动 |
| SystemPromptBuilder | 系统提示词设计（第04章） | 代码级构建流程：指令文件发现 → Git 上下文注入 → 预算截断 |
| WorkerBoot | 未覆盖 | 完整状态机：Spawning → TrustGate → Ready → Running |
| Recovery Recipes | 未覆盖 | 七种故障场景、恢复步骤、升级策略 |

---

## Rust 语法速查（本章新增）

| 符号 / 写法 | 含义 | 本章出现的场景 |
|---|---|---|
| `trait Name { fn method(&mut self, ...); }` | 定义接口（类似 Java interface 或 Python Protocol） | `ApiClient`、`ToolExecutor`、`PermissionPrompter` |
| `impl Trait for Struct { ... }` | 为类型实现接口 | `impl ApiClient for ScriptedApiClient` |
| `impl<T> Struct<T> where T: Trait { ... }` | 为泛型结构体实现方法，要求泛型参数满足约束 | `ConversationRuntime<C, T>` 的方法 |
| `Box<dyn Trait>` | 堆上的动态分派对象（类似 Python 的多态引用） | `Box<dyn HookProgressReporter>` |
| `Arc<Mutex<T>>` | 线程安全的共享引用（Arc = 原子引用计数，Mutex = 互斥锁） | `WorkerRegistry`、`TaskRegistry` |
| `impl Into<String>` | "任何能转成 String 的类型" | `run_turn(user_input: impl Into<String>)` |
| `#[must_use]` | 编译器警告：如果忽略返回值会报警 | 所有不可丢弃的结果类型 |
| `BTreeMap<K, V>` | 有序键值映射（类似 Python 的 OrderedDict） | `PermissionPolicy.tool_requirements` |

---

## 第一部分：问题——为什么需要 Runtime？

### 1.1 没有 Runtime 的世界

想象你正在构建一个 Agent。最直觉的写法可能是这样：

```python
# 伪代码：没有 Runtime 的朴素实现
response = model.chat(user_input)
if response.has_tool_calls:
    for call in response.tool_calls:
        result = execute_tool(call)
        response = model.chat(result)
```

这段代码能跑，但有无数问题：

| 问题 | 后果 |
|---|---|
| 没有会话管理 | 每次调用都是无状态的，模型没有历史记忆 |
| 没有权限检查 | 工具可以执行任何操作，包括 `rm -rf /` |
| 没有 Hook 机制 | 无法在工具执行前后插入自定义逻辑 |
| 没有用量追踪 | 不知道花了多少钱、用了多少 token |
| 没有错误恢复 | 模型调用失败、工具执行出错时直接崩溃 |
| 没有上下文压缩 | 长对话超出 token 限制后直接报错 |

**Runtime 的核心价值，就是把这些横切关注点（cross-cutting concerns）统一管理起来，让核心的"模型 → 工具 → 模型"循环保持简洁和正确。**

### 1.2 设计目标

claw-code 的 Agent Runtime 追求五个目标：

| 目标 | 具体含义 |
|---|---|
| **解耦** | 模型调用、工具执行、权限策略都是可替换的 trait，不绑死具体实现 |
| **可观测** | 每个阶段都发出遥测事件，用量、耗时、错误全部可追踪 |
| **可扩展** | Hook 系统允许外部脚本拦截工具调用，不需要修改 Runtime 代码 |
| **安全** | 权限策略 + Hook 双重保护，工具执行前必须经过授权 |
| **可靠** | 错误恢复系统让 Agent 能从常见故障中自愈 |

---

## 第二部分：全局架构——Runtime 的模块地图

Agent Runtime 不是一个大文件，而是一组协作的模块。让我们先看全貌：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Runtime 架构                           │
│                                                                     │
│  ┌─────────────┐     ┌──────────────────────┐     ┌──────────────┐ │
│  │ SystemPrompt │────▶│ ConversationRuntime   │◀────│ UsageTracker │ │
│  │ Builder      │     │                      │     │ (token/cost) │ │
│  └─────────────┘     │  ┌──── Session ────┐ │     └──────────────┘ │
│                      │  │ messages         │ │                      │
│  ┌─────────────┐     │  │ compaction       │ │     ┌──────────────┐ │
│  │ WorkerBoot   │────▶│  │ fork             │ │────▶│ TaskRegistry │ │
│  │ StateMachine │     │  └─────────────────┘ │     │ (子任务管理) │ │
│  └─────────────┘     │                      │     └──────────────┘ │
│                      │  run_turn() 核心循环   │                      │
│  ┌─────────────┐     │     │                 │     ┌──────────────┐ │
│  │ Config       │────▶│     ▼                 │────▶│ Recovery     │ │
│  │ (RuntimeConfig)│  │  [ApiClient trait]    │     │ Recipes      │ │
│  └─────────────┘     │     │                 │     │ (故障自愈)   │ │
│                      │     ▼                 │     └──────────────┘ │
│                      │  [HookRunner]         │                      │
│                      │     │                 │                      │
│                      │     ▼                 │                      │
│                      │  [PermissionPolicy]   │                      │
│                      │     │                 │                      │
│                      │     ▼                 │                      │
│                      │  [ToolExecutor trait] │                      │
│                      └──────────────────────┘                      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Lane Events (事件总线)                                       │   │
│  │ started → ready → blocked/green → commit → finished/failed  │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

这些模块分布在 `rust/crates/runtime/src/` 目录下，共 44 个文件。按职责分为七层：

| 层 | 模块 | 职责 |
|---|---|---|
| **核心循环** | `conversation.rs` | Turn 执行、消息流、自动压缩 |
| **会话管理** | `session.rs`, `session_control.rs` | Session 状态、持久化、Store |
| **上下文压缩** | `compact.rs`, `summary_compression.rs` | Token 压缩、摘要生成（第07章已覆盖） |
| **提示词构建** | `prompt.rs` | 系统提示词组装、指令文件发现 |
| **工具层** | `bash.rs`, `file_ops.rs`, `bash_validation.rs` | 具体工具实现 |
| **安全层** | `permissions.rs`, `permission_enforcer.rs`, `hooks.rs`, `policy_engine.rs` | 权限校验、Hook 拦截、策略引擎 |
| **引导与恢复** | `worker_boot.rs`, `recovery_recipes.rs`, `bootstrap.rs` | 进程启动、信任建立、故障恢复 |

让我们从最核心的 `ConversationRuntime` 开始，逐层深入。

---

## 第三部分：核心——ConversationRuntime 的构造与结构

### 3.1 三个核心 Trait：解耦的基石

在深入 `ConversationRuntime` 之前，必须先理解三个 trait。它们是整个 Runtime 解耦设计的关键：

```rust:rust/crates/runtime/src/conversation.rs
/// 最小化的流式 API 协议——Runtime 只需要知道"发送请求、拿到事件流"
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}

/// 工具执行器的接口——Runtime 不关心工具具体怎么实现
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}

/// 权限提示器的接口——Runtime 不关心是弹 GUI 对话框还是命令行确认
pub trait PermissionPrompter {
    fn decide(&mut self, request: &PermissionRequest) -> PermissionPromptDecision;
}
```

**读懂这段 Rust**

- `trait` 定义接口，类似于 Java 的 `interface` 或 Python 的 `Protocol`。
- `&mut self` 表示"可变借用 self"——这个方法可能会修改对象的状态。
- `Result<T, E>` 是带错误处理的返回值——`Ok(value)` 成功，`Err(error)` 失败。

这三个 trait 的意义：

| Trait | 代表什么 | 为什么需要抽象 |
|---|---|---|
| `ApiClient` | 模型 API 调用 | 不同模型（Claude、GPT、本地模型）有不同协议 |
| `ToolExecutor` | 工具执行 | 测试时用 mock，生产时用真实工具 |
| `PermissionPrompter` | 权限确认 | CLI 用命令行确认，GUI 用弹窗，无人值守时自动拒绝 |

### 3.2 ConversationRuntime 的构造

```rust:rust/crates/runtime/src/conversation.rs
pub struct ConversationRuntime<C, T> {
    session: Session,
    api_client: C,
    tool_executor: T,
    permission_policy: PermissionPolicy,
    system_prompt: Vec<String>,
    max_iterations: usize,
    usage_tracker: UsageTracker,
    hook_runner: HookRunner,
    auto_compaction_input_tokens_threshold: u32,
    hook_abort_signal: HookAbortSignal,
    hook_progress_reporter: Option<Box<dyn HookProgressReporter>>,
    session_tracer: Option<SessionTracer>,
}
```

**读懂这段 Rust**

- `<C, T>` 是泛型参数——`C` 代表 ApiClient 的具体类型，`T` 代表 ToolExecutor 的具体类型。
- `Box<dyn HookProgressReporter>` 是堆上的动态分派对象——在运行时才确定具体类型。
- `Option<SessionTracer>` 表示遥测追踪器"可能有、可能没有"。

这 12 个字段可以分为四层：

**第一层：对话核心**

| 字段 | 类型 | 角色 |
|---|---|---|
| `session` | `Session` | 对话状态的唯一真相源 |
| `api_client` | `C` (ApiClient) | 模型调用的执行者 |
| `tool_executor` | `T` (ToolExecutor) | 工具调用的执行者 |
| `system_prompt` | `Vec<String>` | 每次调用都发送的系统提示词 |

**第二层：安全控制**

| 字段 | 类型 | 角色 |
|---|---|---|
| `permission_policy` | `PermissionPolicy` | 工具执行的权限策略 |
| `hook_runner` | `HookRunner` | 工具执行前后的 Hook 拦截器 |
| `hook_abort_signal` | `HookAbortSignal` | 用于取消正在运行的 Hook |
| `hook_progress_reporter` | `Option<Box<dyn HookProgressReporter>>` | Hook 执行进度的回调 |

**第三层：资源管理**

| 字段 | 类型 | 角色 |
|---|---|---|
| `usage_tracker` | `UsageTracker` | Token 用量与成本追踪 |
| `auto_compaction_input_tokens_threshold` | `u32` | 自动压缩的触发阈值 |
| `max_iterations` | `usize` | 单次 Turn 的最大迭代次数 |

**第四层：可观测性**

| 字段 | 类型 | 角色 |
|---|---|---|
| `session_tracer` | `Option<SessionTracer>` | 遥测事件的发射器 |

### 3.3 Builder 模式构造

构造 Runtime 不是直接 `new()` 一把梭，而是用 **Builder 模式**逐步装配：

```rust:rust/crates/runtime/src/conversation.rs
// 基础构造
let runtime = ConversationRuntime::new(
    session,           // Session: 已创建的会话
    api_client,        // C: 实现了 ApiClient 的类型
    tool_executor,     // T: 实现了 ToolExecutor 的类型
    permission_policy, // PermissionPolicy: 权限策略
    system_prompt,     // Vec<String>: 系统提示词
);

// 可选配置（每一步都返回新的 runtime，用链式调用）
let runtime = ConversationRuntime::new_with_features(
    session, api_client, tool_executor, permission_policy, system_prompt,
    &feature_config,
)
    .with_max_iterations(50)                                    // 限制最多 50 轮迭代
    .with_auto_compaction_input_tokens_threshold(80_000)        // 80K token 时触发压缩
    .with_hook_abort_signal(abort_signal)                       // 设置取消信号
    .with_hook_progress_reporter(progress_reporter)             // 设置进度回调
    .with_session_tracer(tracer);                               // 设置遥测追踪器
```

**读懂这段 Rust**

- `with_max_iterations(mut self, ...) -> Self` 消费 self，修改字段，返回 self。这是经典的 Builder 模式。
- `mut self` 表示这个方法会消费并修改 self——调用后旧变量不可用。

Builder 模式的优势：

| 方式 | 优点 | 缺点 |
|---|---|---|
| 构造函数传所有参数 | 一步到位 | 参数多了难以阅读，缺省值难以处理 |
| **Builder 模式** ✅ | 可选参数清晰、链式调用可读 | 稍多代码 |
| `Default` trait + 字段赋值 | 简单 | 不能强制必填参数 |

claw-code 的选择是**混合方式**：必填参数（session、api_client、tool_executor、permission_policy、system_prompt）放在构造函数中，可选参数通过 Builder 方法设置。

---

## 第四部分：Turn 生命周期——从用户输入到最终响应

这是 Runtime 的核心。`run_turn()` 方法是整个系统最复杂也最关键的一段代码。让我们逐段拆解：

### 4.1 完整的 Turn 流程图

```
用户输入 "修复这个 bug"
      │
      ▼
┌──────────────────────┐
│ ① 健康探针            │  session 曾被压缩？→ 探测工具执行器是否可用
│    (health probe)     │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ ② 记录 Turn 开始      │  遥测：turn_started { user_input }
│    push_user_text()   │  将用户输入推入 session.messages
└──────────┬───────────┘
           ▼
┌──────────────────────────────────────────────────────┐
│ ③ Agent Loop（核心循环）                               │
│                                                       │
│   ┌───────────┐                                      │
│   │ 构建 API  │  ApiRequest { system_prompt, messages }│
│   │ 请求      │                                      │
│   └─────┬─────┘                                      │
│         ▼                                             │
│   ┌───────────┐                                      │
│   │ ApiClient │  发送请求 → 接收流式事件               │
│   │ .stream() │  AssistantEvent::TextDelta / ToolUse  │
│   └─────┬─────┘  / Usage / PromptCache / MessageStop │
│         ▼                                             │
│   ┌───────────────────┐                              │
│   │ 事件聚合           │  TextDelta 拼接文本           │
│   │ build_assistant_   │  ToolUse 变成 ContentBlock   │
│   │ message()          │  Usage 记录到 usage_tracker  │
│   └─────┬─────────────┘                              │
│         ▼                                             │
│   有 ToolUse？                                        │
│   ┌─── 是 ──────────────────────────────────────┐    │
│   │                                             │    │
│   │  ┌─────────────────┐                        │    │
│   │  │ ④ PreToolUse    │  Hook 脚本拦截          │    │
│   │  │    Hook          │  可：允许/拒绝/修改输入  │    │
│   │  └──────┬──────────┘                        │    │
│   │         ▼                                    │    │
│   │  ┌─────────────────┐                        │    │
│   │  │ ⑤ Permission    │  权限策略校验            │    │
│   │  │    Check         │  Allow / Deny / Ask     │    │
│   │  └──────┬──────────┘                        │    │
│   │         ▼                                    │    │
│   │  ┌─────────────────┐                        │    │
│   │  │ ⑥ ToolExecutor  │  执行工具               │    │
│   │  │    .execute()    │  返回结果               │    │
│   │  └──────┬──────────┘                        │    │
│   │         ▼                                    │    │
│   │  ┌─────────────────┐                        │    │
│   │  │ ⑦ PostToolUse   │  Hook 脚本处理结果      │    │
│   │  │    Hook          │  或 PostToolUseFailure  │    │
│   │  └──────┬──────────┘                        │    │
│   │         ▼                                    │    │
│   │  ToolResult 推入 session.messages            │    │
│   │         │                                    │    │
│   └─────────┼────────────────────────────────────┘    │
│            │ 回到循环顶部，再次调用 API                  │
│            ▼                                         │
│   没有 ToolUse → 退出循环                              │
│                                                       │
└───────────────────────────┬──────────────────────────┘
                            ▼
┌──────────────────────┐
│ ⑧ 自动压缩检查        │  累计 token 超过阈值？
│    maybe_auto_compact │  → 是：压缩 session
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ ⑨ 返回 TurnSummary   │  assistant_messages, tool_results,
│                      │  iterations, usage, auto_compaction
└──────────────────────┘
```

### 4.2 代码实现——逐步拆解

#### 阶段 ①②：健康探针 + 用户输入

```rust:rust/crates/runtime/src/conversation.rs
pub fn run_turn(
    &mut self,
    user_input: impl Into<String>,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> Result<TurnSummary, RuntimeError> {
    let user_input = user_input.into();

    // ① 健康探针：如果 session 曾被压缩，检查是否还在健康状态
    if self.session.compaction.is_some() {
        if let Err(error) = self.run_session_health_probe() {
            return Err(RuntimeError::new(format!(
                "Session health probe failed after compaction: {error}. \
                 The session may be in an inconsistent state. \
                 Consider starting a fresh session with /session new."
            )));
        }
    }

    // ② 记录 Turn 开始，推入用户消息
    self.record_turn_started(&user_input);
    self.session
        .push_user_text(user_input)
        .map_err(|error| RuntimeError::new(error.to_string()))?;
```

**读懂这段 Rust**

- `impl Into<String>` 表示"任何能转成 String 的类型"——可以传 `&str`、`String`、`&String` 等。
- `mut prompter: Option<&mut dyn PermissionPrompter>` 是"可变的可选动态分派引用"——`None` 表示不需要交互确认，`Some(&mut impl)` 表示有确认界面。
- `.map_err(|error| ...)` 转换错误类型——把 Session 的错误转换成 RuntimeError。

#### 阶段 ③：Agent Loop 主体

```rust:rust/crates/runtime/src/conversation.rs
    let mut assistant_messages = Vec::new();
    let mut tool_results = Vec::new();
    let mut prompt_cache_events = Vec::new();
    let mut iterations = 0;

    loop {
        iterations += 1;
        // 安全阀：防止无限循环
        if iterations > self.max_iterations {
            let error = RuntimeError::new(
                "conversation loop exceeded the maximum number of iterations",
            );
            self.record_turn_failed(iterations, &error);
            return Err(error);
        }

        // 构建 API 请求
        let request = ApiRequest {
            system_prompt: self.system_prompt.clone(),
            messages: self.session.messages.clone(),
        };

        // 发送请求，接收流式事件
        let events = match self.api_client.stream(request) {
            Ok(events) => events,
            Err(error) => {
                self.record_turn_failed(iterations, &error);
                return Err(error);
            }
        };

        // 聚合事件为一条完整的 assistant 消息
        let (assistant_message, usage, turn_prompt_cache_events) =
            match build_assistant_message(events) {
                Ok(result) => result,
                Err(error) => {
                    self.record_turn_failed(iterations, &error);
                    return Err(error);
                }
            };

        // 记录用量
        if let Some(usage) = usage {
            self.usage_tracker.record(usage);
        }
        prompt_cache_events.extend(turn_prompt_cache_events);

        // 提取待执行的工具调用
        let pending_tool_uses = assistant_message
            .blocks
            .iter()
            .filter_map(|block| match block {
                ContentBlock::ToolUse { id, name, input } => {
                    Some((id.clone(), name.clone(), input.clone()))
                }
                _ => None,
            })
            .collect::<Vec<_>>();

        // 推入 assistant 消息
        self.session
            .push_message(assistant_message.clone())
            .map_err(|error| RuntimeError::new(error.to_string()))?;
        assistant_messages.push(assistant_message);

        // 没有工具调用 → 退出循环
        if pending_tool_uses.is_empty() {
            break;
        }
```

**读懂这段 Rust**

- `loop { ... }` 是无限循环，靠 `break` 或 `return` 退出。
- `.clone()` 在 Rust 中是深拷贝——因为 session 需要"拥有"消息的所有权，但我们也需要在 TurnSummary 中保留副本。
- `.filter_map(|block| match ...)` 同时过滤和映射——只保留 `ToolUse` 类型的块，并提取 `id/name/input`。

#### 阶段 ④⑤⑥⑦：工具执行管线

```rust:rust/crates/runtime/src/conversation.rs
        // 逐个执行工具调用
        for (tool_use_id, tool_name, input) in pending_tool_uses {
            // ④ PreToolUse Hook
            let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
            let effective_input = pre_hook_result
                .updated_input()
                .map_or_else(|| input.clone(), ToOwned::to_owned);

            // Hook 的结果影响权限上下文
            let permission_context = PermissionContext::new(
                pre_hook_result.permission_override(),
                pre_hook_result.permission_reason().map(ToOwned::to_owned),
            );

            // ⑤ 权限决策（综合 Hook 结果和 Prompter）
            let permission_outcome = if pre_hook_result.is_cancelled() {
                PermissionOutcome::Deny { /* ... */ }
            } else if pre_hook_result.is_failed() {
                PermissionOutcome::Deny { /* ... */ }
            } else if pre_hook_result.is_denied() {
                PermissionOutcome::Deny { /* ... */ }
            } else if let Some(prompt) = prompter.as_mut() {
                // 有交互式确认器 → 允许弹出确认
                self.permission_policy.authorize_with_context(
                    &tool_name, &effective_input,
                    &permission_context, Some(*prompt),
                )
            } else {
                // 无确认器 → 只用静态策略
                self.permission_policy.authorize_with_context(
                    &tool_name, &effective_input,
                    &permission_context, None,
                )
            };

            // ⑥ 根据 权限结果 决定是执行还是拒绝
            let result_message = match permission_outcome {
                PermissionOutcome::Allow => {
                    self.record_tool_started(iterations, &tool_name);
                    // 执行工具
                    let (mut output, mut is_error) =
                        match self.tool_executor.execute(&tool_name, &effective_input) {
                            Ok(output) => (output, false),
                            Err(error) => (error.to_string(), true),
                        };

                    // 合并 Hook 的反馈信息
                    output = merge_hook_feedback(pre_hook_result.messages(), output, false);

                    // ⑦ PostToolUse Hook（成功或失败各走不同路径）
                    let post_hook_result = if is_error {
                        self.run_post_tool_use_failure_hook(
                            &tool_name, &effective_input, &output,
                        )
                    } else {
                        self.run_post_tool_use_hook(
                            &tool_name, &effective_input, &output, false,
                        )
                    };

                    // Post Hook 也可以否决结果
                    if post_hook_result.is_denied()
                        || post_hook_result.is_failed()
                        || post_hook_result.is_cancelled()
                    {
                        is_error = true;
                    }
                    output = merge_hook_feedback(
                        post_hook_result.messages(), output,
                        /* is_error */ true,
                    );

                    ConversationMessage::tool_result(tool_use_id, tool_name, output, is_error)
                }
                PermissionOutcome::Deny { reason } => ConversationMessage::tool_result(
                    tool_use_id, tool_name,
                    merge_hook_feedback(pre_hook_result.messages(), reason, true),
                    true,  // is_error = true
                ),
            };

            // 工具结果推入 session
            self.session
                .push_message(result_message.clone())
                .map_err(|error| RuntimeError::new(error.to_string()))?;
            self.record_tool_finished(iterations, &result_message);
            tool_results.push(result_message);
        }
    }  // ← loop 结束
```

**读懂这段 Rust**

- `for (tool_use_id, tool_name, input) in pending_tool_uses` 是解构遍历——每次拿到一个三元组。
- `match permission_outcome` 是模式匹配——`Allow` 执行工具，`Deny` 直接返回拒绝消息。
- `&mut output` 借用并允许修改——`output` 在多处被追加信息。

这段代码揭示了一个关键的**四阶段管线**设计：

```
ToolUse 请求
    │
    ▼
[PreToolUse Hook] ── 拒绝 ──▶ ToolResult(is_error=true)
    │ 允许
    ▼
[PermissionPolicy] ── 拒绝 ──▶ ToolResult(is_error=true)
    │ 允许
    ▼
[ToolExecutor] ── 错误 ──▶ PostToolUseFailure Hook ──▶ ToolResult(is_error=true)
    │ 成功                │
    ▼                     ▼
[PostToolUse Hook] ◀──────┘
    │
    ▼
ToolResult(is_error=false/true)
```

每个阶段都有**否决权**——只要任一阶段拒绝，工具就不会执行（或结果会被标记为错误）。这是**防御纵深**的设计：即使某一层被绕过，其他层仍然提供保护。

#### 阶段 ⑧⑨：压缩检查 + 返回

```rust:rust/crates/runtime/src/conversation.rs
    // ⑧ 自动压缩检查
    let auto_compaction = self.maybe_auto_compact();

    // ⑨ 构建并返回 TurnSummary
    let summary = TurnSummary {
        assistant_messages,
        tool_results,
        prompt_cache_events,
        iterations,
        usage: self.usage_tracker.cumulative_usage(),
        auto_compaction,
    };
    self.record_turn_completed(&summary);

    Ok(summary)
}
```

`TurnSummary` 携带了一次 Turn 的所有信息：

| 字段 | 类型 | 含义 |
|---|---|---|
| `assistant_messages` | `Vec<ConversationMessage>` | 模型产生的所有消息（可能多轮） |
| `tool_results` | `Vec<ConversationMessage>` | 所有工具的执行结果 |
| `prompt_cache_events` | `Vec<PromptCacheEvent>` | 缓存命中率的变化事件 |
| `iterations` | `usize` | Agent Loop 迭代了几次 |
| `usage` | `TokenUsage` | 累计的 token 用量 |
| `auto_compaction` | `Option<AutoCompactionEvent>` | 是否触发了自动压缩 |

---

## 第五部分：流式事件聚合——`build_assistant_message`

模型 API 返回的不是一条完整消息，而是一系列**流式事件（streaming events）**。`build_assistant_message` 负责把它们聚合成一条结构化的 `ConversationMessage`：

```rust:rust/crates/runtime/src/conversation.rs
fn build_assistant_message(
    events: Vec<AssistantEvent>,
) -> Result<(ConversationMessage, Option<TokenUsage>, Vec<PromptCacheEvent>), RuntimeError> {
    let mut text = String::new();
    let mut blocks = Vec::new();
    let mut prompt_cache_events = Vec::new();
    let mut finished = false;
    let mut usage = None;

    for event in events {
        match event {
            AssistantEvent::TextDelta(delta) => text.push_str(&delta),
            AssistantEvent::ToolUse { id, name, input } => {
                flush_text_block(&mut text, &mut blocks);  // 先把累积的文本刷成 Text block
                blocks.push(ContentBlock::ToolUse { id, name, input });
            }
            AssistantEvent::Usage(value) => usage = Some(value),
            AssistantEvent::PromptCache(event) => prompt_cache_events.push(event),
            AssistantEvent::MessageStop => {
                finished = true;
            }
        }
    }

    flush_text_block(&mut text, &mut blocks);  // 最后刷一次

    if !finished {
        return Err(RuntimeError::new("assistant stream ended without a message stop event"));
    }
    if blocks.is_empty() {
        return Err(RuntimeError::new("assistant stream produced no content"));
    }

    Ok((ConversationMessage::assistant_with_usage(blocks, usage), usage, prompt_cache_events))
}

fn flush_text_block(text: &mut String, blocks: &mut Vec<ContentBlock>) {
    if !text.is_empty() {
        blocks.push(ContentBlock::Text { text: std::mem::take(text) });
    }
}
```

**读懂这段 Rust**

- `std::mem::take(text)` 把 `text` 的内容移走，留下一个空 `String`——避免了分配新内存。
- `match event` 每个事件类型有不同的处理逻辑。

这个函数的关键设计是**文本累积 + 块刷新**模式：

```
事件流：TextDelta("让我") → TextDelta("看看") → ToolUse{name: "Read", ...} → TextDelta("好的") → MessageStop

聚合过程：
  text = "让我看看"            ← 两个 TextDelta 拼接
  → flush → blocks = [Text("让我看看")]
  text = ""
  → blocks = [Text("让我看看"), ToolUse("Read", ...)]
  text = "好的"
  → flush → blocks = [Text("让我看看"), ToolUse("Read", ...), Text("好的")]
  → finished = true

最终消息：
  assistant { blocks: [Text("让我看看"), ToolUse("Read", ...), Text("好的")] }
```

流式事件有五种类型：

| 事件 | 何时出现 | 处理方式 |
|---|---|---|
| `TextDelta` | 模型输出文本片段 | 累积到 text 缓冲区 |
| `ToolUse` | 模型决定调用工具 | 刷出 Text 块，创建 ToolUse 块 |
| `Usage` | 模型报告 token 用量 | 保存到 usage |
| `PromptCache` | 缓存命中率变化 | 保存事件 |
| `MessageStop` | 模型输出结束 | 标记完成 |

---

## 第六部分：Hook 系统——不侵入核心逻辑的拦截器

### 6.1 为什么需要 Hook？

假设你想要在每次工具执行前记录日志，或者在 `bash` 命令执行前做安全检查。最直觉的做法是修改 `run_turn()` 的代码。但这有几个问题：

1. **侵入性**：每次新增需求都要改核心代码
2. **不可配置**：所有用户都被迫使用相同的逻辑
3. **不可组合**：多个需求之间的交互难以管理

Hook 系统的设计哲学是：**让核心逻辑保持纯净，通过外部脚本来扩展行为**。

### 6.2 三阶段生命周期

```rust:rust/crates/runtime/src/hooks.rs
pub enum HookEvent {
    PreToolUse,           // 工具执行前
    PostToolUse,          // 工具执行后（成功）
    PostToolUseFailure,   // 工具执行后（失败）
}
```

每个 Hook 是一个外部脚本（shell 命令），通过环境变量和 JSON stdin/stdout 与 Runtime 通信：

```
┌────────────┐                         ┌──────────────┐
│ Runtime    │  设置环境变量 + JSON stdin │ Hook Script  │
│            │─────────────────────────▶│ (外部脚本)    │
│            │◀─────────────────────────│              │
│            │  读取 JSON stdout + 退出码 │              │
└────────────┘                         └──────────────┘
```

**环境变量**（Runtime → Hook）：

| 变量 | 内容 |
|---|---|
| `HOOK_EVENT` | `PreToolUse` / `PostToolUse` / `PostToolUseFailure` |
| `HOOK_TOOL_NAME` | 工具名称，如 `bash`、`write_file` |
| `HOOK_TOOL_INPUT` | 工具的输入参数（JSON 字符串） |
| `HOOK_TOOL_OUTPUT` | 工具的输出（仅 PostToolUse/PostToolUseFailure） |
| `HOOK_TOOL_IS_ERROR` | `0`（成功）或 `1`（失败） |

**stdin**：完整的 JSON payload，包含事件详情。

**stdout**：Hook 的响应（JSON 格式）。

**退出码**：

| 退出码 | 含义 |
|---|---|
| 0 | 允许（除非 JSON 中有 `decision: "block"`） |
| 2 | 拒绝 |
| 其他 | Hook 执行失败 |

### 6.3 HookRunner 的执行逻辑

```rust:rust/crates/runtime/src/hooks.rs
impl HookRunner {
    fn run_commands(
        event: HookEvent,
        commands: &[String],       // 配置的 Hook 命令列表
        tool_name: &str,
        tool_input: &str,
        tool_output: Option<&str>,
        is_error: bool,
        abort_signal: Option<&HookAbortSignal>,
        reporter: Option<&mut dyn HookProgressReporter>,
    ) -> HookRunResult {
        // 没有 Hook 配置 → 直接放行
        if commands.is_empty() {
            return HookRunResult::allow(Vec::new());
        }

        // 检查是否已取消
        if abort_signal.is_some_and(HookAbortSignal::is_aborted) {
            return HookRunResult { cancelled: true, ... };
        }

        let payload = hook_payload(event, tool_name, tool_input, tool_output, is_error)
            .to_string();
        let mut result = HookRunResult::allow(Vec::new());

        // 逐个执行 Hook 命令
        for command in commands {
            match Self::run_command(command, event, tool_name, ...) {
                HookCommandOutcome::Allow { parsed } => {
                    merge_parsed_hook_output(&mut result, parsed);
                }
                HookCommandOutcome::Deny { parsed } => {
                    merge_parsed_hook_output(&mut result, parsed);
                    result.denied = true;
                    return result;  // 一旦拒绝，立即短路
                }
                HookCommandOutcome::Failed { parsed } => {
                    merge_parsed_hook_output(&mut result, parsed);
                    result.failed = true;
                    return result;  // 一旦失败，立即短路
                }
                HookCommandOutcome::Cancelled { message } => {
                    result.cancelled = true;
                    result.messages.push(message);
                    return result;
                }
            }
        }

        result
    }
}
```

**读懂这段 Rust**

- `for command in commands` 遍历所有配置的 Hook 命令。
- `return result` 在 `Deny` 和 `Failed` 分支中出现——这意味着 Hook 执行是**短路**的，第一个拒绝就停止。

关键设计决策：

| 决策 | 理由 |
|---|---|
| **短路执行** | 一旦有 Hook 拒绝，不再执行后续 Hook——拒绝的优先级最高 |
| **Allow 不可覆盖 Deny** | 即使前一个 Hook 允许了，后续的 Hook 仍然可以拒绝 |
| **失败等于拒绝** | Hook 脚本出错（非 0/2 退出码）被视为失败，安全起见拒绝工具执行 |
| **Abort Signal** | 用户可以在 Hook 执行期间按 Ctrl+C 取消 |

### 6.4 Hook 的输出解析

Hook 脚本通过 JSON stdout 返回结构化响应：

```rust:rust/crates/runtime/src/hooks.rs
fn parse_hook_output(event, tool_name, command, stdout, stderr) -> ParsedHookOutput {
    if stdout.is_empty() {
        return ParsedHookOutput::default();  // 空输出 = 静默允许
    }

    let root = serde_json::from_str::<Value>(stdout);

    // 解析 JSON 输出中的字段：
    let mut parsed = ParsedHookOutput::default();

    if let Some(message) = root.get("systemMessage").and_then(Value::as_str) {
        parsed.messages.push(message.to_string());  // 附加到工具结果的消息
    }
    if let Some(message) = root.get("reason").and_then(Value::as_str) {
        parsed.messages.push(message.to_string());  // 拒绝原因
    }
    if root.get("continue").and_then(Value::as_bool) == Some(false)
        || root.get("decision").and_then(Value::as_str) == Some("block")
    {
        parsed.deny = true;  // 明确拒绝
    }
    // ... 还有 permissionOverride, updatedInput 等
}
```

Hook 的 JSON 输出可以包含：

| 字段 | 含义 |
|---|---|
| `systemMessage` | 附加到工具结果的消息 |
| `reason` | 拒绝或修改的原因 |
| `decision: "block"` 或 `continue: false` | 拒绝工具执行 |
| `permissionOverride` | 覆盖权限决策（`allow`/`deny`/`ask`） |
| `updatedInput` | 替换工具的输入参数 |

这意味着 Hook 不仅仅是"拦截"，它还能**修改工具的行为**。例如，一个 PreToolUse Hook 可以把 `bash` 命令中的危险参数替换为安全版本。

---

## 第七部分：权限系统——工具执行的安全网

### 7.1 PermissionMode 五级模型

```rust:rust/crates/runtime/src/permissions.rs
pub enum PermissionMode {
    ReadOnly,           // 只读：只能执行不修改文件系统的操作
    WorkspaceWrite,     // 工作区可写：可以修改工作区内文件
    DangerFullAccess,   // 完全访问：可以做任何事（包括 rm -rf）
    Prompt,             // 交互提示：每次工具执行都问用户
    Allow,              // 全部允许：跳过所有权限检查
}
```

这五个模式构成一个**安全层级**：

```
安全性从高到低：

ReadOnly ──▶ WorkspaceWrite ──▶ DangerFullAccess
   │                                     │
   └────── Prompt ◀──────────────────────┘
                  │
              Allow（跳过一切）
```

| 模式 | 适用场景 | 风险 |
|---|---|---|
| `ReadOnly` | 代码审查、只读分析 | 最低——不能修改任何东西 |
| `WorkspaceWrite` | 日常开发、代码修改 | 中等——只能改工作区内文件 |
| `DangerFullAccess` | 系统管理、跨项目操作 | 高——可以执行任意命令 |
| `Prompt` | 不确定时逐次确认 | 安全但繁琐 |
| `Allow` | 自动化、CI/CD | 最高——无任何限制 |

### 7.2 PermissionPolicy 的三层规则

权限策略不仅仅是"模式匹配"，它有三层规则叠加：

```rust:rust/crates/runtime/src/permissions.rs
pub struct PermissionPolicy {
    active_mode: PermissionMode,                      // 第一层：当前模式
    tool_requirements: BTreeMap<String, PermissionMode>, // 第二层：工具级别要求
    allow_rules: Vec<PermissionRule>,                  // 第三层：自定义规则
    deny_rules: Vec<PermissionRule>,
    ask_rules: Vec<PermissionRule>,
}
```

授权决策的流程：

```
工具请求执行
    │
    ▼
[检查 deny_rules] ── 命中 ──▶ Deny
    │ 未命中
    ▼
[检查 ask_rules] ── 命中 ──▶ Ask（弹出确认）
    │ 未命中
    ▼
[检查 allow_rules] ── 命中 ──▶ Allow
    │ 未命中
    ▼
[检查 tool_requirements] ── 模式不够 ──▶ Deny
    │ 模式满足
    ▼
Allow
```

### 7.3 PermissionEnforcer——专用检查器

`PermissionEnforcer` 是 `PermissionPolicy` 的专用包装，提供场景化的检查方法：

```rust:rust/crates/runtime/src/permission_enforcer.rs
pub fn check_file_write(&self, path: &str, workspace_root: &str) -> EnforcementResult {
    match self.policy.active_mode() {
        PermissionMode::ReadOnly => EnforcementResult::Denied { /* ... */ },
        PermissionMode::WorkspaceWrite => {
            if is_within_workspace(path, workspace_root) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied { /* 路径超出工作区 */ }
            }
        }
        PermissionMode::Allow | PermissionMode::DangerFullAccess => {
            EnforcementResult::Allowed
        }
        PermissionMode::Prompt => EnforcementResult::Denied { /* 需要确认 */ },
    }
}

pub fn check_bash(&self, command: &str) -> EnforcementResult {
    match self.policy.active_mode() {
        PermissionMode::ReadOnly => {
            if is_read_only_command(command) {
                EnforcementResult::Allowed   // ls, cat, git status 等
            } else {
                EnforcementResult::Denied { /* ... */ }
            }
        }
        // WorkspaceWrite, Allow, DangerFullAccess: 允许 bash
        _ => EnforcementResult::Allowed,
    }
}
```

**读懂这段 Rust**

- `match self.policy.active_mode()` 根据当前权限模式走不同分支。
- `is_within_workspace(path, workspace_root)` 检查路径是否在工作区内。
- `is_read_only_command(command)` 用启发式判断命令是否只读（如 `ls`、`cat`、`git status`）。

这里有一个精巧的设计：`ReadOnly` 模式下不是完全禁止 `bash`，而是用启发式判断命令是否只读。像 `ls -la`、`cat file.txt`、`git status` 这些命令是允许的，但 `rm`、`mkdir`、`cargo build` 会被拒绝。

---

## 第八部分：SystemPromptBuilder——系统提示词的装配线

### 8.1 提示词不是"一坨文本"

第04章我们从设计角度分析了系统提示词。现在让我们从代码角度看看它是如何被组装的：

```rust:rust/crates/runtime/src/prompt.rs
pub struct SystemPromptBuilder {
    output_style_name: Option<String>,       // 输出风格名称
    output_style_prompt: Option<String>,     // 输出风格指令
    os_name: Option<String>,                 // 操作系统名称
    os_version: Option<String>,              // 操作系统版本
    append_sections: Vec<String>,            // 追加的自定义段落
    project_context: Option<ProjectContext>, // 项目上下文
    config: Option<RuntimeConfig>,           // 运行时配置
}
```

`build()` 方法按固定顺序组装段落：

```rust:rust/crates/runtime/src/prompt.rs
pub fn build(&self) -> Vec<String> {
    let mut sections = Vec::new();
    sections.push(get_simple_intro_section(...));       // ① 简介
    if let (Some(name), Some(prompt)) = (...) {
        sections.push(format!("# Output Style: {name}\n{prompt}"));  // ② 输出风格
    }
    sections.push(get_simple_system_section());          // ③ 系统规则
    sections.push(get_simple_doing_tasks_section());     // ④ 任务执行规则
    sections.push(get_actions_section());                // ⑤ 工具描述
    sections.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY);       // ⑥ 动态边界标记
    sections.push(self.environment_section());           // ⑦ 环境信息
    if let Some(project_context) = ... {
        sections.push(render_project_context(...));      // ⑧ 项目上下文
        sections.push(render_instruction_files(...));    // ⑨ 指令文件
    }
    if let Some(config) = ... {
        sections.push(render_config_section(config));    // ⑩ 运行时配置
    }
    sections.extend(self.append_sections.iter().cloned()); // ⑪ 自定义追加
    sections
}
```

最终生成的是一个 **`Vec<String>`**——每一段是一个独立的字符串，最终用 `"\n\n"` 连接。

### 8.2 指令文件发现——从项目根到当前目录的向上搜索

```rust:rust/crates/runtime/src/prompt.rs
fn discover_instruction_files(cwd: &Path) -> std::io::Result<Vec<ContextFile>> {
    let mut directories = Vec::new();
    let mut cursor = Some(cwd);
    // 从当前目录向上遍历到根目录
    while let Some(dir) = cursor {
        directories.push(dir.to_path_buf());
        cursor = dir.parent();
    }
    directories.reverse();  // 从根到叶排序

    let mut files = Vec::new();
    for dir in directories {
        for candidate in [
            dir.join("CLAUDE.md"),
            dir.join("CLAUDE.local.md"),
            dir.join(".claw").join("CLAUDE.md"),
            dir.join(".claw").join("instructions.md"),
        ] {
            push_context_file(&mut files, candidate)?;
        }
    }
    Ok(dedupe_instruction_files(files))
}
```

**读懂这段 Rust**

- `dir.parent()` 返回父目录的 `Option<&Path>`——到根目录后返回 `None`。
- `directories.reverse()` 反转顺序——从项目根开始加载，确保内层目录的指令可以覆盖外层的。

搜索的文件名和优先级：

```
/Users/fan/workspace/my-project/         ← 项目根
  CLAUDE.md                              ← 全局项目指令
  CLAUDE.local.md                        ← 本地覆盖（不入 Git）
  .claw/CLAUDE.md                        ← .claw 子目录中的指令
  .claw/instructions.md                  ← 额外指令

/Users/fan/workspace/my-project/src/     ← 子目录
  CLAUDE.md                              ← 子目录级指令
  ...
```

### 8.3 预算截断——防止指令文件吃光 token

```rust:rust/crates/runtime/src/prompt.rs
const MAX_INSTRUCTION_FILE_CHARS: usize = 4_000;
const MAX_TOTAL_INSTRUCTION_CHARS: usize = 12_000;
```

每个指令文件最多 4,000 字符，所有文件总计最多 12,000 字符。超过预算的部分被截断，并追加提示：

```
_Additional instruction content omitted after reaching the prompt budget._
```

### 8.4 项目上下文注入

```rust:rust/crates/runtime/src/prompt.rs
fn render_project_context(project_context: &ProjectContext) -> String {
    let mut lines = vec!["# Project context".to_string()];
    lines.extend(prepend_bullets(vec![
        format!("Today's date is {}.", project_context.current_date),
        format!("Working directory: {}", project_context.cwd.display()),
    ]));
    // Git status
    if let Some(status) = &project_context.git_status {
        lines.push("Git status snapshot:".to_string());
        lines.push(status.clone());
    }
    // Git diff
    if let Some(diff) = &project_context.git_diff {
        lines.push("Git diff snapshot:".to_string());
        lines.push(diff.clone());
    }
    // ...
}
```

`ProjectContext` 包含六类动态信息：

| 信息 | 来源 | 何时更新 |
|---|---|---|
| 当前日期 | 系统时钟 | 每次 build |
| 工作目录 | `cwd` | 每次 build |
| Git status | `git status --short --branch` | 每次 build |
| Git diff | `git diff` + `git diff --cached` | 每次 build |
| 最近提交 | `GitContext::detect()` | 每次 build |
| 指令文件 | 文件系统扫描 | 每次 build |

**注意**：系统提示词是"每次 Turn 开始时构建"的，而不是"Session 创建时构建一次"。这意味着 Git 状态会随着用户的操作实时更新。

---

## 第九部分：WorkerBoot 状态机——Agent 进程的启动管理

### 9.1 为什么需要 WorkerBoot？

在自动化场景（如 CI/CD、后台任务）中，Agent 是作为一个**独立进程**启动的。这个进程需要经历一系列初始化步骤，任何一步失败都会导致任务无法开始。

`WorkerBoot` 是一个**内存中的状态机**，追踪 Agent 进程从启动到就绪的完整过程：

```
Spawning ──▶ TrustRequired ──▶ ReadyForPrompt ──▶ Running ──▶ Finished
                   │                                      │
                   └────── Failed ◀───────────────────────┘
```

### 9.2 WorkerStatus 六状态模型

```rust:rust/crates/runtime/src/worker_boot.rs
pub enum WorkerStatus {
    Spawning,           // 进程已启动，正在初始化
    TrustRequired,      // 需要用户确认信任（首次在这个目录运行）
    ReadyForPrompt,     // 初始化完成，等待接收任务
    Running,            // 正在执行任务
    Finished,           // 任务完成
    Failed,             // 任务失败
}
```

### 9.3 信任门控（Trust Gate）

当 Agent 第一次在一个新目录中运行时，需要用户确认"信任这个目录"。`WorkerBoot` 通过**观察终端输出来检测信任提示**：

```rust:rust/crates/runtime/src/worker_boot.rs
pub fn observe(&self, worker_id: &str, screen_text: &str) -> Result<Worker, String> {
    // 检测信任提示
    if !worker.trust_gate_cleared && detect_trust_prompt(&lowered) {
        worker.status = WorkerStatus::TrustRequired;
        worker.last_error = Some(WorkerFailure {
            kind: WorkerFailureKind::TrustGate,
            message: "worker boot blocked on trust prompt".to_string(),
            created_at: now_secs(),
        });

        // 如果目录在信任列表中 → 自动通过
        if worker.trust_auto_resolve {
            worker.trust_gate_cleared = true;
            worker.last_error = None;
            worker.status = WorkerStatus::Spawning;
            // 记录自动解决事件
        } else {
            return Ok(worker.clone());  // 等待人工确认
        }
    }
    // ...
}
```

**读懂这段 Rust**

- `detect_trust_prompt(&lowered)` 扫描终端输出中是否包含信任提示的特征文本。
- `trust_auto_resolve` 是在创建 Worker 时根据 `trusted_roots` 配置设置的——如果目录在信任列表中，自动通过信任门控。

信任门控的两种解决方式：

| 方式 | 场景 | 对应枚举 |
|---|---|---|
| `AutoAllowlisted` | 目录在 `trusted_roots` 配置中 | `WorkerTrustResolution::AutoAllowlisted` |
| `ManualApproval` | 用户手动确认 | `WorkerTrustResolution::ManualApproval` |

### 9.4 提示投递检测（Prompt Misdelivery）

在自动化场景中，一个常见的故障是**提示被投递到了错误的目标**——比如本应发送给 Agent 的任务描述被发送到了普通 shell。`WorkerBoot` 通过观察终端输出来检测这种情况：

```rust:rust/crates/runtime/src/worker_boot.rs
if let Some(observation) = detect_prompt_misdelivery(
    screen_text, &lowered,
    worker.last_prompt.as_deref(),
    &worker.cwd,
    worker.expected_receipt.as_ref(),
) {
    // 检测到提示被投递到错误的目标
    worker.last_error = Some(WorkerFailure {
        kind: WorkerFailureKind::PromptDelivery,
        message: format!("worker prompt landed in shell instead of coding agent: ..."),
        created_at: now_secs(),
    });

    // 如果启用了自动恢复 → 准备重放提示
    if worker.auto_recover_prompt_misdelivery {
        worker.replay_prompt = worker.last_prompt.clone();
        worker.status = WorkerStatus::ReadyForPrompt;
    }
}
```

四种提示投递故障：

| 类型 | 含义 |
|---|---|
| `Shell` | 提示落入了普通 shell 而不是 Agent |
| `WrongTarget` | 提示发送到了错误的工作目录 |
| `WrongTask` | 提示的任务上下文不匹配 |
| `Unknown` | 提示投递失败但原因不明 |

### 9.5 WorkerRegistry——多 Worker 管理

```rust:rust/crates/runtime/src/worker_boot.rs
pub struct WorkerRegistry {
    inner: Arc<Mutex<WorkerRegistryInner>>,
}

struct WorkerRegistryInner {
    workers: HashMap<String, Worker>,
    counter: u64,
}
```

`WorkerRegistry` 使用 `Arc<Mutex<...>>` 模式——这是 Rust 中**线程安全的共享状态**的标准做法：

- `Arc`（Atomic Reference Counted）允许多个线程共享所有权
- `Mutex` 确保同一时间只有一个线程能访问内部数据

每个 Worker 有 15 个字段，记录了完整的状态：

```rust:rust/crates/runtime/src/worker_boot.rs
pub struct Worker {
    pub worker_id: String,
    pub cwd: String,
    pub status: WorkerStatus,
    pub trust_auto_resolve: bool,
    pub trust_gate_cleared: bool,
    pub auto_recover_prompt_misdelivery: bool,
    pub prompt_delivery_attempts: u32,
    pub prompt_in_flight: bool,
    pub last_prompt: Option<String>,
    pub expected_receipt: Option<WorkerTaskReceipt>,
    pub replay_prompt: Option<String>,
    pub last_error: Option<WorkerFailure>,
    pub created_at: u64,
    pub updated_at: u64,
    pub events: Vec<WorkerEvent>,
}
```

---

## 第十部分：错误恢复——Recovery Recipes

### 10.1 七种已知故障场景

```rust:rust/crates/runtime/src/recovery_recipes.rs
pub enum FailureScenario {
    TrustPromptUnresolved,   // 信任提示未解决
    PromptMisdelivery,       // 提示投递错误
    StaleBranch,             // 分支落后于 main
    CompileRedCrossCrate,    // 编译失败
    McpHandshakeFailure,     // MCP 握手失败
    PartialPluginStartup,    // 插件部分启动失败
    ProviderFailure,         // 模型服务故障
}
```

每种故障都有对应的**恢复配方（Recovery Recipe）**：

```rust:rust/crates/runtime/src/recovery_recipes.rs
pub fn recipe_for(scenario: &FailureScenario) -> RecoveryRecipe {
    match scenario {
        FailureScenario::TrustPromptUnresolved => RecoveryRecipe {
            steps: vec![RecoveryStep::AcceptTrustPrompt],
            max_attempts: 1,
            escalation_policy: EscalationPolicy::AlertHuman,
        },
        FailureScenario::StaleBranch => RecoveryRecipe {
            steps: vec![RecoveryStep::RebaseBranch, RecoveryStep::CleanBuild],
            max_attempts: 1,
            escalation_policy: EscalationPolicy::AlertHuman,
        },
        FailureScenario::McpHandshakeFailure => RecoveryRecipe {
            steps: vec![RecoveryStep::RetryMcpHandshake { timeout: 5000 }],
            max_attempts: 3,
            escalation_policy: EscalationPolicy::LogAndContinue,
        },
        // ...
    }
}
```

**读懂这段 Rust**

- `RecoveryRecipe` 包含三要素：`steps`（恢复步骤序列）、`max_attempts`（最大自动尝试次数）、`escalation_policy`（超出尝试次数后的策略）。

### 10.2 恢复步骤

八种恢复步骤：

| 步骤 | 含义 |
|---|---|
| `AcceptTrustPrompt` | 自动接受信任提示 |
| `RedirectPromptToAgent` | 将提示重新发送到正确的目标 |
| `RebaseBranch` | 将当前分支 rebase 到 main |
| `CleanBuild` | 清理并重新构建 |
| `RetryMcpHandshake` | 重试 MCP 握手（带超时） |
| `RestartPlugin` | 重启失败的插件 |
| `RestartWorker` | 重启整个 Worker 进程 |
| `EscalateToHuman` | 升级到人工处理 |

### 10.3 三种升级策略

```rust:rust/crates/runtime/src/recovery_recipes.rs
pub enum EscalationPolicy {
    AlertHuman,       // 通知人工干预
    LogAndContinue,   // 记录日志但继续运行
    Abort,            // 终止执行
}
```

不同故障场景使用不同的升级策略：

| 场景 | max_attempts | 升级策略 | 理由 |
|---|---|---|---|
| 信任提示未解决 | 1 | AlertHuman | 安全决策必须人工确认 |
| 提示投递错误 | 1 | AlertHuman | 需要人工检查目标 |
| 分支落后 | 1 | AlertHuman | rebase 可能需要解决冲突 |
| 编译失败 | 1 | AlertHuman | 可能需要代码修改 |
| MCP 握手失败 | 3 | LogAndContinue | 网络问题可能自行恢复 |
| 插件部分失败 | 2 | LogAndContinue | 非关键插件可以降级运行 |
| 模型服务故障 | 2 | Abort | 没有模型，Agent 无法工作 |

### 10.4 恢复执行流程

```rust:rust/crates/runtime/src/recovery_recipes.rs
pub fn attempt_recovery(
    scenario: &FailureScenario,
    context: &mut RecoveryContext,
) -> RecoveryResult {
    let recipe = recipe_for(scenario);
    let attempts = context.attempt_count(scenario);

    // 超出最大尝试次数 → 升级
    if attempts >= recipe.max_attempts {
        context.events.push(RecoveryEvent::Escalated);
        return RecoveryResult::EscalationRequired {
            reason: format!("exceeded max_attempts ({}) for {:?}", recipe.max_attempts, scenario),
        };
    }

    // 执行恢复步骤
    context.record_attempt(scenario);
    // ... 逐个执行 steps

    RecoveryResult::Recovered { steps_taken: recipe.steps.len() as u32 }
}
```

恢复系统的设计原则：

| 原则 | 实现 |
|---|---|
| **先自动后升级** | `max_attempts` 允许多次自动尝试，失败后才升级 |
| **每步可观测** | `RecoveryEvent` 记录每次尝试的结果 |
| **按场景定制** | 不同故障有不同的步骤和升级策略 |
| **安全优先** | 安全相关的故障（信任提示）只允许一次自动尝试 |

---

## 第十一部分：事件系统——Lane Events

### 11.1 Lane 的概念

在 claw-code 中，一个 **Lane（车道）** 代表一条完整的工作流——从 Agent 启动到任务完成。Lane Events 是这条工作流的状态机：

```rust:rust/crates/runtime/src/lane_events.rs
pub enum LaneEventName {
    Started,           // 工作流启动
    Ready,             // Worker 就绪
    PromptMisdelivery, // 提示投递错误
    Blocked,           // 被阻塞
    Red,               // 失败（红灯）
    Green,             // 成功（绿灯）
    CommitCreated,     // 创建了 commit
    PrOpened,          // 创建了 PR
    MergeReady,        // 可以合并
    Finished,          // 完成
    Failed,            // 失败
    Reconciled,        // 冲突已解决
    Merged,            // 已合并
    Superseded,        // 被新的 commit 替代
    Closed,            // 已关闭
}
```

这些事件构成一条**状态流**：

```
Started → Ready → Green → CommitCreated → PrOpened → MergeReady → Merged → Closed
              │                           │
              └── PromptMisdelivery ──────┘
              │
              └── Blocked → Red → Failed
              │
              └── Green → CommitCreated → Superseded (被新 commit 替代)
```

### 11.2 十一种故障分类

```rust:rust/crates/runtime/src/lane_events.rs
pub enum LaneFailureClass {
    PromptDelivery,    // 提示投递失败
    TrustGate,         // 信任门控失败
    BranchDivergence,  // 分支分叉
    Compile,           // 编译失败
    Test,              // 测试失败
    PluginStartup,     // 插件启动失败
    McpStartup,        // MCP 启动失败
    McpHandshake,      // MCP 握手失败
    GatewayRouting,    // 网关路由失败
    ToolRuntime,       // 工具运行时错误
    WorkspaceMismatch, // 工作区不匹配
    Infra,             // 基础设施故障
}
```

每种故障分类对应不同的恢复策略。`LaneEvent` 携带 `failure_class` 和 `detail`，让上层系统可以根据故障类型采取不同的处理方式。

### 11.3 Commit 追溯——`LaneCommitProvenance`

```rust:rust/crates/runtime/src/lane_events.rs
pub struct LaneCommitProvenance {
    pub commit: String,
    pub branch: String,
    pub worktree: Option<String>,
    pub canonical_commit: Option<String>,
    pub superseded_by: Option<String>,
    pub lineage: Vec<String>,
}
```

这个结构追踪一个 commit 的完整"家谱"：它是在哪个分支上创建的、是否在 worktree 中、是否被后续 commit 替代、它的祖先链是什么。这对于自动化场景（如 CI 集成）至关重要——系统需要知道一个 commit 是从哪条工作流中产生的。

---

## 第十二部分：TaskRegistry——子任务的生命周期管理

### 12.1 从 Turn 到 Task

一个 Agent 的工作可能涉及多个子任务。`TaskRegistry` 管理这些子任务的生命周期：

```rust:rust/crates/runtime/src/task_registry.rs
pub enum TaskStatus {
    Created,    // 已创建，等待执行
    Running,    // 正在执行
    Completed,  // 执行完成
    Failed,     // 执行失败
    Stopped,    // 被手动停止
}
```

```rust:rust/crates/runtime/src/task_registry.rs
pub struct Task {
    pub task_id: String,
    pub prompt: String,
    pub description: Option<String>,
    pub task_packet: Option<TaskPacket>,
    pub status: TaskStatus,
    pub created_at: u64,
    pub updated_at: u64,
    pub messages: Vec<TaskMessage>,
    pub output: String,
    pub team_id: Option<String>,
}
```

### 12.2 Task 的创建和状态转换

```
Created ──▶ Running ──▶ Completed
    │           │
    │           └──▶ Failed
    │
    └──▶ Stopped
```

关键操作：

```rust:rust/crates/runtime/src/task_registry.rs
impl TaskRegistry {
    // 创建任务
    pub fn create(&self, prompt: &str, description: Option<&str>) -> Task;

    // 从 TaskPacket 创建（带验证）
    pub fn create_from_packet(&self, packet: TaskPacket) -> Result<Task, TaskPacketValidationError>;

    // 更新任务状态
    pub fn set_status(&self, task_id: &str, status: TaskStatus) -> Result<(), String>;

    // 追加输出
    pub fn append_output(&self, task_id: &str, output: &str) -> Result<(), String>;

    // 停止任务
    pub fn stop(&self, task_id: &str) -> Result<Task, String>;
}
```

### 12.3 TaskPacket——标准化的任务描述

```rust:rust/crates/runtime/src/task_packet.rs
pub struct TaskPacket {
    // 任务的目标描述
    pub objective: String,
    // 工作范围
    pub scope: String,
    // 来源（哪个 surface 创建的）
    pub source_surface: String,
    // 预期产物
    pub expected_artifacts: Vec<String>,
}
```

`TaskPacket` 是一种**标准化的任务描述格式**，确保无论任务从哪里创建（CLI、API、定时任务），都有统一的结构。在创建前会经过验证：

```rust:rust/crates/runtime/src/task_packet.rs
pub fn validate_packet(packet: TaskPacket) -> Result<ValidatedPacket, TaskPacketValidationError>;
```

---

## 第十三部分：UsageTracker——Token 用量与成本追踪

### 13.1 四维度计费模型

```rust:rust/crates/runtime/src/usage.rs
pub struct TokenUsage {
    pub input_tokens: u32,                      // 输入 token
    pub output_tokens: u32,                     // 输出 token
    pub cache_creation_input_tokens: u32,       // 缓存创建 token
    pub cache_read_input_tokens: u32,           // 缓存读取 token
}
```

不同类型 token 的价格差异巨大（以 Sonnet 为例）：

| Token 类型 | 每百万 token 价格 | 相对倍数 |
|---|---|---|
| 输入 | $15.00 | 基准 |
| 输出 | $75.00 | 5× |
| 缓存创建 | $18.75 | 1.25× |
| 缓存读取 | $1.50 | **0.1×** |

**缓存读取比普通输入便宜 10 倍！** 这就是为什么 `PromptCacheEvent` 被仔细追踪——缓存命中率直接影响成本。

### 13.2 模型定价

```rust:rust/crates/runtime/src/usage.rs
pub fn pricing_for_model(model: &str) -> Option<ModelPricing> {
    if normalized.contains("haiku") {
        return Some(ModelPricing { input: 1.0, output: 5.0, ... });
    }
    if normalized.contains("opus") {
        return Some(ModelPricing { input: 15.0, output: 75.0, ... });
    }
    if normalized.contains("sonnet") {
        return Some(ModelPricing::default_sonnet_tier());
    }
    None  // 未知模型
}
```

定价差异对比：

| 模型 | 输入 | 输出 | 缓存创建 | 缓存读取 |
|---|---|---|---|---|
| Haiku | $1.00 | $5.00 | $1.25 | $0.10 |
| Sonnet | $15.00 | $75.00 | $18.75 | $1.50 |
| Opus | $15.00 | $75.00 | $18.75 | $1.50 |

### 13.3 从 Session 恢复用量

```rust:rust/crates/runtime/src/conversation.rs
// ConversationRuntime 构造时从 session 恢复用量
let usage_tracker = UsageTracker::from_session(&session);
```

`UsageTracker` 支持从已有的 Session 中恢复累计用量——这意味着即使 Agent 重启，用量统计也不会丢失。这对于长时间运行的任务和成本控制至关重要。

---

## 第十四部分：Config——三层配置合并

### 14.1 三级配置优先级

```rust:rust/crates/runtime/src/config.rs
pub enum ConfigSource {
    User,      // 用户级：~/.claude/settings.json
    Project,   // 项目级：.claude/settings.json
    Local,     // 本地级：.claude/settings.local.json
}
```

优先级从低到高：

```
User（全局默认） → Project（项目覆盖） → Local（个人覆盖）
```

### 14.2 RuntimeFeatureConfig——功能配置集

```rust:rust/crates/runtime/src/config.rs
pub struct RuntimeFeatureConfig {
    hooks: RuntimeHookConfig,                    // Hook 配置
    plugins: RuntimePluginConfig,                // 插件配置
    mcp: McpConfigCollection,                    // MCP 服务器配置
    oauth: Option<OAuthConfig>,                  // OAuth 配置
    model: Option<String>,                       // 模型选择
    aliases: BTreeMap<String, String>,           // 模型别名
    permission_mode: Option<ResolvedPermissionMode>, // 权限模式
    permission_rules: RuntimePermissionRuleConfig,   // 权限规则
    sandbox: SandboxConfig,                      // 沙箱配置
    provider_fallbacks: ProviderFallbackConfig,  // 模型降级链
    trusted_roots: Vec<String>,                  // 信任根目录列表
}
```

`RuntimeFeatureConfig` 包含了 Runtime 运行所需的全部配置。它从三级配置文件中合并而来，每级可以覆盖上一级的设置。

### 14.3 ProviderFallbackConfig——模型降级链

```rust:rust/crates/runtime/src/config.rs
pub struct ProviderFallbackConfig {
    primary: Option<String>,      // 首选模型
    fallbacks: Vec<String>,       // 降级模型列表
}
```

当首选模型不可用（429 限速、500 服务器错误、503 维护）时，按顺序尝试降级模型：

```
claude-opus-4 → claude-sonnet-4 → claude-haiku-4 → 报错
```

---

## 总结：Agent Runtime 的设计哲学

### 核心架构回顾

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Runtime 的七大设计决策                 │
│                                                              │
│  ① Trait 解耦         ApiClient / ToolExecutor / Prompter   │
│                       ↕ 不绑死具体实现                         │
│                                                              │
│  ② 管线式工具执行     PreHook → Permission → Execute → PostHook│
│                       ↕ 每层有否决权                           │
│                                                              │
│  ③ Builder 模式       必填参数构造 + 可选参数链式配置            │
│                       ↕ 灵活且类型安全                         │
│                                                              │
│  ④ 事件驱动           LaneEvents / RecoveryEvents / Telemetry │
│                       ↕ 全链路可观测                           │
│                                                              │
│  ⑤ 防御性编程         健康探针 / 短路 Hook / 安全阀迭代限制      │
│                       ↕ 永远不假设上游完美                      │
│                                                              │
│  ⑥ 故障自愈           RecoveryRecipes / PromptReplay          │
│                       ↕ 先自动恢复，再升级人工                  │
│                                                              │
│  ⑦ 配置分层           User → Project → Local                 │
│                       ↕ 个人偏好不覆盖团队规范                  │
└──────────────────────────────────────────────────────────────┘
```

### 模块间的依赖关系

```
                    Config (三层合并)
                       │
                       ▼
               SystemPromptBuilder ──▶ system_prompt: Vec<String>
                       │
                       ▼
             ┌─────────────────────┐
             │ ConversationRuntime  │◀─── Session (唯一真相源)
             │                     │◀─── UsageTracker (用量)
             │   ┌─ ApiClient      │◀─── PermissionPolicy (权限)
             │   ├─ ToolExecutor   │◀─── HookRunner (拦截)
             │   ├─ HookAbortSignal│
             │   └─ SessionTracer │
             └─────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     WorkerBoot    TaskRegistry   LaneEvents
     (启动管理)    (子任务)       (事件流)
          │
          ▼
     RecoveryRecipes
     (故障恢复)
```

### 关键数据流

| 数据 | 从哪来 | 到哪去 | 经过了什么 |
|---|---|---|---|
| 用户输入 | CLI | Session.messages | push_user_text → Agent Loop |
| API 响应 | ApiClient.stream() | Session.messages | 事件聚合 → assistant_message |
| 工具调用 | 模型输出 | ToolExecutor | PreHook → Permission → Execute → PostHook |
| 工具结果 | ToolExecutor.execute() | Session.messages | ToolResult 消息 |
| Token 用量 | ApiClient 的 Usage 事件 | UsageTracker | 累计记录 + 成本估算 |
| 系统提示词 | SystemPromptBuilder.build() | ApiRequest.system_prompt | 每次调用都构建 |
| 配置 | 三级配置文件 | RuntimeFeatureConfig | 合并 + 解析 + 验证 |
| 故障事件 | WorkerBoot / LaneEvents | RecoveryRecipes | 自动恢复 → 升级 |

### 从"能用"到"工业级"

如果把 Agent Runtime 比作一个微型的操作系统内核，那么：

| 操作系统概念 | Agent Runtime 对应 | 本章覆盖 |
|---|---|---|
| 进程管理 | WorkerBoot 状态机 | 第九部分 |
| 系统调用 | ToolExecutor trait | 第五部分 |
| 文件系统 | Session + JSONL 持久化 | 第06章 |
| 权限管理 | PermissionPolicy + Enforcer | 第七部分 |
| 中断处理 | Hook 系统 | 第六部分 |
| 设备驱动 | ApiClient trait | 第三部分 |
| 系统配置 | Config 三层合并 | 第十四部分 |
| 错误恢复 | Recovery Recipes | 第十部分 |
| 审计日志 | LaneEvents + Telemetry | 第十一部分 |

这就是 Agent Runtime 的全貌——它不是一个简单的循环，而是一套精心设计的工程系统，让 Agent 能在真实世界中**安全、可靠、可观测**地运行。
