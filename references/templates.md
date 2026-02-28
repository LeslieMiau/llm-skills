# Templates

Input templates for each workflow step and output format specifications.

All step outputs are saved to the `rounds/` directory with sequence-number prefixes. Subsequent steps read from these files directly — no manual copy-paste needed.

---

## Step 1: Kickoff — Author Input Template

```
STEP = 1 (kickoff)
ROLE = Author

IDEA:
(user's idea description)

USER_CONTEXT: (optional)
(supplementary constraints, preferences, existing tech stack, etc.)

INSTRUCTION:
Act as the proposal Author. Before starting, execute the Bootstrap Protocol:

1. List files in rounds/ directory.
2. Verify rounds/ is empty (or doesn't exist). If rounds/1-proposal.md
   already exists, STOP and inform the user that V1 already exists.
3. Assess whether the idea has critical ambiguities that would significantly
   affect the design direction.
4. IF AMBIGUITIES EXIST: output ONLY your clarifying questions (3–5 max),
   then STOP. Do NOT generate V1 in the same response.
   End your response with: "Please provide your answers so I can proceed to generate V1."
5. IF THE REQUIREMENT IS CLEAR ENOUGH: proceed directly to generate V1.
6. Begin V1 with a Project Brief containing:
   - Original requirement (user's idea)
   - Key constraints (confirmed during Q&A, or inferred)
   - Excluded approaches (ruled out, with reasons)
   - Core decisions (key choices made, with rationale)
7. Determine section structure based on idea complexity.
8. Save the complete V1 output to rounds/1-proposal.md.
```

---

## Step 2: Review — Reviewer Input Template

```
STEP = 2 (review)
ROLE = Reviewer
model = (your lowercase model identifier, e.g., claude, gpt, gemini)

INSTRUCTION:
Act as proposal Reviewer. Before starting, execute the Bootstrap Protocol:

1. List files in rounds/ directory.
2. Verify rounds/1-proposal.md exists. If not, STOP and inform the user
   that the Author must complete Step 1 first.
3. Verify rounds/2-review-{model}.md does NOT exist yet.
   If it does, STOP and inform the user this Reviewer has already submitted.
4. Read rounds/1-proposal.md — pay close attention to the Project Brief.
5. If other rounds/2-review-*.md files exist, read them as well.
6. Review across 4 mandatory dimensions:
   - Architecture soundness
   - Technical feasibility
   - Missed risks
   - Alternative approaches
7. Provide specific suggestions, each tagged with priority (high/medium/low).
8. Do not suggest approaches the user explicitly excluded in the Project Brief.
9. If another Reviewer's opinion conflicts with yours, mark it with
   ⚠️ conflict.
10. Use the Review Output Format defined below.
11. Save output to rounds/2-review-{model}.md.
```

---

## Human Decision (between Step 2 and Step 3)

No template needed. The user reviews all `2-review-*.md` files, arbitrates conflicts, confirms all expected Reviewers have submitted, and tells the Author to proceed.

---

## Step 3: Revise — Author Input Template

```
STEP = 3 (revise)
ROLE = Author

HUMAN_DECISIONS: (optional)
(user's conflict arbitration decisions and any supplementary requirements)

INSTRUCTION:
Act as the proposal Author. Before starting, execute the Bootstrap Protocol:

1. List files in rounds/ directory.
2. Verify rounds/1-proposal.md exists AND at least one rounds/2-review-*.md
   exists. If not, STOP and explain what's missing.
3. Verify rounds/3-proposal.md does NOT exist yet. If it does, STOP and
   inform the user that V2 has already been written.
4. List all rounds/2-review-*.md files. Acknowledge the number of review files
   found, and proceed immediately to generate V2.
5. Read rounds/1-proposal.md.
6. Read ALL rounds/2-review-*.md files. Do not skip any review file.
7. Evaluate each Reviewer suggestion individually.
8. Incorporate user decisions (if any) when accepting or rejecting suggestions.
9. Generate V2 as a complete proposal (not a diff).
10. Append a Changelog table at the end (see Changelog Format below).
11. Save the complete V2 output to rounds/3-proposal.md.
```

---

## Step 4: Final Review — Reviewer Input Template

```
STEP = 4 (final-review)
ROLE = Reviewer
model = (your lowercase model identifier, e.g., claude, gpt, gemini)

INSTRUCTION:
Act as proposal Reviewer for the final review. Before starting, execute the
Bootstrap Protocol:

1. List files in rounds/ directory.
2. If rounds/5-proposal.md exists, STOP — the workflow is already complete
   (V3 hard stop).
3. Verify rounds/3-proposal.md exists. If not, STOP and inform the user
   that the Author must complete Step 3 first.
4. Verify rounds/2-review-{model}.md exists. If not, STOP and inform
   the user this reviewer did not participate in Step 2 and cannot submit
   a final review.
5. Verify rounds/4-final-review-{model}.md does NOT exist yet.
   If it does, STOP and inform the user this Reviewer has already submitted.
6. Read rounds/3-proposal.md (including the Changelog at the end).
7. Read your own prior review from rounds/2-review-{model}.md.
8. Verify prior suggestions were handled reasonably
   (accepted, or rejected with sufficient rationale).
9. Check whether V2 introduced new issues.
10. Output using the Final Review Output Format below. The FIRST LINE of
   the file must be the verdict heading — either:
   # Verdict: ✅ pass
   or:
   # Verdict: ⚠️ remaining issues
11. Remaining issues must be genuine blockers, not nice-to-have improvements.
12. Save output to rounds/4-final-review-{model}.md.
```

---

## V3 Fix — Author Input Template (only if needed)

```
STEP = V3 fix
ROLE = Author

HUMAN_DECISIONS: (optional)
(user's arbitration regarding remaining issues, e.g., instructing you to ignore certain invalid warnings)

INSTRUCTION:
Act as the proposal Author for targeted V3 fixes. Before starting, execute
the Bootstrap Protocol:

1. List files in rounds/ directory.
2. Verify rounds/3-proposal.md exists AND at least one
   rounds/4-final-review-*.md with ⚠️ verdict exists.
3. Verify rounds/5-proposal.md does NOT exist yet. If it does, STOP —
   V3 is already sealed, workflow is complete.
4. Read rounds/3-proposal.md.
5. Read ALL rounds/4-final-review-*.md files.
6. Apply targeted fixes ONLY for the ⚠️ issues raised. Incorporate any HUMAN_DECISIONS to override or ignore specific issues.
7. Generate V3 as a complete proposal (not a diff).
8. Place the Changelog (V2 → V3) table above the existing V1 → V2 Changelog
   (newest first; see Changelog Format below for V3 example).
9. Save the complete V3 output to rounds/5-proposal.md.
10. This is a HARD STOP. No further iterations after V3.
```

---

## Review Output Format

```markdown
# Review — {Model Name} (Reviewer)

## Review Summary
One-sentence overall assessment of the proposal.

## Dimension Reviews
### Architecture Soundness
...
### Technical Feasibility
...
### Missed Risks
...
### Alternative Approaches
...

## Specific Suggestions
### Suggestion 1: [title]
- **Priority**: high / medium / low
- **Issue**: ...
- **Suggestion**: ...
- **Rationale**: ...

### Suggestion 2: [title]
...

## ⚠️ Conflicts with Other Reviewers (if applicable)
(mark specific points of disagreement with other Reviewers)
```

---

## Final Review Output Format

The first line MUST be the verdict heading. This enables automated state detection.

```markdown
# Verdict: ✅ pass

## Review of Changes
(Brief assessment of how prior suggestions were handled)
```

Or, if there are remaining issues:

```markdown
# Verdict: ⚠️ remaining issues

## Findings
### 1. [Priority] Issue title
- **Location**: (section/line reference)
- **Issue**: ...
- **Suggestion**: ...

## Handled Well
(Brief acknowledgment of suggestions that were properly addressed)
```

---

## Changelog Format

Append to the end of V2:

```markdown
## Changelog (V1 → V2)
| # | Reviewer Suggestion | Disposition | Rationale |
|---|---|---|---|
| 1 | Claude: Use PostgreSQL instead of MongoDB | ✅ accepted | Relational data better fits transactional workload |
| 2 | GPT: Add message queue | ❌ rejected | Current scale does not require async decoupling |
| 3 | User decision: Deploy with Docker | ✅ accepted | Explicit user requirement |
```

If V3 is needed, place the V3 changelog **above** the V1 → V2 changelog (newest first):

```markdown
## Changelog (V2 → V3)
| # | Final Review Issue | Disposition | Change Location |
|---|---|---|---|
| F1 | GPT: Settlement cycle incorrectly modeled as sell restriction | ✅ fixed | §4 MarketRules |
| F2 | GPT: LIMIT order unsupported by Bar schema | ✅ fixed | §2.2, §6.1 |

---

## Changelog (V1 → V2)
(preserved from V2, unchanged)
```
