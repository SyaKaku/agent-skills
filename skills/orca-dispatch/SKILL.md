---
name: orca-dispatch
description: >-
  Supervise tracked Orca dispatches from a coordinator to Codex or Kimi workers:
  write work orders, dispatch tasks, handle gates, wait, verify results, and
  clean up. Use when the user explicitly asks to supervise, monitor, wait for
  or collect worker results, coordinate a DAG or ask/reply flow, use decision
  gates, or explicitly invokes $orca-dispatch. Do not use for an ownership
  handoff that starts another worker/model/effort without waiting for results;
  use orca-cli.
---

# Orca 派单

此 skill 面向 Orca coordinator，需要具备 terminal 与 orchestration 能力的 Orca
CLI、Codex CLI 和/或 Kimi CLI；使用独立 worktree 时还需要 Git。具体模型、参数
和能力取决于已安装 CLI、账号及仓库策略。缺少兼容依赖时不得模拟 task、dispatch
或生命周期事件；报告缺失项和可执行的安装或配置前置条件。

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
与 worker 不匹配或运行环境不支持指定配置时，派单前直接询问用户。仅加载所选
手册：

- Codex（默认）：[`references/codex-worker.md`](references/codex-worker.md)
- Kimi：[`references/kimi-worker.md`](references/kimi-worker.md)

## 确定流程深度

- **handoff**：用户要转交所有权且无需监督结果。按当前 `orca-cli` handoff
  流程发送后结束，不创建 task。
- **Tier 1 — work order**：目标、范围、约束和验证方式已明确，可交给一个 worker。
  用户显式调用 `$orca-dispatch` 或要求派单并回收结果时，即使任务简单也不得改为
  coordinator 本地执行。
- **Tier 2 — freeze spec**：写工作单仍会迫使 worker 做产品或架构决策。Coordinator
  先澄清并落盘规格、验收标准和依赖，再拆成 work order 或 DAG。

## 规格流水线

从想法到派单的完整链路中，前置规格阶段由 coordinator 与用户交互完成，
只有拆解后的执行阶段交给 worker：

1. **拷问与定稿（coordinator 本地，不派单）**：用 `grill-with-docs` 逐问压测
   方案并就地更新 `CONTEXT.md` / ADR，再用 `to-spec` 把讨论综合为 spec 落盘
   （本地模式 `.scratch/<feature>/PRD.md`，或发布到项目配置的 issue
   tracker）。这两个 skill 强交互且禁止模型自动调用；worker 是零历史
   agent，拿不到对话上下文，不得把拷问或写 spec 派给 worker。
2. **拆 ticket（coordinator 本地，用户批准后生效）**：用 `to-tickets` 把 spec
   拆成 tracer-bullet ticket；本地模式每 ticket 一个
   `.scratch/<feature>/issues/<NN>-<slug>.md`，含 `Blocked by:` 边与验收
   标准。粒度和依赖边由用户 quiz 确认后才发布。
3. **映射为派单 DAG**：每个 ticket 一个 task，`Blocked by` 边写入
   `task-create --deps <json_array>`。批量建单前先从 issues 目录枚举文件，
   把编号到真实路径的清单写进 DAG，不得凭任务摘要猜文件名。标题显式传
   `--task-title`（人类可读），不依赖 spec 首行派生。从 frontier（无未完成
   blocker 的 ticket）起按波次派单，依赖链深度不超过 3–4 层。
4. **执行与验收**：为每个 frontier ticket 按 Tier 1 写 work order，`[规格]`
   引用 PRD 与 ticket 文件的精确路径；实现与验收按「实现类工单」和「监督式
   流程」执行。

用户直接给出已批准的 spec 或 tickets 时跳过前两步，从 DAG 映射开始。

## 裁决与 gate

- 派单前缺少用户选择时直接询问，不为尚未创建的 task 建空 gate。
- Worker 的 `ask` 或 `decision_gate` 消息按动态 orchestration 指南用消息 ID
  `reply`；答复后继续等待当前 dispatch。
- 已有 DAG 中由 coordinator 管理的阻塞决策才使用 `gate-create` / `gate-resolve`。
  不存在抽象的 `decision_gate` 命令。

## 权限与无人值守

- Worker 默认遵守 CLI、用户和目标仓库现有的审批与沙箱策略。不要仅为了自动化而
  主动关闭审批或沙箱。
- 监督式 dispatch 要求 worker 能从自己的 terminal 发送 `worker_done`；coordinator
  不能代发。派单前按所选 worker 手册确认 Orca CLI 可解析且生命周期命令有一条
  可执行的审批路径。两者任一不满足时先解决或升级，不得创建一个注定无法闭环的
  dispatch。
- 只有用户已明确授权，或目标仓库/运行时策略已明确选择无人值守模式时，才可使用
  对应 CLI 的审批绕过参数；并把授权范围写入 work order 的 `[权限]`。
- 无法在现有权限下继续时保留原始错误，并按上述裁决规则决定提升权限、更换执行
  方式或停止。不得把无人值守参数当作解决权限错误的默认办法。
- 无人值守参数不扩大工作单范围，也不覆盖仓库对 commit、push、部署、密钥、
  数据和破坏性操作的限制。

## Worker 与 worktree 隔离

- 同一问题的审查修复、补测或二次验证可复用原 worker terminal；每轮仍创建新的
  task/dispatch。模块、目标、工具链不相关，或旧上下文可能干扰判断时，创建新的
  terminal 和 agent session，不因方便而复用。
- 只读审计或顺序执行的代码任务默认使用任务所依赖的现有 worktree。并行且会修改
  代码的独立任务，默认每个 worker 使用独立 git worktree；需要共享未提交修改时
  不得拆 worktree，先留在原 worktree 或解决可复现的基线。
- 新建 worktree 的 worker 优先 agent-first 创建：
  `worktree create --agent <id> --prompt <brief>` 让 agent 直接占首个
  terminal。`--agent`、`--activate`、`--run-hooks` 会把 worktree 带到前台，
  plain create 留在后台。需要自定义 argv（Codex 的 model/effort、Kimi 的
  `--yolo` 或 effort 环境变量）时只能走两步法，bare create 会多开一个
  fallback shell，属预期行为而非异常：只操作 agent handle，经
  `terminal list` / `terminal show` 确认 fallback shell 未使用后才允许关闭。
- 一个 worktree 可挂多个 terminal；terminal 在创建时用 `--worktree` 绑定，
  创建后不可移动。review、repair 或补测 worker 必须落在目标 worktree 时，用
  `terminal create --worktree id:<repoId>::<path>` 创建新 agent terminal——
  完整两段式 id 从 `worktree create --json` 或 `worktree list --json` 复制，
  不得凭 sidebar 位置或目录名推断归属。
- 创建 worktree 前检查 `git status`、当前 HEAD、目标 base ref 及依赖提交。不得假设
  Orca 的默认 base 已包含本地分支进度；需要当前分支内容时显式传入对应 branch/ref。
- Worktree 只隔离 checkout、分支与构建目录，不隔离端口、数据库、测试账号和外部
  服务。并行工单必须分配互不冲突的端口、数据或环境，无法隔离时改为串行。
- 独立 worktree 必须在工单和报告中记录完整 worktree ID、分支、起始 HEAD 与集成
  方式。未明确集成方式的并行代码任务不得开工。

## 可用性与额度

- 派单前用最小成本信号确认 worker 账户可用（CLI 启动、模型加载、一次平凡
  短任务的提交信号）。账户级用量上限不能靠新建 terminal 或降低模型/effort
  档位绕过；识别到 account-wide limit 后不得连续创建注定失败的 worker。
- Worker 因额度在任务中途终止时保留现场：原 terminal、非干净 worktree、
  base SHA、task/dispatch ID、已通过证据与未闭合 finding，禁止 reset/clean。
  替换工单用 `task-create --parent` 与原工单建立父子关系，并写明禁止动作。
- CLI 提示的 `usage limit reset` 与购买 credits 同属账户状态变更，只有用户
  裁决后才能消耗；coordinator 不得自行输入 `/usage` 或消耗 reset，需要时按
  裁决规则给出等待/消耗/放弃的选项。

## 实现类工单

- 修改代码的工单优先选择能发现 `implement` skill 的 Codex worker，并在 `[工具]`
  中显式要求先调用该 skill；不得依赖模型从“实现”语义自动触发。发现顺序：
  用户显式提供的路径 > 目标仓库 `.agents/skills/` > 用户全局
  `~/.agents/skills/`；以实际读取到的绝对路径为准写入 `[工具]`，会话技能
  目录未列出不代表文件不存在。目标 worker 无该 skill 时，明确展开测试优先、
  持续类型检查/单测、最终充分验证和代码审查要求，不得声称已调用。
- `implement` 自带的 commit 步骤不覆盖仓库安全规则。未经用户明确授权，工单必须
  写明“禁止 commit/push，本条覆盖 implement 的 commit 步骤”；若采用独立任务
  分支提交来集成，先取得对应的本地 commit 授权，push 仍需单独授权。
- 工单必须显式声明 review 所有权：默认 worker 不启动 Standards/Spec 子审阅，
  implement 的独立审阅要求由 coordinator 的固定提交外审满足；worker 内部
  code review 只是交付前自检，不能替代 coordinator 的独立审查与复验。确需
  worker 预审时，coordinator 的第二层限定为对首轮 finding、组合基线和残余
  风险的 targeted review，不再次通读全部 diff。

## 派发后启动核验

`dispatch --inject` 返回 `injected: true` 只表示运行时接受了终端注入，不证明
agent 已提交 prompt。每个 dispatch 在进入长时滚动等待前执行一次有界核验：

1. 派发前用 `terminal read` 记录 `nextCursor`；派发后读取该 cursor 的增量和当前
   terminal 尾部，并用短时、带显式 `--timeout-ms` 的
   `terminal wait --for tui-idle` 检查状态。
2. 所选 worker 手册列出的本轮提交信号，以及新的 agent 输出、工具调用或
   lifecycle 消息，均可证明已启动；此时保持只读。无法明确确认 `tui-idle` 时
   也不得发送输入，继续观察并保留原始状态或错误。
3. 仅当当前 dispatch 仍为 `dispatched`、`tui-idle` 明确满足，且有界观察后没有
   本轮提交或活动证据时，补发一次。除 terminal 尾部已有明确待提交输入外，至少
   在两次 `terminal read` 之间完成一个短时、带显式 `--timeout-ms` 的
   `check --wait` 窗口，并确认两次 `nextCursor` 均未增长作为静默证据：

   ```text
   orca terminal send --terminal <worker-handle> --text '' --enter --json
   ```

4. 补发后重新读取增量并确认 worker 已启动。仍无活动证据时保存 `dispatch-show`
   与 terminal 输出并升级；同一 dispatch 不得再次补回车、重新 `task-create` 或
   重复 `dispatch`。

只认派发前 cursor 之后的本轮证据；复用 terminal 时不得把旧工单的输出或 hook
当作当前 dispatch 已启动。

## 滚动等待与交付缺口

`check --wait` 返回 `count: 0` 只是检查点。每轮超时后读取当前 task/dispatch、
派发后 cursor 增量和 terminal 状态：

- 有新的推理、工具输出、heartbeat 或其他活动时继续等待；不得仅因缺少 heartbeat
  或一次静默超时停止 worker。
- 已确认本轮启动后，若 terminal 回到 agent 输入态却没有当前 dispatch 的
  `worker_done`，检查尾部是否有审批提示、Automatic approval rejection、CLI
  解析错误或 worker 最终回复。审批提示按权限规则升级；明确拒绝或解析错误是
  lifecycle delivery failure，不是注入失败。
- Worker 已输出最终回复并回到输入态，却仍无 `worker_done`，同样属于 lifecycle
  delivery failure；没有最终回复或错误证据的暂时空闲保持观察，不据此判定失败。
- Delivery failure 时保存 `dispatch-show`、terminal 增量和原始错误；不得把任务
  视为完成、补发空回车、由 coordinator 代发 `worker_done`，也不得重复当前
  dispatch。Automatic approval rejection、CLI 解析错误或 turn 已结束且消息未送达
  时，按动态 orchestration 指南将 task 标记为 `failed`；仍在等待用户审批或外部
  条件时标记为 `blocked`。若仍需回收已有产物，先修复新 worker 的生命周期前置
  条件，再创建新的验证或返工 task/dispatch。
- terminal 仍在工作、状态含糊或只有暂时空闲时继续有界观察。只有明确的完成消息、
  delivery failure、terminal 退出/消失或用户裁决才能结束滚动等待。

## 监督式流程

1. 运行 `orca skills get orca-cli`、`orca skills get orchestration` 和
   `orca status --json`，以当前动态指南为命令与生命周期的事实来源。
2. 选择 worker、模型和 effort，完整读取唯一匹配的 worker 手册；需要确认的
   高成本配置、Orca CLI 解析方式或权限提升在派单前按裁决规则解决。
3. 创建 worker terminal，等待 `tui-idle` 后检查终端已越过登录、信任或启动提示。
   `tui-idle` 单独成立不够；MCP server 或其他启动日志仍增长时继续有界等待。
   默认在当前 worktree 工作；并行前确认文件、端口、数据库和服务互不冲突。
4. 创建 task，以 `dispatch --inject` 派发并核验 task/dispatch provenance；按上述
   规则确认 worker 已启动后，滚动等待 `worker_done`、`escalation`、
   `decision_gate`，及时 reply 后继续等待。
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
[前提] 基线事实（测试数、构建退出码等，带来源）与核对命令；worker 动工前
  先核对，对不上立即停止并 escalation，不带错误前提施工。
[范围] 允许修改的路径白名单；白名单之外一律只读，顺手发现的无关问题记入
  报告待裁决，不直接改。
[规格] 权威文件及优先级；先读根级与范围内的 AGENTS.md。
[约束] 项目安全红线、本工单限制和允许的外部操作；coordinator 未逐项确认的
  默认决定逐条标注“猜的”及猜错代价。
[权限] 审批/沙箱模式，以及 commit、push、部署和外部系统操作的授权边界。
[工具] 必须使用的工具及确定路径；例如 implement skill 的绝对路径、E2E 明确
  指定 Playwright CLI。
[隔离] Worker 是否复用、worktree/base、端口和集成方式。
[验证] 命令、场景、预期结果和需要保留的证据。验收命令与测试由本工单指定并
  冻结：不得修改验收脚本、放宽断言、skip 或 mock 被测对象来凑绿；基线不可退
  （测试数/覆盖率不低于 [前提]，skipped=0）。“坏了没人知道”的检查先故意
  制造一次失败，给出红→绿证据。
[审查] review 所有权：默认 worker 不启动独立 reviewer，由 coordinator 做
  固定提交对抗式外审；工单另有约定时从其约定。
[止损] 同一验收连败 3 次停止并 escalation；结果比基线差则回滚并如实报告；
  “没做成但说清了”是合格交付，“做了但更糟”不是。
[报告] 结论 / 偏差 / worktree 与改动文件 / 验证证据（命令、退出码、关键数字）。
```

Worker 视为零历史；长规格落盘后引用路径。报告以工作区文件和真实命令为依据，
并使用当前 dispatch 注入的 taskId/dispatchId 回报。代码修改只有在审查问题修复且
coordinator 复验通过后闭环；只读审计保持只读，发现的实现缺口另建工作单。
