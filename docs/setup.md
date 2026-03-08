# Setup Guide | 安装指南

This repository supports two installation styles:  
这个仓库支持两种安装方式：

- Install an individual skill only.  
  只安装单个 skill。
- Install a skill plus lightweight global guidance when the workflow benefits from it.  
  当工作流确实需要时，同时安装 skill 和轻量全局指令。

## Install a Skill | 安装单个 Skill

For any skill:  
对于任意 skill：

1. Copy `skills/<skill-slug>/SKILL.md` into the target assistant's skills directory or skill registry.  
   将 `skills/<skill-slug>/SKILL.md` 复制到目标助手的 skills 目录或技能注册位置。
2. If the platform supports UI metadata, also copy `skills/<skill-slug>/agents/openai.yaml`.  
   如果目标平台支持 UI 元数据，也一并复制 `skills/<skill-slug>/agents/openai.yaml`。
3. Keep `references/` and `examples/` alongside the skill if you want the full packaged experience.  
   如果你想保留完整体验，可以同时带上 `references/` 和 `examples/`。

## Install `token-guard` | 安装 `token-guard`

`token-guard` is designed as a two-layer setup:  
`token-guard` 设计为双层结构：

- Global lightweight precheck in `CLAUDE.md`  
  `CLAUDE.md` 中的全局轻量预检
- Detailed high-risk escalation workflow in `skills/token-guard/SKILL.md`  
  `skills/token-guard/SKILL.md` 中的高风险升级工作流

### Claude Code

Recommended setup:  
推荐安装方式：

1. Install `skills/token-guard/SKILL.md` into your Claude Code skills directory.  
   把 `skills/token-guard/SKILL.md` 安装到 Claude Code 的 skills 目录。
2. Copy the repository root `CLAUDE.md` into your Claude Code global instructions.  
   把仓库根目录的 `CLAUDE.md` 复制到 Claude Code 的全局指令中。

Why this split:  
这样拆分的原因：

- `CLAUDE.md` stays short and always-on.  
  `CLAUDE.md` 保持短小并常驻生效。
- `token-guard` only loads when the task is actually risky or explicitly requested.  
  `token-guard` 只在任务真的高风险或被显式调用时加载。
- Normal tasks stay fast and low-friction.  
  普通任务依旧保持快速、低干扰。

### Codex

Recommended setup:  
推荐安装方式：

1. Use `skills/token-guard/SKILL.md` as the skill body.  
   用 `skills/token-guard/SKILL.md` 作为 skill 正文。
2. Use `skills/token-guard/agents/openai.yaml` for the interface metadata.  
   用 `skills/token-guard/agents/openai.yaml` 提供界面元数据。
3. Also carry over the repository root `CLAUDE.md` if you want the same lightweight precheck behavior before high-risk tasks.  
   如果你希望高风险任务前也具备同样的轻量预检行为，请同时带上仓库根目录的 `CLAUDE.md`。

## TokenGuard Usage Pattern | TokenGuard 使用方式

The intended flow is:  
推荐使用流程：

```text
new task
  -> lightweight precheck
  -> low/medium risk: do the task
  -> high/extreme risk: invoke token-guard
  -> token-guard intercepts or allows with a coarse estimate
```

Use explicit invocation when needed:  
需要时可以显式调用：

```text
Use $token-guard to assess this request before doing it:
"Continue this long session, scan the repo, run tools, and give me a complete report."
```

## Updating a Skill | 更新 Skill

When you update a skill:  
当你更新一个 skill：

1. Keep `SKILL.md` and `agents/openai.yaml` aligned.  
   保持 `SKILL.md` 与 `agents/openai.yaml` 一致。
2. Update repository docs if the positioning or installation flow changes.  
   如果定位或安装流程变了，也同步更新仓库文档。
3. If the skill depends on a root-level companion file like `CLAUDE.md`, document that relationship clearly in both places.  
   如果 skill 依赖类似 `CLAUDE.md` 这样的根目录配套文件，就在两个位置都把关系说明清楚。
