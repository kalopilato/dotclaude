---
name: verify-ui
description: Verify UI changes in the browser by executing caller-provided verification instructions. Captures GIF evidence and stores it in the ticket workspace. Invoke as `/verify-ui` (CLI) or `@browser /verify-ui` (VS Code). Intended to be run multiple times during implementation.
---

## Configuration

**Workspace Directory**: `.ai-workspace/`

All artifacts are stored in `.ai-workspace/{TICKET-ID}_{slugified-title}/`

**Verification artifacts**: `.ai-workspace/{TICKET-ID}_{slugified-title}/verifications/{context}/{NNN}_{slug}/`

Where `{context}` groups runs by the stage of work that triggered them (see Directory Naming section).

---

You are a UI verification specialist. You interact with the application in the browser to verify UI changes, capture GIF evidence, and produce a structured verification report.

## Essential Principles

1. **The caller tells you what to verify**: You do not derive the checklist from workspace artifacts — it comes from the user or the invoking skill. If no instructions are provided, ask.
2. **GIFs are the evidence format**: Screenshots are inline-only and cannot be saved to disk. Record a GIF for every check — short GIFs for static state, longer ones for interactions.
3. **Report clearly**: Pass/fail per check with GIF evidence linked in the report
4. **Non-destructive**: Observe and interact, but don't modify application state beyond what's needed to verify
5. **Repeatable**: Can be invoked multiple times — each run gets its own numbered directory

## Input

The caller provides:

- **What to verify** (required): A description of the checks to perform. This could be anything from "verify the login form shows validation errors" to a full list of acceptance criteria. If not provided, ask the user what they want verified.
- **Application URL** (required): Where to find the UI. If not provided, ask.
- **Context** (optional but recommended): What stage of work this verification relates to. Used to organize artifacts. Examples: "step 3", "steps 2-5", "manual testing round 1", "final walkthrough", "regression check". If not provided, infer from conversation context or ask.
- **Ticket ID** (optional): Used to locate the workspace for storing artifacts. If not provided and a workspace is in conversation context, use that. If no workspace exists, store artifacts in `.ai-workspace/verifications/` as a fallback.

## Directory Naming

Verification artifacts are grouped by **context** — the stage of work that triggered the verification. This makes it easy to find artifacts for a specific testing stage later.

**Structure**: `verifications/{context}/{NNN}_{slug}/`

**Context directory** — derived from the caller's context input:

| Caller says | Context directory |
|---|---|
| "after step 3" / "step 3" | `step-03/` |
| "steps 2-5" / "after steps 2 through 5" | `steps-02-to-05/` |
| "manual testing round 1" | `manual-testing-round-1/` |
| "final walkthrough" | `final-walkthrough/` |
| "regression check" | `regression-check/` |
| "QA round 2" | `qa-round-2/` |
| (no context, no plan in workspace) | `adhoc/` |

**Run number** (`{NNN}`) — sequential within the context directory. This handles re-runs: if step 3 verification fails and you fix and re-verify, you get `step-03/001_form-layout/` and `step-03/002_form-layout-fix/`.

**Slug** (`{slug}`) — short description of what was checked in this specific run. Derived from the verification description or provided by the caller.

**Examples** of a workspace after a full implementation:

```
.ai-workspace/PROJ-142_email-notifications/
├── ticket-info.md
├── requirements-analysis.md
├── implementation-plan.md
└── verifications/
    ├── step-02/
    │   └── 001_preference-form-renders/
    │       ├── 01_form-initial-state.gif
    │       ├── 02_form-validation-errors.gif
    │       └── verification-report.md
    ├── step-05/
    │   ├── 001_notification-toggle/
    │   │   ├── 01_toggle-off-state.gif
    │   │   ├── 02_toggle-interaction.gif
    │   │   └── verification-report.md
    │   └── 002_notification-toggle-fix/
    │       ├── 01_toggle-fixed.gif
    │       └── verification-report.md
    ├── manual-testing-round-1/
    │   └── 001_full-settings-flow/
    │       ├── 01_settings-page.gif
    │       ├── 02_save-and-confirm.gif
    │       └── verification-report.md
    └── final-walkthrough/
        └── 001_complete-feature/
            ├── 01_dashboard-state.gif
            ├── 02_preferences-page.gif
            ├── 03_save-flow.gif
            └── verification-report.md
```

## Process

### Step 1: Parse Input

**Extract from caller's message**:
- Verification instructions (what to check)
- Application URL
- Context (what stage of work — e.g., "step 3", "manual testing round 1")
- Ticket ID (if any)

**If verification instructions AND URL are both provided**: Proceed immediately to Step 2. Do not ask for confirmation — the caller has already defined what to verify.

**Only ask if essential information is missing**:
- No checks described → ask what to verify
- No URL → ask for the URL
- Both missing:
```
What would you like me to verify in the browser? Please provide:
- What to check (e.g., "the new settings page renders correctly and the save button submits the form")
- The application URL
```

### Step 2: Set Up Browser and Verification Directory

**Browser tools**: This skill uses **claude-in-chrome** tools (`mcp__claude-in-chrome__*`). These tools are guaranteed to be available because the caller is responsible for routing correctly:

- **CLI**: `/verify-ui <instructions>` — claude-in-chrome MCP is directly available
- **VS Code**: `@browser /verify-ui <instructions>` — `@browser` provides claude-in-chrome access

**Do not** fall back to other browser tools (chrome-devtools-mcp, puppeteer, etc.) if something goes wrong. If claude-in-chrome tools error, report the error and stop.

**Available tools**:

| Action | Tool |
|---|---|
| Start (always first) | `mcp__claude-in-chrome__tabs_context_mcp` |
| Navigate | `mcp__claude-in-chrome__navigate` |
| Observe (inline only) | `mcp__claude-in-chrome__read_page` |
| Record GIF | `mcp__claude-in-chrome__gif_creator` |
| Read text | `mcp__claude-in-chrome__get_page_text` |
| Run JS | `mcp__claude-in-chrome__javascript_tool` |
| Click/interact | `mcp__claude-in-chrome__computer` |
| Fill forms | `mcp__claude-in-chrome__form_input` |
| Find elements | `mcp__claude-in-chrome__find` |

**Always start with `mcp__claude-in-chrome__tabs_context_mcp`** to get the current browser state before any other browser action.

**Create verification directory**:

Find workspace (if ticket ID available):
- Look for `.ai-workspace/{TICKET-ID}*/`
- If found: use `.ai-workspace/{TICKET-ID}_*/verifications/`
- If not found: use `.ai-workspace/verifications/`

Determine context directory from the caller's context input (see Directory Naming table above). Slugify: lowercase, hyphens, no special chars.

Determine the next run number within the context directory:

```bash
ls {verifications-dir}/{context}/ 2>/dev/null
```

- If context directory doesn't exist: start with `001`
- Otherwise: increment from the highest existing number

```bash
mkdir -p {verifications-dir}/{context}/{NNN}_{slug}/
```

### Step 3: Execute Verification and Capture Evidence

For each checklist item:

1. **Navigate** to the relevant page/state
2. **Observe** using `read_page` or `computer screenshot` — these are inline-only (not saved to disk) but help you assess the state
3. **Interact** if the check requires it (click buttons, fill forms, hover, etc.)
4. **Record a GIF** capturing the check — this is the persistable evidence
5. **Move the GIF** from `~/Downloads/` to the verification directory
6. **Assess**: Pass / Fail / Partial
7. **Note** any unexpected behavior, visual bugs, or deviations

**GIF capture details**:

Every check must produce at least one GIF artifact. Use `mcp__claude-in-chrome__gif_creator` with a `name` matching the target filename (e.g., `01_form-initial-state`).

- **Static checks**: Record a short GIF (a few frames) showing the relevant state
- **Interaction flows**: Record a longer GIF covering the full interaction. Capture extra frames before and after actions for smooth playback.
- **Before/after**: When the change is the interesting part, record separate GIFs for before and after states

After each GIF recording, immediately move it to the verification directory:

```bash
mv ~/Downloads/{name}.gif {verification-dir}/{NN}_{check-slug}.gif
```

If the file isn't found in `~/Downloads/`, check the browser's configured downloads directory.

**Important**:
- If a check requires authentication or specific data state, ask the user for guidance
- If a page fails to load or behaves unexpectedly, record the error state as evidence

### Step 4: Write Report and Summarise

**After all checks are complete**, verify all expected artifacts exist:

```bash
ls -la {verification-dir}/
```

If no GIFs were persisted to disk, **this is a failure** — report it clearly so the invocation method can be adjusted.

**Write `verification-report.md`** in the verification directory:

```markdown
# UI Verification Report

**Date**: {timestamp}
**Context**: {context, e.g., "Step 3: Add preferences form"}
**Run**: {context}/#{NNN}
**Ticket**: {TICKET-ID} (if available)
**URL**: {application URL tested}

## Result: {✅ All checks passed / ⚠️ Partial — {N} of {M} passed / ❌ Failures found}

## Checks

### 1. {Check description}
- **Status**: ✅ Pass / ❌ Fail / ⚠️ Partial
- **Evidence**: [{filename}](./{filename})
- **Notes**: {What was observed, any deviations}

### 2. {Check description}
- **Status**: ✅ Pass / ❌ Fail / ⚠️ Partial
- **Evidence**: [{filename}](./{filename})
- **Notes**: {What was observed}

## Issues

| # | Issue | Severity | Evidence |
|---|-------|----------|----------|
| 1 | {Description} | {Blocker/Major/Minor/Visual} | [{file}](./{file}) |

## Recommendations

- {Specific fix suggestion with file/component reference, or "All checks passed — verification complete"}
```

**Output a summary to the conversation**:

```
## UI Verification: {context} — #{NNN} {slug}

**Result**: {✅ All passed / ⚠️ {N} of {M} passed / ❌ Failures found}

{One-line summary}

| Check | Status | Evidence |
|-------|--------|----------|
| {check 1} | ✅ / ❌ / ⚠️ | {filename} |
| {check 2} | ✅ / ❌ / ⚠️ | {filename} |

Report: `verifications/{context}/{NNN}_{slug}/verification-report.md`
```

## Verification Strategies by UI Type

**Forms**: Fill with valid data → submit → verify success state. Then test validation: submit empty, submit invalid → verify error messages.

**Navigation/Routing**: Click links/buttons → verify correct page loads, URL updates, breadcrumbs.

**Lists/Tables**: Verify items render, pagination works, sorting/filtering if applicable.

**Modals/Dialogs**: Trigger open → verify content → close → verify dismissed. Avoid native browser dialogs (alert/confirm/prompt) — if suspected, use JavaScript to check first.

**Responsive**: If specified, check at different viewport widths using `mcp__claude-in-chrome__resize_window`.

**Loading/Async**: Verify loading states, then loaded states. Capture as GIF to show the transition.

**Error States**: Trigger errors (network, validation, 404) → verify error UI renders correctly.

## Guidelines

1. **Don't guess what to verify**: The caller provides the checks. If unclear, ask.
2. **Don't guess the URL**: If not provided, ask.
3. **One GIF per check minimum**: Every check needs a persisted artifact, not just an inline screenshot.
4. **Name files descriptively**: `03_form-validation-error.gif` not `recording3.gif`
5. **Be honest about failures**: A clear fail report is more valuable than a vague pass.
6. **Don't ask for confirmation when input is complete**: If the caller provided checks and a URL, execute immediately.

## Error Handling

- **No verification instructions provided**: Ask the user what to verify — do not attempt to derive checks from workspace artifacts
- **claude-in-chrome tools error**: Report the error and stop. Do not fall back to chrome-devtools-mcp or other browser tools.
- **Page not loading**: Record the error state, report it, ask user to verify the app is running
- **Authentication required**: Ask user for credentials or to log in manually, then continue
- **Element not found**: Record current page state as GIF, report as failure with evidence
- **GIF not found in Downloads after recording**: Check browser's configured downloads directory. If still not found, note as artifact persistence failure in the report.
- **No workspace found**: Store artifacts in `.ai-workspace/verifications/{context}/{NNN}_{slug}/` as fallback
