# llm-skills

Reusable, production-minded skills for LLM coding assistants.  
面向 LLM 编码助手的可复用、可落地技能仓库。

This repository is for people who want assistant workflows that are sharper than generic prompt snippets: reusable instructions, platform metadata, examples, and companion docs that work across both Claude Code and Codex.  
这个仓库面向那些不满足于“提示词片段”的用户：它提供可复用指令、平台元数据、示例和配套文档，并同时兼容 Claude Code 与 Codex。

## Why This Repo | 为什么做这个仓库

- One repository, multiple skills, one predictable layout.  
  一个仓库，多个 skill，一套稳定目录结构。
- Shared `SKILL.md` instructions across platforms.  
  用同一份 `SKILL.md` 适配多个平台。
- Optional Codex metadata via `agents/openai.yaml`.  
  通过 `agents/openai.yaml` 提供可选的 Codex 元数据。
- Lightweight global guidance where it helps, detailed workflows only when needed.  
  该常驻的保持轻量，该详细的只在需要时加载。
- Checked-in examples and references instead of hidden tribal knowledge.  
  用仓库内示例和参考文档代替口口相传的隐性知识。

## Featured Skills | 重点技能

### `roundtable-design-review`

Structured multi-model design workflow for proposal, critique, revision, and final review. Use it when one model should author, peers should review, and the human should arbitrate without mixing design artifacts into the main codebase.  
面向多模型协作设计评审的结构化工作流。适合让一个模型产出方案、多个模型审查、由人来裁决，同时避免把设计过程文件混进业务代码库。

- Skill definition: `skills/roundtable-design-review/SKILL.md`
- Codex metadata: `skills/roundtable-design-review/agents/openai.yaml`
- Templates: `skills/roundtable-design-review/references/templates.md`
- Sample sessions: `skills/roundtable-design-review/examples/sessions/`

Workflow | 工作流:

```text
User idea -> Author drafts V1 -> Reviewers critique -> Human arbitrates
  -> Author revises V2 -> Reviewers give final verdict
  -> All approved -> Done / Any blocker -> Author fixes V3 -> Done
```

### `token-guard`

Escalation-only token budget guardrail for expensive sessions. Use it to catch the patterns that quietly burn context and budget: long-session bloat, repo-wide scans, tool loops, oversized tool output, repeated background, heavy MCP exposure, or mid-session switching of model, thinking mode, or tool strategy.  
只在高风险场景启用的 token 成本守门 skill。它专门拦截那些悄悄吞噬上下文和预算的模式，例如长会话膨胀、全仓扫描、工具循环、超大工具输出、重复背景、过重的 MCP 暴露，以及中途切模型、切 thinking、切工具策略。

- Skill definition: `skills/token-guard/SKILL.md`
- Codex metadata: `skills/token-guard/agents/openai.yaml`
- Companion global precheck: `CLAUDE.md`
- Setup guide: `docs/setup.md`

Layering model | 分层模型:

```text
CLAUDE.md lightweight precheck
  -> low/medium risk: proceed normally
  -> high/extreme risk or explicit TokenGuard request: invoke token-guard
  -> token-guard intercepts, narrows scope, or allows with a coarse estimate
```

Example prompts | 示例提示:

```text
Use $token-guard to assess this request before doing it:
"Continue this long session, scan the whole repo, fix issues, run tests, and give me a full report."

Use $token-guard for an explicit check:
"Evaluate whether this task is too expensive before proceeding: review these 20 files and summarize everything."
```

## Quick Start | 快速开始

### Claude Code

1. Copy the desired `skills/<skill-slug>/SKILL.md` into your Claude Code skills directory.  
   将目标 `skills/<skill-slug>/SKILL.md` 复制到 Claude Code 的 skills 目录。
2. If you install `token-guard`, also install the repository root `CLAUDE.md` as your lightweight global precheck.  
   如果你安装 `token-guard`，也请把仓库根目录的 `CLAUDE.md` 作为轻量全局预检一起安装。
3. Keep examples and references in the repo; load them only when needed.  
   示例和参考文档可以保留在仓库里，需要时再按需读取。

### Codex

1. Use `skills/<skill-slug>/SKILL.md` as the skill body.  
   用 `skills/<skill-slug>/SKILL.md` 作为 skill 正文。
2. Use `skills/<skill-slug>/agents/openai.yaml` for Codex/OpenAI interface metadata.  
   用 `skills/<skill-slug>/agents/openai.yaml` 提供 Codex/OpenAI 的界面元数据。
3. For `token-guard`, also carry over the root `CLAUDE.md` if you want the always-on lightweight precheck layer.  
   对于 `token-guard`，如果你也想要常驻的轻量预检层，请同时带上根目录 `CLAUDE.md`。

### Manual Use | 手动使用

1. Load `skills/<skill-slug>/SKILL.md` into the target model.  
   将 `skills/<skill-slug>/SKILL.md` 加载到目标模型。
2. Read any referenced files from that skill's `references/` directory on demand.  
   按需读取该 skill 下 `references/` 目录中的参考文件。
3. If the skill generates workflow artifacts, create a working directory exactly as described by the skill.  
   如果该 skill 会生成流程文件，请严格按 skill 的说明创建工作目录。
4. Treat `examples/` as checked-in samples, not as the live working directory unless the skill explicitly says otherwise.  
   `examples/` 是仓库内示例，不是实际工作目录，除非 skill 明确说明可以直接使用。

## Repository Layout | 仓库结构

Each skill lives under `skills/<skill-slug>/` and owns its own instructions, metadata, references, and examples.  
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
├── LICENSE
└── README.md
```

## Platform Support | 平台支持

| Platform | Consumes | Notes |
| --- | --- | --- |
| Claude Code | `skills/<skill-slug>/SKILL.md` | Shared skill body; root `CLAUDE.md` is optional global guidance |
| Codex / OpenAI-compatible agents | `skills/<skill-slug>/SKILL.md` + `skills/<skill-slug>/agents/openai.yaml` | `openai.yaml` provides UI-facing metadata |

## Docs | 文档

- Setup and installation / 安装说明: `docs/setup.md`
- Contribution rules / 贡献规范: `CONTRIBUTING.md`

Note: the root `CLAUDE.md` remains concise and single-language on purpose, because it is loaded into model context and should stay as small as possible.  
说明：根目录 `CLAUDE.md` 有意保持精简且单语，因为它会直接进入模型上下文，应该尽量短。

## Contributing | 贡献

Contributions should preserve the shared multi-skill layout and keep skills usable across platforms. See `CONTRIBUTING.md` for the directory contract and contribution workflow.  
欢迎贡献，但请保持多 skill 共用的仓库结构，并确保 skill 在多个平台上都可用。具体规则见 `CONTRIBUTING.md`。

## License | 许可证

[MIT](LICENSE)
