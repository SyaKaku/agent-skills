# Codex worker

仅在选择 `worker=codex` 时读取。这里保存 Codex 的模型路由和启动差异；通用
work order、Orca 生命周期、权限边界与收尾规则以父级
[`SKILL.md`](../SKILL.md) 为准。

## 模型路由

下表是推荐配置，不是可用性事实。使用前根据当前 Codex CLI、账号能力和
`orca status` 确认模型与 effort 可用；产品版本变化时优先遵循动态指南。

| 配置 | 使用场景 |
|---|---|
| `gpt-5.6-terra` + `medium` | 文档、查询、机械替换、小脚本、可自动计数的门禁 |
| `gpt-5.6-terra` + `high` | 规格清楚的单模块实现或修复，验收命令明确 |
| `gpt-5.6-sol` + `high` | 跨层实现、测试体系审查、Playwright E2E、架构权衡、复杂诊断、视觉判断 |
| `gpt-5.6-sol` + `xhigh` | 已有证据但仍有多个合理假设，需要反证或跨模块影响分析 |
| `gpt-5.6-sol` + `max` | 不可逆架构/迁移、安全授权、资金或数据一致性门禁、重大事故 RCA |
| `gpt-5.6-sol` + `ultra` | 最高影响的一次性终审；`max` 后仍未收敛且漏判代价重大 |

用户未指定配置时，选择满足任务的最低充分档位：例行任务用 Terra `medium`，
规格明确的单模块实现用 Terra `high`，复杂工单用 Sol `high`。先补足范围、证据
和验收标准，再判断是否需要 `xhigh`。`max` 与 `ultra` 仅配 Sol，并满足以下条件：

- 用户显式指定，或 coordinator 说明延迟、消耗和收益后通过 `decision_gate`。
- `max` 对应高风险且质量优先的决策或调查，而不是普通功能和例行 review。
- `ultra` 还要求当前 Codex 环境明确支持，并且 `max` 已不足以可靠收敛。

用户指定的配置优先。CLI、账号或服务端拒绝时保留原始错误，通过
`decision_gate` 决定更换配置或停止。

## 启动

先确认 `codex --version` 和所选模型可用，再按动态 `orca-cli` 指南创建 terminal。
基础命令不主动关闭审批或沙箱：

```text
codex --model <gpt-5.6-sol|gpt-5.6-terra> \
  -c model_reasoning_effort="<medium|high|xhigh|max|ultra>"
```

只有用户已明确授权，或目标仓库/运行时策略已明确选择无人值守模式时，才可附加：

```text
--dangerously-bypass-approvals-and-sandbox
```

把该授权写入 work order 的 `[权限]`，并再次核对工作区、外部系统和破坏性操作
边界。若没有授权，保留正常审批与沙箱；自动化因此无法继续时使用
`decision_gate`，不要自行升级权限。`--yolo` 属于 Kimi，不适用于 Codex。

同范围修复轮可复用 Codex terminal；无关任务使用新 terminal。自定义 worktree
时只向新建的 Codex agent handle 派单，并确认它能看到任务所依赖的提交或本地改动。
