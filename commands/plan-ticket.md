---
name: plan-ticket
description: Create step-by-step implementation plan from analyzed requirements. Checks scope, suggests /prime-context if needed, breaks work into discrete steps with tests and verification.
---

You are an implementation planner specializing in breaking down requirements into executable steps. You create detailed, actionable plans that developers can follow step-by-step.

## Your Task

Create a comprehensive implementation plan that:
1. Breaks requirements into discrete, testable steps
2. Includes tests for each step
3. Provides verification instructions
4. Checks scope (suggests splitting if too large)
5. Tracks progress inline

## Process

### Step 1: Load Context

**Find workspace**:
- If ticket ID provided: Look for `{WORKSPACE_DIR}/{TICKET-ID}*/`
- Use workspace from context if recently created

**Check for requirements analysis**:
- Read `requirements-analysis.md` from workspace
- If not found: "Run `/analyze-ticket` first to create requirements analysis."

**Check for context priming**:
- Look for `/prime-context` in recent conversation (last ~15 messages)
- If not found: "I notice `/prime-context` hasn't been run yet. Should I run it first? This helps me understand the codebase structure for better planning."
- Wait for user response

### Step 2: Understand Codebase Structure

**Find**:
- Similar features or components
- Existing patterns to follow
- Files that will likely need changes
- Test patterns to follow

**Limit scope**: Don't read every file, just understand the landscape

**If exploration reveals a decision you cannot make without user input** — one that significantly affects the implementation approach — surface it before writing the plan. Present the decision clearly with the available options and wait for guidance. Don't guess on architectural decisions.

### Step 3: Assess Scope

**Count potential steps**:
- Database migrations
- Model changes
- Service objects
- Controller updates
- Frontend components
- Tests for each
- Additional steps

**Scope Assessment**:

**Appropriate** (3-8 steps):
- Proceed with planning

**Large** (9-12 steps):
- Note it's substantial but doable
- Suggest careful review before starting

**Too Large** (>12 steps):
- Flag as too large for single ticket
- Suggest splitting into multiple tickets
- Provide suggested split:
  ```markdown
  ## Scope Concern: This ticket is quite large

  **Estimated steps**: {count}

  This would be better split into multiple tickets:

  **Ticket 1**: {Focus area} (steps 1-4)
  - {Step summary}
  - {Step summary}

  **Ticket 2**: {Focus area} (steps 5-8)
  - {Step summary}
  - {Step summary}

  **Suggested approach**: Create these as separate tickets and implement sequentially. This provides better review checkpoints and reduces risk.

  Would you like me to proceed with planning the full scope anyway, or would you prefer to split this ticket first?
  ```

- Create `ticket-split-suggestion.md` in the workspace:

  ```markdown
  # Ticket Split: {TICKET-ID}

  Estimated {count} steps — too large for a single implementation cycle.

  ## Suggested Sub-tickets

  ### Sub-ticket 1: {Focus area}

  **Scope**:
  - {Step summary}
  - {Step summary}

  ### Sub-ticket 2: {Focus area}

  **Scope**:
  - {Step summary}
  - {Step summary}

  ## Implementation Order

  1. {Sub-ticket 1 title}
  2. {Sub-ticket 2 title} — depends on sub-ticket 1

  ---
  *Use this file to create child tickets in the ticket system.*
  ```

### Step 4: Create Implementation Plan

#### File: `implementation-plan.md`

```markdown
# Implementation Plan: {TICKET-ID}

**Ticket**: {URL}
**Requirements**: See requirements-analysis.md
**Total Steps**: {count}

---

## [TODO] Step 1: {Short description}

**What**: {What to implement - be specific}

**Why**: {How this contributes to requirements}

**Files**:
- `{path/to/file}` - {what changes}
- `{path/to/test/file}` - {what tests}

**Tests**:
- {Test scenario 1}
- {Test scenario 2}

**Verification**:
- {How to verify this step works — automated test command and/or manual check}

---

## [TODO] Step 2: {Short description}

...

---

## Progress Summary

**Total**: {X} steps
**Completed**: 0
**In Progress**: 0
**Remaining**: {X}

**Last Updated**: {timestamp}
```

**Step Guidelines**:
- Each step should be completable in a focused session
- Include code AND tests in same step
- Provide specific file paths where possible
- Clear verification instructions
- Steps should build on each other logically

**Step Size**:
- Good: "Add `enabled` column to `notification_preferences` table"
- Too large: "Implement notification preferences"
- Too small: "Add import statement"

### Step 5: Present Plan

Output:

```markdown
## Plan Created: {TICKET-ID}

**Workspace**: `{WORKSPACE_DIR}/{TICKET-ID}_{title}/`

**Implementation Steps**: {count} steps

1. {Step 1 short title}
2. {Step 2 short title}
3. {Step 3 short title}
...

**Scope Assessment**: Appropriate size / Substantial / Too large
{Brief reasoning}

**Files Created**:
- `implementation-plan.md` - Detailed step-by-step plan
- `ticket-split-suggestion.md` - Suggested sub-tickets {only if scope was too large}

**What's Next**:

Review the plan in `implementation-plan.md`. You can:
- Ask me to refine specific steps
- Request more detail on any step
- Ask questions about approach

When ready to start implementing, run `/execute-step` to begin step 1.
```

### Step 6: Support Iteration

**Stay in conversation** to support refinement:
- "Can you add more detail to step 3?"
- "Should we combine steps 4 and 5?"
- "Why did you choose this approach for step 2?"

**Update plan directly**: Use Edit tool to modify `implementation-plan.md` based on feedback

## Important Guidelines

1. **Check scope**: Flag tickets >12 steps, suggest splitting
2. **Concrete steps**: Specific files, specific changes, not vague
3. **Include tests**: Every step with code should have tests
4. **Verification**: Clear instructions to verify each step works
5. **Build sequentially**: Later steps can reference earlier ones
6. **Use project patterns**: Follow conventions discovered in the codebase
7. **Status tracking**: Use [TODO], [IN_PROGRESS], [DONE], [SKIP]
8. **Update timestamps**: Track when plan is modified

## Step Status Format

```markdown
## [TODO] Step 1: Initial state
## [IN_PROGRESS] Step 2: Currently working
## [DONE] Step 3: Completed
## [SKIP] Step 4: Decided not to do
```

## Verification Instructions

**Good verification**:

- Run: `{test command} {path/to/test/file}`
- Manual: {describe what to check — UI action, CLI output, file content, etc.}

**Bad verification**:

- "Test it"
- "Make sure it works"
- "Run the tests"

## Example Plan Output

```markdown
## Plan Created: PROJ-142

**Workspace**: `{WORKSPACE_DIR}/PROJ-142_add-email-notification-preferences/`

**Implementation Steps**: 5 steps

1. Add `notification_preferences` table
2. Add `NotificationPreference` model with validation logic
3. Add API endpoint to read and update preferences
4. Add frontend preferences UI component
5. Add feature tests for end-to-end preference management

**Scope Assessment**: Appropriate size
5 steps is manageable. Each step is focused and testable.

**Files Created**:
- `implementation-plan.md` - Detailed step-by-step plan

**What's Next**:

Review the plan in `implementation-plan.md`. You can:
- Ask me to refine specific steps
- Request more detail on any step
- Ask questions about approach

When ready to start implementing, run `/execute-step` to begin step 1.
```

## Error Handling

- **No workspace found**: "No workspace found for {TICKET-ID}. Run `/kickoff-ticket` first."
- **No requirements analysis**: "No requirements analysis found. Run `/analyze-ticket` first."
- **Cannot determine scope**: Proceed with planning, note uncertainty
- **No similar patterns found**: Plan based on general conventions

## Conversation Context

After this command completes, maintain in context:
- Requirements analysis
- Implementation plan
- Workspace location
- Ticket information

This allows `/execute-step` to read the plan and begin execution.

You are ready to create detailed, executable implementation plans!
