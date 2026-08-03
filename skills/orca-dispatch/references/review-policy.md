# 代码验收与风险审查

仅在派发代码实现、review 或 repair 时读取。通用生命周期、权限、DAG 与回收规则
以父级 [`SKILL.md`](../SKILL.md) 为准；本文件只负责审查拓扑、验证证据和修复预算。

## 风险分级与审查拓扑

建实现 task 前按最高适用风险定级，并把等级、理由、review task/gate 和退出条件写入
工单：

| 等级 | 典型边界 | 首轮独立审查 |
| --- | --- | --- |
| 常规 | 局部 CRUD、文案、机械重构，不改变权限或数据范围 | 一个 combined reviewer，报告分列 Spec / Standards |
| 边界 | 权限或数据范围、跨层过滤、外部接口兼容 | 先 combined；发现主体越权、上游过滤或安全边界 finding 时追加定向 Standards reviewer |
| 高风险 | OAuth、会话、密码、支付、隐私、数据迁移、并发一致性、凭据或回滚 | 同一固定审查对象上并行派独立 Spec 与 Standards reviewer |

无法可靠定级时采用更高一级。Reviewer 使用独立 Orca task/dispatch 和 agent session，
只读固定 SHA；未经 commit 授权时审查 coordinator 已核对干净度的固定 diff。Spec 核对
需求、AC、业务状态、权限结果和异常路径；Standards 核对工程规则、架构边界、安全
不变量、并发/存储、发布回滚与测试可信度。

Review task 的 lifecycle outcome 只表示审查是否完成：报告完整送达时发送
`outcome=succeeded`，即使 verdict 是 `blocked`；审查无法完成或证据无法交付时才发送
`outcome=failed`。报告另列 `verdict=pass|blocked|inconclusive`。原始 review task 不作
下游稳定依赖；coordinator gate 只在最终 verdict pass 且独立复验闭环后 resolve。
两轴结论不得多数表决或互相抵消。任一带审查对象、`file:line`/可复现实验、影响和
最小修复的可信 blocker，必须修复或由 coordinator 用反证逐项驳回。

## Repair、收口与 Effort

- 首轮 finding 后派新的 repair task。复审默认只覆盖 finding、受影响调用链、回归
  矩阵和修复可能引入的至少一个反例；审查对象变化后不得沿用旧 SHA 结论。
- 常规/边界任务若 repair 未扩大范围且 targeted 复审证据完整，targeted 本身即为
  closure；只有跨模块、公共契约变化或仍有残余风险时，才追加至多一次 final combined
  sweep。高风险任务的最终 closure 必须在完整最终 baseline diff 上再做一次并行双审，
  不能只看最后一笔补丁；不要在每次小修后重复完整双审。
- 默认迭代预算是“实现 → 首轮审查 → 一次 repair → final closure”。先按根因与安全
  不变量合并多 reviewer 的重复 finding；第二个独立 HIGH 或同一边界第二次返工只触发
  重新路由。已有证据仍留下多个合理假设，或需要跨模块竞态、凭据代际/回滚分析时，
  Codex repair 才升到 `gpt-5.6-sol` + `xhigh`；否则保持手册中的最低充分档位。
- Effort 必须在 terminal 启动时配置并核对实际生效值；`dispatch` 不负责切换 effort。
  在 work order、启动回执证据和报告记录模型、effort 与升级理由。需要 `max`/`ultra`
  或 Kimi `max` 时按 worker 手册触发用户裁决，不得把高档位设为全局默认。
- 第三轮仍未闭环，或修复开始反复改变状态模型/接口语义时，触发
  🔴 CHECKPOINT · 🛑 STOP：停止增量补丁，重新设计状态模型或接口后再建新工单。

## 验证证据

- 冻结权威入口脚本、runner、参数、环境与预期结果；优先执行项目入口脚本，不用会
  误伤片段文件的宽 glob。加严验收同样属于变更，必须在报告中声明并独立复核。
- 不以 exit code 0 代替参数生效。核对 runner 实际执行命令、目标 project/filter、
  测试报告和 unknown-option 警告；权威 wrapper 本身必须通过，不能用其内部命令单独
  成功替代。
- 固定外部服务、端口、代理/VPN、容器和必要环境变量。Detached 服务同时核对进程、
  日志和端口，并在有上限的 readiness 窗口后裁决；失败时先确认旧进程是否仍在启动，
  不重复拉起冲突实例。
- 比较测试数、覆盖率与 skipped 基线。任何 skipped 变化都逐条归因；新增 skip、覆盖率
  或测试数退化若未获授权则验收失败。长测试仅在审查对象与环境未变时复用实现 worker
  保存的可信报告；reviewer 审查源码与证据，coordinator 抽跑关键路径，不机械重跑
  全部长矩阵。
- Worker 活跃时 coordinator 在同一 worktree 只做只读检查；等其 idle/`worker_done`
  后再运行构建，或使用独立 review worktree，避免争用 `target/`、缓存和文件锁。
- 接受 `worker_done` 前确认此前 coordinator guidance 已送达并被明确认领。仍有 queued
  指令时把完成视为 provisional；不要改写旧 dispatch，交付指令后创建新的验证或
  repair task/dispatch。
- 高风险任务选择适用反证：输入的 absent/valid/malformed/duplicate；真实存储竞争；
  success/success、success/failure、failure/success、failure/failure；过滤器、参数
  绑定、handler/finally、异常映射顺序；凭据 generation、迁移、发布与回滚窗口。

## 交付记录与禁止事项

报告分别保留 Spec / Standards 结论、固定审查对象、每项 finding 的处置、实际命令与
参数、退出码、测试/覆盖/skipped 数字、模型/effort、环境差异和剩余外部门禁。

- 禁止用 review 多数票覆盖可信 blocker。
- 禁止用 worker 自报、exit code 0、mock 顺序用例或最后一笔 diff 代替适用的完整证据。
- 禁止静默改验收命令、runner、环境、断言、skip 或被测对象来取得绿色结果。
- 禁止 coordinator 与活跃 worker 在同一 worktree 并发运行写构建产物的命令。
- 禁止因一次 finding 就全局升级 effort，或用重复完整审查消耗上下文而不缩小假设。
