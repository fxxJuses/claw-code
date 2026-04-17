# Agent 学习计划（30 天）

> 目标：以 `claw-code` 为主线，系统掌握 Agent 工程实践，并沉淀可用于简历和面试的项目成果。

## 0. 学习终局（一个月后你要达到）

- 能独立复现并讲清 `claw` 主流程：输入 -> Prompt -> Tool 调度 -> 权限 -> 会话 -> 输出。
- 能在 `rust/` 工作区完成 1-2 个中等改动，并补齐测试。
- 能产出自己的 Agent 方法论：多模式使用、Prompt 调优、优化策略。
- 能把实践包装成简历项目，并完成可复述的技术讲稿。

## 1. 项目复现基线（Day 1 必做）

```bash
cd rust
cargo build --workspace
./target/debug/claw
# 进入 REPL 后
/doctor
```

```bash
cd rust
cargo test --workspace
./scripts/run_mock_parity_harness.sh
```

```shell
export ANTHROPIC_AUTH_TOKEN="6a439d754b86445aaef5066f24ad29bb.ZbbTnOd7K5W2PxNh"
export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
```

## 2. 周度计划（Week 1 ~ Week 4）

### Week 1：复现 + 架构地图

目标：跑通项目、熟悉目录、画出架构图。

- 学习材料：`USAGE.md`、`rust/README.md`、`PARITY.md`、`PHILOSOPHY.md`
- 重点目录：`rust/crates/rusty-claude-cli`、`rust/crates/runtime`、`rust/crates/tools`、`rust/crates/api`
- 你要回答的 4 个问题：
  - 用户输入在哪里解析？
  - 系统 Prompt 在哪里构建（重点 `runtime/src/prompt.rs`）？
  - 工具如何分发和执行（重点 `tools/src/lib.rs`）？
  - 权限如何判定（`runtime/src/permissions.rs` / `permission_enforcer.rs`）？

本周产出：

- `learning/week1-repro-notes.md`（复现记录 + 踩坑）
- `learning/week1-architecture.md`（模块关系 + 请求链路）

### Week 2：Agent 多模式实践

目标：掌握模式切换与边界条件，不只会用命令。

建议按 5 类模式做实验：

- 交互模式：REPL vs one-shot `prompt`
- 输出模式：`text` vs `json`
- 权限模式：`read-only` / `workspace-write` / `danger-full-access` / `prompt`
- 工作模式：`EnterPlanMode` / `ExitPlanMode`
- 协作模式：`/review`、`/advisor`、`/subagent`、`/team`、`/cron`

每种模式记录：

- 适用场景
- 典型命令
- 常见失败
- 安全风险和回退方案

本周产出：

- `learning/mode-playbook.md`（可直接给面试官讲）

### Week 3：Prompt 调优

目标：从“会写 Prompt”升级到“会做 Prompt 实验”。

实验方法：

- 建立 20 条基线任务（理解/修改/测试/排错/文档）
- 固定变量（模型、权限、任务集）做 A/B 对比
- 定义指标：
  - 任务成功率
  - 首次通过率
  - 工具调用次数
  - 平均耗时
  - token 成本

模板迭代建议：

- v0：原始提示词
- v1：增加任务分解步骤
- v2：增加约束（不猜测、不越权、失败先诊断）
- v3：增加输出自检清单

本周产出：

- `learning/prompt-tuning-report.md`（含表格和结论）

### Week 4：优化落地 + 简历包装

目标：做出“可证明提升”的工程改进。

可选优化方向（选 1-2 个）：

- 工具执行稳定性（错误分类/重试/降级）
- 权限治理（allow/deny/ask 规则细化 + 审计）
- 会话质量（压缩、恢复、长会话稳定性）
- 可观测性（延迟/失败原因/工具成功率统计）
- Parity 扩展（新增 harness 场景）

验收标准：

- 有前后对比数据（至少 1 个核心指标改善）
- 有对应测试，`cargo test --workspace` 可通过
- 有文档说明设计权衡和失败复盘

本周产出：

- `learning/optimization-case-study.md`
- `learning/resume-bullets.md`
- `learning/interview-storyline.md`

## 3. 30 天日程建议（可打卡）

- Day 1-3：环境复现 + 命令熟悉 + 文档速读
- Day 4-7：核心 crate 走读 + 架构图 + 首次讲解稿
- Day 8-14：多模式实验 + 失败案例复盘 + playbook
- Day 15-21：Prompt A/B 测试 + 指标统计 + 调优报告
- Day 22-27：选题优化 + 编码实现 + 测试验证
- Day 28-30：简历包装 + 项目讲稿 + 模拟面试

## 4. 每周固定节奏（建议）

- 周一：设目标（3 个最重要结果）
- 周二到周四：编码/实验
- 周五：复盘 + 文档沉淀
- 周末：面试演练（讲架构、讲取舍、讲数据）

## 5. 简历项目模板（可直接改写）

项目名：

- Claw Code Agent Runtime 深度实践（Rust）

一句话定位：

- 基于开源 Agent CLI 工程完成全链路复现与能力增强，覆盖 Prompt、Tool、权限、会话与观测体系。

职责亮点（示例）：

- 复现并跑通 `rust/` 工作区核心流程，构建标准化诊断与验证路径（doctor + tests + parity harness）。
- 设计多模式实验矩阵（交互/权限/计划/协作），沉淀团队可复用的 Agent 操作手册。
- 搭建 Prompt 调优实验体系，通过 A/B 对比提升任务成功率并降低工具无效调用。
- 完成 runtime/tooling 改进并补齐测试，提升 Agent 执行稳定性与可解释性。

量化成果（示例写法）：

- 在 20 条基线任务上，将端到端成功率从 `X%` 提升至 `Y%`。
- 将平均工具调用次数从 `A` 降低到 `B`，减少冗余轮次 `N%`。
- 将失败任务中的权限误拒比例从 `P%` 降低到 `Q%`。

## 6. 面试回答框架（3 分钟）

- 背景：为什么选这个项目（真实 Agent 工程，不是纯 demo）。
- 方案：我把能力拆成“模式、Prompt、优化”三条线并行推进。
- 实现：在哪些模块动手（`runtime`、`tools`、`cli`）以及如何验证。
- 结果：用指标说话（成功率、耗时、稳定性）。
- 反思：哪些问题仍未解决（如会话压缩、MCP 全链路深度），下一步怎么做。

---

如果你愿意，我可以下一步在 `learning/` 里继续帮你补齐这 5 个配套文档骨架，变成可直接打卡执行的学习资料包。