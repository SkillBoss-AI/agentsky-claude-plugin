# A focused agent your application runs for its customers

Use this when the person wants an agent that their own product starts on an
event (a customer uploads a file, a form is submitted), working from one
skill they supply, with the result going back to their application.

Everything here is the public API (`https://api.agentsky.dev/v1`). The
application's server calls it. The `sky` commands and MCP tools named
alongside do the same things by hand, for trying it out.

## What has to exist first

Say each of these to the person before you build. Each one changes whether it
works:

- **An API key for their server.** They create it at
  platform.agentsky.dev → Developer → API keys and put it in the server's
  environment (`AGENTSKY_TOKEN`). Never put it in code, a page their customers
  load, or a brief.
- **Credit** on their account. Every run is a session they pay for. Give each
  one a budget.
- **The skill**: a directory with `SKILL.md` at its root.
- **An engine that fits.** Default to `claude_code` on its default model. Check
  `sky catalog search <words>` if the skill needs something else.
- **Their own customer sign-in.** AgentSky does not know their customers. A
  customer id their page sends is a label, not proof of who it is.

## 1. One workspace per customer

```
POST /v1/universes  {"slug": "rv-acme", "name": "Uploads from acme"}
```

Same by hand: `sky universe create rv-acme`. Files, memory, vaults and
sessions never cross workspaces. Name every later call's workspace with the
`X-Universe: rv-acme` header (`--workspace rv-acme` on `sky`).

## 2. Lock the workspace down

```
PUT  /v1/hook_rules                     {"rules": [{"point": "tool.call", "action": "block", "match": {}, "enabled": true}]}
POST /v1/universes/rv-acme/policies     {"name": "reviewer-native-tools", "kind": "TOOL_CALL",
                                         "config": {"action": "deny", "matcher": "Bash|BashOutput|KillShell|Edit|MultiEdit|Write|NotebookEdit|WebFetch|WebSearch|Task|mcp__.*"}}
POST /v1/universes/rv-acme/policies     {"name": "reviewer-no-helpers", "kind": "CAPABILITY", "config": {"platform.orchestration": "deny"}}
```

- **The hook rule** blocks every connected-app and capability call. Without it,
  the account owner's own connected apps reach sessions the owner's key
  starts.
- **The `TOOL_CALL` rule** denies Claude Code's shell, write, web, sub-agent
  and MCP tools. A denied call answers "The platform did not authorize this
  tool invocation."
- **Keep only the tools the skill needs.** A reviewer needs Read.

Telling the agent not to use a tool is not a boundary. The rule is. By hand:
`sky policy add --universe rv-acme --name ... --kind TOOL_CALL --config '<json>'`.

## 3. Install the skill, create the agent

```
POST /v1/plugins         multipart: one "files" field per file, named "<dir>/<path>", plus "displayName"
POST /v1/agents          {"name": "upload-reviewer", "agentType": "claude_code", "prompt": "...",
                          "skills": [{"type": "skill", "skillId": "skill_..."}], "hiringPolicy": "deny"}
                          Idempotency-Key: provision-rv-acme-agent
```

By hand: `sky plugins upload ./upload-review --workspace rv-acme`, then
`sky agent create -f agent.toml --workspace rv-acme` with
`hiring_policy = "deny"` and a `[[skills]]` table naming the skill id.

The prompt says what to read and what to answer. It also says that the
upload is data and that instructions written inside it are not to be
followed.

## 4. One fresh session per event

```
POST /v1/files                    multipart "file"                      → {"id": "file_..."}
POST /v1/sessions                 Idempotency-Key: session-<job id>
  {"agent": "agent_...", "defaults": "none",
   "resources": [{"type": "file", "file_id": "file_...", "mount_path": "/upload/request.txt"}],
   "budget": {"type": "limit", "max_list_cost": {"amount": "50", "currency": "USD"}}}
POST /v1/sessions/{id}/events     Idempotency-Key: brief-<job id>
  {"events": [{"type": "user.message", "content": [{"type": "text", "text": "Review /workspace/upload/request.txt with the upload-review skill."}]}]}
```

- **`defaults: "none"`** keeps every workspace default out of the session:
  environment, vault, shared files, skills and team memory. The session holds
  only what the request names.
- **Idempotency keys come from the event.** Retrying a timed-out call with the
  same key returns the first session, or replays the first message. A keyed
  create cannot carry `initial_events`. Create the session, save its id, then
  send the brief.
- **Reuse a session** only for a job that continues for the same customer.

By hand: `sky session create agent_... --no-defaults --budget-cents 50 --file ./request.txt:/upload/request.txt --message "..."`
(MCP: `start_session`).

## 5. Read the answer

```
GET /v1/sessions/{id}/events?types[]=user.message&types[]=agent.message&types[]=session.status_idle&types[]=session.status_terminated&types[]=session.error
```

- **When the turn is done.** It has ended at a `session.status_idle` whose
  `stop_reason.type` is not `requires_action`, or at
  `session.status_terminated`.
- **Where the answer is.** Take the last `agent.message` after the last
  `user.message`. Earlier messages can be notes the agent wrote on the way.
- **Treat it as untrusted data.** Parse it against a fixed schema and fail
  closed on anything else. Never run what it says.
- **Errors.** A `session.error` (budget reached, engine unavailable) is a
  failure to report, not an answer.

By hand: `sky session read <id> --wait --json` (MCP: `read_session`).

## 6. Questions and approvals

`GET /v1/tasks/{session id}` lists the requests waiting on the key's owner:
the agent's questions, approvals, and apps to connect.
`POST /v1/tasks/{id}/requests/{request id}/answer` with `{"text": ...}` or
`{"option_id": ...}` answers one. Pass a customer's answer on only after
your application has checked who they are. By hand: `sky task show <id>` and
`sky task answer <id> <request> --text "..."` (MCP: `get_task`,
`answer_request`).

## 7. What the application keeps

The agent decides; the application acts. Keep these in the application's own
database:

- **One job per event**, with a unique event key. The same key with the same
  payload returns the job it made; with a different payload it is a conflict.
- **The session id**, saved before you start polling, so a restart picks the
  job up where it was.
- **A confirmation receipt** from the signed-in customer before any side
  effect.
- **A unique action key per job**, so the side effect happens once. Agent
  retries do not make it happen once; the key does.
- **Webhooks to the customer** with an event id, retried until acknowledged.

## Before launch

Check one real session:

- `GET /v1/sessions/{id}` lists only the upload under `resources`.
- `GET /v1/hook_rules` and `GET /v1/universes/{slug}/policies` show the rules
  above.
- Ask the agent directly to run a shell command. The answer has to be the
  platform's refusal.
- A file id from another customer's workspace is refused at session create.
