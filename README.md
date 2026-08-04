# opencode-foreman

**Your orchestrator keeps the plan. OpenCode does the typing. You review the diff.**

A ~450-line, dependency-free CLI that dispatches implementation work to
[OpenCode](https://opencode.ai) through its HTTP server API — one session per
task, each isolated in its own git worktree, each with server-enforced write
permissions.

Point it at [OpenCode Go](https://opencode.ai/go) and your executor is DeepSeek
V4 (or Kimi, GLM, Qwen, Grok) at subscription prices, while Claude Code, Codex,
or a shell script stays the orchestrator.

```bash
foreman dispatch --worktree --deny 'apps/worker/**' \
  --check 'python3 -m pytest tests/test_retry.py -q' \
  "Add retry with exponential backoff to the HTTP client in src/http.py …"

foreman ls          # t1  running  opencode-go/deepseek-v4-pro  Add retry…
foreman verify t1   # runs the acceptance checks — evidence, not the model's word
foreman diff t1     # review before anything lands on your branch
```

There is no server to start first — `foreman` brings one up the moment it needs
one.

## Why this exists instead of a plugin

Every Claude-Code-to-OpenCode bridge I could find is a wrapper around the same
thing: the HTTP API that `opencode serve` already exposes. That API has 162
endpoints, an OpenAPI 3.1 spec at `/doc`, and primitives the wrappers do not
surface — per-session permission rulesets, revert, session diff, programmatic
permission replies.

So this talks to the API directly. No plugin to go stale, no proxy, no
`ANTHROPIC_BASE_URL` swap, and nothing that takes the orchestrator role away
from the agent you actually trust with design decisions.

## Install

Requires Python 3.10+, git, and `opencode` on your PATH with at least one
provider authenticated (`opencode auth login`).

```bash
git clone https://github.com/takeshita-0x0201/opencode-foreman
ln -s "$PWD/opencode-foreman/foreman" ~/.local/bin/foreman
```

## Do you actually need this?

Often not. `opencode run "..."` already runs the agent in-process — it opens no
socket and needs no server — and for a single sequential task it is the simpler
tool. Measured on a trivial task: `opencode run` ~3.0s, `foreman dispatch
--wait` ~2.7s. **The server buys no speed.**

What it buys is control that `opencode run` has no way to express:

| | `opencode run` | server API |
| --- | --- | --- |
| Per-**task** permission ruleset | ✗ (only `--auto`) | ✓ |
| Project-wide permissions | ✓ via `opencode.json` | ✓ |
| Non-blocking dispatch | ✗ blocks | ✓ `prompt_async` |
| Status / abort a live run | ✗ | ✓ |
| Roll back the agent's edits | ✗ | ✓ `revert` |
| Inspect messages, diff, todo, fork | limited | ✓ |

`opencode.json` can already deny `edit` or `bash` — but for the whole project,
for every session. Running two tasks side by side in one repo where one may
write `apps/web` and the other is read-only requires a ruleset attached to the
session, and that only exists on the API.

So: reach for `opencode run` for one-off work. Reach for this when you are
fanning out tasks with different blast radii and want to review each diff
before it lands.

## Running the server

You never have to think about it. Any command that needs `opencode serve`
starts it on demand and reuses it, so one server covers every task, repo, and
terminal. `foreman down` stops it; `FOREMAN_AUTOSTART=0` turns a missing server
into an error instead of a launch.

If you want it resident across reboots, write a launchd plist or a systemd user
unit for `opencode serve --port 4096` — that is your init system's job, not
this tool's.

## Commands

| Command | What it does |
| --- | --- |
| `foreman dispatch "<task>"` | Create a session and fire `prompt_async`; returns immediately |
| `foreman ls` | All tasks with live `running` / `starting` / `idle` state |
| `foreman wait <id>` | Block until the session goes idle |
| `foreman result <id>` | The executor's final message |
| `foreman verify <id>` | Run the task's acceptance checks; non-zero if any fail |
| `foreman diff <id>` | `git diff` of what it changed |
| `foreman abort <id>` | Stop a running session |
| `foreman clean <id>` | Remove the worktree and forget the task |
| `foreman down` | Stop the server |

That is the whole surface. Everything the OpenCode API can do that this does
not is [documented](docs/api-notes.md) so you can call it directly.

`dispatch` flags worth knowing:

- `--worktree [NAME]` — run in a fresh git worktree on branch `foreman/<name>`.
  Nothing touches your working tree until you decide it should.
- `--deny PATH` (repeatable) — refuse edits under that path, enforced by the
  server, not by prompt wording.
- `--read-only` — analysis only.
- `--allow-bash` — shell is **denied by default**; opt in per task.
- `--check 'CMD'` (repeatable) — an acceptance criterion. Must exit 0.
- `--profile flash|pro|max` — ladder rungs: `deepseek-v4-flash` (bulk,
  mechanical, search) → `deepseek-v4-pro` (default) → `grok-4.5` (hardest).
- `--model provider/model` — pin any of the 18 OpenCode Go models directly;
  `opencode models` lists them.
- `--wait` — dispatch synchronously, print the result, then run the checks.

## Specs, not vibes

A task with no `--check` cannot be verified, so `dispatch` says so on stderr.
The criteria are what let a cheap model be trusted: it is not judgment that
makes the output safe, it is that you can prove the output meets a bar you set
in advance.

```
$ foreman dispatch --check 'test -f calc.py' \
    --check 'python3 -c "import calc; assert calc.factorial(5)==120"' \
    "Create calc.py with a function add(a,b) returning a+b."
[PASS] test -f calc.py
[FAIL] python3 -c "import calc; assert calc.factorial(5)==120"
       AttributeError: module 'calc' has no attribute 'factorial'
1/2 passed
```

Re-dispatching that at `--profile pro` produced the identical failure at higher
cost. The prompt asked for `add`; the criterion demanded `factorial`. **A
stronger model cannot read a requirement that was never written.**

So diagnose before you reach for a bigger rung. A **spec defect** means fix the
brief and re-dispatch where you are. Only a genuine **capability limit** — a
complete spec the model still could not satisfy — is worth `--profile` one step
up.

## Parallelism

`dispatch` returns as soon as the session is created, so fan-out is just a
loop. Measured on OpenCode Go with V4 Flash: three worktree-isolated tasks
dispatched in 2 seconds, all three running concurrently.

```bash
for f in parser writer reporter; do
  foreman dispatch --worktree "$f" --profile flash "Write tests for src/$f.py"
done
foreman ls
```

## Permissions, and a footgun worth knowing about

Permission rules are enforced by the OpenCode server. A denied write fails
inside the executor and it is told why:

```
$ foreman dispatch --deny 'apps/worker/**' "Create apps/web/hello.txt and apps/worker/nope.txt" --wait
apps/web/hello.txt succeeded; apps/worker/nope.txt was denied by permission rules.
```

**Two ways to write a rule that is silently ignored** (verified against
opencode 1.18.10 — see [docs/api-notes.md](docs/api-notes.md)):

1. The permission name must be `edit`. `write` is accepted by the API and has
   no effect.
2. The pattern matcher does not honour `**/dir/**` or absolute-path prefixes.
   Only substring globs (`*dir*`) match.

Both fail **open** — no error, the rule just never fires. `foreman` normalizes
whatever you pass into the form that actually works and prints the resulting
ruleset on dispatch, so you can see what is really in force:

```
  denied    bash:**, edit:*apps/worker*
```

Treat this as defence in depth, not a sandbox. Pair it with `--worktree`.

## Using it from Claude Code

[`skills/opencode-foreman/SKILL.md`](skills/opencode-foreman/SKILL.md) teaches
Claude Code the dispatch → poll → review → land loop. Install it with:

```bash
mkdir -p ~/.claude/skills/opencode-foreman
ln -s "$PWD/skills/opencode-foreman/SKILL.md" ~/.claude/skills/opencode-foreman/SKILL.md
```

The same loop works from Codex, a Makefile, or CI — nothing here is
Claude-specific.

## What this is not

- **Not a sandbox.** Permission rules and worktrees raise the cost of a
  mistake; they do not contain a determined process. Do not point it at
  production credentials or irreversible operations.
- **Not an autonomous agent.** There is no auto-merge. Landing work is your
  call, on purpose.
- **Not a model router.** One model per task, chosen by you. If you want
  automatic routing across a fleet, [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)
  does that inside OpenCode and composes fine with this.

## Known rough edges

- `GET /session/{id}/diff` returned `[]` for newly created untracked files in
  testing, so `foreman diff` shells out to git instead (using
  `git add -AN` so new files show up).
- Right after dispatch a session is briefly absent from `/session/status`.
  `foreman` reports `starting` for 5 seconds rather than a misleading `idle`.
- `foreman down` only stops a server that `foreman` itself started.
- The server runs unsecured unless `OPENCODE_SERVER_PASSWORD` is set (it logs a
  warning saying so). It binds `127.0.0.1`, so the exposure is to other
  processes on the same machine — worth a password if that matters to you.
- `foreman` never emits an `ask` permission rule, only `allow`/`deny`, so a
  session cannot stall waiting for a human. If your project `opencode.json`
  sets `ask`, a run can block; the API routes for answering that are in
  [docs/api-notes.md](docs/api-notes.md).

Verified against opencode 1.18.10 on macOS. Issues and PRs welcome.

## License

MIT
