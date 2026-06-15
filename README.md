# nice-insights-skills

Shared agent skills for Nice Insights clients and the internal team.

These skills follow the open [Agent Skills](https://agentskills.io) standard, so
the same `SKILL.md` works across Claude Code, OpenAI Codex, and OpenClaw with no
runtime-specific copies.

## Skills

| Skill | Description |
|-------|-------------|
| [`nice-insights-metrics`](./nice-insights-metrics) | Query ecommerce metrics: ad spend, sales, orders, margins, CAC, retention/LTV, checkout funnel, and email activity |

## MCP server

The Nice Insights metrics MCP server is available over streamable HTTP:

```text
https://api.niceinsights.io/mcp-metrics
```

The server uses OAuth. When your MCP client asks you to authenticate, sign in
with your Nice Insights account. The metrics server uses the `read:metrics`
scope by default.

### Claude Code CLI

Add the remote HTTP server:

```bash
claude mcp add --transport http nice-insights-metrics --scope user https://api.niceinsights.io/mcp-metrics
```

Then start Claude Code and run `/mcp` to authenticate and verify that the server
is connected.

### Claude Desktop

Claude Desktop uses Claude custom connectors for remote MCP servers:

1. Open Claude Desktop and go to **Customize > Connectors**.
2. Click **+**, then **Add custom connector**.
3. Enter the remote MCP server URL:
   `https://api.niceinsights.io/mcp-metrics`
4. Finish adding the connector, click **Connect**, and complete the browser
   authentication flow.
5. In a conversation, use **+ > Connectors** to enable the connector when you
   want Claude to use it.

For Team and Enterprise plans, an Owner may need to add the custom connector
from **Organization settings > Connectors** before members can connect it.

### Codex CLI

Add the server to `~/.codex/config.toml`:

```toml
[mcp_servers.nice-insights-metrics]
url = "https://api.niceinsights.io/mcp-metrics"
```

Authenticate the server:

```bash
codex mcp login nice-insights-metrics
```

In the Codex TUI, run `/mcp` to confirm the server is active.

### Codex Desktop

Codex Desktop shares MCP configuration with Codex CLI through
`~/.codex/config.toml`.

Use **Settings > Integrations & MCP** to add a custom MCP server with this URL:

```text
https://api.niceinsights.io/mcp-metrics
```

If you prefer editing the config directly, use the same `config.toml` snippet
from the Codex CLI section. Codex Desktop starts the OAuth flow when the server
requires authentication.

## Install the skill

The skill gives your agent the Nice Insights-specific usage guide for the MCP
tools. Install the skill in addition to configuring the MCP server.

### Claude Code

Install with [`npx skills`](https://github.com/obra/skills):

```bash
npx skills add nice-insights/nice-insights-skills -g -a claude-code -y
```

Update later with:

```bash
npx skills update
```

Or symlink manually:

```bash
git clone https://github.com/nice-insights/nice-insights-skills ~/src/nice-insights-skills
ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.claude/skills/nice-insights-metrics
```

### Codex / OpenClaw

Both read skills from `~/.agents/skills/`:

```bash
git clone https://github.com/nice-insights/nice-insights-skills ~/src/nice-insights-skills
mkdir -p ~/.agents/skills
ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.agents/skills/nice-insights-metrics
```

OpenClaw also reads `~/.openclaw/skills/` if you prefer to scope the skill
there. Update a manual install with `git pull`.
