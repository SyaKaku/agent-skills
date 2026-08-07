# Codex worker

仅在选择 `worker=codex` 时读取。这里保存 Codex 的模型路由和启动差异；通用
work order、Orca 生命周期、权限边界与收尾规则以父级
[`SKILL.md`](../SKILL.md) 为准。

## 模型路由

下表以 OpenAI 当前
[GPT-5.6 模型指南](https://developers.openai.com/api/docs/guides/latest-model) 为能力基线，
但不是账号可用性事实。使用前根据当前 Codex CLI、账号能力和 `orca status` 确认模型
与 effort 可用；产品版本变化时优先遵循动态指南。

| 配置 | 使用场景 |
|---|---|
| `gpt-5.6-terra` + `medium` | 文档、查询、机械替换、小脚本、可自动计数的门禁 |
| `gpt-5.6-terra` + `high` | 规格清楚的单模块实现或修复，验收命令明确 |
| `gpt-5.6-sol` + `high` | 跨层实现、测试体系审查、Playwright E2E、架构权衡、复杂诊断、视觉判断 |
| `gpt-5.6-sol` + `xhigh` | 已有证据但仍有多个合理假设，需要反证或跨模块影响分析 |
| `gpt-5.6-sol` + `max` | 不可逆架构/迁移、安全授权、资金或数据一致性门禁、重大事故 RCA |

用户未指定配置时，选择满足任务的最低充分档位：例行任务用 Terra `medium`，
规格明确的单模块实现用 Terra `high`，复杂工单用 Sol `high`。先补足范围、证据
和验收标准，再判断是否需要 `xhigh`。`max` 仅配 Sol，并满足以下条件：

- 用户显式指定，或 coordinator 说明延迟、消耗和收益后按父级裁决规则取得确认。
- `max` 对应高风险且质量优先的决策或调查，而不是普通功能和例行 review。

监督式 worker 不使用 `ultra`：它会引入 Codex 自动任务委派，绕过 coordinator 的
Orca provenance、并发预算和 reviewer 所有权。用户为监督式任务指定 `ultra` 时，派单前
请其在 `max` 与不监督的 full handoff 之间选择，不得静默降级。

用户指定的配置优先。CLI、账号或服务端拒绝时保留原始错误，按父级裁决规则决定
更换配置或停止。

## 启动

先确认 `codex --version`、所选模型和生命周期发送路径可用，再按动态 `orca-cli`
指南创建 terminal。Coordinator 用 `(Get-Command orca -ErrorAction Stop).Source`
记录实际 Orca 可执行文件；若 worker PowerShell 不能解析裸 `orca`，在 `[工具]`
中提供该完整路径并要求 lifecycle 命令直接使用它，不让 worker 扫描安装目录。
不得用 coordinator shell 的解析结果代替 worker 实测。原生 Windows 的 Codex
sandbox 可能保留 Orca bin 的 `PATH` 项，却仍拒绝读取安装目录或连接 Orca runtime；
因此完整路径只解决定位问题，不证明命令可执行。

基础命令不主动关闭审批或沙箱：

```text
codex --model <gpt-5.6-sol|gpt-5.6-terra> \
  -c model_reasoning_effort="<medium|high|xhigh|max>"
```

### 生命周期审批

Codex 的 `approvals_reviewer="auto_review"` 可能把 `orca orchestration send` 判为高风险
并自动拒绝；完整 Orca 路径只解决命令解析，不解决审批。不要缓存某个 CLI 版本的结论；
每次派单前按以下优先级选择并验证一条路径：

1. 优先复用用户或仓库策略中已授权的精确 execpolicy `allow` rule，只匹配实际
   Orca 可执行文件的 `orchestration send` 前缀；用
   `codex execpolicy check --rules <file> -- <orca-executable> orchestration send ...`
   分别验证 `heartbeat` 与 `worker_done` 命令命中 `allow`。未经用户授权不得新增
   或扩大 rule。
2. 需要用户逐次裁决时，用
   `--ask-for-approval on-request -c approvals_reviewer="user"` 启动，并在 terminal
   出现审批提示时请用户处理；lifecycle shell 调用必须显式请求
   `sandbox_permissions="require_escalated"` 并说明只执行当前 Orca 命令。
   仅启动 user reviewer、仅传完整路径或仅出现审批提示都不算路径可用；用户批准且
   命令成功返回才算通过。这不是无人值守路径。
3. 只有父级无人值守条件满足时，才使用
   `--dangerously-bypass-approvals-and-sandbox`。`--ask-for-approval never`
   不等于授权被策略拒绝的命令，不得把它当作替代。

没有可用路径时在 `task-create` / `dispatch` 前停止并升级。不得先完成长任务，再把
`worker_done` 能否发送留到收尾阶段验证。

### Worker-side 生命周期预检

在 `task-create` 前，coordinator 生成本轮唯一的 `preflightId`，记录 coordinator
与候选 worker 的具体 terminal handle，再从最终候选 Codex terminal 使用正式派单
将采用的 executable、sandbox、approval reviewer 和执行方式完成预检：

1. 运行 `<orca-executable> status --json`，确认 `runtime.reachable=true`。
2. 由 worker 使用同一审批路径向具体 coordinator handle 发送一条不带 task /
   dispatch ID 的 `status`，`--from` 使用候选 worker handle，subject 包含唯一
   `preflightId`。该消息既是 outbound orchestration 探针，也是 worker 对
   coordinator 的主动预检回报。
3. Coordinator 用带显式 timeout 的
   `orchestration check --wait --types status` 接收 FIFO Delivery，核对 sender handle
   与 `preflightId`；保存证据后处理整批消息并 ack。不得用旧 status、worker TUI
   最终回复或 coordinator 的 terminal 轮询代替。
4. 保存命令、审批方式、退出码、关键 JSON 字段和匹配的 status message。审批仍
   pending、命令在 sandbox 内失败、runtime 不可达、消息未送达或来源不匹配时，
   关闭候选 terminal 或保留诊断现场，并在创建 task 前停止。

```text
<orca-executable> orchestration send \
  --from <worker-handle> --to <coordinator-handle> --type status \
  --subject "Codex lifecycle preflight passed: <preflightId>" \
  --body "runtime reachable; outbound orchestration probe passed" --json

orca orchestration check --terminal <coordinator-handle> \
  --wait --types status --timeout-ms 30000 --json
```

Coordinator 本地成功、`Get-Command` 成功、`PATH` 含 Orca bin 或审批框成功弹出均
不能替代上述 worker-side 成功证据。预检只证明 transport 和审批路径可用；正式
`heartbeat` / `worker_done` 仍须由 worker 使用相同路径发送。预检通过后按父级流程
创建 task，并用 `worker-start --terminal <candidate-handle>` 绑定该 terminal；只有
当前动态指南证明组合命令无法表达所需拓扑时才回退低层 `dispatch --inject`。

### 派发后信号

按父级启动核验处理恢复动作。Codex 可能把长文本显示为
`[Pasted Content <N> chars]`；本轮 terminal 尾部停在该占位符，且没有新的
`UserPromptSubmit`、推理或工具输出，是 prompt 仍待提交的强证据。
`UserPromptSubmit` 只在对应 hook 可用时作为正向证据，缺少该 hook 本身不能判定
卡住。复用 terminal 时只检查派发前 cursor 之后的信号，不得用旧工单留下的
`UserPromptSubmit` 或输出通过本轮核验。

`--yolo` 不适用于 Codex。自定义 worktree 时只向新建的 Codex agent handle
派单，并确认它能看到任务依赖的提交或本地改动。
