# llm-skills

Custom skills for LLM-powered coding assistants. Currently includes **Roundtable Design Council** — a structured multi-model collaborative design workflow.

## Overview

This repository hosts reusable skill definitions that can be loaded into LLM coding assistants (Claude Code, OpenAI Codex, etc.) to extend their capabilities with specialized workflows.

### Roundtable Design Council v6.2

A multi-model Author-Reviewer workflow for technical design. One model (Author) drafts a technical proposal, other models (Reviewers) critique it, and the human arbitrates conflicts. The process converges to a final design in at most 3 versions.

**Workflow:**

```
User idea → Author drafts V1 → Reviewers critique → Human arbitrates
  → Author revises V2 → Reviewers give final verdict
  → All ✅ → Done  /  Any ⚠️ → Author fixes V3 → Done (hard stop)
```

**Roles:**
- **Author** — Drafts proposals, revises based on feedback
- **Reviewer** — Reviews across architecture, feasibility, risks, and alternatives

## Repository Structure

```
.
├── SKILL.md                  # Skill definition (Roundtable Design Council)
├── agents/
│   └── openai.yaml           # OpenAI-compatible agent config
├── references/
│   └── templates.md          # Input templates and output format specs
├── rounds/                   # Workflow artifacts (proposals & reviews)
│   ├── 1-proposal.md         # Author's V1 proposal
│   ├── 2-review-*.md         # Reviewer feedback on V1
│   ├── 3-proposal.md         # Author's V2 proposal (revised)
│   ├── 4-final-review-*.md   # Reviewer final verdicts on V2
│   └── 5-proposal.md         # V3 fix (only if any verdict has ⚠️)
├── LICENSE                   # MIT License
└── README.md
```

## Usage

### With Claude Code

Place `SKILL.md` in the Claude Code skill configuration directory. The skill will be automatically available when the user requests a multi-model design review.

### With OpenAI-compatible Agents

Use the `agents/openai.yaml` as the agent configuration. Paste the content of `SKILL.md` as the system prompt.

### Manual (Any LLM)

1. Copy the content of `SKILL.md` into the model's system prompt or context.
2. Use the input templates from `references/templates.md` to drive each step.
3. Save outputs to the `rounds/` directory following the naming convention.

### Starting a New Design

1. Clear or rename the `rounds/` directory.
2. Provide your idea to the Author model using the Step 1 template.
3. Pass the V1 proposal to each Reviewer model using the Step 2 template.
4. Arbitrate any conflicts, then have the Author revise using the Step 3 template.
5. Have each Reviewer submit a final verdict using the Step 4 template.
6. If all verdicts are `✅ pass`, V2 is final. If any `⚠️`, proceed to V3 fix.

## File Naming Convention

All workflow artifacts in `rounds/` use sequence-number prefixes:

| File Pattern | Description |
|---|---|
| `1-proposal.md` | Author's V1 draft |
| `2-review-{model}.md` | Reviewer feedback (e.g., `2-review-claude.md`) |
| `3-proposal.md` | Author's V2 revision |
| `4-final-review-{model}.md` | Final verdict per reviewer |
| `5-proposal.md` | V3 targeted fix (if needed) |

## License

[MIT](LICENSE)
