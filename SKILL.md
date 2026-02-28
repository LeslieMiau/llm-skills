---
name: roundtable-design-council
description: This skill should be used when the user wants to "design a system with multiple models", "run a design review", "get a technical proposal reviewed", "multi-model design collaboration", "Author-Reviewer design workflow", or mentions generating a technical design that benefits from multi-model cross-review. It orchestrates a structured Author-Reviewer workflow where one model drafts a proposal and others review it.
version: "6.2"
---

# Roundtable Design Council v6.2

Multi-model collaborative design workflow. One model (Author) drafts a technical proposal, other models (Reviewers) critique it, and the human arbitrates conflicts. Converges to a final design in at most 3 versions.

## Roles

- **Author** — Drafts proposals, asks clarifying questions, revises based on feedback. Only the Author writes `*-proposal.md` files.
- **Reviewer** — Reviews proposals across multiple dimensions, suggests improvements. Only Reviewers write `*-review-*.md` and `*-final-review-*.md` files.

## File Convention

All workflow artifacts live in `rounds/`. File names encode a sequence number, type, and source:

```
rounds/
  1-proposal.md              ← Author's V1 proposal
  2-review-{model}.md        ← Each Reviewer's feedback on V1
  3-proposal.md              ← Author's V2 proposal (revised)
  4-final-review-{model}.md  ← Each Reviewer's final verdict on V2
  5-proposal.md              ← V3 fix (only if any verdict has ⚠️)
```

`{model}` = lowercase model identifier (e.g., `claude`, `gpt`, `gemini`).

Rules:
- **Sequence prefix is mandatory.** It makes workflow state visible from a directory listing.
- **Each role writes only its own file type.** Author writes `*-proposal.md`. Reviewer writes `*-review-*.md` or `*-final-review-*.md`. Never cross-write.
- **Files are append-only.** Once written, never modified. To redo a step, the user deletes the file and re-runs.
- **New workflow**: To start a new design, clear or rename the `rounds/` directory first.
- **No file-system access**: If the model cannot read/write files directly, the user must provide the file directory listing and paste relevant file contents into the prompt. The model treats pasted content as having read the corresponding file, and outputs file content for the user to save manually.

## Bootstrap Protocol (CRITICAL)

**Every model MUST execute this protocol before taking any action:**

1. List files in `rounds/` directory.
2. If any `4-final-review-*.md` files exist, read each one and check whether the first line is `# Verdict: ✅ pass` or `# Verdict: ⚠️ remaining issues`.
3. Determine the current workflow state using the State Detection Table below. **Match from top to bottom; use the first matching row.**
4. Determine what action the user is requesting.
5. Verify the requested action matches the current state. If not, warn the user and suggest the correct next step.
6. Only then proceed.

### State Detection Table

Match top-to-bottom. First match wins.

| # | Condition | State | Valid next actions |
|---|---|---|---|
| 1 | `5-proposal.md` exists | **Done (V3 hard stop)** | None. Workflow complete. |
| 2 | Any `4-final-review-*.md` exists, AND `3-proposal.md` does not exist | Invalid state (final review without V2) | None automatically. Warn the user to remove stray final-review files or restore the missing `3-proposal.md`. |
| 3 | `3-proposal.md` exists, AND at least one `4-final-review-*.md` exists, AND the set of models in `2-review-*.md` exactly matches the set of models in `4-final-review-*.md`, AND all verdicts are `✅` | **Done (V2 approved)** | None. Workflow complete. |
| 4 | `3-proposal.md` exists, AND at least one `4-final-review-*.md` exists, AND the set of models in `2-review-*.md` exactly matches the set of models in `4-final-review-*.md`, AND any verdict is `⚠️` | V2 needs fix | User arbitrates issues → Author → write `5-proposal.md` |
| 5 | `3-proposal.md` exists, AND any `4-final-review-{model}.md` exists without a corresponding `2-review-{model}.md` | Invalid state (unmatched final reviews) | None automatically. Warn the user to remove unmatched final-review files or restore the missing Step 2 review files. |
| 6 | `3-proposal.md` exists, AND no `2-review-*.md` exist | Invalid state (V2 without prior reviews) | None automatically. Warn the user to restore missing Step 2 review files or restart the workflow. |
| 7 | `3-proposal.md` exists, AND one or more `4-final-review-{model}.md` are missing for the existing `2-review-{model}.md` | Final reviews pending | Only a Reviewer with an existing `2-review-{model}.md` and no `4-final-review-{model}.md` may write `4-final-review-{model}.md`. |
| 8 | One or more `2-review-*.md` exist | Reviews available | User confirms all reviews in → Author → write `3-proposal.md` |
| 9 | `1-proposal.md` exists | V1 drafted | Reviewer → write `2-review-{model}.md` |
| 10 | `rounds/` empty or missing | Not started | Author → write `1-proposal.md` |

### Enforcement Rules

- If the workflow is complete (rows 1–2), **refuse to generate new files**. Inform the user the design is finalized.
- If a Reviewer is asked to act but the state expects an Author action (or vice versa), **refuse and explain** which role should act next.
- If a file required for the current step is missing, **warn the user** before proceeding.
- If no row matches, treat the workspace as an invalid state. Do not write new files until the user resolves the inconsistency.

---

## Workflow

```
User idea
    │
    ▼
Step 1: Kickoff ── Author asks clarifying questions (if needed)
                    → Author writes 1-proposal.md
    │
    ▼
Step 2: Review ─── Each Reviewer reads 1-proposal.md
                    → writes 2-review-{model}.md
    │
    ▼
Human Decision ─── User arbitrates conflicts, confirms all reviews are in
    │
    ▼
Step 3: Revise ─── Author reads 1-proposal.md + all 2-review-*.md
                    → writes 3-proposal.md (with Changelog)
    │
    ▼
Step 4: Final Review ── Each Reviewer who participated in Step 2 reads 3-proposal.md
                         → writes 4-final-review-{model}.md
    │
    ├─ All ✅ → 3-proposal.md is final. Done.
    └─ Any ⚠️ → User arbitrates issues → Author fixes → writes 5-proposal.md → Done (hard stop).
```

## Interaction Rules

1. **User may inject ideas at any time** — Absorb additional information into the current step's output.
2. **Act autonomously first, ask only when truly uncertain** — Attempt to resolve ambiguity using context and professional judgment. Only escalate genuinely uncertain key decisions to the user.

---

## Step 1: Kickoff — Author Drafts V1

**Precondition**: `rounds/` is empty or doesn't exist.
**Output**: `rounds/1-proposal.md`

Process:
1. Assess whether the user's idea has critical ambiguities that would significantly affect design direction.
2. **If ambiguities exist**: output only the clarifying questions (3–5 max), then **STOP. Do not generate V1 in the same turn.** End your response with: *"Please provide your answers so I can proceed to generate V1."*
3. **If the requirement is clear enough**: proceed directly to generate V1.
4. Save the complete V1 to `rounds/1-proposal.md`.

V1 must begin with a **Project Brief**:

```markdown
## Project Brief
- **Original requirement**: (user's idea)
- **Key constraints**: (confirmed during Q&A, or inferred)
- **Excluded approaches**: (ruled out, with reasons)
- **Core decisions**: (key choices made, with rationale)
```

### Suggested Dimensions (non-mandatory, select as needed)

- Technology stack selection with rationale
- System architecture (components, data flow, deployment topology)
- Module decomposition
- Data model / API design
- Security and access control
- Performance and scalability
- Risks and open questions
- Implementation roadmap

---

## Step 2: Review — Reviewers Critique V1

**Precondition**: `rounds/1-proposal.md` exists; this Reviewer's `2-review-{model}.md` does not yet exist.
**Input**: Read `rounds/1-proposal.md`. If other `2-review-*.md` exist, read those too.
**Output**: `rounds/2-review-{model}.md`

Four mandatory review dimensions (additional findings welcome):

1. **Architecture soundness** — Over-engineered or under-designed? Clear component boundaries?
2. **Technical feasibility** — Mature technology choices? Simpler alternatives?
3. **Missed risks** — Security, performance, maintainability, operational cost gaps?
4. **Alternative approaches** — Better options for key decision points?

Mark disagreements with other Reviewers using `⚠️ conflict`.

Output format: see `references/templates.md` § Review Output Format.

---

## Human Decision (between Step 2 and Step 3)

User reviews all `2-review-*.md` files, arbitrates conflict points, and **confirms all expected reviews are in** before proceeding. Skip if no conflicts and user trusts Author's judgment.

---

## Human Decision (between Step 4 and V3, if any ⚠️)

If any Reviewer raised `⚠️ remaining issues`, the human arbitrates whether these issues are valid. The user can instruct the Author to ignore hallucinated or overly strict warnings using the `HUMAN_DECISIONS` field.

---

## Step 3: Revise — Author Generates V2

**Precondition**: `rounds/1-proposal.md` and at least one `2-review-*.md` exist; `3-proposal.md` does not exist.
**Input**: Read `rounds/1-proposal.md` + **all** `rounds/2-review-*.md` + user decisions (if any).
**Output**: `rounds/3-proposal.md` (complete V2, not a diff, with Changelog appended).

Before writing, list all `2-review-*.md` files and report how many reviews were found.

Evaluate each suggestion. Accept or reject with explicit rationale in the Changelog.

Output format: see `references/templates.md` § Changelog Format.

---

## Step 4: Final Review — Reviewers Approve V2

**Precondition**: `rounds/3-proposal.md` exists; this Reviewer's `2-review-{model}.md` exists; this Reviewer's `4-final-review-{model}.md` does not yet exist; `5-proposal.md` does not exist.
**Input**: Read `rounds/3-proposal.md` + own `rounds/2-review-{model}.md`.
**Output**: `rounds/4-final-review-{model}.md`

**The first line of the output must be the verdict heading** (see Final Review Output Format in `references/templates.md`):
- `# Verdict: ✅ pass`
- `# Verdict: ⚠️ remaining issues`

Remaining issues must be genuine blockers, not nice-to-haves.

### Termination

Once all Reviewers who participated in Step 2 have submitted final reviews (i.e., every `2-review-{model}.md` has a corresponding `4-final-review-{model}.md`):

- All verdicts `✅ pass` → **`3-proposal.md` is the final design. Workflow ends.**
- Any verdict `⚠️ remaining issues` → User arbitrates issues → Author reads all `4-final-review-*.md`, applies targeted fixes → **`5-proposal.md` → hard stop.**

---

## Additional Resources

- **`references/templates.md`** — Input templates for each step and output format specifications.
