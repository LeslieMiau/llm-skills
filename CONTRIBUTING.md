# Contributing | 贡献指南

Contributions should make the repository more reusable, not more cluttered.  
贡献应当让仓库更可复用，而不是更杂乱。

This repo is organized around portable skills: each skill owns its instructions, optional platform metadata, optional references, and optional examples. Keep that contract intact.  
这个仓库围绕“可移植 skill”组织：每个 skill 独立维护自己的指令、可选平台元数据、可选参考文档和可选示例。请保持这套结构不被破坏。

## Repository Contract | 仓库约定

Add each skill under `skills/<skill-slug>/`.  
每个 skill 都应放在 `skills/<skill-slug>/` 下。

```text
skills/<skill-slug>/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── references/
└── examples/
```

Rules | 规则:

- `SKILL.md` is the canonical skill body.  
  `SKILL.md` 是唯一权威的 skill 正文。
- Do not add skill bodies, references, or sample workflow files to the repository root.  
  不要把 skill 正文、参考文档或示例流程文件放到仓库根目录。
- If a skill supports Codex or OpenAI-compatible interfaces, keep `agents/openai.yaml` in the same skill directory and keep it aligned with `SKILL.md`.  
  如果一个 skill 支持 Codex 或 OpenAI 兼容接口，请把 `agents/openai.yaml` 放在同一个 skill 目录，并保持与 `SKILL.md` 一致。
- Use lowercase kebab-case for the skill slug.  
  skill slug 使用小写 kebab-case。
- Add `references/` only for material that should be loaded on demand.  
  `references/` 只放那些需要按需加载的材料。
- Add `examples/` only for checked-in examples that help explain or validate the skill.  
  `examples/` 只放那些真正有助于说明或验证 skill 的仓库内示例。

## Creating a New Skill | 新建 Skill

1. Create `skills/<skill-slug>/SKILL.md` with YAML frontmatter.  
   创建带 YAML frontmatter 的 `skills/<skill-slug>/SKILL.md`。
2. Add `agents/openai.yaml` if the skill should surface cleanly in Codex/OpenAI interfaces.  
   如果这个 skill 需要在 Codex/OpenAI 界面中良好展示，就添加 `agents/openai.yaml`。
3. Add `references/` and `examples/` only when they carry real reusable value.  
   只有在 `references/` 和 `examples/` 确实具有复用价值时才添加。
4. Update `README.md` so the new skill is discoverable.  
   更新 `README.md`，让新 skill 可被发现。
5. If the skill relies on a root-level companion file such as `CLAUDE.md`, document that relationship in both the skill and the repo docs.  
   如果 skill 依赖根目录的配套文件，例如 `CLAUDE.md`，请在 skill 和仓库文档中都写清楚这种关系。

Minimal frontmatter example | 最小 frontmatter 示例:

```markdown
---
name: your-skill-name
description: What the skill does and when it should be used.
---
```

## Updating an Existing Skill | 更新已有 Skill

- Keep links and relative paths valid from the skill directory.  
  保证相对路径和链接从 skill 目录出发仍然有效。
- Update `agents/openai.yaml` whenever the skill's positioning, name, or invocation guidance changes.  
  只要 skill 的定位、命名或调用方式变了，就同步更新 `agents/openai.yaml`。
- Keep docs in sync when installation flow or usage expectations change.  
  安装流程或使用预期发生变化时，同步更新文档。
- Prefer tightening instructions over adding verbose explanation.  
  优先收紧说明，不要堆砌冗长解释。

## Pull Requests | 提交 PR

1. Make the skill change and the supporting documentation change in the same PR.  
   在同一个 PR 中同时提交 skill 变更和配套文档更新。
2. Explain which skills changed and whether metadata, examples, or setup docs changed.  
   说明改了哪些 skill，以及元数据、示例或安装文档是否也发生了变化。
3. Keep commits focused and reviewable.  
   保持提交聚焦、易于审查。
