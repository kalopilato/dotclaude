---
name: start-ticket
description: Start working on a ticket. Fetches ticket, creates workspace, analyzes requirements, and primes context in one command.
---

## Configuration

**Workspace Directory**: `.ai-workspace/`

All artifacts are stored in `.ai-workspace/{TICKET-ID}_{slugified-title}/`

---

# Start Ticket

Run this once at the start of a ticket to set up everything needed for planning. It fetches the ticket, creates a workspace, delegates analysis to `analyze-ticket`, and primes project context.

## Input

Ticket ID or URL (required). Examples:
- `PROJ-142`
- `https://ticketsystem.example.com/issue/PROJ-142`

## Process

Execute these three phases in sequence, continuing automatically unless an error occurs.

---

### Phase 1: Kickoff

**Goal**: Fetch ticket, create workspace, check branch.

1. Parse ticket ID from input (accept ticket ID like `PROJ-142` or full URL)
2. Fetch ticket (issue + comments) using whatever ticket system tool is available (CLI, MCP, etc.)
3. Create workspace directory:
   - **Slugify title**: lowercase, replace spaces and special chars with hyphens, max 50 chars
   - Path: `.ai-workspace/{TICKET-ID}_{slugified-title}/`
   - Example: `PROJ-142: Add email notification preferences` → `.ai-workspace/PROJ-142_add-email-notification-preferences/`
   - If workspace already exists: reuse it without prompting
4. Write `ticket-info.md` to workspace using this template:

```markdown
# {TICKET-ID}: {Title}

**URL**: {Ticket URL}
**Status**: {ticket status}
**Workspace**: `.ai-workspace/{TICKET-ID}_{slugified-title}/`

## Description

{full ticket description}

## Acceptance Criteria

{acceptance criteria if present, otherwise omit section}

## Comments

{all comments, preserving author and date — omit section if no comments}
```

5. Check git branch:
   - If branch name contains the ticket ID (case-insensitive): note as matching
   - If not: suggest branch name `{ticket-id}_{concise-slug}` (max ~40 chars for slug). Do not switch branches — just note the suggestion

**On error**: Stop and report. Common errors:
- Ticket not found
- No ticket system tool available

---

### Phase 2: Analyze

**Goal**: Extract requirements, identify gaps, assess readiness.

Invoke the `analyze-ticket` skill with the ticket ID via the Skill tool. It runs as a forked subagent (`context: fork`) and reads `ticket-info.md` from the workspace created in Phase 1.

Wait for it to complete, then read `requirements-analysis.md` and `questions.md` (if created) from the workspace to extract:
- Complexity assessment
- Readiness status
- Summary of requirements
- Any blocking questions

---

### Phase 3: Prime Context

**Goal**: Load project structure into conversation context.

Invoke the `prime-context` skill via the Skill tool. It runs inline (not forked) so its output stays in the main conversation context.

---

## Output

```markdown
## Ready: {TICKET-ID} - {Title}

**Workspace**: `.ai-workspace/{TICKET-ID}_{slug}/`
**Ticket**: {URL}
**Branch**: `{current}` ✓ | ⚠️ suggested: `{ticket-id}_{slug}`

**Complexity**: {Simple/Medium/Complex}
**Readiness**: Ready | Needs refinement

**Files Created**:
- `ticket-info.md`
- `requirements-analysis.md`
- `questions.md` (if questions exist)
- `related-tickets.md` (if related tickets were fetched)

---

### Summary

{2-3 sentences: what needs to be built}

### Requirements

- {Requirement 1}
- {Requirement 2}
- {Requirement 3}

### Questions (if any)

{List question titles, or "No blocking questions"}

---

**Next**:

{If questions exist:}
Review questions above. Answer them or say "proceed" to continue with current understanding.

Then invoke `plan-ticket` to create an implementation plan.

{If no questions:}
Invoke `plan-ticket` to create an implementation plan.
```

## Error Handling

- **Ticket not found**: Stop after Phase 1. "Could not find ticket {ID}. Verify the ticket ID or URL."
- **No ticket system tool**: Stop after Phase 1. "No ticket system tool available. Configure a CLI or MCP before proceeding."
- **Cannot create workspace**: Stop after Phase 1. "Could not create workspace. Check directory permissions."

## Notes

- This skill does not pause between phases — it runs straight through
- Workspace reuse is automatic (no confirmation prompt)
- Branch switching is not automatic — only suggestions are provided
- All three phases must complete for the skill to succeed
