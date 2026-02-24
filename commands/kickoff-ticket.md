---
name: kickoff-ticket
description: Initialize ticket workspace by fetching ticket and creating organized workspace directory. Loads ticket into conversation context for subsequent commands.
---

You are initializing a new ticket implementation session. Your job is simple and focused: fetch the ticket, create a workspace, and load the ticket into context for the next steps.

## Your Task

1. Fetch the ticket (issue + comments) using the configured ticket system
2. Create an organized workspace directory
3. Write basic ticket info to workspace
4. Check the current git branch matches the ticket ID, or suggest one
5. Keep ticket in conversation context for subsequent commands

## Process

### Step 1: Parse Ticket Input

Accept either:
- Ticket ID: e.g., `PROJ-752`
- Full ticket URL: e.g., `https://ticketsystem.example.com/issue/PROJ-752/...`

Extract the ticket ID from the input.

### Step 2: Fetch Ticket

Use whichever ticket system MCP tool is available to fetch the ticket's full details including title, description, url, status, and comments.

Store in conversation context:
- Ticket title
- Ticket description
- Ticket URL
- Comments (for later analysis)

### Step 3: Create Workspace Directory

Create workspace in the project's `{WORKSPACE_DIR}` directory: `{WORKSPACE_DIR}/{TICKET-ID}_{slugified-title}/`

**Slugify title**: lowercase, replace spaces/special chars with hyphens, max 50 chars

Example:

- Ticket: `PROJ-142: Add email notification preferences`
- Workspace: `{WORKSPACE_DIR}/PROJ-142_add-email-notification-preferences/`

```bash
# Create {WORKSPACE_DIR} if it doesn't exist, then the ticket subdirectory
mkdir -p {WORKSPACE_DIR}/{TICKET-ID}_{slugified-title}/
```


**If workspace already exists**:
- Check if it's from a previous session
- Ask user: "Workspace already exists at `{WORKSPACE_DIR}/{TICKET-ID}_*/`. Continue with existing workspace? (y/n)"
- If yes: use existing workspace
- If no: suggest deleting or using different workspace

### Step 4: Write Ticket Info

Create `ticket-info.md` in workspace:

```markdown
# {TICKET-ID}: {Title}

**URL**: {Ticket URL}
**Status**: {ticket status}
**Workspace**: `{WORKSPACE_DIR}/{TICKET-ID}_{title}/`

## Description

{ticket description}

## Acceptance Criteria

{acceptance criteria if present in description}

---
*Workspace created: {timestamp}*
```

### Step 5: Check Git Branch

Check the current branch:

```bash
git branch --show-current
```

- If the branch name starts with the ticket ID (case-insensitive): all good, proceed.
- If not: suggest a branch name of the form `{ticket-id}_{concise-slugified-title}` (lowercase, hyphens, max ~40 chars for the slug portion), e.g. `PROJ-142_email-notification-preferences`. Do not switch branches automatically — just inform the user.

### Step 6: Confirm and Provide Next Steps

Output:

```markdown
Workspace initialized for **{TICKET-ID}: {Title}**

**Workspace**: `{WORKSPACE_DIR}/{TICKET-ID}_{title}/`
**Ticket URL**: {Ticket URL}
**Branch**: `{current-branch}` ✓  (or: ⚠️ Not on a ticket branch — suggested: `{ticket-id}_{concise-slug}`)

**Files Created**:
- `ticket-info.md` - Basic ticket information

**What's Next**:

If you haven't already, run `/prime-context` in the project to load its structure into context.

Run `/analyze-ticket` to extract requirements and identify gaps.

The ticket and comments are now loaded in conversation context and will be available for subsequent commands without re-fetching.
```

## Important Guidelines

1. **Keep it simple**: This command just sets up, it doesn't analyze
2. **Load ticket into context**: Fetch both issue and comments so they're cached
3. **Don't create unnecessary files**: Just `ticket-info.md` for reference
4. **Clear next steps**: Tell user to run `/analyze-ticket` next
5. **Handle existing workspaces gracefully**: Ask before overwriting

## Error Handling

- **Ticket not found**: "Could not find ticket {ID}. Please verify the ticket ID or URL."
- **Ticket system tool not available**: "No ticket system MCP tool is available. Please ensure a ticket system MCP is configured and connected."
- **Cannot create workspace**: "Could not create workspace directory. Please check permissions."

## Example Usage

```
User: /kickoff-ticket PROJ-142

You: Fetching ticket PROJ-142...

Workspace initialized for **PROJ-142: Add email notification preferences**

**Workspace**: `{WORKSPACE_DIR}/PROJ-142_add-email-notification-preferences/`
**Ticket URL**: {ticket URL from ticket system}
**Branch**: `main` ⚠️ Not on a ticket branch — suggested: `PROJ-142_email-notification-preferences`

**Files Created**:
- `ticket-info.md` - Basic ticket information

**What's Next**:

If you haven't already, run `/prime-context` in the project to load its structure into context.

Run `/analyze-ticket` to extract requirements and identify gaps.

The ticket and comments are now loaded in conversation context and will be available for subsequent commands without re-fetching.
```

You are ready to kickoff tickets efficiently and set up organized workspaces!
