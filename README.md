# dotclaude

My current, (mostly) portable Claude-assisted engineering workflow to help turn tickets into pull requests based around a simple idea: implementation isn't the bottleneck - specification and verification are. That's where my attention should be.

**[Full philosophy and lessons learned →](docs/philosophy.md)**

---

## Principles (the short version)

- **Specification is the product** — Clear requirements in, good results out
- **Verification builds trust** — AI earns autonomy, we don't trust it by default
- **Artifacts over memory** — Context is hard to manage and hard to reason about, files on disk endure
- **Human-in-the-loop (for now)** — Review agent output at checkpoints, widen the checkpoints as confidence grows
- **Portable by default** — Discover tools at runtime, don't bake them into the workflow

---

## Skills

| Skill | Purpose | Output |
| --- | --- | --- |
| `start-ticket` | Fetch ticket, create workspace, analyze requirements, prime context | `ticket-info.md`, `requirements-analysis.md`, `questions.md` |
| `plan-ticket` | Create step-by-step implementation plan | `implementation-plan.md` |
| `execute-step` | Implement one step with verification | Updates `implementation-plan.md` |
| `review-code` | Self-review against requirements and standards | `code-review.md` |
| `change-request-writer` | Draft PR/MR description from diff and ticket context | `change-request-draft.md` |
| `prime-context` | Load project structure and docs into context | — |
| `analyze-ticket` | Extract requirements, surface gaps and blockers | `requirements-analysis.md`, `questions.md` |

---

## Workflow

```mermaid
flowchart TD
    ST(["start-ticket"])
    PT(["plan-ticket"])
    ES(["execute-step"])
    RC(["review-code"])
    CRW(["change-request-writer"])

    U1["review analysis · answer questions"]
    U2["review plan · iterate if needed"]
    U3["review changes · iterate if needed"]
    U4["fix issues"]

    D1["ticket-info.md · requirements-analysis.md · questions.md"]
    D2["implementation-plan.md"]
    D3["code-review.md"]
    D4["change-request-draft.md"]

    ST --> U1
    U1 --> PT
    PT --> U2
    U2 --> ES
    ES --> U3
    U3 -->|"next step"| ES
    U3 -->|"steps completed"| RC
    RC --> U4
    U4 -->|"re-review"| RC
    RC -.-> CRW

    ST -.-> D1
    PT -.-> D2
    ES -. updates .-> D2
    RC -.-> D3
    CRW -.-> D4

    classDef skill fill:#dbeafe,stroke:#3b82f6,color:#1e3a5f
    classDef user fill:#f3f4f6,stroke:#9ca3af,color:#374151
    classDef doc fill:#fef9c3,stroke:#ca8a04,color:#713f12
    class ST,PT,ES,RC,CRW skill
    class U1,U2,U3,U4 user
    class D1,D2,D3,D4 doc
```

## Typical session

```bash
# Start ticket: fetches ticket, creates workspace, analyzes requirements, primes context
start-ticket PROJ-142

# → review analysis, answer any questions, add any context missing from the ticket

plan-ticket                     # Create implementation plan

# → review plan, refine if needed, iterate until it's solid

execute-step                    # Implement step 1

# → review, iterate if needed, skill updates plan as we go

execute-step                    # Implement step 2 ...

# → review, iterate if needed, skill updates plan as we go

# → repeat until complete

review-code                     # Self-review before PR

# → invoke change-request-writer
```

## Generated workspace structure

Each ticket creates a workspace directory in your project, with artifacts generated as we progress through the workflow:

```text
.ai-workspace/
└── PROJ-142_add-email-preferences/
    ├── ticket-info.md              # Ticket details (for session resume)
    ├── requirements-analysis.md    # Requirements + readiness assessment
    ├── questions.md                # Blockers and open questions
    ├── related-tickets.md          # Related/dependent tickets (if any)
    ├── implementation-plan.md      # Steps with [TODO]/[DONE] tracking and evaluation of magnitude/complexity
    └── change-request-draft.md     # Generated PR description
```

These files enable session resume - return to a ticket in a new conversation and `execute-step` can pick up where you left off.

---

## Setup

Copy the files into your user-level Claude configuration:

```bash
# Clone the repo
git clone https://github.com/kalopilato/dotclaude.git
cd dotclaude

# Copy to your Claude config
cp -r skills ~/.claude/
cp CLAUDE.md ~/.claude/
```

Skills are available in any new conversation.

### Prerequisites

- **Ticket system**: An MCP connection to your ticket system (Linear, Jira, GitHub Issues, etc.)
- **Git CLI**: `gh` (GitHub) or `glab` (GitLab) for PR/MR operations

### Global gitignore

Workspace files shouldn't be committed to your projects:

```bash
echo ".ai-workspace/" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

---

## Current status

**What works:**

- Core workflow is effective - has been my daily driver for around a year now (though it has iteratively improved over that time)
- I estimate approximately 2-3x efficiency gains on well-specified tickets
- Single-conversation context accumulation maintains the level of context I want as I move through the workflow
- Artifacts for persistence and review-ability work well - I'm no longer concerned about what I'll lose if I `/compact` or `/clear`

**What's rough:**

- Manual invocation at each step - this is a presentation of **Verification builds trust** and is reducing over time
- No automated verification layer - I see this as key to increased autonomy and efficiency, but non-trivial to do well
- Some tool-specific references need abstraction

---

## Roadmap

**Next iteration:**

- Verification agents (requirements, standards, adversarial review)
- Execution logging with drift summary
- Clearer documentation of decisions made during execution (for better review and session resume)
- Auto-correction loop before surfacing for human review
- Tiered execution modes (stepwise → checkpoint → autonomous)

For more thinking on where this is heading, see [docs/philosophy.md](docs/philosophy.md).

---

## Claude Directory structure

```text
~/.claude/
├── CLAUDE.md                       # Workspace directory config
└── skills/
    ├── start-ticket/
    │   └── SKILL.md
    ├── analyze-ticket/
    │   └── SKILL.md
    ├── plan-ticket/
    │   └── SKILL.md
    ├── execute-step/
    │   └── SKILL.md
    ├── review-code/
    │   └── SKILL.md
    ├── change-request-writer/
    │   └── SKILL.md
    ├── prime-context/
    │   └── SKILL.md
    └── verify-ui/
        └── SKILL.md
```

---

## Contributing

This is a personal workflow shared in case it's useful. If you try it and something doesn't work, or you extend it in interesting ways, I'd love to hear about it.

---

## License

MIT
