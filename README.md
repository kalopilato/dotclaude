# dotclaude

A portable collection of Claude Code commands and agents for AI-assisted software engineering. Implements a structured ticket-to-change-request workflow that adapts to any tech stack, ticket system, or git host.

---

## Philosophy

**Intent-based, not prescriptive.** Commands describe *what* to achieve, not *how*. The AI discovers your project's tools, patterns, and conventions at runtime from the project's own `CLAUDE.md` and codebase — so the same workflow adapts to a Rails monolith, a Go microservice, or a TypeScript frontend without reconfiguration.

**Human in the loop.** Each command is an explicit invocation. The implementation pipeline waits for human review between steps — the AI does the work, you decide when to continue.

**Conversation as glue.** Ticket data and requirements flow through conversation context across pipeline steps, avoiding redundant fetches. Workspace files persist state across sessions so work can be resumed in a new conversation without losing progress.

**Tool-agnostic.** Ticket system, git CLI, and workspace location are configured in a gitignored `local/tools.md` — keeping the workflow portable across teams and setups.

---

## Workflow

```mermaid
flowchart LR
    PC("/prime-context")
    KT("/kickoff-ticket")
    AT("/analyze-ticket")
    PT("/plan-ticket")
    ES("/execute-step")
    RC("/review-code")
    CRW(["change-request-writer\nagent"])

    PC -.->|"run in project first"| KT
    KT --> AT
    AT --> PT
    PT --> ES
    ES -->|"repeat per step"| ES
    ES --> RC
    RC -->|"fix & repeat"| RC
    RC --> CRW
```

### Commands

| Command | What it does | Creates |
|---------|-------------|---------|
| `/prime-context` | Load project structure and docs into context | — |
| `/kickoff-ticket` | Fetch ticket, create workspace directory | `ticket-info.md` |
| `/analyze-ticket` | Extract requirements, assess ticket readiness, identify blockers | `requirements-analysis.md`, `questions.md` |
| `/plan-ticket` | Create step-by-step implementation plan | `implementation-plan.md` |
| `/execute-step` | Implement one step: code + tests + verification | progress in `implementation-plan.md` |
| `/review-code` | Self or peer code review against ticket requirements | — |

### Agent

| Agent | What it does | Creates |
|-------|-------------|---------|
| `change-request-writer` | Draft change request description from git diff and ticket context | `change-request-draft.md` |

### Typical session

```
/prime-context                  # Load project structure
/kickoff-ticket PROJ-142        # Fetch ticket, create workspace
/analyze-ticket                 # Extract requirements, flag gaps
# → answer any blocker questions
/plan-ticket                    # Create implementation plan
# → review plan, refine if needed
/execute-step                   # Implement step 1
# → review changes, iterate
/execute-step                   # Implement step 2
# → ... repeat until all steps done
/review-code PROJ-142           # Self-review before submitting
# → fix any issues, re-review
# → run change-request-writer agent to draft description
```

---

## Setup

### Fresh install

If you don't have an existing `~/.claude/` directory:

```bash
git clone <repo-url> ~/.claude
```

Claude Code loads commands and agents from `~/.claude/` automatically.

### Merge into an existing `~/.claude/`

If you already manage `~/.claude/` as a git repository with your own configuration:

```bash
cd ~/.claude
git remote add dotclaude <repo-url>
git fetch dotclaude
git merge dotclaude/main --allow-unrelated-histories
```

Pull future updates:

```bash
git fetch dotclaude
git merge dotclaude/main
```

Resolve any merge conflicts by keeping your local customisations where they diverge.

### Global gitignore

Workspace files are generated per-project and shouldn't be committed to your projects. Add `.ai-workspace/` to your global gitignore:

```bash
echo ".ai-workspace/" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

---

## Configuration

### 1. Copy the tools template

```bash
cp ~/.claude/local/tools.md.template ~/.claude/local/tools.md
```

### 2. Fill in your tools

Edit `local/tools.md` with your actual setup:

```markdown
## Ticket System
- Tool: Linear MCP
- MCP fetch command: `mcp__linear-server__get_issue`
- MCP comments command: `mcp__linear-server__list_comments`
- ID format: PB-752

## Workspace (optional)
- Directory: tmp    ← override default (.ai-workspace/)
```

The git provider CLI (`gh` or `glab`) is auto-detected from `git remote get-url origin`. Override only if you're using an alternative provider or self-hosted instance.

`local/tools.md` is gitignored — your credentials and org-specific config stay out of the shared repo.

---

## Directory structure

```
~/.claude/
├── CLAUDE.md                       # Root config: workspace dir, git provider detection
├── local/
│   ├── tools.md.template           # Copy this and fill in your tools
│   └── tools.md                    # Your config (gitignored)
├── commands/
│   ├── prime-context.md
│   ├── kickoff-ticket.md
│   ├── analyze-ticket.md
│   ├── plan-ticket.md
│   ├── execute-step.md
│   └── review-code.md
└── agents/
    └── change-request-writer.md
```

### Workspace anatomy

Each ticket creates a directory in `.ai-workspace/` inside your project:

```
.ai-workspace/
└── PROJ-142_add-email-notification-preferences/
    ├── ticket-info.md              # Ticket details (for session resume)
    ├── requirements-analysis.md   # Requirements + readiness assessment
    ├── questions.md                # Blocker questions (if any)
    ├── implementation-plan.md      # Step-by-step plan with [TODO]/[DONE] tracking
    └── change-request-draft.md    # Generated change request description
```

These files double as session resume artifacts — if you start a ticket on Monday and return Wednesday in a new conversation, `/execute-step` reads them to restore context automatically.
