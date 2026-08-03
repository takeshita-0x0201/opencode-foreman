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
foreman dispatch --worktree --deny 'apps/worker/**' "Add retry with backoff to the HTTP client."
foreman ls        # t1  running  opencode-go/deepseek-v4-pro  Add retry with backoff…
foreman diff t1   # review before anything lands on your branch
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

`opencode serve` is a background HTTP server, but you never have to think about
it. Any command that needs it starts it on demand and reuses it afterwards, so
one server serves every task, every repo, and every terminal.

If you would rather have it always resident — surviving reboots and crashes:

```bash
foreman agent install     # launchd on macOS, systemd --user on Linux
foreman agent status
foreman agent uninstall
```

`foreman up` / `foreman down` remain for starting and stopping it by hand.
Set `FOREMAN_AUTOSTART=0` if you want an unstarted server to be an error
instead of a launch.

## Commands

| Command | What it does |
| --- | --- |
| `foreman dispatch "<task>"` | Create a session and fire `prompt_async`; returns immediately |
| `foreman ls` | All tasks with live `running` / `starting` / `idle` state |
| `foreman wait <id>` | Block until the session goes idle |
| `foreman result <id>` | The executor's final message |
| `foreman diff <id>` | `git diff` of what it changed |
| `foreman perms <id>` / `reply <id> <req> once\|always\|reject` | Answer a permission prompt without a TUI |
| `foreman abort <id>` | Stop a running session |
| `foreman revert <id>` | Ask OpenCode to roll back its own edits |
| `foreman clean <id>` | Remove the worktree and forget the task |
| `foreman up` / `down` | Start / stop the server by hand |
| `foreman agent install` / `uninstall` / `status` | Keep the server resident across reboots |

`dispatch` flags worth knowing:

- `--worktree [NAME]` — run in a fresh git worktree on branch `foreman/<name>`.
  Nothing touches your working tree until you decide it should.
- `--deny PATH` (repeatable) — refuse edits under that path, enforced by the
  server, not by prompt wording.
- `--read-only` — analysis only.
- `--allow-bash` — shell is **denied by default**; opt in per task.
- `--profile pro|flash` — `deepseek-v4-pro` (implementation) or
  `deepseek-v4-flash` (search, classification, bulk mechanical work).
- `--model provider/model` — anything `opencode models` lists.
- `--wait` — dispatch synchronously and print the result.

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
- `foreman down` only stops a server that `foreman` itself started, and refuses
  while `agent install` is active — launchd would just restart it.
- After `agent install`, killing the server looks like KeepAlive is broken for
  about ten seconds. It isn't: launchd throttles restarts. Wait, then re-check.
- The server runs unsecured unless `OPENCODE_SERVER_PASSWORD` is set (it logs a
  warning saying so). It binds `127.0.0.1`, so the exposure is to other
  processes on the same machine — worth a password if that matters to you.

Verified against opencode 1.18.10 on macOS. Issues and PRs welcome.

## License

MIT
