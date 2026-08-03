---
name: opencode-foreman
description: Use when delegating implementation work to a cheaper executor model - dispatches self-contained coding tasks to OpenCode sessions via foreman, isolated per git worktree, then reviews the diff before anything lands
---

# Delegating implementation to OpenCode

You keep the plan and the review. OpenCode's executor model does the typing.
Work lands only after you have read the diff.

Delegate when the task is **mechanical and self-contained**: a scripted
transformation, tests for an existing module, a bounded refactor, bulk
classification, boilerplate. Do it yourself when the task needs cross-file
reasoning, novel architecture, or touches anything irreversible.

## Preflight

```bash
foreman up          # idempotent; prints the running version
```

If `foreman` is missing, say so and stop — do not fall back to editing the
files yourself without telling the user.

## The loop

**1. Write a self-contained prompt.** The executor gets none of this
conversation. State the file paths, the expected behaviour, and how it can
verify itself. Vague prompts are the main cause of unusable output.

**2. Dispatch.** Always isolate, always deny by default:

```bash
foreman dispatch --worktree --deny '<paths that must not change>' \
  --profile pro "<the full task>"
```

- `--profile pro` for implementation, `--profile flash` for search,
  classification, or high-volume mechanical passes.
- `--allow-bash` only when the task genuinely needs to run tests or tooling.
  It is off by default.
- Add `--deny` for every path where a wrong edit would be expensive. The rule
  is enforced by the server, and `dispatch` prints the ruleset actually in
  force — read that line back and confirm it covers what you intended.

**3. Do other work while it runs.** Dispatch returns immediately. Fan out with
a loop when tasks are independent; each `--worktree` gets its own branch.

**4. Collect.**

```bash
foreman ls              # running / starting / idle
foreman wait <id>       # or block
foreman result <id>     # the executor's own summary — a claim, not evidence
foreman diff <id>       # the actual change
```

If `wait` exits 3, a permission prompt is blocking it: `foreman perms <id>`,
then `foreman reply <id> <requestID> once|reject`.

**5. Review the diff yourself.** This is the point of the whole arrangement.
`foreman result` is the executor asserting it succeeded; `foreman diff` is what
it did. Check them against each other. Run the tests yourself.

**6. Land or discard.**

- Good: merge the `foreman/<name>` branch, then `foreman clean <id>`.
- Bad: `foreman clean <id>` throws the worktree and branch away. Prefer
  re-dispatching with a sharper prompt over hand-patching a bad result — if
  the prompt was wrong, fixing the output leaves the prompt wrong.

## Reporting back

Tell the user which model ran it, what the diff actually contains, and what you
verified. If you did not run the tests, say that. Never present the executor's
"Done." as confirmation that the change is correct.
