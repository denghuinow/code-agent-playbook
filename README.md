# Code Agent Playbook

本仓库用于集中管理代码类 AI Agent 的使用规范、`AGENTS.md` 模板、可复用 skills 能力包以及工程实践，适用于 GitHub、GitLab、自建代码平台和内网研发/运维项目。

## 仓库定位

`code-agent-playbook` 面向 Claude Code、Codex、Cursor、OpenCode、Aider 等代码类 AI Agent，沉淀组织级规范、项目级模板和可复用能力包。

主要内容包括：

- `AGENTS.md`：仓库级默认 Agent 行为规范。
- `agents/`：不同技术栈或场景的 `AGENTS.md` 模板。
- `skills/`：可复用 skill 能力包，例如代码审查、Linux 排障、Docker 运维、vLLM 部署等。
- `templates/`：项目接入、评审、排障、变更说明等模板。
- `docs/`：命名规范、skill 编写规范、推广指南。

## 推荐使用方式

### 1. 新项目接入

从 `templates/AGENTS.project.md` 复制到目标项目根目录：

```bash
cp templates/AGENTS.project.md /path/to/project/AGENTS.md
```

然后根据项目技术栈、目录结构、构建命令、测试命令和安全约束做本地化调整。

### 2. 复用组织级规范

项目内的 `AGENTS.md` 可以引用本仓库中的公共规范，例如：

```markdown
本项目遵循 code-agent-playbook 中的基础规范，并补充以下项目特定约束。
```

### 3. 维护 skills

新增 skill 时建议放在：

```text
skills/<skill-name>/SKILL.md
```

每个 skill 应包含：

- 适用场景
- 输入要求
- 执行步骤
- 输出格式
- 风险和边界
- 示例

## 目录结构

```text
code-agent-playbook/
├── README.md
├── AGENTS.md
├── agents/
│   ├── AGENTS.base.md
│   ├── AGENTS.ops.md
│   └── AGENTS.java.md
├── skills/
│   ├── README.md
│   ├── code-review/
│   │   └── SKILL.md
│   ├── docker-ops/
│   │   └── SKILL.md
│   └── linux-troubleshooting/
│       └── SKILL.md
├── templates/
│   ├── AGENTS.project.md
│   ├── CODE_REVIEW.md
│   └── INCIDENT_REPORT.md
└── docs/
    ├── naming-convention.md
    ├── skill-spec.md
    └── rollout-guide.md
```

## 维护原则

- 优先沉淀通用规范，项目特有约束放到目标项目自己的 `AGENTS.md`。
- skill 应尽量小而专，避免一个 skill 覆盖过多场景。
- 涉及生产环境、密钥、数据库、删除操作、权限变更的内容必须明确安全边界。
- 模板应保持可复制、可落地，避免只写抽象原则。

## 版本建议

建议使用语义化版本或日期版本管理组织级规范变更，例如：

```text
v0.1.0  初始规范和基础 skills
v0.2.0  增加运维类 skills
v0.3.0  增加 Java/Spring 项目模板
```
