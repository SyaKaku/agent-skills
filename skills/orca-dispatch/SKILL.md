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
3. **映射为派单 DAG**：每个 ticket 建实现 task；`Blocked by` 边必须落为
   `task-create --deps <json_array>` 或等价验收 gate，不得只记在正文。批量建单前
   先从 issues 目录枚举文件，
   把编号到真实路径的清单写进 DAG，不得凭任务摘要猜文件名。标题显式传
   `--task-title`（人类可读），不依赖 spec 首行派生。按「DAG 与环境门禁」校验后，
   从 frontier（无未完成 blocker 的 ticket）起按波次派单。
4. **执行与验收**：为每个 frontier ticket 按 Tier 1 写 work order，`[规格]`
   引用 PRD 与 ticket 文件的精确路径；实现与验收按「实现类工单」和「监督式
   流程」执行。

用户直接给出已批准的 spec 或 tickets 时跳过前两步，从 DAG 映射开始。

## DAG 与环境门禁

- 3–4 层是未定稿方案的设计启发，不是运行上限。尚未批准的深层 DAG 先提议减少
  非必要顺序边；已批准的 tickets 必须保留真实依赖，不得为压平层数漏边、提前启动
  或合并范围。改变已批准依赖或 ticket 粒度前触发 🔴 CHECKPOINT · 🛑 STOP。
- 建 task 前生成 `ticket ID → 精确路径 → blocker IDs → 验收屏障 → 环境需求` 表，
  检查缺失 ID、重复 ID、自依赖和环。发现环时保持相关 ticket blocked，列出完整环
  路径并请求用户裁决；不得删边绕过。先准备完整 spec，再按拓扑顺序建 task；不要建
  占位 task。误建或重复 task 无法安全删除时标 failed，并在 `task-update --result`
  中记录 `supersededBy` 与原因。
- 每个 ticket 把实现终结与验收解锁分开。可返工的代码 DAG 默认用 coordinator gate
  阻塞下游，直到审查、repair、复验和约定集成通过；不得把每轮 review task 当成稳定
  依赖。只启动已验收 blocker 的 ready frontier；实现 task completed 不得单独解锁。
- 派单前列出环境敏感性：本地工具链、外部服务、端口、代理/VPN、容器、数据库、
  测试账号、共享缓存，以及启动、重启和清理责任人。Coordinator 先执行安全只读
  探针；失败只阻塞受影响 task，保存证据，其他无关 frontier 可继续。
- 并发前为每个 worker 分配资源租约，在工单与报告中记录，并在清理时逐项核对释放。
  Worktree 不隔离 Docker daemon、Compose project、端口、数据库或账号；Compose
  使用唯一 project 名，重型 build/restart 错峰。无法证明隔离时改为单槽串行。
  Worker 只操作工单授权的租约；共享 daemon/服务的启动、重启和全局清理调度由
  coordinator 独占，coordinator 未获对应操作授权时 STOP。

## 裁决与 gate

- 🔴 CHECKPOINT · 🛑 STOP：派单前缺少用户选择时直接询问；收到选择前不得创建
  terminal、task、dispatch 或空 gate。
- Worker 的 `ask` 生成 `question` 消息；按动态 orchestration 指南用消息 ID
  `reply`，确认回复成功后继续等待当前 dispatch。只有动态指南明确标注的 legacy
  assignment 才按其原样处理 `decision_gate`。
- 已有 DAG 中由 coordinator 管理的阻塞决策才使用 `gate-create` / `gate-resolve`。
  不存在抽象的 `decision_gate` 命令。

## Run 与派发路径

- 每个监督目标先创建或绑定一个 Run；Run 只提供 task 命名空间与 coordinator
  inbox，不负责调度。新目标用 `run-create`，继承 task、dispatch 或待处理消息时
  用其原 Run，禁止另建 Run 隔离旧状态。
- 标准路径是 `task-create` 后用 `worker-start` 启动受监督 worker。单任务直接建单；
  DAG 先按拓扑顺序建完已校验 task，再启动当前 ready frontier 的全部 worker，之后
  才进入等待。
- 所选 worker 需要 custom argv 或 worker-side 生命周期预检时，先创建并核验最终
  terminal，再创建 task，并用 `worker-start --terminal <handle>` 绑定。只有当前
  动态指南或 `--help` 证明 `worker-start` 无法表达所需拓扑时，才回退到低层
  `worktree create` / `terminal create` / `dispatch --inject`。
- 盘点任务时限定当前 Run，并先用 `task-list --brief` 配合 `--ready` 或 `--status`；
  仅在需要完整 spec 时读取非 brief 输出。
- `worker-start` 代表正式 dispatched。只有隔离规则已允许且长 setup 确有收益时，才可
  提前创建空闲 worktree/terminal 作为 provisioned 资源；正式启动前重新核对依赖、
  环境门禁与用户 steering。计划取消时只回收已确认 clean、未接收工单且在授权范围内
  的资源。

## 权限与无人值守

- Worker 默认遵守 CLI、用户和目标仓库现有的审批与沙箱策略。不要仅为了自动化而
  主动关闭审批或沙箱。
- 监督式 dispatch 要求 worker 能从自己的 terminal 发送 `worker_done`；coordinator
  不能代发。派单前按所选 worker 手册确认 Orca CLI 可解析且生命周期命令有一条
  可执行的审批路径。两者任一不满足时先解决或升级，不得创建一个注定无法闭环的
  dispatch。
- 🔴 CHECKPOINT · 🛑 STOP：只有用户已明确授权，或目标仓库/运行时策略已明确选择
  无人值守模式时，才可使用对应 CLI 的审批绕过参数；并把授权范围写入 work order
  的 `[权限]`。
- 无法在现有权限下继续时保留原始错误，并按上述裁决规则决定提升权限、更换执行
  方式或停止。不得把无人值守参数当作解决权限错误的默认办法。
- 无人值守参数不扩大工作单范围，也不覆盖仓库对 commit、push、部署、密钥、
  数据和破坏性操作的限制。

## Worker 与 worktree 隔离

- 同一问题的审查修复、补测或二次验证可复用原 worker terminal；每轮仍创建新的
  task/dispatch。模块、目标、工具链不相关，或旧上下文可能干扰判断时，创建新的
  terminal 和 agent session，不因方便而复用。
- 只读审计、顺序任务、依赖当前未提交内容或用户指定当前 worktree 的任务使用现有
  worktree。只有用户明确要求，或已确认共享 checkout 会造成文件、构建或工具冲突
  时才新建 worktree；并行本身不是新建理由。需要共享未提交修改时不得拆 worktree，
  先留在原 worktree 或解决可复现的基线。
- 新 worktree 的标准路径用
  `worker-start --worktree <new-child|new-top-level> --agent <id> --setup run`，并按
  依赖关系选择 lineage 与 base。读取回执中的 setup、
  effects 和 worker 状态；setup 仍在运行不等于失败。需要 custom argv 或预检时才
  走两步法；bare create 可能多开 fallback shell，只操作 agent handle，并经
  `terminal list` / `terminal show` 确认 fallback shell 未使用后才允许关闭。若仓库
  明确要求 wait-for-setup，而两步法不能保持该顺序，触发 🔴 CHECKPOINT · 🛑 STOP，
  不得静默绕过 setup 策略。
- 一个 worktree 可挂多个 terminal；terminal 在创建时用 `--worktree` 绑定，
  创建后不可移动。review、repair 或补测 worker 必须落在目标 worktree 时，向
  `worker-start --worktree id:<repoId>::<path> --agent <id>` 传完整两段式 ID；需要
  custom argv 或预检时先 `terminal create --worktree id:<repoId>::<path>`，再用
  `worker-start --terminal` 绑定。ID 从 `worktree create --json` 或
  `worktree list --json` 复制，不得凭 sidebar 位置或目录名推断归属。
- 创建 worktree 前检查 `git status`、当前 HEAD、目标 base ref 及依赖提交。不得假设
  Orca 的默认 base 已包含本地分支进度；需要当前分支内容时显式传入对应 branch/ref。
- Worktree 只隔离 checkout、分支与构建目录；共享环境按「DAG 与环境门禁」分配租约，
  无法隔离时改为串行。
- 独立 worktree 必须在工单和报告中记录完整 worktree ID、分支、起始 HEAD 与集成
  方式。未明确集成方式的并行代码任务不得开工。

## 可用性与额度

- 派单前用最小成本信号确认 worker 账户可用（CLI 启动、模型加载、一次平凡
  短任务的提交信号）。账户级用量上限不能靠新建 terminal 或降低模型/effort
  档位绕过；识别到 account-wide limit 后不得连续创建注定失败的 worker。
- Worker 因额度在任务中途终止时保留现场：原 terminal、非干净 worktree、
  base SHA、task/dispatch ID、已通过证据与未闭合 finding，禁止 reset/clean。
  同一 task 的替换 attempt 用 `worker-start --retry-of <dispatch_id>`，并重新显式
  指定 worker 与 placement；范围改变、审查或 repair 才创建带 parent 的新 task。
- 🔴 CHECKPOINT · 🛑 STOP：CLI 提示的 `usage limit reset` 与购买 credits 同属
  账户状态变更，只有用户裁决后才能消耗；coordinator 不得自行输入 `/usage` 或
  消耗 reset，需要时按裁决规则给出等待/消耗/放弃的选项。

## 实现类工单

- 代码实现、review 或 repair 在建 task 前完整读取
  [`references/review-policy.md`](references/review-policy.md)；按其中风险等级选择审查
  拓扑、验证证据、repair 预算和 effort。非代码只读工单不加载该 reference。
- 修改代码的工单优先选择能发现 `implement` skill 的 Codex worker，并在 `[工具]`
  中显式要求先调用该 skill；不得依赖模型从“实现”语义自动触发。发现顺序：
  用户显式提供的路径 > 目标仓库 `.agents/skills/` > 用户全局
  `~/.agents/skills/`；以实际读取到的绝对路径为准写入 `[工具]`，会话技能
  目录未列出不代表文件不存在。目标 worker 无该 skill 时，明确展开测试优先、
  持续类型检查/单测、最终充分验证和代码审查要求，不得声称已调用。
- `implement` 自带的 commit 步骤不覆盖仓库安全规则。未经用户明确授权，工单必须
  写明“禁止 commit/push，本条覆盖 implement 的 commit 步骤”；若采用独立任务
  分支提交来集成，先取得对应的本地 commit 授权，push 仍需单独授权。
- 工单必须显式声明 review 所有权：worker 不启动 Standards/Spec 子审阅；内部
  code review 只是交付前自检。独立 reviewer 由 coordinator 按风险策略创建受监督
  task/dispatch，不能用未跟踪的本地子代理替代。

## 低层派发启动核验

仅在按「Run 与派发路径」回退到低层 `dispatch --inject` 时执行本节；
`worker-start` 以其 receipt、`worker-show` 和 `worker-read` 为准。`injected: true`
只表示运行时接受了终端注入，不证明 agent 已提交 prompt：

1. 派发前用 `terminal read` 记录 `nextCursor`；派发后读取该 cursor 的增量和当前
   terminal 尾部，并用短时、带显式 `--timeout-ms` 的
   `terminal wait --for tui-idle` 检查状态。
2. 所选 worker 手册列出的本轮提交信号，以及新的 agent 输出、工具调用或
   lifecycle 消息，均可证明已启动；此时保持只读。`terminal wait` 返回
   `satisfied=false` / `blockedReason=codex-interactive-prompt` 通常不代表 idle，
   不得据此补发；只有 terminal 尾部明确停在当前 dispatch 的
   `[Pasted Content <N> chars]`、没有审批/问答菜单，且两次读取的 cursor 均未增长
   时，才把它记为“当前 prompt 待提交”信号。其他无法明确分类的状态继续观察并
   保留原始结果。
3. 仅当当前 dispatch 仍为 `dispatched`，且 `tui-idle` 明确满足或已取得上述
   “当前 prompt 待提交”信号，并在有界观察后没有本轮提交或活动证据时，补发一次。
   除 terminal 尾部已有明确待提交输入外，至少在两次 `terminal read` 之间完成一个
   短时、带显式 `--timeout-ms` 的 `check --wait` 窗口，并确认两次
   `nextCursor` 均未增长作为静默证据；若返回 Delivery，立即按下一节处理，
   不得把未处理消息当作静默：

   ```text
   orca terminal send --terminal <worker-handle> --text '' --enter --json
   ```

4. 补发后重新读取增量并确认 worker 已启动。仍无活动证据时保存 `dispatch-show`
   与 terminal 输出并升级；同一 dispatch 不得再次补回车、重新 `task-create` 或
   重复 `dispatch`。

只认派发前 cursor 之后的本轮证据；复用 terminal 时不得把旧工单的输出或 hook
当作当前 dispatch 已启动。

## Delivery 等待与恢复

1. 用带显式 timeout 的
   `check --wait --types "worker_done,escalation,question"` 等待。`count: 0` 或 timeout
   只是检查点；JSON keepalive 不是 worker heartbeat。每个 Delivery 是最老的完整
   FIFO 批次，type filter 只决定何时唤醒，不会裁剪返回批次。
2. 逐条处理整批消息：`question` 用消息 ID `reply`；`escalation` 取证后裁决；
   `worker_done` 核对当前 task/dispatch ID 和显式 `outcome`。`report-path` 与
   `files-modified` 存在时核对实际交付；零改动省略 `files-modified`，不得传字符串
   `[]`。失败 outcome 仍是有效终态，不得只在正文中隐含失败。
3. 只有整批消息的回复、记录和其他副作用全部成功后才 `check --ack <delivery_id>`；
   ack 前重启会重放同一批次，按 message ID 幂等处理，不重复 reply、裁决或验收。
   ack 后继续等待，直到所有预期 dispatch 都进入终态。
4. 有推理、工具输出、heartbeat 或其他活动时继续等待；不得仅因缺少 heartbeat 或
   一次静默超时停止 worker。优先用 `worker-show --dispatch` 判断状态，用
   `worker-read --dispatch` 读取有界输出；source 改变时丢弃旧 cursor 后重新读取。

按 `worker-show` 的可证明状态恢复：

- `ready`：继续等待或读取输出。
- `failed` / `stopped`：同一 task 的替换 attempt 用
  `worker-start --retry-of <dispatch_id>`，并重新显式指定 worker 与 placement；
  retry 不继承原 placement。
- `outcome_unknown`：先处理原 Run 中 pending `question`，再执行 receipt 或动态指南
  返回的精确 recovery。需要停止且可能丢失仍在进行的工作时，触发
  🔴 CHECKPOINT · 🛑 STOP，由用户在继续等待、`worker-stop` 或保留现场之间裁决。
  `worker-stop` 后重新检查；若选择 `worker-abandon`，必须声明进程和资源可能仍存活，
  不得在同一 worktree、Compose project 或其他冲突资源上启动替代 worker。
- Runtime 或契约更新时遵循动态指南返回的 authority label 与精确恢复参数；不得仅因
  runtime 重启就启动替代 editor，也不得把 legacy read-only 状态升级为写权限。

低层派发已确认启动后，若 terminal 回到输入态却没有 `worker_done`，检查审批提示、
自动拒绝、CLI 解析错误和最终回复。明确拒绝、解析错误或最终回复已结束但消息未送达
属于 lifecycle delivery failure：保存 `dispatch-show`、terminal 增量和原始错误，
按动态指南收敛 task/dispatch 状态；不得补发空回车、重复 dispatch 或由 coordinator
代发 `worker_done`。状态含糊或只有暂时空闲时继续有界观察。

## 验收与对抗审查

- 固定审查点是 worker 交付报告的最终 SHA（`worker_done` 后的 hash，不是中途
  HEAD）；未授权 commit 的工单改为审查干净度核对后的工作树 diff，并在审查
  记录中写明审查对象形态。
- 按 `review-policy.md` 的风险等级派 combined 或并行双轴 reviewer；优先遵循目标
  仓库或可发现的 `code-review` 规则。Coordinator 独立核对关键 finding、命令、
  数字和环境证据，不代替 worker 修改，也不以 reviewer 多数票裁决可信 blocker。
- 外审期间冻结审查对象：reviewer 启动后 worker 不得 amend；发生的 amend 使原
  SHA 结论失效。按风险策略重做 targeted/closure review；外审未完成时用 gate 或
  明确的 review-complete 状态挡住验收解锁。
- 验收分别核对 task 状态、pending/resolved gate、消息的 task/dispatch ID 与
  固定 SHA：pending gate 不会阻止 worker 发送 `worker_done`，三者状态不一定
  同步，不得把 worker 自报完成或单条 `worker_done` 当作验收通过。
- 有效 `worker_done` 会按显式 `outcome` 自动收敛 task/dispatch：`succeeded` 置为
  completed，`failed` 置为 failed；不要再手动 `task-update completed`。终态只表示
  attempt 结束，不是 coordinator 验收。下游不得仅因实现 task completed 就开工；
  可返工代码的下游用 coordinator gate 保持阻塞；不要依赖可能被替换的单轮 review
  task。只有固定审查点、repair、复验与约定集成全部闭环后才 resolve gate。
- Issue/ticket 状态只在 coordinator 验收通过且用户已授权对应 tracker 写入后推进；
  未授权时只报告建议状态和证据，不替 worker 或用户修改外部状态机。
- 发现 finding 时在同一 worktree 派 repair（复用 terminal 或按隔离节新建绑定
  terminal），每轮创建新的 task/dispatch；按 review policy 的预算、反例和 effort
  门禁修复，并以新的最终 SHA 重走对应审查点。

## 监督式流程

1. 运行 `orca skills get orca-cli`、`orca skills get orchestration` 和
   `orca status --json`，以当前动态指南为命令与生命周期的事实来源。
2. 选择 worker、模型和 effort，完整读取唯一匹配的 worker 手册；需要确认的
   高成本配置、Orca CLI 解析方式或权限提升在派单前按裁决规则解决。
3. 新监督目标创建 Run；继承 task、dispatch 或消息时核实并绑定原 Run。枚举当前
   Run 的 brief/ready task 与已有 dispatch，避免重复创建。
4. 按「Run 与派发路径」选择启动方式。标准路径先创建 task，再用 `worker-start`
   启动；需要 custom argv 或 worker-side 预检时，先创建最终 terminal、等待其越过
   登录/信任/启动提示并完成预检，再创建 task，用 `worker-start --terminal` 绑定。
   只有组合命令无法表达时才走低层派发及其启动核验。
5. 读取 `worker-start` receipt 并核对 task/dispatch provenance。`ready` 才进入等待；
   failed 或 outcome_unknown 按「Delivery 等待与恢复」处理，不猜测或自动重试。
6. 按 FIFO Delivery 循环等待并处理 `worker_done`、`escalation`、`question`；完成
   整批副作用后 ack，直到所有预期 dispatch 终结。
7. 按「验收与对抗审查」节执行固定审查点的风险分级外审与独立复验；finding 按该节
   派 repair 闭环。
8. 同范围返工可复用 terminal，但每轮创建新的 task/dispatch；无关任务使用新 terminal。
   以当前 active dispatch 的 ID 对应的有效 `worker_done` 作为本轮完成信号。

## 禁止事项

- 禁止在继承 task、dispatch、Delivery 或 legacy assignment 时另建 Run 逃避旧状态。
- 禁止在 Delivery 未全部处理成功时 ack，或在重放后重复 reply、裁决、验收和外部写入。
- 禁止把 `outcome_unknown`、runtime 重启、一次 timeout 或暂时 idle 当成失败并盲目重派。
- 禁止在 `worker-abandon` 后向可能冲突的 worktree、Compose project、端口或共享服务
  派替代 worker；abandon 不证明进程已停止。
- 禁止 coordinator 代发 `worker_done`，或把 task completed、worker 自报和单条
  `worker_done` 当成 coordinator 验收及下游解锁依据。

## Worker 关闭与 worktree 回收

- `worker_done` 只表示 worker 进入空闲交付态，不是立即关闭信号。Coordinator
  复验期间保留 terminal，以便对同范围问题创建新的 task/dispatch 返工。
- 同时满足以下条件后用 `worker-stop --dispatch` 关闭受监督 worker并确认结果；
  低层派发才按 terminal handle 关闭：当前 dispatch 的 `worker_done` 有效；
  coordinator 已检查交付物并完成关键复验；报告、日志和失败证据已记录；不存在
  未决 `question`、gate 或同范围返工；代码改动已经安全集成或明确保留在
  可恢复的 worktree 中。
- 只读 worker 在报告复核通过后即可关闭。失败或阻塞的 worker 要先保存诊断证据并
  完成继续、替换或放弃的裁决；不得仅因等待超时、暂时无输出或只有 heartbeat
  就关闭、停止或重启。
- 🔴 CHECKPOINT · 🛑 STOP：Terminal 与 worktree 分开回收；terminal 达到上述条件
  即可关闭。独立 worktree 必须保留到改动集成并完成最终回归；删除前必须核对用户
  授权、精确目标和可恢复性。未集成、工作树非干净或恢复方式不明确时禁止删除。
- 已验收且预计无同范围返工的 worker 应及时关闭，避免长期保留无关上下文。之后若
  再发现问题，创建新 worker，并把原 task、审查结论和验证证据写入新工单。
- `worker-abandon` 只封存 lifecycle authority，不关闭进程或回收资源；不得把它
  记为 worker 已关闭，也不得据此删除 worktree。

## Work order

```text
[角色] EXECUTOR：只完成本工作单；二次派单需要 coordinator 明确授权。
[目标] 要完成的结果及可观察的完成判据。
[前提] 基线事实（测试数、构建退出码等，带来源）、本地工具链与环境敏感性清单、
  核对命令和清理责任；worker 动工前先核对，对不上立即停止并 escalation。
[范围] 允许修改的路径白名单；白名单之外一律只读，顺手发现的无关问题记入
  报告待裁决，不直接改。
[规格] 权威文件及优先级；先读根级与范围内的 AGENTS.md。
[约束] 项目安全红线、本工单限制和允许的外部操作；coordinator 未逐项确认的
  默认决定逐条标注“猜的”及猜错代价。
[权限] 审批/沙箱模式，以及 commit、push、部署和外部系统操作的授权边界。
[工具] 必须使用的工具及确定路径；例如 implement skill 的绝对路径、E2E 明确
  指定 Playwright CLI。
[隔离] Worker 是否复用、worktree/base、资源租约、端口/Compose project/数据隔离、
  共享服务操作权和集成方式。
[验证] 冻结的权威入口、runner/参数、环境、场景、预期结果与证据；变更需声明并
  独立复核。基线不可退；skipped 变化逐条归因，未授权新增 skip 视为失败。
  “坏了没人知道”的检查先故意制造一次失败，给出红→绿证据。
[审查] 风险等级、review 所有权与拓扑、固定审查对象、verdict/gate、repair/closure
  预算；worker 不自行启动独立 reviewer，由 coordinator 按 review policy 派发。
[止损] 同一验收连败 3 次停止并 escalation；结果比基线差则回滚并如实报告；
  “没做成但说清了”是合格交付，“做了但更糟”不是。
[报告] outcome / 结论 / 偏差 / taskId 与 dispatchId / worktree 与改动文件 /
  审查对象 / 实际模型与 effort / 验证证据（命令、退出码、关键数字）/
  可选 report path。零改动时不传 files-modified；不得用字符串 [] 冒充空列表。
```

Worker 视为零历史；长规格落盘后引用路径。报告以工作区文件和真实命令为依据，
并使用当前 dispatch 注入的 taskId/dispatchId 回报。代码修改只有在审查问题修复且
coordinator 复验通过后闭环；只读审计保持只读，发现的实现缺口另建工作单。
