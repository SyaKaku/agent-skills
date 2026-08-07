# 规格与 DAG 调度

仅在 Tier 2 或多 task DAG 中读取。通用 Run、派发、Delivery、权限、验收与回收规则
以父级 [`SKILL.md`](../SKILL.md) 为准。

## 人工规格 checkpoint

`grill-with-docs`、`to-spec` 和 `to-tickets` 禁止模型自动调用。按已有产物进入最靠后的
适用步骤；缺少产物时停止建 task，并请用户显式调用对应 skill：

1. 想法仍需产品或架构裁决：请用户调用 `$grill-with-docs`，直到关键选择写入
   `CONTEXT.md` / ADR。
2. 讨论已定稿但没有权威 spec：请用户调用 `$to-spec`，取得 PRD 或 issue tracker
   中的精确 spec。
3. Spec 已批准但没有 tickets：请用户调用 `$to-tickets`，取得带 `Blocked by:` 边和
   验收标准的 tickets。

只在对应产物存在且用户已批准后继续。不得模拟 skill 调用，不得把拷问、写 spec 或
拆 ticket 派给零历史 worker。用户直接提供已批准 tickets 时从 DAG 映射开始。

## Task 与验收屏障

先从 tickets 的真实目录或 tracker 枚举文件/记录，生成：

`ticket ID → 精确路径 → blocker IDs → implementation task → acceptance task → 写集 → 环境需求`

检查缺失 ID、重复 ID、自依赖和完整环路径。发现环时保持相关 ticket blocked 并请求
用户裁决；不得删边、猜文件名或创建占位 task。已批准依赖和 ticket 粒度不得为压平
DAG 而改变。

按拓扑顺序为每个 ticket 建两类 task，标题显式传 `--task-title`（人类可读），不依赖
spec 首行派生：

1. **Implementation task**：只有它会 dispatch 给 worker；依赖必须落为
   `task-create --deps <json_array>`，取值为其 blocker tickets 的 acceptance task
   IDs，不得只记在正文。
2. **Acceptance task**：依赖本 ticket 的 implementation task；创建后立即在它上面
   `gate-create`。它由 coordinator 管理，不启动 terminal、不 dispatch worker。

Implementation 的有效 `worker_done` 只收敛 implementation task。固定审查点、repair、
独立复验和约定集成全部闭环后，coordinator 才 resolve acceptance gate，并执行
`task-update --id <acceptance-task-id> --status completed --result <json>`，记录固定
SHA/diff、review verdict 和
验证证据。下游只依赖 acceptance task；验收前 `task-list --ready` 不得出现下游。

误建或重复 task 无法安全删除时标 failed，并在 `task-update --result` 中记录
`supersededBy` 与原因。

## Context cluster 与波次

Task 粒度与 agent 粒度分离。每个已批准 ticket 保留独立 task/provenance，再建立两种
边：

- **Conflict edge**：预期写集、构建目录、端口、数据库、Compose project、账号、缓存
  或共享服务冲突。相连 task 不得在同一波次；无法确认写集或隔离性时视为 conflict。
- **Affinity edge**：共享规格、模块、调用链或验证入口。相连的顺序 task 归入同一
  context cluster，默认复用一个 terminal 逐 task dispatch。

每个 implementation task 恰好属于一个 context cluster；同一 cluster 同时最多运行
一个 task。`ready` 只进入候选池。每个波次从候选池选取不含 conflict edge 的 task，
最多使用 2 个活跃 worker terminal，并优先推进关键路径。用户显式指定更低上限时收紧，
只有用户显式指定更高上限时才扩大。当前波次终结后重新计算 ready、conflict 与预算。

创建 task 前输出派单预算：implementation/acceptance task 数、context cluster/terminal
数、独立 reviewer 数、最大同时运行 worker 数及 full/targeted review 次数。所有
implementation task 必须恰好出现一次；任何同波次 conflict 或超预算使门禁失败。

## 环境与租约

列出本地工具链、外部服务、端口、代理/VPN、容器、数据库、测试账号、共享缓存，及其
启动、重启和清理责任人。Coordinator 先执行安全只读探针；失败只阻塞受影响 task。

为每个活跃 worker 分配资源租约并写入工单和报告。Worktree 不隔离 Docker daemon、
Compose project、端口、数据库或账号；Compose 使用唯一 project 名，重型 build/restart
错峰。共享 daemon/服务的启动、重启和全局清理由 coordinator 独占；缺少对应授权时
停止。清理时逐项核对租约释放。
