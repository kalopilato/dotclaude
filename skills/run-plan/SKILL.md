---
name: run-plan
description: Orchestrate plan execution with checkpoint mode. Loops over implementation-plan.md, detecting human corrections, invoking execute-step for each step, committing at group boundaries, and pausing or continuing based on [auto]/[review] classification.
---

## Configuration

**Workspace Directory**: `.ai-workspace/`

All artifacts are stored in `.ai-workspace/{TICKET-ID}_{slugified-title}/`

---

`run-plan` is an inline orchestrator for checkpoint-mode execution. It loops over the implementation plan, invoking `execute-step` for each step, detecting human corrections between invocations, committing at group boundaries, and either continuing automatically (`[auto]`) or pausing for review (`[review]`).

It does not implement steps itself — all implementation work is delegated to `execute-step`.

**To resume after any pause:** invoke `run-plan` again. It reads the current plan and git state, picks up where it left off, and the correction check captures anything changed during review.

---

## Inputs

| Input                    | Where to find it                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `implementation-plan.md` | Ticket workspace — must exist before run-plan is invoked                                 |
| `correction-log.md`      | Ticket workspace — created by run-plan on first correction; absent if no corrections yet |
| Git history              | Via `git diff HEAD` and `git log`                                                        |

---

## Precondition check

At the start of every invocation, before anything else:

1. Look for `implementation-plan.md` in the ticket workspace. If not found: halt with the message:

   > "implementation-plan.md is missing or was created before v2.0.0 — run `plan-ticket` first, or add a `**Last agent commit:** (none yet)` line to the top of the plan."

2. Check that `implementation-plan.md` contains a `**Last agent commit:**` header line. If absent: halt with the same message above.

If both checks pass, proceed to the execution loop.

---

## Execution loop

The loop runs the following phases in sequence on each iteration. It stops when:

- Implementation is complete (no `[TODO]` steps remain)
- A `[review]` boundary is reached
- A step fails after retry (execute-step halted)
- A group-level test run fails

---

### Phase 1 — Correction check

Runs at the start of every loop iteration, before invoking `execute-step`.

**Step 1:** Run `git diff HEAD`. Any output means uncommitted changes exist.

**Step 2:** Read `**Last agent commit:**` from `implementation-plan.md`. If the value is `(none yet)`, skip this step entirely. Otherwise run `git log --oneline <hash>..HEAD`. Any commits listed are ones the orchestrator did not make — these are human corrections.

**If corrections are found** (uncommitted changes or human-authored commits since the last agent commit):

1. Analyze the diff: what changed, in which files
2. Assign a category: `naming | convention | approach | missing-test | missing-edge-case | architecture | other`
3. Assign severity: `trivial | minor | significant`
4. **If the reason is clear** from the diff or conversation context: log without asking
5. **If ambiguous:** ask exactly one short question, then halt with the message:

   > "Invoke `run-plan` to resume once you have answered the question above."

   On the next invocation, the human's answer will be in conversation context — use it to populate the inferred reason before logging.

6. Append an entry to `correction-log.md` in the ticket workspace (see format below)
7. Stage and commit:

   ```
   git add -A
   git commit -m "fix: corrections to [brief description]

   [what was corrected and why]"
   ```

8. Update `**Last agent commit:**` in `implementation-plan.md` to this new commit hash

**If no corrections are found:** proceed to Phase 2.

---

### Phase 2 — Check for remaining work

Read `implementation-plan.md`. If no `[TODO]` steps remain: report to the human that implementation is complete and stop. Do not proceed to Phase 3.

---

### Phase 3 — Identify the current step and group position

Read the next `[TODO]` step and determine:

1. **Group position:**
   - **Last step in group:** The next `**Commit**: After this step` line immediately follows this step with no intervening `[TODO]` steps
   - **Mid-group step:** One or more `[TODO]` steps exist between this step and the next `After this step` boundary

2. **Group classification:** Read `[auto]` or `[review]` from the `After this step` line that closes this group. Carry this forward to Phase 5 and group boundary handling.

---

### Phase 4 — Invoke execute-step

Invoke the `execute-step` skill. It handles all implementation work for the current step: marking it `[IN_PROGRESS]`, writing code and tests, running the test suite, marking it `[DONE]`, and annotating the plan.

After `execute-step` completes, read `implementation-plan.md` to check the outcome:

- **If the step is `[DONE]`:** proceed to Phase 5.
- **If the step is still `[IN_PROGRESS]`:** execute-step halted due to a test failure. Stop with the message:
  > "Step halted due to test failure. Fix the issue and invoke `run-plan` to resume."

---

### Phase 5 — Determine next action

Use the group position determined in Phase 3:

- **Mid-group step:** Loop back to Phase 1. The correction check will detect anything changed since execute-step ran, then the next step will be executed.
- **Last step in group:** Proceed to group boundary handling.

---

## Group boundary handling

Only reached when the current step was the last in its commit group.

1. Run the full test suite to confirm all steps in the group pass together
2. **If failing:** stop with the message:

   > "Group-level test run failed — individual steps passed but the group does not. Fix the failures and invoke `run-plan` to resume."

   Do not commit.

3. **If passing:** use the commit message suggested by execute-step in its output for the last step in the group as a starting point. If the group spanned multiple steps and the commit message doesn't cover them all, broaden the subject line to reflect the combined scope. Follow the project's commit conventions (`git log --oneline` to check).

4. Record the commit hash on the `After this step` line in `implementation-plan.md`:

   ```
   **Commit**: After this step [auto] — reason (abc1234)
   ```

5. Update `**Last agent commit:** abc1234` in the plan header

6. Check whether any `[TODO]` steps remain:
   - If none remain: report that implementation is complete. Stop.

7. Act on the group's classification:
   - **`[auto]`:** Loop back to Phase 1. Continue without pausing.
   - **`[review]`:** Present a summary of what the group implemented. Include the commit hash so the human can run `git show <hash>` or review the diff in their editor. Then stop with the message:
     > "Review complete — invoke `run-plan` to continue."

---

## Correction log format

File: `.ai-workspace/{TICKET-ID}_{slug}/correction-log.md`

Append-only. Created on first correction; not created if no corrections occur.

```markdown
# Correction Log — {TICKET-ID}

## Correction {N} — Step: {step number or description}

**Detected at:** Start of loop iteration before Step {next step number}
**Agent commit:** {hash of last agent commit before the correction}
**Correction commit:** {hash assigned to this correction commit}
**Files changed:** {list of files}
**Category:** {naming | convention | approach | missing-test | missing-edge-case | architecture | other}
**Severity:** {trivial | minor | significant}

**What the agent produced:** {brief description}
**What was corrected:** {brief description}
**Inferred reason:** {from diff analysis, conversation context, or human's answer}

---
```
