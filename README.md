# agent-skills

Reusable AI agent skills for daily development workflows.

这是一个用于维护日常 AI 编程协作能力的 skill 集合。每个 skill 都放在
`skills/` 下的独立目录中，并包含自己的 `SKILL.md` 与按需加载的资源。

## 来源与致谢

本仓库的日常工程 skill 组织方式与部分工作流约定，基于并参考
[Matt Pocock 的 Engineering Skills](https://github.com/mattpocock/skills/tree/main/skills/engineering)。
`orca-dispatch` 是结合 Orca coordinator/worker 生命周期形成的独立扩展，会在
可用时配合其中的 `implement`、`code-review` 等 skills 使用；它不是上游仓库的
官方组成部分。

The organization and workflow conventions in this repository are based on and
inspired by Matt Pocock's Engineering Skills. `orca-dispatch` is an independent
Orca-specific extension and is not an official upstream skill.

## Skills

| Skill | 用途 | 运行环境 |
|---|---|---|
| [`orca-dispatch`](skills/orca-dispatch/) | 通过 Orca 创建工作单、派发 Codex 或 Kimi worker，并监督、复核和回收结果 | Orca CLI，以及 Codex CLI 或 Kimi CLI |

## 使用

每个 `skills/<name>/` 目录都是一个自包含 skill。将所需目录复制或链接到目标
agent 运行时的 skills 目录；具体安装位置和发现机制以该运行时的文档为准。

使用前请先阅读目标 skill 的 `compatibility` 和安全说明，确认本机具备依赖的
CLI、账号权限及隔离能力。

## 安全

- 仓库不应包含密钥、Token、密码、真实客户数据或本机私有路径。
- 涉及无人值守、跳过审批或关闭沙箱的参数必须由用户或仓库策略明确授权。
- Skill 不覆盖目标仓库的安全规则；提交、推送、部署和破坏性操作仍需遵守目标
  仓库及用户给出的授权边界。

## 目录结构

```text
agent-skills/
├── README.md
├── LICENSE
└── skills/
    └── orca-dispatch/
        ├── SKILL.md
        └── references/
            ├── codex-worker.md
            └── kimi-worker.md
```

## License

[MIT](LICENSE)
