# Kimi worker

仅在选择 `worker=kimi` 时读取。这里保存 Kimi 的模型路由和启动差异；通用
work order、Orca 生命周期、权限边界与收尾规则以父级
[`SKILL.md`](../SKILL.md) 为准。

## 模型路由

下表是推荐配置，不是可用性事实。使用前根据当前 Kimi CLI、账号能力和
`orca status` 确认模型与 effort 可用；产品版本变化时优先遵循动态指南。

| 配置 | 使用场景 |
|---|---|
| `kimi-code/kimi-for-coding` | 文档、查询、小改、规格明确的实现和机械工单 |
| `kimi-code/kimi-for-coding-highspeed` | 输出量大且重复、可机械验证的批量工单 |
| `kimi-code/k3` + `high` | 架构权衡、疑难根因、大范围审查、设计判断或长上下文 |
| `kimi-code/k3` + `max` | 已充分取证仍长期不收敛、不可逆决策或一次性高影响设计 |

用户未指定配置时，例行任务使用 `kimi-for-coding`，仅在输出量大且可机械验证时
使用 `kimi-for-coding-highspeed`，复杂工单使用 K3 `high`。K3 `max` 仅在用户
显式指定，或 coordinator 解释延迟、消耗和收益后通过 `decision_gate` 时使用。
输入缺少范围、证据或验收标准时，先补齐工作单。用户指定的配置不可用时保留
原始错误并通过 `decision_gate` 处理。

## 启动

先确认 Kimi CLI、登录状态和所选模型可用，再按动态 `orca-cli` 指南创建
terminal。基础命令不主动关闭审批或沙箱：

```text
kimi -m kimi-code/kimi-for-coding
kimi -m kimi-code/kimi-for-coding-highspeed
kimi -m kimi-code/k3 --effort high
kimi -m kimi-code/k3 --effort max
```

只有用户已明确授权，或目标仓库/运行时策略已明确选择无人值守模式时，才可附加：

```text
--yolo
```

把该授权写入 work order 的 `[权限]`，并再次核对工作区、外部系统和破坏性操作
边界。若没有授权，保留正常审批与沙箱；自动化因此无法继续时使用
`decision_gate`，不要自行升级权限。Codex 的审批绕过参数不适用于 Kimi。

同范围修复轮可复用 Kimi terminal；无关任务使用新 terminal。自定义 worktree
时确认 worker 能看到任务依赖的提交或本地改动，并避开共享端口与服务冲突。
