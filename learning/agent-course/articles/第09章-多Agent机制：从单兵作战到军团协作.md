# 第09章 多 Agent 机制：从单兵作战到军团协作

> 本文是 claw-code 学习系列的第九篇，聚焦于 Agent 系统中最具扩展性的架构——**多 Agent 机制**。
> 前一章我们理解了 Agent Runtime 如何驱动单个 Agent 的完整生命周期。但现实世界的问题往往不是一个人能解决的：代码审查需要同时检查安全性、性能、可读性；大规模重构需要并行修改多个模块；复杂调研需要从不同角度搜索和验证。
> **多 Agent 机制让 claw-code 从"一个人的工具"进化为"一个协作团队"。** 主 Agent 可以按需派出专门的子 Agent，每个子 Agent 拥有独立的能力、隔离的状态和明确的目标。
> **本文面向不懂 Rust 的读者**：每段代码后都有「读懂这段 Rust」小节，只解释理解多 Agent 所必需的语法。

读完本章，你应该能回答八件事：

1. 为什么需要多 Agent？单 Agent 的局限性和多 Agent 的优势是什么？
2. claw-code 定义了哪些 Agent 类型？每种类型的工具集和能力边界是什么？
3. 子 Agent 的创建流程是什么？`execute_agent` → `spawn_agent_job` → `run_agent_job` 三级调用链如何工作？
4. `SubagentToolExecutor` 如何实现工具隔离？为什么子 Agent 不能使用父 Agent 的全部工具？
5. 子 Agent 如何与主 Agent 通信？`AgentInput`/`AgentOutput`/`AgentJob` 三个核心结构体的关系是什么？
6. Team 和 Task 如何支持多 Agent 的编排与并行执行？
7. Agent 的状态如何持久化？`manifest_file` 和 `output_file` 的作用是什么？
8. Worktree 隔离如何让多个 Agent 同时修改代码而不冲突？

---

## 与前几章的关系

| 主题 | 前几章已覆盖 | 本章新增深入 |
|---|---|---|
| Agent Runtime | 单 Agent 的 Turn 生命周期 | 多 Agent 的创建、调度、隔离 |
| ToolExecutor | trait 抽象和 Hook 集成 | `SubagentToolExecutor` 的工具过滤与权限隔离 |
| TaskRegistry | 子任务的状态管理 | Team 编排、并行执行、Cron 调度 |
| PermissionPolicy | 权限模式与规则引擎 | 子 Agent 的权限继承与限制 |
| Session | 会话状态与持久化 | 子 Agent 的独立会话与状态持久化 |
| WorkerBoot | Worker 启动状态机 | 多 Worker 并行管理 |

---

## Rust 语法速查（本章新增）

| 符号 / 写法 | 含义 | 本章出现的场景 |
|---|---|---|
| `BTreeSet<String>` | 有序字符串集合（自动去重） | `allowed_tools`——子 Agent 可用的工具名集合 |
| `std::thread::Builder::new().name(...).spawn(...)` | 创建命名线程 | 子 Agent 的独立执行线程 |
| `std::panic::catch_unwind(AssertUnwindSafe(\|\| ...))` | 捕获 panic，防止线程崩溃影响父线程 | 子 Agent 线程的 panic 恢复 |
| `match Ok(Ok(())) => {}` | 嵌套 Result 匹配 | 子 Agent 线程执行结果的三层解包 |
| `trait Canonicalize { fn canonicalize(&self) -> String; }` | 规范化接口 | Agent 类型名称的标准化 |

---

## 第一部分：问题——为什么需要多 Agent？

### 1.1 单 Agent 的局限性

假设你让 Agent "审查这个 PR"。一个 Agent 需要同时关注：

| 维度 | 需要检查的内容 | 所需工具 |
|---|---|---|
| 正确性 | 逻辑错误、边界条件、空指针 | Read、Bash（运行测试） |
| 安全性 | SQL 注入、XSS、权限绕过 | Read、Grep（模式匹配） |
| 性能 | N+1 查询、内存泄漏、死循环 | Read、Bash（profiling） |
| 风格 | 命名规范、注释质量、代码重复 | Read、Grep |

问题是：**一个 Agent 的上下文窗口是有限的**。它不可能同时深入所有维度。结果往往是"每个维度都检查了一点，但没有一个做到位"。

### 1.2 多 Agent 的核心思想

多 Agent 的设计哲学可以用一句话概括：

> **让每个 Agent 专注于一个明确的任务，用隔离保证安全，用编排实现协作。**

```
                ┌─────────────────┐
                │   主 Agent       │
                │  (协调者/调度器)  │
                └────────┬────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ 安全审查员  │ │ 性能分析师  │ │ 风格审查员  │
   │ Agent      │ │ Agent      │ │ Agent      │
   │ (只读+Grep)│ │ (只读+Bash)│ │ (只读+Read)│
   └────────────┘ └────────────┘ └────────────┘
         │              │              │
         └──────────────┼──────────────┘
                        ▼
                 汇总报告给主 Agent
```

每个子 Agent 的特点：

| 特性 | 说明 |
|---|---|
| **类型明确** | 预定义的 `subagent_type`，决定了可用工具和系统提示词 |
| **工具受限** | 只能使用被授权的工具集，不能越权 |
| **状态隔离** | 独立的 Session，不与主 Agent 共享对话历史 |
| **线程独立** | 在自己的线程中运行，不阻塞主 Agent |
| **目标单一** | 系统提示词明确限定"只做被委派的任务" |

### 1.3 设计目标

claw-code 的多 Agent 系统追求五个目标：

| 目标 | 具体含义 |
|---|---|
| **专业化** | 不同类型的 Agent 有不同的工具集和提示词，各司其职 |
| **隔离性** | 子 Agent 不能访问父 Agent 的全部工具，防止越权 |
| **并行性** | 多个子 Agent 可以同时运行，缩短总耗时 |
| **可观测** | 每个 Agent 的状态、输出、事件都有独立的持久化记录 |
| **容错性** | 子 Agent 崩溃不影响主 Agent，panic 被捕获并记录 |

---

## 第二部分：全局架构——多 Agent 的模块地图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     多 Agent 系统架构                                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 主 Agent (Main Agent)                                        │  │
│  │                                                              │  │
│  │  用户请求: "审查这个 PR 的安全性、性能和风格"                   │  │
│  │       │                                                      │  │
│  │       ▼                                                      │  │
│  │  ┌─────────────────────────────────────┐                     │  │
│  │  │ execute_agent(AgentInput)            │                     │  │
│  │  │   ├─ description: "安全性审查"        │                     │  │
│  │  │   ├─ prompt: "检查 SQL 注入..."      │                     │  │
│  │  │   ├─ subagent_type: "Explore"        │                     │  │
│  │  │   └─ model: None (继承主模型)        │                     │  │
│  │  └─────────────┬───────────────────────┘                     │  │
│  │                │                                              │  │
│  │                ▼                                              │  │
│  │  ┌─────────────────────────────────────┐                     │  │
│  │  │ 创建 AgentOutput (元数据/清单)       │                     │  │
│  │  │ 创建 AgentJob (完整执行包)           │                     │  │
│  │  │   ├─ manifest: AgentOutput           │                     │  │
│  │  │   ├─ prompt: 具体任务描述            │                     │  │
│  │  │   ├─ system_prompt: 系统提示词        │                     │  │
│  │  │   └─ allowed_tools: 允许的工具集     │                     │  │
│  │  └─────────────┬───────────────────────┘                     │  │
│  │                │                                              │  │
│  │     ┌──────────┴──────────┐                                   │  │
│  │     ▼                     ▼                                   │  │
│  │  ┌────────────┐    ┌────────────┐                            │  │
│  │  │ 线程 1      │    │ 线程 2      │                           │  │
│  │  │ 安全审查员  │    │ 性能分析师  │                            │  │
│  │  │            │    │            │                            │  │
│  │  │ Subagent   │    │ Subagent   │                            │  │
│  │  │ ToolExec.  │    │ ToolExec.  │                            │  │
│  │  │ ├─ allowed │    │ ├─ allowed │                            │  │
│  │  │ │  _tools  │    │ │  _tools  │                            │  │
│  │  │ └─ enforcer│    │ └─ enforcer│                            │  │
│  │  └────────────┘    └────────────┘                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 全局注册表 (Global Registries)                                │  │
│  │                                                              │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │  │
│  │  │TaskRegistry│  │TeamRegistry│  │CronRegistry│             │  │
│  │  │(任务管理)  │  │(团队编排)  │  │(定时调度)  │             │  │
│  │  └────────────┘  └────────────┘  └────────────┘             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ 持久化层 (Persistence)                                       │  │
│  │                                                              │  │
│  │  manifest_file: .claude/agents/{id}/manifest.json            │  │
│  │  output_file:   .claude/agents/{id}/output.txt               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

关键模块分布在两个 crate 中：

| Crate | 文件 | 职责 |
|---|---|---|
| `tools` | `lib.rs` | Agent 工具实现：`execute_agent`、`spawn_agent_job`、`SubagentToolExecutor` |
| `runtime` | `task_registry.rs` | 任务生命周期管理 |
| `runtime` | `team_cron_registry.rs` | Team 编排和 Cron 调度 |
| `runtime` | `worker_boot.rs` | Worker 启动和状态管理 |
| `runtime` | `prompt.rs` | 系统提示词构建（含子 Agent 专用提示词） |

---

## 第三部分：Agent 类型——专业化的基石

### 3.1 六种预定义 Agent 类型

claw-code 定义了六种 Agent 类型，每种类型对应不同的工具集和使用场景：

```rust:rust/crates/tools/src/lib.rs
fn normalize_subagent_type(subagent_type: Option<&str>) -> String {
    let trimmed = subagent_type.map(str::trim).unwrap_or_default();
    if trimmed.is_empty() {
        return String::from("general-purpose");
    }

    match canonical_tool_token(trimmed).as_str() {
        "general" | "generalpurpose" | "generalpurposeagent" => String::from("general-purpose"),
        "explore" | "explorer" | "exploreagent" => String::from("Explore"),
        "plan" | "planagent" => String::from("Plan"),
        "verification" | "verificationagent" | "verify" | "verifier" => {
            String::from("Verification")
        }
        "clawguide" | "clawguideagent" | "guide" => String::from("claw-guide"),
        "statusline" | "statuslinesetup" => String::from("statusline-setup"),
        _ => trimmed.to_string(),
    }
}
```

**读懂这段 Rust**

- `Option<&str>` 是"可能有的字符串引用"——`None` 表示未指定类型。
- `canonical_tool_token(trimmed).as_str()` 把输入标准化为小写无符号的形式——用户输入 "Explore"、"explore"、"EXPLORER" 都能匹配。
- `_ => trimmed.to_string()` 是通配分支——未知类型保持原样，允许自定义 Agent 类型。

**类型标准化表**：

| 用户输入 | 标准化结果 | 含义 |
|---|---|---|
| 未指定 / `""` | `general-purpose` | 默认的通用 Agent |
| `general` / `GeneralPurpose` | `general-purpose` | 通用 Agent |
| `explore` / `Explore` / `Explorer` | `Explore` | 探索型 Agent |
| `plan` / `Plan` | `Plan` | 规划型 Agent |
| `verification` / `verify` | `Verification` | 验证型 Agent |
| `claw-guide` / `guide` | `claw-guide` | 引导型 Agent |
| `statusline` | `statusline-setup` | 状态栏配置 Agent |

### 3.2 每种 Agent 类型的工具集

不同类型的 Agent 拥有不同的工具集。这是**最小权限原则**的体现——每个 Agent 只能访问完成其任务所必需的工具：

```
┌─────────────────────────────────────────────────────────────────┐
│                     工具集分层                                    │
│                                                                  │
│  Layer 0: 基础只读工具（所有 Agent 共享）                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Read, Glob, Grep, WebFetch, WebSearch, ToolSearch,       │  │
│  │ Skill, StructuredOutput                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│  Layer 1: Plan 增强工具    │  Layer 1: Verification 增强工具      │
│  ┌──────────────────┐    │  ┌──────────────────────────────┐  │
│  │ TodoWrite,       │    │  │ bash, PowerShell,            │  │
│  │ SendUserMessage  │    │  │ TodoWrite, SendUserMessage   │  │
│  └──────────────────┘    │  └──────────────────────────────┘  │
│                           │                                      │
│  Layer 2: general-purpose 全部工具                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 所有工具：bash, Read, Write, Edit, Glob, Grep,            │  │
│  │ NotebookEdit, WebFetch, WebSearch, Agent, Skill, ...     │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

| Agent 类型 | 可用工具 | 典型用途 |
|---|---|---|
| `Explore` | Read, Glob, Grep, WebFetch, WebSearch, ToolSearch, Skill, StructuredOutput | 代码搜索、文档查询、信息收集 |
| `Plan` | Explore 的全部 + TodoWrite, SendUserMessage | 制定实施计划、与用户确认方案 |
| `Verification` | Explore 的全部 + bash, PowerShell, TodoWrite, SendUserMessage | 运行测试、验证修复、检查结果 |
| `claw-guide` | 受限的工具集 | Claude Code 使用指南和配置帮助 |
| `statusline-setup` | bash, Read, Write, Edit, Glob, Grep, ToolSearch | 配置状态栏设置 |
| `general-purpose` | **所有工具** | 任何不需要特殊限制的任务 |

**为什么要限制工具？**

| 场景 | 如果不限制 | 限制后的效果 |
|---|---|---|
| 探索型 Agent | 可能意外修改文件 | 只能读，不可能破坏 |
| 验证型 Agent | 可能引入新的变更 | 能运行测试但不能写代码 |
| 规划型 Agent | 可能直接开始实施 | 只能做计划，不能动手 |

### 3.3 Agent 类型的选择策略

```
用户请求
    │
    ▼
┌─────────────────────────────┐
│ 需要修改文件？               │── 是 ──▶ general-purpose
└──────────┬──────────────────┘
           │ 否
           ▼
┌─────────────────────────────┐
│ 需要运行命令？               │── 是 ──▶ Verification
└──────────┬──────────────────┘
           │ 否
           ▼
┌─────────────────────────────┐
│ 需要制定计划？               │── 是 ──▶ Plan
└──────────┬──────────────────┘
           │ 否
           ▼
       Explore（纯搜索/分析）
```

---

## 第四部分：三级调用链——从请求到执行

子 Agent 的创建和执行是一个**三级调用链**：

```
execute_agent(input)           ← 第一级：入口，创建元数据
       │
       ▼
execute_agent_with_spawn(...)  ← 第二级：构建 AgentJob，确定工具集
       │
       ▼
spawn_agent_job(job)           ← 第三级：创建线程，启动执行
       │
       ▼
run_agent_job(job)             ← 第四级：在独立线程中运行 Agent Loop
```

### 4.1 第一级：`execute_agent`——入口与元数据创建

```rust:rust/crates/tools/src/lib.rs
struct AgentInput {
    description: String,        // 任务描述（用于日志和 UI 显示）
    prompt: String,             // 具体的任务指令（发送给子 Agent 的 prompt）
    subagent_type: Option<String>,  // Agent 类型（None = general-purpose）
    name: Option<String>,       // 可选的自定义名称
    model: Option<String>,      // 可选的模型覆盖
}
```

`execute_agent` 的核心工作是：

1. **标准化 Agent 类型**：调用 `normalize_subagent_type` 把用户输入转为规范名称
2. **生成唯一 Agent ID**：每个子 Agent 有唯一的标识符
3. **创建 AgentOutput**：记录元数据（ID、类型、状态、时间戳等）
4. **委托给 `execute_agent_with_spawn`**

```rust:rust/crates/tools/src/lib.rs
fn execute_agent(input: AgentInput) -> Result<AgentOutput, String> {
    execute_agent_with_spawn(input, spawn_agent_job)
}
```

**读懂这段 Rust**

- `execute_agent_with_spawn(input, spawn_agent_job)` 把 `spawn_agent_job` 函数作为参数传入——这是**依赖注入**，让测试时可以用 mock 函数替换真实的线程创建。

### 4.2 第二级：`execute_agent_with_spawn`——构建 AgentJob

```rust:rust/crates/tools/src/lib.rs
fn execute_agent_with_spawn<F>(
    input: AgentInput,
    spawn_fn: F,
) -> Result<AgentOutput, String>
where
    F: FnOnce(AgentJob) -> Result<(), String>,
{
    // ① 标准化 Agent 类型
    let subagent_type = normalize_subagent_type(input.subagent_type.as_deref());

    // ② 确定可用工具集
    let allowed_tools = resolve_allowed_tools(&subagent_type)?;

    // ③ 构建子 Agent 的系统提示词
    let system_prompt = build_agent_system_prompt(&subagent_type)?;

    // ④ 创建 AgentOutput（元数据/清单）
    let manifest = create_agent_manifest(
        &subagent_type,
        &input.description,
        input.model.as_deref(),
    )?;

    // ⑤ 组装 AgentJob（完整执行包）
    let job = AgentJob {
        manifest,
        prompt: input.prompt,
        system_prompt,
        allowed_tools,
    };

    // ⑥ 持久化初始状态
    persist_agent_manifest(&job.manifest)?;

    // ⑦ 委托给 spawn 函数
    spawn_fn(job)?;

    Ok(job.manifest)
}
```

**读懂这段 Rust**

- `where F: FnOnce(AgentJob) -> Result<(), String>` 是泛型约束——`F` 必须是"只调用一次的函数"，参数是 `AgentJob`，返回 `Result<(), String>`。
- `input.subagent_type.as_deref()` 把 `Option<String>` 转为 `Option<&str>`——因为 `normalize_subagent_type` 接受引用。

**六个步骤的意义**：

| 步骤 | 做了什么 | 为什么重要 |
|---|---|---|
| ① 标准化类型 | `"explore"` → `"Explore"` | 确保后续查找工具集时能匹配 |
| ② 确定工具集 | 根据 Agent 类型返回 `BTreeSet<String>` | 子 Agent 的能力边界 |
| ③ 构建提示词 | 加载基础提示词 + 追加子 Agent 类型说明 | 让子 Agent 知道自己的角色 |
| ④ 创建清单 | 生成唯一的 agent_id、时间戳、文件路径 | 追踪和持久化 |
| ⑤ 组装 AgentJob | 把 manifest + prompt + tools 打包 | 传递给执行线程的完整上下文 |
| ⑥ 持久化 | 写入 manifest.json | 即使崩溃也能恢复状态 |

### 4.3 第三级：`spawn_agent_job`——线程创建与隔离

```rust:rust/crates/tools/src/lib.rs
fn spawn_agent_job(job: AgentJob) -> Result<(), String> {
    let thread_name = format!("clawd-agent-{}", job.manifest.agent_id);
    std::thread::Builder::new()
        .name(thread_name)
        .spawn(move || {
            let result =
                std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| run_agent_job(&job)));
            match result {
                Ok(Ok(())) => {
                    // 正常完成：状态已在 run_agent_job 中更新
                }
                Ok(Err(error)) => {
                    // Agent 执行出错（非 panic）：记录错误
                    let _ =
                        persist_agent_terminal_state(&job.manifest, "failed", None, Some(error));
                }
                Err(_) => {
                    // Agent 线程 panic：记录崩溃
                    let _ = persist_agent_terminal_state(
                        &job.manifest,
                        "failed",
                        None,
                        Some(String::from("sub-agent thread panicked")),
                    );
                }
            }
        })
        .map(|_| ())
        .map_err(|error| error.to_string())
}
```

**读懂这段 Rust**

- `std::thread::Builder::new().name(thread_name).spawn(move || { ... })` 创建一个命名线程。
  - `move` 关键字把 `job` 的所有权移入闭包——闭包"拥有"这个 job。
  - `.name(thread_name)` 给线程命名，方便在调试器和日志中识别。
- `std::panic::catch_unwind(AssertUnwindSafe(|| run_agent_job(&job)))` 捕获 panic。
  - `AssertUnwindSafe` 告诉编译器"我确认这段代码在 panic 后状态仍然安全"。
  - 如果不捕获，子 Agent 的 panic 会导致整个线程崩溃，但不会影响主线程。
- `match result` 有三种情况：
  - `Ok(Ok(()))`：正常运行，没有错误
  - `Ok(Err(error))`：运行了但返回了错误（如工具执行失败）
  - `Err(_)`：发生了 panic（如数组越界、除以零）

**三层防护的意义**：

```
run_agent_job(&job)
       │
       ├─ 正常完成 ──▶ 状态已在内部更新为 "completed"
       │
       ├─ 返回 Err ──▶ persist_agent_terminal_state("failed", error)
       │                  状态被记录，主 Agent 可以看到失败原因
       │
       └─ panic ─────▶ persist_agent_terminal_state("failed", "panicked")
                          即使崩溃，也不会影响主 Agent
```

### 4.4 第四级：`run_agent_job`——Agent Loop 的执行

```rust:rust/crates/tools/src/lib.rs
fn run_agent_job(job: &AgentJob) -> Result<(), String> {
    // ① 创建子 Agent 专用的 ToolExecutor
    let tool_executor = SubagentToolExecutor::new(
        job.allowed_tools.clone(),
    );

    // ② 创建子 Agent 的 Session（独立于主 Agent）
    // ③ 创建 ApiClient（使用配置的模型或继承主模型）
    // ④ 创建 PermissionPolicy（子 Agent 的权限策略）
    // ⑤ 构建 ConversationRuntime
    // ⑥ 调用 runtime.run_turn(job.prompt)
    // ⑦ 收集结果，写入 output_file
    // ⑧ 更新 manifest 状态为 "completed"
    ...
}
```

这个函数的完整实现包含：

| 步骤 | 做了什么 |
|---|---|
| ① 创建 SubagentToolExecutor | 限制子 Agent 只能使用 `allowed_tools` 中的工具 |
| ② 创建独立 Session | 子 Agent 的对话历史与主 Agent 完全隔离 |
| ③ 创建 ApiClient | 可以使用不同的模型（如果指定了 `model` 覆盖） |
| ④ 创建 PermissionPolicy | 继承主 Agent 的权限模式，但工具集受限 |
| ⑤ 构建 ConversationRuntime | 复用第08章的 Runtime 架构 |
| ⑥ 执行 run_turn | 子 Agent 开始执行被委派的任务 |
| ⑦ 写入 output_file | 结果持久化到 `.claude/agents/{id}/output.txt` |
| ⑧ 更新状态 | manifest.json 中的状态更新为 `completed` 或 `failed` |

**关键洞察**：子 Agent 复用了完整的 `ConversationRuntime`！这意味着第08章学到的所有机制——Agent Loop、Hook 系统、权限策略、上下文压缩——在子 Agent 中同样生效。

---

## 第五部分：三个核心数据结构

子 Agent 的创建涉及三个核心数据结构，它们之间有清晰的层次关系：

```
AgentInput (用户的请求)
       │
       │ execute_agent() 处理
       ▼
AgentOutput (Agent 的元数据/清单)
       │
       │ 与 prompt + system_prompt + allowed_tools 组合
       ▼
AgentJob (完整的执行包)
       │
       │ 传递给 spawn_agent_job()
       ▼
独立线程中的 run_agent_job()
```

### 5.1 AgentInput——用户的请求

```rust:rust/crates/tools/src/lib.rs
struct AgentInput {
    description: String,            // 任务的简短描述（显示给用户）
    prompt: String,                 // 发送给子 Agent 的完整指令
    subagent_type: Option<String>,  // Agent 类型（None = general-purpose）
    name: Option<String>,           // 可选的自定义名称
    model: Option<String>,          // 可选的模型覆盖（如 "haiku"）
}
```

| 字段 | 谁提供的 | 用途 |
|---|---|---|
| `description` | 主 Agent | 在 UI 中显示"正在执行：安全性审查" |
| `prompt` | 主 Agent | 子 Agent 收到的实际任务描述 |
| `subagent_type` | 主 Agent 或用户 | 决定工具集和系统提示词 |
| `name` | 可选 | 用于标识和日志 |
| `model` | 可选 | 让子 Agent 用不同的模型（如用 Haiku 做简单任务省钱） |

### 5.2 AgentOutput——Agent 的元数据

```rust:rust/crates/tools/src/lib.rs
struct AgentOutput {
    agent_id: String,                   // 唯一标识符
    name: String,                       // 名称
    description: String,                // 任务描述
    subagent_type: Option<String>,      // Agent 类型
    model: Option<String>,              // 使用的模型
    status: String,                     // 当前状态："created"/"running"/"completed"/"failed"
    output_file: String,                // 输出文件路径
    manifest_file: String,              // 清单文件路径
    created_at: String,                 // 创建时间
    started_at: Option<String>,         // 开始执行时间
    completed_at: Option<String>,       // 完成时间
    lane_events: Vec<LaneEvent>,        // Lane 事件列表
    current_blocker: Option<LaneEventBlocker>,  // 当前阻塞事件
    derived_state: String,              // 派生状态（如 "green"/"red"）
    error: Option<String>,              // 错误信息（如果失败）
}
```

**读懂这段 Rust**

- `agent_id` 是全局唯一的——每次创建子 Agent 都会生成新的 ID。
- `status` 是字符串而非枚举——为了 JSON 序列化的灵活性。
- `lane_events: Vec<LaneEvent>` 记录了子 Agent 的完整事件流（第08章已覆盖）。

### 5.3 AgentJob——完整的执行包

```rust:rust/crates/tools/src/lib.rs
struct AgentJob {
    manifest: AgentOutput,             // 元数据/清单
    prompt: String,                    // 任务指令
    system_prompt: Vec<String>,        // 系统提示词（按段落）
    allowed_tools: BTreeSet<String>,   // 允许的工具名集合
}
```

**读懂这段 Rust**

- `BTreeSet<String>` 是有序的字符串集合——自动去重，查找效率为 O(log n)。
- `system_prompt: Vec<String>` 是按段落的提示词列表——与主 Agent 的 `SystemPromptBuilder.build()` 输出格式一致。

**三个结构体的关系**：

```
AgentInput ──────▶ (处理) ──────▶ AgentOutput
                                      │
                                      │ + prompt
                                      │ + system_prompt
                                      │ + allowed_tools
                                      ▼
                                  AgentJob
                                      │
                                      │ 传递给线程
                                      ▼
                                 run_agent_job()
```

---

## 第六部分：SubagentToolExecutor——工具隔离的核心

### 6.1 为什么需要工具隔离？

假设子 Agent 是一个 `Explore` 类型的探索员，它的任务是搜索代码中的安全漏洞。如果不限制工具集：

```
Explore Agent → 发现漏洞 → 想要"顺手修复" → 调用 Write/Edit 工具 → 修改了代码！
```

这违反了"探索员只探索，不修改"的原则。`SubagentToolExecutor` 就是防止这种情况的安全网。

### 6.2 SubagentToolExecutor 的结构

```rust:rust/crates/tools/src/lib.rs
struct SubagentToolExecutor {
    allowed_tools: BTreeSet<String>,              // 允许的工具名白名单
    enforcer: Option<PermissionEnforcer>,         // 权限执行器（可选）
}
```

### 6.3 工具执行的双重检查

```rust:rust/crates/tools/src/lib.rs
impl ToolExecutor for SubagentToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError> {
        // 第一重检查：工具是否在白名单中？
        if !self.allowed_tools.contains(tool_name) {
            return Err(ToolError::PermissionDenied {
                tool: tool_name.to_string(),
                reason: format!(
                    "Tool '{}' is not in the allowed tools set for this sub-agent",
                    tool_name
                ),
            });
        }

        // 第二重检查：权限执行器是否允许？
        if let Some(enforcer) = &self.enforcer {
            let result = enforcer.check(tool_name, input);
            if !result.is_allowed() {
                return Err(ToolError::PermissionDenied {
                    tool: tool_name.to_string(),
                    reason: result.reason().to_string(),
                });
            }
        }

        // 双重检查通过，执行工具
        self.execute_internal(tool_name, input)
    }
}
```

**读懂这段 Rust**

- `impl ToolExecutor for SubagentToolExecutor` 是为 `SubagentToolExecutor` 实现 `ToolExecutor` 接口——这意味着它可以替换主 Agent 的 `ToolExecutor`，无缝接入 `ConversationRuntime`。
- `self.allowed_tools.contains(tool_name)` 检查工具是否在白名单中——BTreeSet 的查找是 O(log n)。

**双重检查的设计**：

```
工具调用请求: Write("/etc/passwd", "hacked!")
       │
       ▼
┌─────────────────────────┐
│ 第一重：allowed_tools    │── "Write" 不在集合中 ──▶ PermissionDenied
│ (工具白名单)             │
└──────────┬──────────────┘
           │ 在白名单中
           ▼
┌─────────────────────────┐
│ 第二重：PermissionEnforcer│── 路径超出工作区 ──▶ PermissionDenied
│ (权限策略)               │
└──────────┬──────────────┘
           │ 允许
           ▼
      执行工具
```

### 6.4 各 Agent 类型的 allowed_tools

```rust:rust/crates/tools/src/lib.rs
fn resolve_allowed_tools(subagent_type: &str) -> Result<BTreeSet<String>, String> {
    // 基础只读工具集（所有 Agent 共享）
    let mut tools = BTreeSet::from([
        "Read".into(),
        "Glob".into(),
        "Grep".into(),
        "WebFetch".into(),
        "WebSearch".into(),
        "ToolSearch".into(),
        "Skill".into(),
        "StructuredOutput".into(),
    ]);

    match subagent_type {
        "Explore" => {
            // Explore: 只有基础工具
        }
        "Plan" => {
            // Plan: 基础工具 + 计划相关
            tools.insert("TodoWrite".into());
            tools.insert("SendUserMessage".into());
        }
        "Verification" => {
            // Verification: 基础工具 + 命令执行
            tools.insert("bash".into());
            tools.insert("PowerShell".into());
            tools.insert("TodoWrite".into());
            tools.insert("SendUserMessage".into());
        }
        "general-purpose" | _ => {
            // general-purpose: 所有工具
            return Ok(all_tools());
        }
    }

    Ok(tools)
}
```

工具集的层次关系一目了然：

```
                      所有工具
                         │
            ┌────────────┼────────────┐
            │            │            │
     general-purpose  Verification   Plan
      (全部工具)      (+bash等)     (+TodoWrite等)
                         │            │
                         └─────┬──────┘
                               │
                           Explore
                          (基础工具)
```

---

## 第七部分：子 Agent 的系统提示词

### 7.1 子 Agent 提示词的构建

```rust:rust/crates/tools/src/lib.rs
fn build_agent_system_prompt(subagent_type: &str) -> Result<Vec<String>, String> {
    let cwd = std::env::current_dir().map_err(|error| error.to_string())?;
    // 加载基础系统提示词（与主 Agent 相同的模板）
    let mut prompt = load_system_prompt(
        cwd,
        DEFAULT_AGENT_SYSTEM_DATE.to_string(),
        std::env::consts::OS,
        "unknown",
    )
    .map_err(|error| error.to_string())?;

    // 追加子 Agent 的角色定义
    prompt.push(format!(
        "You are a background sub-agent of type `{subagent_type}`. \
         Work only on the delegated task, use only the tools available to you, \
         do not ask the user questions, and finish with a concise result."
    ));

    Ok(prompt)
}
```

**读懂这段 Rust**

- `std::env::current_dir()` 获取当前工作目录——与主 Agent 在同一目录下工作。
- `std::env::consts::OS` 是编译时确定的操作系统名称——与平台相关的指令会被注入。
- `format!("...", subagent_type)` 用变量填充模板字符串。

这段追加的提示词有四个关键约束：

| 约束 | 含义 | 为什么 |
|---|---|---|
| "background sub-agent" | 你是后台运行的子 Agent | 子 Agent 不直接面向用户 |
| "type `{subagent_type}`" | 你的类型是 X | 子 Agent 知道自己的能力边界 |
| "Work only on the delegated task" | 只做被委派的任务 | 防止子 Agent "跑题" |
| "do not ask the user questions" | 不要问用户问题 | 子 Agent 没有交互能力 |

### 7.2 子 Agent 提示词与主 Agent 提示词的对比

| 方面 | 主 Agent | 子 Agent |
|---|---|---|
| 基础模板 | 相同 | 相同 |
| 环境信息 | 完整 | 完整 |
| 角色定义 | "你是 Claude Code" | "你是 background sub-agent of type X" |
| 用户交互 | 允许提问 | **禁止提问** |
| 任务范围 | 用户决定 | **仅限委派任务** |
| 工具描述 | 全部工具 | **仅允许的工具** |

---

## 第八部分：状态持久化——Agent 的"黑匣子"

### 8.1 为什么要持久化？

在多 Agent 系统中，持久化有三个目的：

1. **可恢复性**：即使主进程崩溃，子 Agent 的状态不会丢失
2. **可观测性**：用户和调试者可以查看每个子 Agent 的完整历史
3. **协调性**：主 Agent 可以查询子 Agent 的状态来决定下一步

### 8.2 持久化的文件结构

```
.claude/agents/
    ├── {agent_id_1}/
    │   ├── manifest.json      ← AgentOutput 的 JSON 序列化
    │   └── output.txt         ← 子 Agent 的输出结果
    ├── {agent_id_2}/
    │   ├── manifest.json
    │   └── output.txt
    └── ...
```

### 8.3 manifest.json 的内容

`manifest.json` 是 `AgentOutput` 的 JSON 序列化，记录了子 Agent 的完整元数据：

```json
{
  "agent_id": "agent-a1b2c3d4",
  "name": "security-reviewer",
  "description": "安全性审查",
  "subagent_type": "Explore",
  "model": null,
  "status": "completed",
  "output_file": ".claude/agents/agent-a1b2c3d4/output.txt",
  "manifest_file": ".claude/agents/agent-a1b2c3d4/manifest.json",
  "created_at": "2024-01-15T10:30:00Z",
  "started_at": "2024-01-15T10:30:01Z",
  "completed_at": "2024-01-15T10:32:45Z",
  "lane_events": [
    { "event": "started", "timestamp": "..." },
    { "event": "green", "timestamp": "..." },
    { "event": "finished", "timestamp": "..." }
  ],
  "current_blocker": null,
  "derived_state": "green",
  "error": null
}
```

### 8.4 状态更新的时机

```rust:rust/crates/tools/src/lib.rs
fn persist_agent_terminal_state(
    manifest: &AgentOutput,
    status: &str,
    output: Option<String>,
    error: Option<String>,
) -> Result<(), String> {
    let mut updated = manifest.clone();
    updated.status = status.to_string();
    updated.error = error;

    if status == "completed" {
        updated.completed_at = Some(now_iso8601());
    }

    if let Some(output) = output {
        // 写入 output 文件
        std::fs::write(&updated.output_file, output)
            .map_err(|error| error.to_string())?;
    }

    // 写入 manifest 文件
    let json = serde_json::to_string_pretty(&updated)
        .map_err(|error| error.to_string())?;
    std::fs::write(&updated.manifest_file, json)
        .map_err(|error| error.to_string())?;

    Ok(())
}
```

**读懂这段 Rust**

- `serde_json::to_string_pretty(&updated)` 把结构体序列化为格式化的 JSON 字符串。
- `std::fs::write(&path, content)` 是原子的文件写入——要么全部写入，要么不写入。

状态更新的生命周期：

```
created ──▶ manifest.json 写入初始状态
    │
    ▼
running ──▶ manifest.json 更新 started_at
    │
    ├─▶ completed ──▶ manifest.json 更新 completed_at + output.txt 写入
    │
    └─▶ failed ──▶ manifest.json 更新 error 字段
```

---

## 第九部分：Team 编排——多 Agent 的协作模式

### 9.1 从单个 Task 到 Team 编排

单个子 Agent 解决的是"一个任务"。但现实中，一个请求往往需要多个子 Agent 协作。`Team` 和 `Task` 提供了编排能力：

```
用户请求: "全面审查这个 PR"
           │
           ▼
    ┌─────────────┐
    │ Team "PR审查" │
    │              │
    │  ┌────────┐  │
    │  │ Task 1 │  │──▶ Agent: 安全性审查 (Explore)
    │  └────────┘  │
    │  ┌────────┐  │
    │  │ Task 2 │  │──▶ Agent: 性能分析 (Verification)
    │  └────────┘  │
    │  ┌────────┐  │
    │  │ Task 3 │  │──▶ Agent: 风格检查 (Explore)
    │  └────────┘  │
    └─────────────┘
```

### 9.2 Team 的数据结构

```rust:rust/crates/runtime/src/team_cron_registry.rs
struct Team {
    pub team_id: String,         // 团队唯一标识
    pub name: String,            // 团队名称
    pub task_ids: Vec<String>,   // 属于这个团队的 Task ID 列表
    pub status: TeamStatus,      // 团队状态
    pub created_at: u64,         // 创建时间
    pub updated_at: u64,         // 更新时间
}

enum TeamStatus {
    Created,    // 已创建，等待启动
    Running,    // 正在执行（至少一个 Task 在运行）
    Completed,  // 全部完成
    Deleted,    // 已删除
}
```

### 9.3 Task 的数据结构

```rust:rust/crates/runtime/src/task_registry.rs
struct Task {
    pub task_id: String,                // 任务唯一标识
    pub prompt: String,                 // 任务指令
    pub description: Option<String>,    // 任务描述
    pub task_packet: Option<TaskPacket>, // 标准化任务描述
    pub status: TaskStatus,             // 任务状态
    pub created_at: u64,
    pub updated_at: u64,
    pub messages: Vec<TaskMessage>,     // 任务消息历史
    pub output: String,                 // 任务输出
    pub team_id: Option<String>,        // 所属团队 ID
}

enum TaskStatus {
    Created,    // 已创建
    Running,    // 运行中
    Completed,  // 已完成
    Failed,     // 已失败
    Stopped,    // 已停止
}
```

### 9.4 Team 与 Task 的生命周期

```
Team 创建
    │
    ▼
┌──────────────────────────────────────────┐
│ TeamStatus: Created                       │
│   ┌──────┐ ┌──────┐ ┌──────┐            │
│   │Task 1│ │Task 2│ │Task 3│            │
│   │Created│ │Created│ │Created│           │
│   └──┬───┘ └──┬───┘ └──┬───┘            │
└──────┼────────┼────────┼────────────────┘
       │        │        │
       ▼        ▼        ▼
┌──────────────────────────────────────────┐
│ TeamStatus: Running                       │
│   ┌──────┐ ┌──────┐ ┌──────┐            │
│   │Task 1│ │Task 2│ │Task 3│            │
│   │Running│ │Running│ │Running│           │
│   └──┬───┘ └──┬───┘ └──┬───┘            │
└──────┼────────┼────────┼────────────────┘
       │        │        │
       ▼        ▼        ▼
┌──────────────────────────────────────────┐
│ TeamStatus: Running                       │
│   ┌──────┐ ┌──────┐ ┌──────┐            │
│   │Task 1│ │Task 2│ │Task 3│            │
│   │Compl.│ │Running│ │Compl.│           │
│   └──────┘ └──┬───┘ └──────┘            │
└──────────────┼──────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ TeamStatus: Completed                     │
│   ┌──────┐ ┌──────┐ ┌──────┐            │
│   │Task 1│ │Task 2│ │Task 3│            │
│   │Compl.│ │Compl.│ │Compl.│           │
│   └──────┘ └──────┘ └──────┘            │
└──────────────────────────────────────────┘
```

Team 的状态转换规则：

```
Created ──▶ 至少一个 Task 进入 Running ──▶ Running
Running ──▶ 全部 Task 进入 Completed ──▶ Completed
任意 ──▶ 手动删除 ──▶ Deleted
```

### 9.5 Task 的创建方式

```rust:rust/crates/runtime/src/task_registry.rs
impl TaskRegistry {
    // 方式一：直接创建
    pub fn create(&self, prompt: &str, description: Option<&str>) -> Task {
        // 生成唯一 task_id，初始化状态为 Created
    }

    // 方式二：从 TaskPacket 创建（带验证）
    pub fn create_from_packet(&self, packet: TaskPacket) -> Result<Task, TaskPacketValidationError> {
        // 验证 TaskPacket 的字段
        // 创建 Task 并关联 TaskPacket
    }

    // 状态更新
    pub fn set_status(&self, task_id: &str, status: TaskStatus) -> Result<(), String>;

    // 追加输出
    pub fn append_output(&self, task_id: &str, output: &str) -> Result<(), String>;

    // 停止任务
    pub fn stop(&self, task_id: &str) -> Result<Task, String>;
}
```

### 9.6 TaskPacket——标准化的任务描述

```rust:rust/crates/runtime/src/task_packet.rs
struct TaskPacket {
    pub objective: String,             // 任务目标
    pub scope: String,                 // 工作范围
    pub source_surface: String,        // 创建来源（CLI / API / Cron）
    pub expected_artifacts: Vec<String>, // 预期产物
}
```

`TaskPacket` 的设计意图是**统一任务的描述格式**。无论任务是从哪里创建的，都有相同的结构：

| 字段 | 示例 | 用途 |
|---|---|---|
| `objective` | "审查 PR #42 的安全性" | 明确目标 |
| `scope` | "src/auth/ 目录下的所有修改文件" | 限定范围 |
| `source_surface` | "cli" | 追溯来源 |
| `expected_artifacts` | ["security-report.md"] | 预期产出 |

---

## 第十部分：Cron 调度——定时 Agent

### 10.1 为什么需要定时 Agent？

在持续集成场景中，你可能需要 Agent 定期执行某些任务：

| 场景 | 调度 | Agent 类型 |
|---|---|---|
| 每日依赖检查 | `0 9 * * *`（每天 9:00） | `Explore` |
| 每小时健康检查 | `0 * * * *`（每小时） | `Verification` |
| 每周代码审计 | `0 9 * * 1`（每周一 9:00） | `general-purpose` |

### 10.2 CronEntry 的数据结构

```rust:rust/crates/runtime/src/team_cron_registry.rs
struct CronEntry {
    pub cron_id: String,            // Cron 任务唯一标识
    pub schedule: String,           // Cron 表达式（如 "0 9 * * *"）
    pub prompt: String,             // 每次触发的任务指令
    pub description: Option<String>, // 描述
    pub enabled: bool,              // 是否启用
    pub created_at: u64,
    pub updated_at: u64,
    pub last_run_at: Option<u64>,   // 上次运行时间
    pub run_count: u64,             // 累计运行次数
}
```

**读懂这段 Rust**

- `schedule: String` 存储标准的 5 字段 cron 表达式。
- `enabled: bool` 允许暂停/恢复定时任务而不删除它。
- `run_count: u64` 追踪累计运行次数——用于监控和限制。

### 10.3 Cron 的时间模型

```
Cron 表达式: "0 9 * * 1-5"
              │ │ │ │ │
              │ │ │ │ └─── 星期几（1-5 = 周一到周五）
              │ │ │ └───── 月份（* = 每月）
              │ │ └─────── 日期（* = 每天）
              │ └───────── 小时（9 = 上午 9 点）
              └─────────── 分钟（0 = 整点）

含义：每个工作日的上午 9:00 触发
```

---

## 第十一部分：Worktree 隔离——并行修改的安全网

### 11.1 问题：多个 Agent 同时修改同一个文件

假设两个子 Agent 同时被派去修改不同的功能，但它们都需要改同一个配置文件：

```
Agent A: 读取 config.yaml → 修改端口为 8080 → 写入
Agent B: 读取 config.yaml → 修改超时为 30s → 写入
                                    ↑
                    Agent B 的写入覆盖了 Agent A 的修改！
```

### 11.2 解决方案：Git Worktree 隔离

Git Worktree 允许在同一个仓库中创建多个工作目录，每个目录可以检出不同的分支：

```
主仓库: /project/
    ├── main 分支
    │
    ├── .claude/worktrees/agent-a1b2c3/
    │   └── feature-auth 分支     ← Agent A 在这里工作
    │
    └── .claude/worktrees/agent-d4e5f6/
        └── feature-perf 分支     ← Agent B 在这里工作
```

**Worktree 的关键特性**：

| 特性 | 说明 |
|---|---|
| **物理隔离** | 每个子 Agent 在独立的目录中工作 |
| **分支独立** | 每个子 Agent 在独立的分支上提交 |
| **共享 .git** | 所有 worktree 共享同一个 Git 对象库，节省空间 |
| **无冲突** | 不同目录的文件修改互不影响 |

### 11.3 Worktree 的创建流程

```
主 Agent 决定派两个子 Agent 并行工作
       │
       ▼
┌──────────────────────────────┐
│ ① 创建 Worktree A            │  git worktree add .claude/worktrees/agent-xxx feature-auth
│    (Agent A 的工作目录)       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ ② 创建 Worktree B            │  git worktree add .claude/worktrees/agent-yyy feature-perf
│    (Agent B 的工作目录)       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ ③ 分别创建子 Agent           │
│    Agent A → cwd: Worktree A │
│    Agent B → cwd: Worktree B │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ ④ 并行执行                   │
│    Agent A: 修改 auth 模块    │
│    Agent B: 优化性能瓶颈      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ ⑤ 合并结果                   │
│    主 Agent 合并两个分支      │
│    git merge feature-auth     │
│    git merge feature-perf     │
└──────────────────────────────┘
```

### 11.4 Worktree 隔离的权衡

| 方面 | 优点 | 代价 |
|---|---|---|
| **安全** | 完全隔离，不可能互相覆盖 | 需要 git 操作 |
| **空间** | 共享 .git 对象，比 clone 轻 | 每个目录有独立的工作文件 |
| **合并** | 可以用 git merge 解决冲突 | 合并时可能需要解决冲突 |
| **创建速度** | 比完整 clone 快很多 | 仍有几十毫秒的开销 |

---

## 第十二部分：完整的并发模型

### 12.1 从线程到注册表

claw-code 的多 Agent 并发模型可以用一张图概括：

```
┌──────────────────────────────────────────────────────────────┐
│                        主线程 (Main Thread)                    │
│                                                              │
│   ConversationRuntime                                       │
│       │                                                      │
│       ├─ 用户输入 "全面审查 PR"                                │
│       │                                                      │
│       ├─── execute_agent(Task1) ──▶ spawn thread 1           │
│       ├─── execute_agent(Task2) ──▶ spawn thread 2           │
│       ├─── execute_agent(Task3) ──▶ spawn thread 3           │
│       │                                                      │
│       ▼                                                      │
│   继续处理其他事务（不阻塞）                                    │
│       │                                                      │
│       ├─── TaskOutput(task_1) ──▶ 读取结果                    │
│       ├─── TaskOutput(task_2) ──▶ 读取结果                    │
│       ├─── TaskOutput(task_3) ──▶ 读取结果                    │
│       │                                                      │
│       ▼                                                      │
│   汇总三个 Agent 的报告，呈现给用户                             │
└──────────────────────────────────────────────────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ 线程 1            │  │ 线程 2            │  │ 线程 3            │
│ clawd-agent-xxx  │  │ clawd-agent-yyy  │  │ clawd-agent-zzz  │
│                  │  │                  │  │                  │
│ SubagentTool     │  │ SubagentTool     │  │ SubagentTool     │
│ Executor         │  │ Executor         │  │ Executor         │
│  ├ allowed:      │  │  ├ allowed:      │  │  ├ allowed:      │
│  │ Explore 工具  │  │  │ Verify 工具   │  │  │ Explore 工具   │
│  └ enforcer      │  │  └ enforcer      │  │  └ enforcer      │
│                  │  │                  │  │                  │
│ Conversation     │  │ Conversation     │  │ Conversation     │
│ Runtime          │  │ Runtime          │  │ Runtime          │
│  ├ Session       │  │  ├ Session       │  │  ├ Session       │
│  ├ ApiClient     │  │  ├ ApiClient     │  │  ├ ApiClient     │
│  └ ToolExecutor  │  │  └ ToolExecutor  │  │  └ ToolExecutor  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 12.2 线程安全：`Arc<Mutex<T>>` 模式

全局注册表使用 `Arc<Mutex<T>>` 模式确保线程安全：

```rust:rust/crates/runtime/src/task_registry.rs
struct TaskRegistryInner {
    tasks: HashMap<String, Task>,
    counter: u64,
}

struct TaskRegistry {
    inner: Arc<Mutex<TaskRegistryInner>>,
}
```

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 线程 1    │  │ 线程 2    │  │ 线程 3    │
│          │  │          │  │          │
│ Arc ─────┼──┼── Arc ───┼──┼── Arc    │  ← 引用计数（共享所有权）
│   │      │  │    │     │  │    │     │
│   ▼      │  │    ▼     │  │    ▼     │
│  Mutex ──┼──┼── Mutex ─┼──┼── Mutex  │  ← 互斥锁（同一时间只有一个线程能访问）
│   │      │  │    │     │  │    │     │
│   ▼      │  │    ▼     │  │    ▼     │
│  TaskRegistryInner                   │
│   ├─ tasks: HashMap<String, Task>    │
│   └─ counter: u64                    │
└──────────────────────────────────────┘
```

**为什么需要 `Arc<Mutex<T>>`？**

| 问题 | `Arc` 解决 | `Mutex` 解决 |
|---|---|---|
| 多线程共享所有权 | ✅ 引用计数自动管理 | — |
| 并发读写安全 | — | ✅ 互斥锁保证串行访问 |
| 生命周期管理 | ✅ 最后一个引用消失时释放 | — |

### 12.3 四个全局注册表

| 注册表 | 管理对象 | 线程安全 |
|---|---|---|
| `TaskRegistry` | Task 的生命周期 | `Arc<Mutex<TaskRegistryInner>>` |
| `TeamRegistry` | Team 的编排 | `Arc<Mutex<TeamRegistryInner>>` |
| `CronRegistry` | 定时任务 | `Arc<Mutex<CronRegistryInner>>` |
| `WorkerRegistry` | Worker 进程状态 | `Arc<Mutex<WorkerRegistryInner>>` |

这四个注册表都是**全局单例**，在整个进程生命周期内存在。它们使用 `Arc<Mutex<T>>` 确保在多线程环境下的安全访问。

---

## 第十三部分：从入口到输出的完整数据流

让我们追踪一次完整的多 Agent 调用：

### 13.1 场景：安全性审查

```
用户: "审查 src/auth/ 目录的安全性"
        │
        ▼
┌─── 主 Agent 的 Agent Loop ────────────────────────────────┐
│                                                            │
│  ① 模型返回 ToolUse:                                       │
│     {                                                      │
│       "tool": "execute_agent",                             │
│       "input": {                                           │
│         "description": "安全性审查",                         │
│         "prompt": "检查 src/auth/ 目录下的安全漏洞...",     │
│         "subagent_type": "Explore",                        │
│         "model": null                                      │
│       }                                                    │
│     }                                                      │
│                                                            │
│  ② PreToolUse Hook: 允许                                   │
│  ③ PermissionPolicy: 允许（execute_agent 不需要特殊权限）    │
│  ④ ToolExecutor.execute("execute_agent", input)            │
│     │                                                      │
│     ▼                                                      │
│  ⑤ execute_agent(input)                                    │
│     │                                                      │
│     ▼                                                      │
│  ⑥ normalize_subagent_type → "Explore"                     │
│     resolve_allowed_tools → {Read, Glob, Grep, WebSearch}  │
│     build_agent_system_prompt → [基础模板 + "你是Explore型"] │
│     create_agent_manifest → AgentOutput {agent_id: "xxx"}  │
│     persist_agent_manifest → 写入 manifest.json             │
│     │                                                      │
│     ▼                                                      │
│  ⑦ spawn_agent_job(job)                                    │
│     │                                                      │
│     ├── 线程 clawd-agent-xxx 启动                           │
│     │                                                      │
│     └── 返回 AgentOutput（状态: "created"）                  │
│                                                            │
│  ⑤ PostToolUse Hook: 允许                                  │
│  ⑥ ToolResult: AgentOutput JSON → 推入 session             │
└────────────────────────────────────────────────────────────┘

        ┌─── 子 Agent 线程 clawd-agent-xxx ──────────────────┐
        │                                                    │
        │  run_agent_job(job)                                │
        │    │                                               │
        │    ├── 创建 SubagentToolExecutor {allowed: Explore} │
        │    ├── 创建独立 Session                             │
        │    ├── 创建 ApiClient                               │
        │    ├── 创建 ConversationRuntime                     │
        │    │                                               │
        │    ├── runtime.run_turn(job.prompt)                 │
        │    │     │                                         │
        │    │     ├── 模型调用: "检查安全漏洞..."              │
        │    │     ├── ToolUse: Grep("sql injection pattern") │
        │    │     │     └── SubagentToolExecutor: allowed ✓  │
        │    │     │         └── 执行 Grep → 返回结果          │
        │    │     ├── ToolUse: Write("fix.sql")              │
        │    │     │     └── SubagentToolExecutor: denied ✗   │
        │    │     │         └── "Write not in allowed tools" │
        │    │     ├── 模型总结: "发现 3 个潜在漏洞..."        │
        │    │     └── 退出 Agent Loop                        │
        │    │                                               │
        │    ├── 写入 output.txt: "发现 3 个潜在漏洞..."       │
        │    └── 更新 manifest.json: status = "completed"     │
        └────────────────────────────────────────────────────┘
```

### 13.2 关键时刻分析

| 时刻 | 发生了什么 | 涉及的组件 |
|---|---|---|
| T1: 模型决定派子 Agent | ToolUse 事件中包含 `execute_agent` | Agent Loop |
| T2: 权限检查 | 主 Agent 的 PermissionPolicy 允许执行 | PermissionPolicy |
| T3: 类型标准化 | `"Explore"` 被规范化 | `normalize_subagent_type` |
| T4: 工具集解析 | 确定 Explore 可用的工具 | `resolve_allowed_tools` |
| T5: 线程创建 | 子 Agent 在独立线程中启动 | `spawn_agent_job` |
| T6: 工具隔离检查 | SubagentToolExecutor 拦截 Write | `SubagentToolExecutor` |
| T7: 结果持久化 | output.txt + manifest.json 写入 | `persist_agent_terminal_state` |

---

## 总结：多 Agent 的设计哲学

### 核心设计原则回顾

```
┌──────────────────────────────────────────────────────────────┐
│                    多 Agent 系统的六大设计原则                   │
│                                                              │
│  ① 最小权限         每个 Agent 只有完成任务所需的最小工具集      │
│                     ↕ SubagentToolExecutor 双重检查            │
│                                                              │
│  ② 线程隔离         每个 Agent 在独立线程中运行                  │
│                     ↕ panic::catch_unwind 防止崩溃传播          │
│                                                              │
│  ③ 状态持久化       manifest.json + output.txt 记录完整历史     │
│                     ↕ 崩溃恢复、可观测性、调试                  │
│                                                              │
│  ④ 类型专业化       六种预定义 Agent 类型 + 自定义类型支持        │
│                     ↕ 不同任务选择不同能力的 Agent               │
│                                                              │
│  ⑤ 复用 Runtime     子 Agent 复用完整的 ConversationRuntime    │
│                     ↕ Hook、权限、压缩等机制无需重新实现         │
│                                                              │
│  ⑥ 编排能力         Team + Task + Cron 支持复杂的多 Agent 协作   │
│                     ↕ 并行执行、定时调度、团队管理               │
└──────────────────────────────────────────────────────────────┘
```

### 与第08章的关联

| 第08章（单 Agent Runtime） | 本章（多 Agent） |
|---|---|
| `ConversationRuntime` | 子 Agent 复用完整的 Runtime |
| `ToolExecutor` trait | `SubagentToolExecutor` 实现了受限版本 |
| `PermissionPolicy` | 子 Agent 继承权限策略但工具集受限 |
| `Session` | 子 Agent 拥有独立的 Session |
| `WorkerBoot` | 多 Worker 并行，每个有独立的状态机 |
| `TaskRegistry` | 从"理论"到"实践"——真正的多任务管理 |

### 架构分层总览

```
┌─────────────────────────────────────────────────────┐
│ Layer 4: 编排层 (Orchestration)                       │
│   Team / Cron / 并行调度                              │
│   ── "派谁去做？同时派几个？"                          │
├─────────────────────────────────────────────────────┤
│ Layer 3: Agent 层 (Agent Lifecycle)                   │
│   execute_agent / spawn_agent_job / AgentJob          │
│   ── "怎么创建和启动一个子 Agent？"                    │
├─────────────────────────────────────────────────────┤
│ Layer 2: 隔离层 (Isolation)                           │
│   SubagentToolExecutor / Worktree / Session            │
│   ── "子 Agent 能做什么？不能做什么？"                 │
├─────────────────────────────────────────────────────┤
│ Layer 1: Runtime 层 (Agent Loop)                      │
│   ConversationRuntime / ApiClient / Hook               │
│   ── "子 Agent 内部怎么运行？"（第08章）               │
├─────────────────────────────────────────────────────┤
│ Layer 0: 基础设施 (Infrastructure)                    │
│   线程 / Arc<Mutex> / JSONL / manifest.json            │
│   ── "底层机制是什么？"                                │
└─────────────────────────────────────────────────────┘
```

### 从"单兵"到"军团"

如果用一句话概括多 Agent 机制：

> **主 Agent 是指挥官，子 Agent 是被派出的专家。每个专家有自己的专长（工具集）、独立的工作空间（Session/Worktree）、明确的目标（prompt），并且随时向指挥官报告进展（manifest/output）。**

这种设计让 claw-code 从一个"智能助手"进化为一个"智能团队"——不是靠一个超级 Agent 解决所有问题，而是靠一群专业化的 Agent 协作完成复杂任务。
