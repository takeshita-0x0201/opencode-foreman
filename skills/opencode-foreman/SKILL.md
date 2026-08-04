---
name: opencode-foreman
description: Use when acting as PM over implementation work - hardens a request into a spec with executable acceptance criteria, dispatches it to an OpenCode Go model, verifies the result against those criteria rather than trusting the executor, and escalates by ladder when the model was the limit
---

# Running implementation as PM

You hold the plan, the spec, and the verdict. An OpenCode Go model writes the
code. Nothing lands on a real branch until you have checked it yourself.

The discipline that makes this work is **the spec, not the model**. A cheap
model executing a tight spec beats an expensive one guessing at a vague one.

## Step 1 — harden the requirement before dispatching

Do not dispatch a request in the shape the user said it. Convert it first. A
task is ready when you can answer all four:

1. **Scope** — exactly which files may change, and which must not.
2. **Behaviour** — what the code must do, in specifics, not adjectives.
3. **Acceptance** — a command that exits 0 iff the task succeeded. If you
   cannot write one, the requirement is still too vague; keep going.
4. **Context** — the executor sees none of this conversation. Anything it
   needs (existing function signatures, file layout, conventions) goes in the
   prompt.

If the user's request cannot survive this, that is a finding. Bring the
ambiguity back to them instead of dispatching a guess.

Track the tasks with TodoWrite so the fan-out stays visible.

## Step 2 — dispatch with the criteria attached

```bash
foreman dispatch --worktree --profile pro \
  --deny 'apps/worker/**' \
  --check 'test -f src/retry.py' \
  --check 'python3 -m pytest tests/test_retry.py -q' \
  "<the hardened spec>"
```

- `--check` is repeatable and is the whole point. `dispatch` warns on stderr
  when a task has none — treat that warning as a bug in your own spec, not as
  noise.
- `--deny` every path where a wrong edit is expensive. Read back the `denied`
  line it prints and confirm it says what you meant.
- `--worktree` always, for anything touching a real repo.
- `--allow-bash` only when the task must run its own tests.
- Rungs: `flash` (bulk, mechanical, search) → `pro` (default) → `max`
  (hardest). `--model opencode-go/<id>` pins any of the 18 models directly;
  `opencode models` lists them.

Independent tasks are a plain loop — dispatch returns immediately.

## Step 3 — verify, do not believe

```bash
foreman wait <id>
foreman verify <id>    # runs the checks; non-zero if any fail
foreman diff <id>      # read this even when the checks pass
```

`foreman result` is the executor asserting it succeeded. `foreman verify` is
evidence. They disagree more often than you would like — a model that says
"Done." while a check fails is the normal case this workflow exists to catch.

Read the diff even on a green run: checks prove the criteria you thought of,
not the ones you missed.

## Step 4 — diagnose before escalating

When `verify` fails, the failure is one of two kinds, and they need opposite
responses:

- **Spec defect** — the criteria demanded something the prompt never asked
  for, context was missing, scope was ambiguous. **Fix the brief and dispatch
  again at the same rung.** Escalating will reproduce the same failure at
  higher cost; a stronger model cannot read a requirement that is not there.
- **Capability limit** — the spec was complete and the model still could not
  do it. Then, and only then, re-dispatch the same brief at the next rung.

Step up at most once per task. If `max` fails on a complete spec, the task is
mis-decomposed — take it back and split it, or implement it yourself.

## Step 5 — land or discard

- Pass: merge `foreman/<name>`, then `foreman clean <id>`.
- Fail: `foreman clean <id>` discards the worktree and branch. Prefer
  re-dispatching over hand-patching — patching the output leaves the defective
  spec in place, and you will hit it again next task.

## Reporting

Say which model ran it, what `verify` returned, and what you read in the diff.
If a check was skipped or a criterion was untestable, say that explicitly.
Never pass the executor's "Done." along as confirmation.
