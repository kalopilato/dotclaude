# The Thinking Behind This Workflow

This document captures the philosophy, principles, and lessons learned from around a year of daily AI-assisted engineering. If you just want to try out the workflow, see the [README](../README.md). If you want to understand why it's shaped this way and where it's heading, read on.

---

## The Core Insight

After a year of iterating on AI-assisted workflows, the biggest realization is this: **implementation was never the bottleneck**.

The hard parts of software engineering are:

- Deciding what to build
- Specifying it precisely enough that the result is predictable
- Knowing whether you got it right

AI is very very good at generating code. What it can't do (yet) is know your requirements better than you do, or verify that the result is correct without clear criteria to check against.

This workflow is built around that thought. The human focuses on specification and judgment, AI handles implementation, and artifacts help to align the two.

---

## Principles

The following principles emerged from what worked (and what didn't) over many iterations.

### Specification is the product

The quality of AI output is a function of the quality of the input. Time spent on clear requirements isn't overhead, it's the actual work.

AI is fast but that doesn't change the fact that vague instructions will produce vague results, you'll just get there faster. Well specified tickets with explicit acceptance criteria are more likely to produce good results.

This is why the workflow front-loads analysis and planning. `/analyze-ticket` extracts requirements and surfaces gaps and `/plan-ticket` creates explicit steps with verification criteria. By the time implementation starts the hardest thinking is done.

### Verification builds trust

The path to more autonomous AI execution isn't hope, it's confidence earned through verification.

Right now I review every step. That's appropriate for a workflow I'm still learning to trust, but each step that passes review is evidence. Over time that evidence accumulates into confidence that certain types of changes, in certain contexts, can run with lighter oversight.

The workflow is designed to make this progression possible. Explicit review points create opportunities to verify, artifacts create an audit trail, and over time the review points can widen. Not because I've arbitrarily decided to trust AI more, but because the evidence supports it.

### Artifacts over memory

Conversations compact and sessions end but files in the workspace persist.

Requirements analysis, implementation plans, questions, and decisions are written to disk. If a conversation gets too long and needs to restart, the artifacts remain. If I put down a ticket Monday and return Wednesday, the workspace has everything needed to resume.

But persistence isn't the only point, it's also about review-ability. I can read the implementation plan before the agent starts writing code. I can see what questions were raised and how they were answered. I can retrospectively look at where things went wrong and why. The thinking is visible, not just the output.

### Conversation as context

A single conversation accumulates understanding as the work progresses. An agent learns the codebase quirks, the decisions made, the false starts abandoned; this context is valuable and can't be fully reconstructed from artifacts alone.

Splitting work across disconnected sessions loses this. Starting fresh means re-explaining constraints, re-discovering patterns and re-learning what was already figured out.

This workflow keeps ticket work in a single conversation where possible, letting context build and course corrections happen naturally. The agent's understanding of the problem deepens as implementation progresses.

### Human-in-the-loop, not human-in-the-way

Each command is an explicit invocation, the workflow pauses for review between steps.

This is deliberate friction but the goal isn't to stay in the loop forever, it's to build enough confidence in verification that the loop can widen.

Today: review every step.
Tomorrow: review every checkpoint.
Eventually: review only the outcome.

The friction should decrease as trust increase - this workflow is designed to support that progression, not prevent it.

### Portable by default

Tools change, products change, organizations change. What shouldn't change is the workflow's core logic.

Commands describe intent, not implementation. "Fetch the ticket" rather than "call the Linear API." "Create a pull request" rather than "run gh pr create."

The ticket system, git provider, and project conventions are discovered at runtime or injected via configuration. The same workflow works at different companies, with different tools, in different codebases, without rewriting the commands.

---

## What I've Learned

### The 2-3x efficiency gain is real, but conditional

On well-specified tickets in familiar codebases, I consistently see 2-3x efficiency improvement. Agents handle the mechanical work while I focus on decisions and review.

But poorly specified tickets don't get faster, they just fail faster, and unfamiliar territory (new codebase, new patterns, uncertain requirements) benefits less from AI assistance because the human judgment parts dominate.

The efficiency gain is real when the preconditions are met - this workflow is partly about ensuring those preconditions exist.

### Small blast radius beats big ambition

Early on I tried to give the AI more rope. "Implement this whole feature", "Figure out the architecture and build it", etc. It didn't go well. AI would make reasonable-looking decisions that compounded into something obviously or, worse yet, subtly wrong. By the time I reviewed the mess was too big to unpick cleanly.

Step-by-step execution with review between steps catches problems when they're small. It's slower in theory but faster in practice because recovery is cheap.

### Course correction matters just as much as perfect plans

Implementation plans are hypotheses. In practice, as we make changes, the codebase reveals truths the plan didn't anticipate.

The workflow supports course correction: plans are updated as implementation progresses, decisions are documented when they diverge from the original approach. The goal isn't to rigidly follow the plan, it's to end up at the right destination while maintaining a record of how you got there.

### Sub-agents are tricky

I've experimented with spawning sub-agents for specialized tasks. The challenge: they start with empty context, and when they complete, their context is lost.

If a sub-agent produces good output, great. If it produces something that needs iteration, you're stuck. You can't work with the sub-agent to refine its output because it's gone.

This is solvable (have sub-agents write their reasoning to artifacts, spawn new instances with that context), but it's friction. For now, I mostly keep work in the main conversation with artifacts making it easier to `/compact` or `/clear` as necessary.

### Parallel execution is less useful than I expected

I spent time thinking about parallelizing work - running multiple agents on different parts of a ticket simultaneously. However, most tickets are inherently sequential once you're inside them. Step 2 depends on step 1, the component needs the types, the test needs the implementation.

The real opportunity for parallelism is across tickets, not within them. And even then, the cognitive overhead of tracking multiple streams often exceeds the time saved.

The better optimization is reducing idle time - working on something else while an agent runs, rather than watching it. That's a workflow and tooling problem, not a parallelization problem.

---

## Where This Is Heading

The current workflow is solid but relies heavily on manual orchestration. The next iteration focuses on **earned autonomy through verification**.

### Verification agents

After each implementation step, specialized sub-agents review the work:

- **Requirements verifier**: Does this implementation satisfy the requirements?
- **Standards verifier**: Does this follow project patterns and coding standards?
- **Adversarial verifier**: How could this break? Edge cases? Security issues?

If all pass, continue. If any raise concerns, attempt correction before surfacing for human review.

The goal: human review shifts from "check everything" to "check the things verification couldn't reliably catch."

### Execution logging

As work progresses, decisions and course corrections are logged to a structured file. What changed from the plan? Why? What did verification catch?

At completion, a drift summary would make end-of-work review faster. Instead of reading every line of diff, I can see "plan deviated at step 3 because X, verification caught issue Y which was corrected."

### Tiered execution modes

Not all tickets need the same ceremony:

- **Stepwise**: Current default. Human invokes each step, reviews each result.
- **Checkpoint**: Steps marked `[auto]` run in sequence, pause at steps marked `[review]`.
- **Autonomous**: Full implementation with verification at each step. Human reviews only the final result.

The right mode depends on the ticket's risk profile, the codebase familiarity, and accumulated trust.

### Reduced orchestration

Some manual steps exist only because I haven't collapsed them yet. For example, setup commands (kickoff, analyze, prime-context) have no decision point between them. They should be one command.

The goal: fewer manual invocations without sacrificing quality. Typing commands isn't where I add value.

---

## Values

These inform not just what the workflow does, but how it evolves:

**Sustainability over speed.** Efficiency gains should reduce cognitive load, not increase it. A workflow that's faster but exhausting isn't better. I optimize for value per unit of energy, not maximum output.

**Confidence over hope.** Don't trust AI output because it looks right, trust it because verification passed. Optimism isn't a testing strategy.

**Transparency over magic.** The workflow should be understandable, artifacts should be readable, decisions should be traceable. If something goes wrong, the debugging path should be clear.

**Adaptability over prescription.** Describe what to achieve, not how. Let the AI discover the project's patterns rather than imposing a one-size-fits-all approach. The same workflow should work across different stacks, tools, and team conventions.

**Progress over perfection.** This workflow is a work in progress. It's better to ship something useful and iterate than to wait for the perfect system. The current version has rough edges. That's fine - they'll get smoother.

---

## Questions I'm Thinking About

- Where exactly is the line between "agent can handle this" and "human judgment required"? It's contextual, but can it be made more systematic?

- How do you maintain quality when verification passes but the output is subtly wrong in ways verification couldn't catch?

- What's the right interface for monitoring multiple agent sessions without overwhelming cognitive load?

- How much of this workflow is specific to my working style vs. generalizable to others?

---

## Feedback Welcome

This is shared in the spirit of "here's what I've learned, take what's useful." If you try it and something doesn't work, I want to know. If you extend it in interesting ways, I want to see.

The repo is at [github.com/kalopilato/dotclaude](https://github.com/kalopilato/dotclaude).
