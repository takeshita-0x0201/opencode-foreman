# OpenCode server API — verified notes

Everything here was checked against **opencode 1.18.10** on macOS by making the
calls, not by reading source. Where behaviour surprised me it is written down
as behaviour, not as intent — a later release may change it.

Start the server and read the spec yourself:

```bash
opencode serve --port 4096
curl http://127.0.0.1:4096/global/health     # {"healthy":true,"version":"1.18.10"}
curl http://127.0.0.1:4096/doc               # OpenAPI 3.1, 162 paths
```

## The dispatch loop

```bash
DIR=/abs/path/to/worktree

SID=$(curl -s -X POST "http://127.0.0.1:4096/session?directory=$DIR" \
  -H 'Content-Type: application/json' \
  -d '{"title":"demo"}' | jq -r .id)

curl -s -X POST "http://127.0.0.1:4096/session/$SID/prompt_async?directory=$DIR" \
  -H 'Content-Type: application/json' \
  -d '{"model":{"providerID":"opencode-go","modelID":"deepseek-v4-pro"},
       "parts":[{"type":"text","text":"..."}]}'      # returns immediately

curl -s "http://127.0.0.1:4096/session/status?directory=$DIR"
# {"ses_...":{"type":"busy"}}   while running
# {}                            when idle — the session simply disappears
```

Notes:

- **`directory` is a query parameter on nearly every route.** One server can
  serve many worktrees concurrently; there is no per-server working directory
  to fight over. This is what makes parallel isolation cheap.
- `prompt_async` returns before the session appears in `/session/status`. Poll
  too early and an unstarted task looks finished. Allow a few seconds of grace.
- The model is `{"providerID": ..., "modelID": ...}` — two fields, not a single
  `provider/model` string.

## Reading results

`GET /session/{id}/message?directory=...` returns messages shaped as
`{info: {role, ...}, parts: [...]}`. Part types seen in practice:
`step-start`, `reasoning`, `tool`, `patch`, `text`, `step-finish`. The final
answer is the last `text` part of the last `assistant` message.

## Permission rulesets

Set on session creation:

```json
{"permission": [{"permission": "edit", "pattern": "*apps/worker*", "action": "deny"}]}
```

`action` is `allow` | `deny` | `ask`. Rules are enforced server-side — the
executor's tool call fails and it is told a rule blocked it.

### Two silent failure modes

Both fail **open**: the API accepts the rule, returns no error, and the rule
never fires.

**1. The permission name must be `edit`.**

| rule | blocks a file write? |
| --- | --- |
| `{"permission":"edit","pattern":"**"}` | yes |
| `{"permission":"write","pattern":"**/protected/**"}` | **no** |
| `{"permission":"write","pattern":"*protected*"}` | **no** |

`write` is the *tool* name; `edit` is the *permission category* that governs
it. Naming the tool gets you nothing.

**2. `**` does not work as a path-recursive glob.**

Tested by asking the executor to write `apps/web/a.txt` and `apps/worker/b.txt`
with only the worker path denied:

| pattern | worker | web | verdict |
| --- | --- | --- | --- |
| `*apps/worker*` | blocked | written | ✅ correct |
| `<abs>/apps/worker/*` | written | written | ❌ ineffective |
| `**/apps/worker/**` | written | written | ❌ ineffective |

Use substring globs. `**` alone (match everything) does work.

### Answering a prompt without a TUI

Listing pending requests only works on the **v2** route:

```
GET  /api/session/{id}/permission              -> {"data": [...]}
POST /api/session/{id}/permission/{reqID}/reply  {"reply": "once"|"always"|"reject"}
```

The v1 path `GET /session/{id}/permission` answers **HTTP 200 with a non-JSON
body**, which is easy to misread as "nothing pending". The v1 reply route
(`POST /session/{id}/permissions/{permissionID}`, note the plural) takes
`{"response": ...}` instead.

## Other useful routes

| Route | Use |
| --- | --- |
| `POST /session/{id}/abort` | Stop a run |
| `POST /session/{id}/revert` / `unrevert` | Roll back the executor's edits |
| `POST /session/{id}/revert/stage` / `commit` | Partial rollback control |
| `GET /session/{id}/diff` | Session diff — **omitted untracked new files in testing**; prefer git |
| `GET /session/{id}/todo` | The executor's task list |
| `POST /session/{id}/fork` | Branch a session |
| `POST /session/{id}/shell` | Run a shell command in session context |
| `POST /session/{id}/summarize` / `compact` | Context management |
| `GET /global/event` | SSE stream of everything |
| `POST /experimental/session/{id}/background` | Background subagents |
| `GET /session/{id}/question` + `.../reply` \| `/reject` | Answer an executor's question programmatically |

## Auth

Bound to `127.0.0.1` by default. For anything else set
`OPENCODE_SERVER_PASSWORD` (and optionally `OPENCODE_SERVER_USERNAME`,
default `opencode`) and send HTTP basic auth.
