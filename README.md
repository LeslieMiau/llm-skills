# llm-skills

[简体中文](README.zh-CN.md)

Reusable, production-minded skills for LLM coding assistants.

This repository is for people who want assistant workflows that are sharper than generic prompt snippets: reusable instructions, platform metadata, examples, and companion docs that work across both Claude Code and Codex.

This is not a giant skill catalog. It is a curated repository of AI productivity workflows that create real leverage in day-to-day AI coding work: planning important changes before coding, controlling token burn, and turning repeated high-value patterns into reusable operating procedures.

## Solve The Pain First

- `roundtable-design-review`: Use cross-model review for first prompts, design proposals, and risky refactors so you think through the important work before you start coding.
- `token-guard`: Stop losing momentum to token black holes, bloated sessions, and expensive Claude limits. Use it to catch high-risk tasks before they burn context and stall the job.

## Why This Repo

- Not a kitchen-sink skill collection. Only keep workflows that materially improve real AI coding work.
- One repository, multiple skills, one predictable layout.
- Shared `SKILL.md` instructions across platforms.
- Optional Codex metadata via `agents/openai.yaml`.
- Lightweight global guidance where it helps, detailed workflows only when needed.
- Checked-in examples and references instead of hidden tribal knowledge.

## Featured Skills

### `roundtable-design-review`

Structured multi-model design workflow for proposal, critique, revision, and final review. Use it when one model should author, peers should review, and the human should arbitrate without mixing design artifacts into the main codebase.

- Skill definition: `skills/roundtable-design-review/SKILL.md`
- Codex metadata: `skills/roundtable-design-review/agents/openai.yaml`
- Templates: `skills/roundtable-design-review/references/templates.md`
- Sample sessions: `skills/roundtable-design-review/examples/sessions/`

Workflow:

```text
User idea -> Author drafts V1 -> Reviewers critique -> Human arbitrates
  -> Author revises V2 -> Reviewers give final verdict
  -> All approved -> Done / Any blocker -> Author fixes V3 -> Done
```

### `token-guard`

Escalation-only token budget guardrail for expensive sessions. Use it to catch the patterns that quietly burn context and budget: long-session bloat, repo-wide scans, tool loops, oversized tool output, repeated background, heavy MCP exposure, or mid-session switching of model, thinking mode, or tool strategy.

- Skill definition: `skills/token-guard/SKILL.md`
- Codex metadata: `skills/token-guard/agents/openai.yaml`
- Companion global precheck: `CLAUDE.md`
- Setup guide: `docs/setup.md`

Layering model:

```text
CLAUDE.md lightweight precheck
  -> low/medium risk: proceed normally
  -> high/extreme risk or explicit TokenGuard request: invoke token-guard
  -> token-guard intercepts, narrows scope, or allows with a coarse estimate
```

Example prompts:

```text
Use $token-guard to assess this request before doing it:
"Continue this long session, scan the whole repo, fix issues, run tests, and give me a full report."

Use $token-guard for an explicit check:
"Evaluate whether this task is too expensive before proceeding: review these 20 files and summarize everything."
```

## Quick Start

### Claude Code

1. Copy the desired `skills/<skill-slug>/SKILL.md` into your Claude Code skills directory.
2. If you install `token-guard`, also install the repository root `CLAUDE.md` as your lightweight global precheck.
3. Keep examples and references in the repo; load them only when needed.

### Codex

1. Use `skills/<skill-slug>/SKILL.md` as the skill body.
2. Use `skills/<skill-slug>/agents/openai.yaml` for Codex/OpenAI interface metadata.
3. For `token-guard`, also carry over the root `CLAUDE.md` if you want the always-on lightweight precheck layer.

### Manual Use

1. Load `skills/<skill-slug>/SKILL.md` into the target model.
2. Read any referenced files from that skill's `references/` directory on demand.
3. If the skill generates workflow artifacts, create a working directory exactly as described by the skill.
4. Treat `examples/` as checked-in samples, not as the live working directory unless the skill explicitly says otherwise.

## Repository Layout

Each skill lives under `skills/<skill-slug>/` and owns its own instructions, metadata, references, and examples.

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

## Platform Support

| Platform | Consumes | Notes |
| --- | --- | --- |
| Claude Code | `skills/<skill-slug>/SKILL.md` | Shared skill body; root `CLAUDE.md` is optional global guidance |
| Codex / OpenAI-compatible agents | `skills/<skill-slug>/SKILL.md` + `skills/<skill-slug>/agents/openai.yaml` | `openai.yaml` provides UI-facing metadata |

## Docs

- Setup and installation: `docs/setup.md`
- Chinese setup guide: `docs/setup.zh-CN.md`
- Contribution rules: `CONTRIBUTING.md`
- Chinese contribution guide: `CONTRIBUTING.zh-CN.md`

Note: the root `CLAUDE.md` remains concise and single-language on purpose, because it is loaded into model context and should stay as small as possible.

## Contributing

Contributions should preserve the shared multi-skill layout and keep skills usable across platforms. See `CONTRIBUTING.md` for the directory contract and contribution workflow.

## License

[MIT](LICENSE)
