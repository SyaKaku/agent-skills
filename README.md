# agent-skills

Reusable AI agent skills for daily development workflows.

这是一个用于维护日常 AI 编程协作能力的 skill 集合。每个 skill 都放在
`skills/` 下的独立目录中，并包含自己的 `SKILL.md` 与按需加载的资源。

本仓库遵循 [Agent Skills 开放标准](https://agentskills.io/home)。Claude Code、
Codex、Qoder、Kimi Code、iFlow、CodeBuddy、Cursor 等 40+ 款支持该标准的 Agent
均可安装本仓库中的 skills。

## 来源与致谢

本仓库的日常工程 skill 组织方式与部分工作流约定，基于并参考
[Matt Pocock 的 Engineering Skills](https://github.com/mattpocock/skills/tree/main/skills/engineering)。
`orca-dispatch` 是结合 Orca coordinator/worker 生命周期形成的独立扩展，会配合
上游的 `grill-with-docs`、`to-spec`、`to-tickets`、`implement`、`code-review` 等
skills 使用；它不是上游仓库的官方组成部分。

The organization and workflow conventions in this repository are based on and
inspired by Matt Pocock's Engineering Skills. `orca-dispatch` is an independent
Orca-specific extension and is not an official upstream skill.

## Skills

| Skill | 用途 | 运行环境 |
|---|---|---|
| [`fable-explain`](skills/fable-explain/) | 用精炼寓言解释抽象概念，并给出定义、元素映射和理解/迁移检验 | Claude Code、Codex、Kimi Code 等 Agent Skills 兼容运行时 |
| [`orca-dispatch`](skills/orca-dispatch/) | 通过 Orca 把想法变成可追踪的多 worker 流水线：从规格/拆票到派单、实现与对抗式验收，并监督、复核和回收结果 | Orca CLI，以及 Codex CLI 或 Kimi CLI |

各 skill 的跨运行时入口均为 `SKILL.md`；`agents/openai.yaml` 仅提供可选的
OpenAI UI metadata，不是 Claude Code 或 Kimi Code 的运行依赖。

## 使用

每个 `skills/<name>/` 目录都是一个自包含 skill。推荐使用
[`skills` CLI](https://skills.sh/docs/cli) 管理，无需全局安装 CLI。

### 安装

安装仓库中的 skill：

```shell
npx skills add syakaku/agent-skills
```

也可以只安装指定 skill：

```shell
npx skills add syakaku/agent-skills --skill orca-dispatch
```

默认安装到当前项目；如需在所有项目中使用，添加 `--global`（或 `-g`）：

```shell
npx skills add syakaku/agent-skills --skill orca-dispatch --global
```

### 更新

更新已安装的全部 skill：

```shell
npx skills update
```

也可以按名称更新，或仅更新全局安装的 skill：

```shell
npx skills update orca-dispatch
npx skills update --global
```

### 卸载

卸载当前项目中的 skill：

```shell
npx skills remove orca-dispatch
```

如果安装在全局范围，卸载时同样需要添加 `--global`：

```shell
npx skills remove --global orca-dispatch
```

使用前请先阅读目标 skill 的依赖与安全说明，确认本机具备依赖的 CLI、账号权限
及隔离能力。

## 安全

- 仓库不应包含密钥、Token、密码、真实客户数据或本机私有路径。
- 涉及无人值守、跳过审批或关闭沙箱的参数必须由用户或仓库策略明确授权。
- Skill 不覆盖目标仓库的安全规则；提交、推送、部署和破坏性操作仍需遵守目标
  仓库及用户给出的授权边界。

## 目录结构

```text
agent-skills/
├── .gitignore
├── AGENTS.md
├── README.md
├── LICENSE
└── skills/
    ├── fable-explain/
    │   ├── SKILL.md
    │   └── agents/
    │       └── openai.yaml
    └── orca-dispatch/
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        └── references/
            ├── codex-worker.md
            └── kimi-worker.md
```

## License

[MIT](LICENSE)
