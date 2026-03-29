---
name: cross-ticket-review
description: Run periodically to aggregate retrospective files across ticket workspaces, identify systemic patterns, track fix effectiveness, and recommend workflow-level improvements. Maintains a persistent issue registry and dated run reports.
---

## Configuration

**Workspace Directory**: `.ai-workspace/`

All artifacts are stored in `.ai-workspace/{TICKET-ID}_{slugified-title}/`

---

You are a workflow diagnostician. You run periodically to surface patterns that individual retrospectives cannot see — recurring issues, fixes that are or aren't working, and systemic improvements to the engineering workflow.

## Your Task

Read retrospective files from completed ticket workspaces, update a persistent issue registry, assess fix effectiveness over time, and produce a dated workflow review report with actionable recommendations.

## When to Run

Periodically — weekly, after every 5 tickets, or on demand.

**Invocation**: `cross-ticket-review`

**Runs**: Inline (not forked). Pattern analysis across multiple retrospectives requires full conversation context.

---

## Inputs

| Source | How to find it |
|--------|---------------|
| `review-state.md` | `.ai-workspace/workflow-review/review-state.md` |
| `issue-registry.md` | `.ai-workspace/workflow-review/issue-registry.md` |
| New retrospectives | Glob `.ai-workspace/*/retrospective.md`, then filter to those whose ticket ID is not listed in `review-state.md`'s processed list |

---

## Process

### Phase 1: Initialise Workspace (first run only)

Check whether `.ai-workspace/workflow-review/` exists.

If it does not exist, create the directory structure and initialise all three files using the templates in the **Initial Templates** section below:

```
.ai-workspace/workflow-review/
    ├── review-state.md
    ├── issue-registry.md
    └── review-reports/
```

### Phase 2: Read State

Read `.ai-workspace/workflow-review/review-state.md`.

Extract:
- **Processed ticket IDs**: the comma-separated list on the `Tickets processed:` line
- **Last run date**: the value on the `Last run:` line

### Phase 3: Find New Retrospectives

Glob `.ai-workspace/*/retrospective.md`.

For each match, extract the ticket ID from the workspace directory name (the segment before the first underscore, e.g. `v2.1.0` from `v2.1.0_retrospective-skill`).

Filter to only those ticket IDs **not** in the processed list from Phase 2.

### Phase 4: Early Exit Check

If no new retrospectives are found after filtering:

> "No new retrospectives since last run on {date}."

Stop. Do not write a new report.

### Phase 5: Check Actions Taken Completeness

For each new retrospective, check whether its **Actions Taken** section is complete:

- **Incomplete**: the table is empty, contains only the template placeholder row, or the section is missing entirely
- **Complete**: at least one row has a real decision recorded

If a retrospective is incomplete, **skip it** and note it in the run report as:

> "Skipped {TICKET-ID}: Actions Taken not yet completed."

If all new retrospectives are incomplete after this check, report all skipped tickets and stop. Do not write a new report.

### Phase 6: Process Each Retrospective

Process complete retrospectives in ticket order, oldest first.

For each:

1. Read the full retrospective file including the Actions Taken section
2. Extract each finding (from the Findings section) and what action was taken in response (from Actions Taken)
3. For each finding, check the issue registry for a matching issue:
   - **Match criteria**: same diagnostic category AND same affected skill or file
   - If a match exists: this is a **recurrence** — note which ticket it recurred in
   - If no match: create a new issue entry (assign the next ISS-NNN ID)

### Phase 7: Assess Fix Effectiveness

For each issue in the registry where:
- A fix was previously applied (status is `Monitoring` or `Resolved`), **and**
- At least one new ticket has been processed since the fix was applied

Assess the fix using the retrospectives processed since the fix was applied:

| Status | Criteria |
|--------|----------|
| **Resolved** | Issue not present in any ticket processed after the fix was applied |
| **Partially resolved** | Issue present in some subsequent tickets but with lower frequency or severity than before the fix |
| **Recurring** | Issue present in subsequent tickets at similar frequency or severity despite the fix |
| **Insufficient data** | Fewer than 3 tickets processed since the fix was applied — too early to assess |

When an issue reaches `Resolved`, note the evidence (which tickets were clean). When it returns to `Open` after a fix, append "(fix ineffective)" to the Fix applied field.

### Phase 8: Identify New Patterns

A **new systemic pattern** is any issue appearing in 2 or more tickets in this processing batch that is not already in the issue registry.

Note each new pattern separately in the run report.

### Phase 9: Write Run Report

Write `.ai-workspace/workflow-review/review-reports/YYYY-MM-DD.md` using the **Run Report Format** below.

### Phase 10: Update Issue Registry

Write the updated `issue-registry.md` with:
- New issues added
- Recurrence counts and last-occurrence columns updated
- Status changes applied (Open → Monitoring, Monitoring → Resolved, Resolved → Open)

### Phase 11: Update Review State

Update `review-state.md`:
- `Last run:` → today's date
- `Tickets processed:` → append the newly processed ticket IDs to the existing list

### Phase 12: Present Summary

Output a brief summary to the user (see **Presenting Results** below) and ask them to review the run report and complete its Actions Taken section.

---

## Issue Registry Format

File: `.ai-workspace/workflow-review/issue-registry.md`

```markdown
# Issue Registry

Last updated: {date}

| ID | Issue | Category | First seen | Status | Fix applied | Fix applied after | Last occurrence |
|----|-------|----------|-----------|--------|-------------|------------------|-----------------|
| ISS-001 | {short description} | {category} | {TICKET-ID} | {Open \| Monitoring \| Resolved} | {description or —} | {TICKET-ID or —} | {TICKET-ID or —} |
```

**Status lifecycle:**

- `Open` — issue identified, no fix applied yet
- `Monitoring` — fix applied; watching for recurrence. Record what fix was applied and after which ticket.
- `Resolved` — fix applied and issue has not recurred across 5+ subsequent tickets. Note the evidence.
- `Open` (returned) — issue recurred despite fix being applied. Append "(fix ineffective)" to the Fix applied field.

---

## Review State Format

File: `.ai-workspace/workflow-review/review-state.md`

```markdown
# Cross-Ticket Review State

Last run: {date}
Tickets processed: {comma-separated TICKET-IDs}
Next scheduled: {date, or "On demand"}
```

---

## Run Report Format

File: `.ai-workspace/workflow-review/review-reports/YYYY-MM-DD.md`

```markdown
# Workflow Review — {date}

**Tickets reviewed this run:** {comma-separated TICKET-IDs}
**Skipped (Actions Taken incomplete):** {comma-separated TICKET-IDs, or "None"}
**New issues identified:** {count}
**Issues now resolved:** {count}
**Issues recurring despite fix:** {count}

## Fix Effectiveness

{One entry per issue where a fix was previously applied and there are now enough
subsequent tickets to assess. If none: write "No fixes to assess yet."}

### {ISS-ID}: {issue description}

**Fix applied:** {what was done, from the retrospective Actions Taken}
**Applied after:** {TICKET-ID where the fix was applied}
**Tickets assessed:** {list of TICKET-IDs processed since the fix}
**Assessment:** {Resolved | Partially resolved | Recurring | Insufficient data}
**Evidence:** {which tickets showed the issue or its absence}

---

## New Patterns

{One entry per new issue identified this run. If none: write "No new patterns
identified this run."}

### {ISS-ID}: {issue description}

**Category:** {diagnostic category — same categories as the retrospective skill}
**Seen in:** {TICKET-IDs}
**Pattern:** {what the common thread is — specific, not generic}
**Recommended fix:** {specific and actionable — exact CLAUDE.md rule text, specific
skill change, or process improvement}

---

## Workflow Recommendations

{Higher-level observations that don't map to a single issue — e.g. "auto/review
boundary should be tightened", "plan-ticket consistently misses migration steps",
"correction rates have dropped significantly over the last 8 tickets".
If none: write "No workflow-level recommendations this run."}

---

## Actions Taken

{The human completes this section after reviewing the report.}

| Recommendation | Decision | Notes |
|----------------|----------|-------|
| {recommendation} | Accepted / Rejected / Modified | {what was actually done} |
```

---

## Initial Templates

Write these when creating the workspace on first run.

### `review-state.md` initial content

```markdown
# Cross-Ticket Review State

Last run: Never
Tickets processed: None
Next scheduled: On demand
```

### `issue-registry.md` initial content

```markdown
# Issue Registry

Last updated: {today's date}

| ID | Issue | Category | First seen | Status | Fix applied | Fix applied after | Last occurrence |
|----|-------|----------|-----------|--------|-------------|------------------|-----------------|
```

---

## Edge Cases

### Fewer than 3 retrospectives available total (including previous runs)

Process what is available. Add a note at the top of the report:

> "Note: Only {N} retrospectives available. Pattern analysis is more reliable after
> 5+ tickets. Findings should be treated as preliminary."

### First run with no prior state

Create the workspace directory and initialise all template files (Phase 1). Process all available retrospectives as the first batch. This is expected — do not treat it as an error condition.

### Issue matching ambiguity

When a new finding could match an existing registry issue, prefer a match if the category and the affected skill or file are the same. If genuinely ambiguous, create a new issue and note "Possible duplicate of {ISS-ID}" in the issue description.

---

## Presenting Results

After writing the run report, output:

```markdown
## Workflow Review Complete — {date}

**Tickets reviewed:** {comma-separated TICKET-IDs}
**Skipped (incomplete Actions Taken):** {list, or "None"}

**New issues identified:** {count}
**Issues now resolved:** {count}
**Issues recurring despite fix:** {count}

### Key Findings

{If high-severity recurring issues exist, flag them prominently here.}

{2–4 bullet points summarising the most important findings from this run.
If no findings of note: "No significant patterns identified this run."}

### Report

Saved to `.ai-workspace/workflow-review/review-reports/{date}.md`

---

**Next**: Review the report and complete the **Actions Taken** table.
Apply accepted proposals (CLAUDE.md edits, skill updates, process notes) before the next review run.
```

---

## Error Handling

- **Workspace not found and cannot be created**: Halt — "Could not create `.ai-workspace/workflow-review/`. Check directory permissions."
- **`review-state.md` missing but workspace exists**: Re-initialise from the initial template and proceed. Note in the report: "review-state.md was missing and has been re-initialised."
- **`issue-registry.md` missing but workspace exists**: Re-initialise from the initial template and proceed. Note in the report: "issue-registry.md was missing and has been re-initialised."
- **Retrospective file unreadable**: Skip it and note in the report: "Skipped {TICKET-ID}: retrospective.md could not be read."
- **Cannot determine ticket ID from workspace directory name**: Skip the retrospective and note it in the report.
