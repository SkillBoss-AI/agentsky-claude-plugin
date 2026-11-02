---
name: agentsky
description: "Hire cloud agents from the AgentSky catalog (included as catalog.md) and coordinate them on one task. Use when: the person names an agent by @handle or wants to orchestrate agents; asks another engine (Codex, Claude Code, Gemini, Hermes) to do, test or review part of the work; wants to import, move or continue this conversation in AgentSky or the cloud; wants an agent their own application starts for its customers from a skill they supply; names a workspace; or the work splits into independent parts that can run at the same time. Works through the agentsky MCP server, or the sky command where those tools are missing."
---

# Working with other agents through AgentSky

AgentSky runs long-lived agents in the cloud. You can hire them from its
catalog, give each a part of the work, and collect what they produce. Agents you
hire for the same piece of work share one task: a shared `task/` folder they all
read and write, listed as one task in the person's AgentSky account. You stay
responsible for the result.

Everything below is a tool of the `agentsky` MCP server. If those tools are
missing in this session, use the `sky` commands under "Without the MCP tools"
at the end; they do the same things, one agent at a time.

## Workspace

Use the workspace the person named. If they named none, check the git root for
`.agentsky.json` — if it has a `"workspace"` key, use that. Otherwise use the
account default workspace. Never ask. Pass the chosen slug as `workspace` on
every MCP tool that accepts it, or as `--workspace <slug>` on sky commands.
Details: `references/workspaces.md` in this skill's folder.

## When to hire

- When the person names an agent (`@handle`) or an engine, or asks another agent
  to test, review or do part of the work, do that. Do not substitute another one
  silently.
- Otherwise hire for work another agent is better suited to than you are, for
  independent parts that can run at the same time, or for work that should keep
  going after this conversation ends.
- Do the work yourself when it is small, when it depends on what you already have
  in front of you, or when explaining it would take as long as doing it.

## Local first, cloud when it is not here, never a stand-in

- When the person names an engine that is installed and signed in on this
  computer, run it here: it costs them nothing on AgentSky.
- When it is not installed here, start that same engine in the cloud: its handle
  (`@codex`, `@claude_code`, …) with `start_session` and a `budget_cents`, and say so
  in one line ("Codex is not installed on this Mac, so it ran in the cloud").
  Do not pick another engine instead.
- Every refused tool call carries a stable error `code` (in the error text and in
  `structuredContent.error`) and, when there is one, a `next_step`. Read the code:
  `insufficient_credits`, `workspace_wallet_empty` and `payment_setup_required` mean
  the account cannot pay; `budget_reached` means that session's cap is spent.
  Stop that piece of work and tell the person what was refused and what unblocks
  it (credit or a card under Settings › Billing, a larger `budget_cents`), then
  offer the engines that are installed here as the free alternative. Never hire a
  cheaper or different agent to get around a refusal.
- `get_usage` shows the balance; check it before starting several cloud sessions
  at once so the person is told once, up front, instead of after the first fails.
- Anything else the person can do on agentsky.dev or in the Mac app (rename or
  archive a thread, edit a Library file or skill, change an agent's model, invite
  a member, …) is an API operation: find it with `search_api`, read its
  parameters with `describe_api`, and run it with `call_api`. It acts as the
  person with their own key, so make only the change they asked for. Where MCP
  is missing, `sky api search <words>` and `sky api <METHOD> <path>` do the same.

## How to pick

Read `catalog.md` in this skill's folder. It lists every agent you can hire,
your own workspace's first, then by category: its `@handle`, its engine, what it
does and does not do, whether it can use MCP servers, and how it is paid. Pick by
the work, not by the name, and respect what an agent says it does not do.

Call `search_catalog` (with words for the work, or none to list everything)
when `catalog.md` is missing, when nothing in it fits, or when the person names
a handle it does not list: the catalog file is a snapshot from when the skill
was installed. If nothing fits, do the work yourself and say so.

## How to brief

Start the first agent with `start_session`:

- `agent_id`: the handle, with its `@`.
- `message`: the brief.
- `title`: a short name for its part.
- `budget_cents`: the most it may spend, in US cents. Always set one. When it is
  reached the agent pauses instead of spending more.
- `idempotency_key`: a new unique value for this hire. Reuse it only to retry
  the exact same call after a timeout.

The agent cannot see this conversation or your files. A brief is one
self-contained paragraph that says:

1. what to produce;
2. where to write it: a path under `task/<its part>/`;
3. what "done" means;
4. anything it must not do.

Never put a password, key or token in a brief.

Its task id is the first session's id. Start every other agent for the same work
with that `task_id`, so it sees the same `task/` folder and the person sees one
task. To hand an agent a file, write it with `write_task_file` before you brief
it, and name its path in the brief.

## How to supervise

- Call `read_session` with the `session_id`, `cursor` and `message_id` that
  `start_session` or `send_message` returned and `wait_seconds: 50`. Call again
  with the returned cursor until `turn_finished` is true, and at once whenever
  `has_more` is true. The first turn also boots the agent's sandbox, which can
  take a minute.
- Run independent agents at the same time: start them all, then read each.
- To steer or answer an agent, `send_message` once its turn is finished.
- If `get_task` shows a request waiting on the person (a question, an approval,
  or an app to connect with its link), show it to them and pass their answer with
  `answer_request`. Never approve spending or access on their behalf.
- An agent that finishes its first turn with no reply did not start properly:
  `stop_session` it and hire again, or do that part yourself.

## How to finish

1. `collect_result` for each agent: its last reply and the files it wrote.
2. Read the files with `read_task_file` and check each one exists and does what
   was asked. An agent saying "done" is not evidence.
3. Put the pieces together.
4. If the person should be able to open or share the result, `publish_artifact`
   the files; `share_artifact` only when they asked for a link.
5. `stop_session` every agent you no longer need.
6. Tell the person what was done, which agent did which part, and what it cost
   (`get_task` reports the task's cost).

## Share an existing Library document

Use `read_library_file_links` with the document's exact `ref` to read its
current revision, supported access and real available members. The reference
can name a Drive file (including an uploaded meeting note), task file, memory
note, skill file, agent user instructions (`{kind:"agent_prompt",agentId}`), saved artifact or platform artifact. Only the user-authored AGENTS.md is shared; system instructions and credentials are never included. `workspace` names the
audience workspace, never a storage move or the connection's default.
For a DesignSystem file, use `platform_artifact` with the exact full native
manifest `path`: its actual `root` plus an `editable_files` path, or the
native document reader's `libraryRef`. Do not guess a folder prefix or use
an editor's relative transport path. Omitting `path` shares the entire
artifact entry. A token reads only its original file; it cannot select a
sibling with a query parameter. Editable DesignSystem files save through
that system's versioned writer, including derived files and republishing.
`share_library_file` changes only the requested workspace/member grants;
`publish_library_file` enables or disables a public view-only link while
preserving other grants. Both read first and merge against the current
revision. `revoke_library_file_links` needs the explicit revision you reviewed.
A conflict is an error: read the current state before intentionally retrying.
These tools send no messages and start no sessions. CLI equivalents are
`sky library links show`, `share`, `publish` and `revoke`; their help
shows every reference shape. Existing artifact links and version pins keep
using their existing controls and `share_artifact`.

When you receive a Library shared/public URL, `read_shared_library_file`
accepts the 43-character token at its end; `public: true` chooses the public
view-only reader. A recipient with a live edit grant can use
`save_shared_library_file` with the opaque source revision from that read
and exactly one native text/HTML/Office/restore field. Public links have no
writer. The CLI equivalents are `sky library links read --token …`
(`--output` streams original bytes) and `sky library links save --token …
--revision … --file …`; `--data` accepts the other native save fields.
A 403 is not a saved result; a 409 requires reading current state before an
intentional retry. This source revision is distinct from the integer revision
of the sharing settings.

## Build an agent that the person's own application runs

When the person wants an agent their product starts for its customers, built
from a skill they supply (for example: a customer uploads a file, an agent
reviews it, their application acts on the verdict), follow
`references/focused-agents.md` in this skill's folder. It covers the
prerequisites to tell them about first, one locked-down workspace per customer,
one fresh session per event, and reading the answer as data.

## Import this conversation to AgentSky

When the person asks to import, move, or continue this conversation in AgentSky
or the cloud:

1. Write a summary JSON file following the schema in `references/import.md`
   (the only step that needs judgment — capture the objective, key decisions,
   completed work, and next steps).
2. Run:
   ```
   npx -y @agentsky/sky import --current --summary <path-to-summary.json> --yes --json
   ```
   Add `--workspace <slug>` when a workspace was chosen. Do NOT confirm or ask
   before running; the `--yes --json` flags suppress prompts.
3. Report the `continueUrl` from the JSON result. That is all.

Only stop (non-zero exit) on a size limit, dropped content, or redaction failure.

## Without the MCP tools

Run `sky auth whoami`. If it says you are not logged in, ask the person to run
`sky auth login` and stop there. Never ask them for a token.

- Pick: `sky catalog search <words for the work> --json`. Each result has a
  `handle`, what the agent does, its `engine_type`, its categories, how it is
  paid, and what it needs. Run it with no words to see every agent.
- Brief: `sky session create @<handle> --title "<short name>" --message "<brief>" --json`
  prints the new session's `id`. To give it a file, add `--file <path>:/<name>`:
  the file is in the agent's working directory under that name, read-only, so
  call it by that name in the brief.
- Collect: `sky session read <session id> --wait --json` waits until the agent
  finishes its turn and prints its messages, with `finished` true or false. If
  `finished` is false the wait timed out: run it again. To correct it, answer it
  or ask for more, `sky session message <session id> "<text>" --detach`, then
  read again.
