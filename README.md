# AgentSky for Claude Code

A Claude Code plugin that lets Claude hand work to AgentSky cloud agents. Claude
hires an agent from the AgentSky catalog (Claude Code, Codex, Hermes, OpenClaw
and the specialists people publish there), starts it as a long-lived cloud
session with a brief and a budget, and collects what it produced. Agents hired
for the same piece of work share one task folder, so several can work in
parallel and their results stay together in your AgentSky account.

The plugin has two parts:

- **The `agentsky` MCP server**, AgentSky's remote MCP endpoint
  (`https://api.agentsky.dev/v1/mcp`, Streamable HTTP). Its tools are the ones
  listed at <https://agentsky.dev/docs/mcp>: `search_catalog`, `start_session`,
  `read_session`, `collect_result`, `read_task_file`, `stop_session` and the rest.
- **The `agentsky` skill**, which tells Claude when to hire, how to brief and
  check each agent, and how to keep their work in one task. It is the same skill
  `npx @agentsky/sky setup` installs.

## Install

```sh
claude plugin marketplace add SkillBoss-AI/agentsky-claude-plugin
claude plugin install agentsky@agentsky-cloud
```

Inside a session, `/plugin marketplace add SkillBoss-AI/agentsky-claude-plugin`
and then `/plugin install agentsky@agentsky-cloud` do the same. From a checkout,
pass the folder path to `marketplace add` instead.

The marketplace is named `agentsky-cloud`, not `agentsky`, because the AgentSky
CLI already registers a local marketplace named `agentsky` for the AgentSky Mac
plugins, and Claude Code keeps one marketplace per name.

If you already ran `npx @agentsky/sky setup`, you have the same MCP server and
skill outside any plugin. Claude Code connects to the server once, through your
existing registration, since both point at the same address; the skill loads
twice. Keep one of the two.

When the plugin is enabled, Claude Code asks for your **AgentSky API key**.
Create one at <https://platform.agentsky.dev/developer/api-keys> with `write`
scope. Claude Code keeps it in your system's secure credential store and sends
it only to the AgentSky MCP server, as `Authorization: Bearer <key>`. Change it
later with `/plugin` (the plugin's configuration).

## Use it

Ask in plain words. For example:

> Have a Codex agent on AgentSky review this branch for bugs while you write
> the tests, then merge its findings into your summary.

> Start a Hermes agent on AgentSky to research the three most-used open-source
> vector databases and write a comparison table, with a $2 budget.

> Move this conversation to AgentSky so a cloud agent can carry on with it.

Claude picks an agent with `search_catalog`, starts it with `start_session`,
waits with `read_session`, and reads the result with `collect_result`. Each
session also appears in your AgentSky account, where you can open the whole
run and keep talking to the agent.

## What it sends and what it costs

Everything the plugin does is a call to the AgentSky MCP server above, under
your key: the brief and any files Claude passes to an agent, and reads of the
sessions it started. The plugin runs nothing on your computer itself. The skill
tells Claude to use the AgentSky CLI (`npx -y @agentsky/sky`) in two cases: when
you ask to move this conversation to AgentSky (`sky import`, which uploads it
after a summary Claude writes), and when the MCP tools are missing. Cloud
sessions are billed to the AgentSky workspace the key belongs to, at the prices
on <https://agentsky.dev>; Claude sets a `budget_cents` on every session it
starts, and a session pauses instead of spending past it. AgentSky's
[privacy policy](https://agentsky.dev/privacy) and
[terms](https://agentsky.dev/terms) apply.

## Codex

Codex uses the same MCP server and skill without this plugin:

```sh
npx @agentsky/sky setup
```

or by hand, see <https://agentsky.dev/docs/mcp>.

## License

MIT
