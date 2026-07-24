---
name: orca-dispatch
description: >-
  Supervise Orca work delegation from a coordinator to Codex or Kimi workers:
  create self-contained work orders, dispatch them, handle decision gates,
  wait for worker completion, verify results, and clean up safely. Use whenever
  the user asks to delegate, dispatch, start or supervise workers, select a
  worker/model/effort, manage a task DAG, or collect delegated results. Orca
  监督式派单：用户要求派单、发工单、起 worker、选择模型或 effort、监督任务 DAG
  或回收结果时使用；纯所有权交接则路由到 Orca handoff。
compatibility: >-
  Requires Orca CLI with skills, status, terminal, task, dispatch, event, and
  reply capabilities; Codex CLI and/or Kimi CLI; and Git when worktree
  isolation is used. Exact models and non-interactive flags depend on the
  installed CLI, account, and repository policy.
---

# Orca 派单

此 skill 面向 Orca coordinator。若本机没有兼容的 Orca CLI，不要模拟 task、
dispatch 或生命周期事件；应报告缺失的依赖和可执行的安装或配置前置条件。

## 选择 worker

Coordinator 保持当前会话与模型；选择器只配置新 worker。
支持 `$orca-dispatch worker=<codex|kimi> [model=<id>] [effort=<level>]：<任务>`，
例如：

```text
$orca-dispatch worker=codex model=gpt-5.6-sol effort=max：审计前后端测试机制
$orca-dispatch worker=kimi：诊断偶发登录故障
```

结构化选择器优先于自然语言；未指定 worker 时使用 Codex。未指定 model 或 effort
时，按对应 worker 手册选择满足任务风险和复杂度的最低充分档位。选择器冲突、模型
与 worker 不匹配或运行环境不支持指定配置时，用 `decision_gate` 请用户裁决。
仅加载所选手册：

- Codex（默认）：[`references/codex-worker.md`](references/codex-worker.md)
- Kimi：[`references/kimi-worker.md`](references/kimi-worker.md)

## 确定流程深度

- **handoff**：用户要转交所有权且无需监督结果。按当前 `orca-cli` handoff
  流程发送后结束，不创建 task。
- **Tier 0 — local**：简单问答、一次只读查询、单行或机械小改，由 coordinator 完成。
- **Tier 1 — work order**：目标、范围、约束和验证方式已明确，可交给一个 worker。
- **Tier 2 — freeze spec**：写工作单仍会迫使 worker 做产品或架构决策。Coordinator
  先澄清并落盘规格、验收标准和依赖，再拆成 work order 或 DAG。

## 权限与无人值守

- Worker 默认遵守 CLI、用户和目标仓库现有的审批与沙箱策略。不要仅为了自动化而
  主动关闭审批或沙箱。
- 只有用户已明确授权，或目标仓库/运行时策略已明确选择无人值守模式时，才可使用
  对应 CLI 的审批绕过参数；并把授权范围写入 work order 的 `[权限]`。
- 无法在现有权限下继续时，保留原始错误并通过 `decision_gate` 决定提升权限、
  更换执行方式或停止。不得把无人值守参数当作解决权限错误的默认办法。
- 无人值守参数不扩大工作单范围，也不覆盖仓库对 commit、push、部署、密钥、
  数据和破坏性操作的限制。

## Worker 与 worktree 隔离

- 同一问题的审查修复、补测或二次验证可复用原 worker terminal；每轮仍创建新的
  task/dispatch。模块、目标、工具链不相关，或旧上下文可能干扰判断时，创建新的
  terminal 和 agent session，不因方便而复用。
- 只读审计或顺序执行的代码任务默认使用任务所依赖的现有 worktree。并行且会修改
  代码的独立任务，默认每个 worker 使用独立 git worktree；需要共享未提交修改时
  不得拆 worktree，先留在原 worktree 或解决可复现的基线。
- 创建 worktree 前检查 `git status`、当前 HEAD、目标 base ref 及依赖提交。不得假设
  Orca 的默认 base 已包含本地分支进度；需要当前分支内容时显式传入对应 branch/ref。
- Worktree 只隔离 checkout、分支与构建目录，不隔离端口、数据库、测试账号和外部
  服务。并行工单必须分配互不冲突的端口、数据或环境，无法隔离时改为串行。
- 独立 worktree 必须在工单和报告中记录完整 worktree ID、分支、起始 HEAD 与集成
  方式。未明确集成方式的并行代码任务不得开工。

## 实现类工单

- 修改代码的工单优先选择能发现 `implement` skill 的 Codex worker，并在 `[工具]`
  中显式要求先调用该 skill；不得依赖模型从“实现”语义自动触发。目标 worker
  无该 skill 时，明确展开测试优先、持续类型检查/单测、最终充分验证和代码审查
  要求，不得声称已调用。
- `implement` 自带的 commit 步骤不覆盖仓库安全规则。未经用户明确授权，工单必须
  写明“禁止 commit/push，本条覆盖 implement 的 commit 步骤”；若采用独立任务
  分支提交来集成，先取得对应的本地 commit 授权，push 仍需单独授权。
- Worker 内部 code review 是交付前自检，不能替代 coordinator 的独立审查与复验。

## 监督式流程

1. 运行 `orca skills get orca-cli`、`orca skills get orchestration` 和
   `orca status --json`，以当前动态指南为命令与生命周期的事实来源。
2. 选择 worker、模型和 effort，完整读取唯一匹配的 worker 手册；需要确认的
   高成本配置或权限提升先完成 `decision_gate`。
3. 创建 worker terminal，等待 `tui-idle` 后检查终端已越过登录、信任或启动提示。
   默认在当前 worktree 工作；并行前确认文件、端口、数据库和服务互不冲突。
4. 创建 task，以 `dispatch --inject` 派发并核验 task/dispatch provenance；滚动等待
   `worker_done`、`escalation`、`decision_gate`，及时 reply 后继续等待。
5. Coordinator 检查实际文件和命令结果，亲自复验关键退出码。代码改动优先按目标
   仓库或可发现的 `code-review` 规则审查；没有该规则时，至少检查需求符合度、
   diff、相关测试、静态检查和仓库文档约束。只读工单抽查来源、数字和浏览器证据。
6. 同范围返工可复用 terminal，但每轮创建新的 task/dispatch；无关任务使用新 terminal。
   以当前 active dispatch 的 ID 对应的有效 `worker_done` 作为本轮完成信号。

## Worker 关闭与 worktree 回收

- `worker_done` 只表示 worker 进入空闲交付态，不是立即关闭信号。Coordinator
  复验期间保留 terminal，以便对同范围问题创建新的 task/dispatch 返工。
- 同时满足以下条件后关闭 worker terminal：当前 dispatch 的 `worker_done` 有效；
  coordinator 已检查交付物并完成关键复验；报告、日志和失败证据已记录；不存在
  未决 `decision_gate`、`ask` 或同范围返工；代码改动已经安全集成或明确保留在
  可恢复的 worktree 中。
- 只读 worker 在报告复核通过后即可关闭。失败或阻塞的 worker 要先保存诊断证据并
  完成继续、替换或放弃的裁决；不得仅因等待超时、暂时无输出或只有 heartbeat
  就关闭、停止或重启。
- Terminal 与 worktree 分开回收：terminal 达到上述条件即可关闭；独立 worktree
  必须保留到改动集成并完成最终回归。未集成、工作树非干净或恢复方式不明确时禁止
  删除 worktree；删除属于独立的破坏性清理动作，仍需符合用户授权和精确目标核对。
- 已验收且预计无同范围返工的 worker 应及时关闭，避免长期保留无关上下文。之后若
  再发现问题，创建新 worker，并把原 task、审查结论和验证证据写入新工单。

## Work order

```text
[角色] EXECUTOR：只完成本工作单；二次派单需要 coordinator 明确授权。
[目标] 要完成的结果及可观察的完成判据。
[范围] 仓库路径、模块，以及允许修改或只读的边界。
[规格] 权威文件及优先级；先读根级与范围内的 AGENTS.md。
[约束] 项目安全红线、本工单限制和允许的外部操作。
[权限] 审批/沙箱模式，以及 commit、push、部署和外部系统操作的授权边界。
[工具] 必须使用的工具；例如 E2E 明确指定 Playwright CLI。
[隔离] Worker 是否复用、worktree/base、端口和集成方式。
[验证] 命令、场景、预期结果和需要保留的证据。
[报告] 结论 / 偏差 / worktree 与改动文件 / 验证证据（命令、退出码、关键数字）。
```

Worker 视为零历史；长规格落盘后引用路径。报告以工作区文件和真实命令为依据，
并使用当前 dispatch 注入的 taskId/dispatchId 回报。代码修改只有在审查问题修复且
coordinator 复验通过后闭环；只读审计保持只读，发现的实现缺口另建工作单。
