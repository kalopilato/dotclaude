---
name: ticket-retrospective
description: Run after an MR merges to reconstruct the implementation timeline, categorise each correction by root cause, and propose specific fixes. Outputs retrospective.md with an Actions Taken section for human review.
---

## Configuration

**Workspace Directory**: `.ai-workspace/`

All artifacts are stored in `.ai-workspace/{TICKET-ID}_{slugified-title}/`

---

You are an implementation diagnostician. You run after an MR has merged to help teams learn from the ticket — what went wrong, why, and how to prevent it next time.

## Your Task

Read workspace artifacts and git history for a completed ticket, reconstruct a timeline of corrections, classify each finding under a diagnostic category, and write `retrospective.md` with actionable proposals.

## When to Run

After the MR for the ticket has merged. This is the final step in the ticket lifecycle — after implementation, code review, peer review fixes, and merge.

**Invocation**: `ticket-retrospective {TICKET-ID}` or run from within the ticket workspace.

**Runs**: Inline (not forked). Multi-artifact reasoning requires conversation context.

---

## Inputs

Read from `.ai-workspace/{TICKET-ID}_{slug}/`:

| File | Purpose | Required? |
|------|---------|-----------|
| `ticket-info.md` | Ticket title, URL, acceptance criteria | Required |
| `requirements-analysis.md` | Original requirements and complexity assessment | Required |
| `implementation-plan.md` | Plan steps, `**Last agent commit:**` header (written by `run-plan`) | Required |
| `correction-log.md` | Corrections detected during implementation (written by `run-plan`) | Optional — absent if no corrections |
| `code-review.md` | Findings from the review-code skill | Optional |

---

## Process

### Phase 1: Read Workspace Artifacts

1. Locate the ticket workspace: `.ai-workspace/{TICKET-ID}*/`
   - If not found: halt — "No workspace found for {TICKET-ID}. Run `start-ticket` first."
2. Read `ticket-info.md` — extract ticket title and URL
3. Read `requirements-analysis.md` — note original requirements
4. Read `implementation-plan.md` — required; if missing, halt with:
   > "`implementation-plan.md` not found. This skill requires a plan created by `plan-ticket` or `run-plan`."
5. From `implementation-plan.md`, find the `**Last agent commit:**` header line — extract the hash
   - If the header is absent or the value is `(none yet)`, halt with: "`implementation-plan.md` has no recorded agent commit — this plan was not run with `run-plan`. Re-run using `run-plan` to generate the commit history needed for a retrospective."
6. Read `correction-log.md` if it exists — count correction groups and extract entries
7. Read `code-review.md` if it exists — note any findings flagged there

### Phase 2: Analyse Git History

8. Run `git log --oneline` to see full history
9. Using the hash from `**Last agent commit:**`, run `git log --oneline <hash>..HEAD`
   - Each commit listed is a **post-implementation change** (made after the agent finished — peer review fixes, late corrections)

10. For each post-implementation commit, read its diff: `git show <hash>`
    - Understand what changed and why (from the commit message)

### Phase 3: Reconstruct Timeline

11. Build a chronological picture:
    - **Planned implementation**: steps marked `[DONE]` in `implementation-plan.md`
    - **Implementation corrections**: entries in `correction-log.md` (corrections detected mid-run by `run-plan`)
    - **Post-implementation changes**: commits after the last agent commit (peer review fixes, human edits)

### Phase 4: Categorise Findings

12. For each correction (from `correction-log.md`) and each post-implementation change (from git), create one finding entry
13. Assign exactly one **primary diagnostic category** per finding (see Diagnostic Categories below). If no category fits well, use `unclassified` and include a suggested new category name with a one-sentence rationale — this may indicate a gap in the taxonomy
14. Assess severity: `trivial` | `minor` | `significant`
15. Draft a specific, actionable proposed fix appropriate to the category

### Phase 5: Write and Present

16. Write `retrospective.md` to the ticket workspace (see Output Format below)
17. Present a brief summary to the user and ask them to review the proposals and complete the Actions Taken section

---

## Diagnostic Categories

Assign the category that best explains *why* the correction was needed:

| Category | When to use | What the proposed fix looks like |
|----------|------------|----------------------------------|
| `CLAUDE.md gap` | A missing rule caused the agent to make a wrong choice it would otherwise have avoided | Exact rule text, ready to copy-paste into CLAUDE.md |
| `plan quality` | The plan step was underspecified, missed a file, or didn't anticipate a dependency | The specific planning pattern to add to `plan-ticket` behaviour |
| `ticket quality` | Requirements were ambiguous, contradictory, or missing — a better ticket would have prevented the issue | Flag for upstream process — note what should have been specified |
| `skill behavior` | A specific skill (e.g. `execute-step`, `run-plan`) produced a recurring type of error | The specific prompt change or instruction to add to the skill |
| `codebase knowledge` | The agent didn't know about an undocumented pattern, convention, or structural constraint | CLAUDE.md rule to document the convention, or prime-context coverage improvement |
| `permissions/tooling` | The agent couldn't perform an action it needed to — missing tool, permission, or CLI | Config change or `update-config` invocation to recommend |
| `unclassified` | The finding doesn't fit any category above — may indicate a gap in the taxonomy | Describe what happened and suggest a candidate category name with a one-sentence rationale |

---

## Output Format

File: `.ai-workspace/{TICKET-ID}_{slug}/retrospective.md`

```markdown
# Retrospective — {TICKET-ID}: {ticket title}

**Date:** {today's date}
**MR:** {MR reference from ticket-info.md or git log, or "not recorded" if unavailable}
**Implementation corrections:** {count of correction groups from correction-log.md, or 0 if file absent}
**Post-implementation changes:** {count of commits from git log <hash>..HEAD}

## Timeline

{2–4 sentence narrative: what the plan covered, where it diverged, what was corrected and when.
Factual, not evaluative. E.g. "The plan had 4 steps. Steps 1–3 ran cleanly. A correction was
detected before step 4 when the agent omitted X. After the agent commit, 2 peer review
commits addressed Y and Z."}

## Findings

{One entry per correction or post-implementation change. If there are no findings, write:
"No corrections or post-implementation changes detected for this ticket."}

### Finding {N}: {short descriptive title}

**Source:** {one of: Correction log — Group N | Post-implementation git history | Code review}
**Category:** {diagnostic category}
**Severity:** {trivial | minor | significant}

**What happened:** {what the agent produced, what was changed, what the gap was}
**Root cause:** {why this happened — be specific}
**Proposed fix:** {specific and actionable:
  - For CLAUDE.md gap: include exact rule text
  - For plan quality: include specific pattern to add
  - For ticket quality: describe what the ticket should have specified
  - For skill behavior: name the skill and the specific prompt change
  - For codebase knowledge: include CLAUDE.md rule text or prime-context coverage note
  - For permissions/tooling: include the config change needed
  - For unclassified: describe what happened and suggest a candidate category name with rationale}
**Suggested category** *(unclassified only)*: {candidate category name} — {one sentence on why this pattern warrants its own category}

---

## Recommendations Summary

### CLAUDE.md Additions

{List each proposed rule with exact text, ready to copy-paste. If none proposed: "none"}

### Skill Adjustments

{List each skill to adjust and what specifically to change. If none proposed: "none"}

### Planning Improvements

{Patterns to build into plan-ticket or execute-step behaviour. If none proposed: "none"}

### Ticket / Process Improvements

{Upstream issues to flag — what should change in how tickets are written or refined.
If none proposed: "none"}

## Actions Taken

{The human completes this section after reviewing the recommendations.
Each proposal should be marked and annotated. Leave blank until reviewed.}

| Proposal | Decision | Notes |
|----------|----------|-------|
| {e.g. CLAUDE.md rule: "Always use full_name not user_name"} | Accepted / Rejected / Modified | {what was actually done} |
```

---

## Edge Cases

### No corrections at all

If `correction-log.md` does not exist (or is empty) and `git log <hash>..HEAD` shows no commits, write the retrospective with zero findings:

> "No corrections or post-implementation changes detected for this ticket."

Still write `retrospective.md` and include an empty Findings section and empty Recommendations Summary (each subsection marked "none"). This is valid and useful data — it tells the cross-ticket review that this ticket ran cleanly.

### Empty correction-log.md

Treat as equivalent to the file not existing: zero implementation corrections.

### Ambiguous post-implementation commits

When a commit after the last planned commit is ambiguous — it could be a planned change or a correction — use `implementation-plan.md` as the reference:

- If the commit's changes correspond to steps marked `[DONE]` in the plan, treat it as a planned commit (not a finding)
- If the changes go beyond what any plan step describes, treat it as a post-implementation change and create a finding

### Last agent commit hash not in current git history

If the hash recorded in `**Last agent commit:**` cannot be found (e.g. history was rewritten), halt with:

> "The agent commit `{hash}` recorded in `implementation-plan.md` is not present in git history — the history may have been rewritten. Restore the commit history or update `**Last agent commit:**` to a valid hash before running the retrospective."

---

## Presenting Results

After writing `retrospective.md`, output:

```markdown
## Retrospective Complete: {TICKET-ID}

**Workspace**: `.ai-workspace/{TICKET-ID}_{slug}/`

**Implementation corrections**: {count}
**Post-implementation changes**: {count}

### Timeline

{Repeat the 2–4 sentence narrative from retrospective.md}

### Findings Summary

{If findings exist:}
{N} findings identified:
- Finding 1: {title} ({category}, {severity})
- Finding 2: {title} ({category}, {severity})
...

{If no findings:}
No corrections or post-implementation changes detected — this ticket ran cleanly.

### Proposed Actions

{List the concrete proposals from Recommendations Summary, grouped by type.
If none: "No changes proposed."}

---

**Next**: Review `retrospective.md` and complete the **Actions Taken** table.
Apply accepted proposals (CLAUDE.md edits, skill updates, process notes) before closing the ticket.
```

---

## Error Handling

- **Workspace not found**: "No workspace found for {TICKET-ID}. Run `start-ticket` to create one."
- **`implementation-plan.md` missing**: "`implementation-plan.md` not found. This skill requires a plan created by `plan-ticket` or `run-plan`."
- **`requirements-analysis.md` missing**: Continue without it — note in the retrospective that original requirements were unavailable.
- **Agent commit hash not in git history**: Halt — "The agent commit `{hash}` recorded in `implementation-plan.md` is not present in git history — the history may have been rewritten. Restore the commit history or update `**Last agent commit:**` to a valid hash before running the retrospective."
- **Git command fails**: Halt and inform the user — git history is required for the retrospective and cannot be skipped.
- **No `**Last agent commit:**` header or value is `(none yet)`**: Halt — "`implementation-plan.md` has no recorded agent commit — this plan was not run with `run-plan`. Re-run using `run-plan` to generate the commit history needed for a retrospective."
