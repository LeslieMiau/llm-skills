# Contributing

Contributions are welcome! Here's how you can help.

## Adding a New Skill

1. Create a `SKILL-NAME.md` file in the repository root following the frontmatter format:

```markdown
---
name: your-skill-name
description: When and how this skill should be used.
version: "1.0"
---

# Skill Title

(Skill instructions here)
```

2. If the skill uses templates or reference files, add them under `references/`.
3. If the skill supports specific agent platforms, add configs under `agents/`.
4. Update `README.md` to include your new skill in the overview.

## Modifying an Existing Skill

- Bump the version number in the frontmatter.
- Document changes in the pull request description.

## Workflow Artifacts

The `rounds/` directory contains example workflow outputs. When submitting changes to the skill definition, consider including example rounds that demonstrate the updated workflow.

## Pull Requests

1. Fork the repository and create a feature branch.
2. Make your changes.
3. Submit a pull request with a clear description of what changed and why.
