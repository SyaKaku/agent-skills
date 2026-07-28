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
terminal。Kimi worker 默认以 `--yolo` 启动（自动批准常规工具调用，不抑制
提问与升级）：监督式 dispatch 要求 lifecycle 消息不被审批提示阻塞，实测
默认审批配置下 `orca orchestration send` 会停在 `▶ Run this command?`。
这属于本仓库策略明确选择的启动模式，已满足父级无人值守条件；`--yolo`
不扩大工单范围，也不覆盖仓库安全红线，但每次派单仍须在工单 `[权限]`
中写明。基础命令：

```text
kimi -m kimi-code/k3-256k --yolo
kimi -m kimi-code/k3 --yolo
kimi -m kimi-code/kimi-for-coding --yolo
kimi -m kimi-code/kimi-for-coding-highspeed --yolo
```

当前 Kimi Code CLI 的 `-m` 只选择模型别名，官方没有 `--effort` 启动旗标。K3 的
effort 有三个来源：TUI 内 `/model`、`config.toml` 的 `[thinking] effort`，以及
每次启动生效的环境变量 `KIMI_MODEL_THINKING_EFFORT`（仅 kimi provider 且
Thinking 开启时生效，会绕过模型声明的 `support_efforts` 直接下发，取值必须在
目标模型支持的档位内）。工单需要固定 effort 时优先用环境变量，不修改用户的
全局配置；Orca 的 Windows 终端是 PowerShell，bash 风格的 `VAR=x cmd` 前缀会
报错，正确写法：

```text
$env:KIMI_MODEL_THINKING_EFFORT='low'; kimi -m kimi-code/k3-256k
```

启动 worker 后确认状态栏 `thinking:` 生效值符合 work order。切换模型或 effort
会使既有上下文缓存失效，需切换时优先创建新 session，不要在长会话中反复切换。

Kimi worker 同样在 Orca 的 PowerShell 终端发送 lifecycle 命令。Coordinator 用
`(Get-Command orca -ErrorAction Stop).Source` 记录实际 Orca 可执行文件；若
worker 不能解析裸 `orca`，在 `[工具]` 中提供该完整路径并要求 lifecycle 命令
直接使用它，不让 worker 扫描安装目录。

默认启动即 `--yolo`（见上文命令），不再按父级无人值守条件逐次判断；Codex
的审批绕过参数不适用于 Kimi。不用 `--auto`：监督式 worker 需要保留向
coordinator 提问和升级的能力，`--yolo` 只自动批准工具调用、不抑制提问，
`--auto` 面向完全自治场景，不作为 worker 启动参数。用户明确要求保留逐次
审批时去掉 `--yolo`；此时 lifecycle 命令可能停在审批提示，按父级权限规则
裁决，不要当作注入失败重新派单。

### 派发后信号

按父级启动核验处理恢复动作。Kimi 不使用 Codex 的
`[Pasted Content ...]` 或 `UserPromptSubmit` 作为信号；只根据派发前 cursor
之后当前工单已作为新一轮对话出现，或新的推理、工具输出、lifecycle 消息确认
本轮已启动。正常提交或无法明确确认 `tui-idle` 时保持只读，不因缺少 Codex
专用标记而补回车。

长工单 `dispatch --inject` 由 Kimi TUI 直接提交，实测无 Codex 式粘贴停滞。
`terminal read` 可视区对会话内容的可见性不稳定：活跃期可能完整显示工单与
输出，也可能只剩空闲输入框，与“尚未开始”相似。可视区只作辅助证据，启动与
完成判断以派发前 cursor 增量和 orchestration 消息为准，恢复动作按父级启动
核验的条件执行。

默认审批配置下，非 `--yolo` 启动的 worker 执行 preamble 的
`orca orchestration send`（heartbeat / `worker_done`）可能停在审批提示
（实测形态为 `▶ Run this command?`），`check --wait` 表现为静默超时，与
注入未提交在消息层面相似。读取 worker 终端出现审批提示说明工单已提交、
worker 在等权限，不是启动核验失败；按父级权限规则裁决（取得用户授权后代为
批准，或升级用户），不要当作注入失败重新派单。
