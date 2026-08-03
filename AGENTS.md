# AGENTS.md

## 仓库定位

本仓库维护可复用的 AI agent skills，不是应用程序。每项改动都应提升 skill 的
触发准确性、执行确定性或上下文效率；避免加入面向人的冗长教程和无关辅助文件。

默认使用中文编写说明，命令、模型 ID、参数、文件名和协议字段保留英文。

## 目录约定

- 每个 `skills/<name>/` 是自包含 skill，目录名必须与 frontmatter 的 `name` 一致。
- `SKILL.md` 只承载触发后的核心流程和必须始终可见的规则。
- `references/` 保存按条件加载的模型、供应商或场景差异；保持为
  `SKILL.md` 的一层直接引用，不做深层嵌套。
- `agents/openai.yaml` 保存 UI metadata；修改 skill 定位或默认用法时同步更新。
- 不在 skill 目录内增加 `README.md`、安装指南、变更日志或快速参考等辅助文档。

## Skill 编写规则

- frontmatter 仅使用 `name` 和 `description`。`description` 必须同时说明能力、
  正向触发条件和容易误触发的排除条件。
- 假定 agent 已具备通用编程能力，只记录本仓库特有、无法可靠推断的流程和约束。
- 使用祈使式规则，明确优先级、失败路径和可观察的完成条件；不要用无法映射到具体
  操作的抽象动词。
- 公共规则只放一处。父级 `SKILL.md` 管通用生命周期、安全和验收；worker
  reference 只保留模型路由、启动方式及 CLI 特有差异。
- 动态事实不得凭印象更新。CLI 参数以本机 `--help` 或动态指南为准；模型名称、
  套餐和能力以官方文档为准，并保留可用性检查。
- 保持安全边界：skill 不得自行扩大 commit、push、部署、无人值守、密钥、数据或
  破坏性操作的授权范围。

## `orca-dispatch` 约束

- 仅在用户明确要求监督、等待、回收结果、协调 DAG 或显式调用
  `$orca-dispatch` 时进入监督式流程；仅转交所有权时路由到 `orca-cli` handoff。
- 用户明确要求派单并回收结果时，即使任务简单也不得改由 coordinator 本地完成。
- 修改 Orca 命令或生命周期前，先读取 `orca skills get orca-cli` 和
  `orca skills get orchestration`；动态指南优先于仓库中的静态示例。
- 派单前缺少选择时直接询问；worker 的 `ask` 生成 `question` 消息并用消息 ID
  `reply`；`decision_gate` 只按动态指南处理 legacy/gate 场景，只有已有 DAG 的
  coordinator 阻塞决策使用 gate 命令。
- 保留 work order、provenance、独立复验、terminal 关闭和 worktree 回收规则；
  这些是高风险流程的必要护栏，不为缩短篇幅而删除。

## 验证

修改后执行最小充分验证：

1. 使用已安装 `$skill-creator` 的 `quick_validate.py` 校验改动过的 skill；Windows
   中文环境使用 `python -X utf8`。
2. 检查 `agents/openai.yaml` 与 `SKILL.md` 一致，且 `default_prompt` 显式包含
   `$<skill-name>`。
3. 运行 `git diff --check`，检查 Markdown 链接、表格、代码块和条件引用。
4. 修改 CLI 或模型规则时，额外验证对应 `--help`、动态指南或官方文档。

若无法运行某项验证，报告原因、替代检查和剩余风险，不得以跳过规则或放宽断言
换取表面通过。
