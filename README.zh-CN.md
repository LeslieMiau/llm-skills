# llm-skills

[English](README.md)

面向 LLM 编码助手的可复用、可落地技能仓库。

这个仓库面向那些不满足于“提示词片段”的用户：它提供可复用指令、平台元数据、示例和配套文档，并同时兼容 Claude Code 与 Codex。

这不是一个大而全的 skill 收集器，而是一个经过筛选的 AI 生产力工作流仓库。这里只保留那些能在日常 AI coding 中真正带来杠杆的东西：先把重要任务想清楚、控制 token 消耗、把高价值且反复出现的模式沉淀成可复用流程。

## 先解决最痛的问题

- `roundtable-design-review`：重要任务别急着做。先让多模型 review 你的 first prompt、设计方案、重构方向，确保想清楚了再动手。
- `token-guard`：还在被 token 黑洞拖垮吗？还在因为 Claude 昂贵又稀少的 token 半路停工吗？用它先拦住高风险会话，别让上下文和预算悄悄流干。

## 为什么做这个仓库

- 不是“什么都收”的 skill 仓库，只收那些能真实提升日常 AI coding 效率的工作流。
- 一个仓库，多个 skill，一套稳定目录结构。
- 用同一份 `SKILL.md` 适配多个平台。
- 通过 `agents/openai.yaml` 提供可选的 Codex 元数据。
- 该常驻的保持轻量，该详细的只在需要时加载。
- 用仓库内示例和参考文档代替口口相传的隐性知识。

## 重点技能

### `roundtable-design-review`

面向多模型协作设计评审的结构化工作流。适合让一个模型产出方案、多个模型审查、由人来裁决，同时避免把设计过程文件混进业务代码库。

- Skill 定义：`skills/roundtable-design-review/SKILL.md`
- Codex 元数据：`skills/roundtable-design-review/agents/openai.yaml`
- 模板：`skills/roundtable-design-review/references/templates.md`
- 示例会话：`skills/roundtable-design-review/examples/sessions/`

工作流：

```text
用户提出想法 -> Author 起草 V1 -> Reviewers 评审 -> 人工裁决
  -> Author 修订 V2 -> Reviewers 给最终结论
  -> 全部通过 -> 完成 / 有阻塞 -> Author 修复 V3 -> 完成
```

### `token-guard`

只在高风险场景启用的 token 成本守门 skill。它专门拦截那些悄悄吞噬上下文和预算的模式，例如长会话膨胀、全仓扫描、工具循环、超大工具输出、重复背景、过重的 MCP 暴露，以及中途切模型、切 thinking、切工具策略。

- Skill 定义：`skills/token-guard/SKILL.md`
- Codex 元数据：`skills/token-guard/agents/openai.yaml`
- 配套全局预检：`CLAUDE.md`
- 安装指南：`docs/setup.zh-CN.md`

分层模型：

```text
CLAUDE.md 轻量预检
  -> 低/中风险：正常执行
  -> 高/极高风险或显式请求 TokenGuard：调用 token-guard
  -> token-guard 拦截、缩小范围，或给出粗粒度放行估算
```

示例提示：

```text
Use $token-guard to assess this request before doing it:
"Continue this long session, scan the whole repo, fix issues, run tests, and give me a full report."

Use $token-guard for an explicit check:
"Evaluate whether this task is too expensive before proceeding: review these 20 files and summarize everything."
```

## 快速开始

### Claude Code

1. 将目标 `skills/<skill-slug>/SKILL.md` 复制到 Claude Code 的 skills 目录。
2. 如果你安装 `token-guard`，也请把仓库根目录的 `CLAUDE.md` 作为轻量全局预检一起安装。
3. 示例和参考文档可以保留在仓库里，需要时再按需读取。

### Codex

1. 用 `skills/<skill-slug>/SKILL.md` 作为 skill 正文。
2. 用 `skills/<skill-slug>/agents/openai.yaml` 提供 Codex/OpenAI 的界面元数据。
3. 对于 `token-guard`，如果你也想要常驻的轻量预检层，请同时带上根目录 `CLAUDE.md`。

### 手动使用

1. 将 `skills/<skill-slug>/SKILL.md` 加载到目标模型。
2. 按需读取该 skill 下 `references/` 目录中的参考文件。
3. 如果该 skill 会生成流程文件，请严格按 skill 的说明创建工作目录。
4. `examples/` 是仓库内示例，不是实际工作目录，除非 skill 明确说明可以直接使用。

## 仓库结构

每个 skill 都放在 `skills/<skill-slug>/` 下，独立维护自己的指令、元数据、参考文档和示例。

```text
.
├── skills/
│   └── <skill-slug>/
│       ├── SKILL.md
│       ├── agents/
│       │   └── openai.yaml
│       ├── references/
│       └── examples/
├── docs/
├── CLAUDE.md
├── CONTRIBUTING.md
├── CONTRIBUTING.zh-CN.md
├── LICENSE
├── README.md
└── README.zh-CN.md
```

## 平台支持

| 平台 | 读取内容 | 说明 |
| --- | --- | --- |
| Claude Code | `skills/<skill-slug>/SKILL.md` | 共享 skill 正文；根目录 `CLAUDE.md` 是可选的全局指令 |
| Codex / OpenAI 兼容代理 | `skills/<skill-slug>/SKILL.md` + `skills/<skill-slug>/agents/openai.yaml` | `openai.yaml` 提供 UI 元数据 |

## 文档

- 安装说明：`docs/setup.zh-CN.md`
- English setup guide: `docs/setup.md`
- 贡献规范：`CONTRIBUTING.zh-CN.md`
- English contribution guide: `CONTRIBUTING.md`

说明：根目录 `CLAUDE.md` 有意保持精简且单语，因为它会直接进入模型上下文，应该尽量短。

## 贡献

欢迎贡献，但请保持多 skill 共用的仓库结构，并确保 skill 在多个平台上都可用。具体规则见 `CONTRIBUTING.zh-CN.md`。

## 许可证

[MIT](LICENSE)
