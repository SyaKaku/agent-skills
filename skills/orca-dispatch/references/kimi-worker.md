# Kimi worker

仅在选择 `worker=kimi` 时读取。这里保存 Kimi 的模型路由和启动差异；通用
work order、Orca 生命周期、权限边界与收尾规则以父级
[`SKILL.md`](../SKILL.md) 为准。

## 模型路由

以下规则以 Kimi Code 官方的
[模型配置](https://www.kimi.com/code/docs/kimi-code/models.html) 为能力基线。
先按上下文、多模态和速度要求选择模型，再仅对 K3 选择 effort。使用前根据当前
Kimi CLI、账号套餐和 `orca status` 确认配置可用；产品版本变化时优先遵循动态
指南。

| 模型 | 匹配规则 |
|---|---|
| `kimi-code/k3-256k` | 默认首选。适合上下文不超过 256K 的日常问答、代码补全、常规功能开发及单文件或少量文件修复；支持图片，不支持视频，需要 Moderato 或更高套餐 |
| `kimi-code/k3` | 仅在预计上下文会超过 256K、必须保留已超过 256K 的会话，或任务需要 K3 处理视频时使用；上下文不超过 256K 且无视频时改用 `k3-256k`，效果相同且消耗约为一半；需要 Moderato 或更高套餐，Allegretto 或更高套餐才支持最高 1M 上下文 |
| `kimi-code/kimi-for-coding` | K2.7 Code 标准版，适合代码补全和常规开发；所有会员可用，可作为无 K3 权限时的回退，并支持图片和视频 |
| `kimi-code/kimi-for-coding-highspeed` | 与 K2.7 Code 标准版编码能力一致，输出约快 5–6 倍但消耗约为 3 倍；仅用于延迟敏感且耗时主要来自模型输出的任务，工具或脚本执行占比高时不升级；需要 Allegretto 或更高套餐 |

K3 与 K3-256K 支持以下 effort：

| Effort | 匹配规则 |
|---|---|
| `low` | 范围窄、证据充分、可机械验证，且明确优先降低推理消耗或延迟 |
| `high` | 默认且推荐；用于常规实现、调试、审查和需要综合判断的任务 |
| `max` | 已充分取证仍难以收敛、不可逆决策或一次性高影响设计 |

K2.7 Code 两个模型不使用 effort 档位，但必须保持 Thinking 开启。对 K3 或 K2.7
关闭 Thinking 会被路由到 K2.6。

用户未指定配置时使用 K3-256K `high`；没有 K3 权限时回退到
`kimi-for-coding`。仅在满足上表的上下文或视频条件时升级到 K3 1M，仅在明确
需要降低输出延迟时改用 HighSpeed。K3 `max` 仅在用户显式指定，或 coordinator
解释延迟、消耗和收益后按父级裁决规则取得确认时使用。输入缺少范围、证据或验收
标准时，先补齐工作单。用户指定的配置不可用时保留原始错误并按父级裁决规则处理。

## 启动

先确认 Kimi CLI、登录状态和所选模型可用，再按动态 `orca-cli` 指南创建
terminal。基础命令不主动关闭审批或沙箱：

```text
kimi -m kimi-code/k3-256k
kimi -m kimi-code/k3
kimi -m kimi-code/kimi-for-coding
kimi -m kimi-code/kimi-for-coding-highspeed
```

当前 Kimi Code CLI 的 `-m` 只选择模型别名，不提供 `--effort` 启动参数。K3 的
effort 应在 `/model` 中选择，或使用现有 `config.toml` 的 `[thinking] effort`
配置；启动 worker 前确认生效值符合 work order，不要为此擅自修改用户的全局
配置。切换模型或 effort 会使既有上下文缓存失效，需切换时优先创建新 session，
不要在长会话中反复切换。

父级无人值守条件满足时，Kimi 对应参数是 `--yolo`；Codex 的审批绕过参数不适用
于 Kimi。
