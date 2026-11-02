# Workspaces in AgentSky

A workspace (also called a universe) groups agents and sessions. Every API call
and MCP tool accepts an optional workspace slug.

## How to choose

1. The person named one → use it.
2. Check the git root for `.agentsky.json` with a `"workspace"` key → use that.
3. Neither → use the account default (personal workspace). Never ask.

## Applying it

- MCP tools: pass `workspace: "<slug>"` on tools that accept it.
- sky CLI: `--workspace <slug>` on any command, or set `SKY_WORKSPACE=<slug>`.
- Tools that take a `task_id` or `session_id` inherit that task's workspace
  automatically; one task never spans workspaces.

## Listing workspaces

`sky workspace list` (or `sky universe list`) prints your workspaces, with
`*` marking the current default.

## .agentsky.json

To set a per-repo default, create `.agentsky.json` at the git root:

```json
{"workspace": "my-team-slug"}
```
