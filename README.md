# nice-insights-skills

Shared agent skills for Nice Insights clients and the internal team.

These skills follow the open [Agent Skills](https://agentskills.io) standard, so
the same `SKILL.md` works across Claude Code, OpenAI Codex, and OpenClaw with no
runtime-specific copies.

## Skills

| Skill | Description |
|-------|-------------|
| [`nice-insights-metrics`](./nice-insights-metrics) | Query ecommerce metrics: ad spend, sales, orders, margins, CAC, retention/LTV, current inventory, checkout funnel, and email activity |
| [`nice-insights-financials`](./nice-insights-financials) | Query and compare scenario-safe monthly financial statement and account hierarchy metrics |

## Install as a plugin (recommended for Claude)

The fastest way to use these skills in Claude is the **Nice Insights plugin
marketplace**. Installing a plugin bundles the skill *and* its MCP connector in
one step — no separate MCP or skill setup — and updates flow from this one
repository.

Two plugins are published so you install only what your access covers:

| Plugin | Bundles | Access |
|--------|---------|--------|
| `nice-insights-metrics` | metrics skill + metrics connector | `read:metrics` |
| `nice-insights-financials` | financials skill + financials connector | `read:financials` + `FinancialsAccess` group |

### Claude.ai account (Chat, Claude Desktop, and Cowork)

Install through claude.ai to add the marketplace and plugins to your Claude
account. Claude Code CLI commands only configure Claude Code locally; they do
not install a plugin to your claude.ai account.

1. Sign in at [claude.ai](https://claude.ai/) and open **Customize > Plugins**.
2. In **Personal plugins**, click **+**, select **Add marketplace**, then choose
   **Add from a repository**.
3. Enter `nice-insights/nice-insights-skills` and sync it.
4. Install `nice-insights-metrics` (and `nice-insights-financials` if you have
   access), then complete the connector's OAuth sign-in when prompted.
5. Type `/` or click **+** in Chat or Cowork to confirm the installed Nice
   Insights skill appears.

### Claude Code CLI (local installation)

```bash
claude plugin marketplace add nice-insights/nice-insights-skills
claude plugin install nice-insights-metrics@nice-insights
claude plugin install nice-insights-financials@nice-insights   # only if you have financials access
```

Then run `/mcp` to complete the OAuth sign-in for each connector.

Update later by refreshing the marketplace and then updating each installed
plugin:

```bash
claude plugin marketplace update nice-insights
claude plugin update nice-insights-metrics@nice-insights
claude plugin update nice-insights-financials@nice-insights   # if installed
```

Or enable background auto-updates in `/plugin` → **Marketplaces** → select
`nice-insights` → **Enable auto-update** (third-party marketplaces are manual by
default).

> **Already added the connector manually?** If you previously ran
> `claude mcp add nice-insights-metrics …` or added a custom connector in Claude
> Desktop, remove it first (for example `claude mcp remove nice-insights-metrics`)
> so you don't end up with a duplicate connector of the same name once the plugin
> provides it.

The sections below cover **manual setup** — configuring the MCP server and skill
separately. Use them for Codex / OpenClaw, or when you can't use the plugin
marketplace.

## MCP servers

The Nice Insights MCP servers are available over streamable HTTP:

```text
https://api.niceinsights.io/mcp-metrics
https://api.niceinsights.io/mcp-financials
```

The servers use OAuth. When your MCP client asks you to authenticate, sign in
with your Nice Insights account. The metrics server uses `read:metrics`; the
financials server separately requests `read:financials` and also requires the
`FinancialsAccess` group.

### Claude Code CLI

Add the remote HTTP server:

```bash
claude mcp add --transport http nice-insights-metrics --scope user https://api.niceinsights.io/mcp-metrics
claude mcp add --transport http nice-insights-financials --scope user https://api.niceinsights.io/mcp-financials
```

Then start Claude Code and run `/mcp` to authenticate and verify that the server
is connected.

### Claude Desktop

Claude Desktop uses Claude custom connectors for remote MCP servers:

1. Open Claude Desktop and go to **Customize > Connectors**.
2. Click **+**, then **Add custom connector**.
3. Enter one remote MCP server URL:
   `https://api.niceinsights.io/mcp-metrics` or
   `https://api.niceinsights.io/mcp-financials`.
4. Finish adding the connector, click **Connect**, and complete the browser
   authentication flow.
5. Repeat for the other server if you need both ecommerce and financial tools.
6. In a conversation, use **+ > Connectors** to enable the connector when you
   want Claude to use it.

For Team and Enterprise plans, an Owner may need to add the custom connector
from **Organization settings > Connectors** before members can connect it.

### Codex CLI

Add the servers to `~/.codex/config.toml`:

```toml
[mcp_servers.nice-insights-metrics]
url = "https://api.niceinsights.io/mcp-metrics"

[mcp_servers.nice-insights-financials]
url = "https://api.niceinsights.io/mcp-financials"
```

Authenticate the server:

```bash
codex mcp login nice-insights-metrics
codex mcp login nice-insights-financials
```

In the Codex TUI, run `/mcp` to confirm the server is active.

### Codex Desktop

Codex Desktop shares MCP configuration with Codex CLI through
`~/.codex/config.toml`.

Use **Settings > Integrations & MCP** to add a custom MCP server with this URL:

```text
https://api.niceinsights.io/mcp-metrics
https://api.niceinsights.io/mcp-financials
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
ln -s ~/src/nice-insights-skills/nice-insights-financials ~/.claude/skills/nice-insights-financials
```

### Codex / OpenClaw

Both read skills from `~/.agents/skills/`:

```bash
git clone https://github.com/nice-insights/nice-insights-skills ~/src/nice-insights-skills
mkdir -p ~/.agents/skills
ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.agents/skills/nice-insights-metrics
ln -s ~/src/nice-insights-skills/nice-insights-financials ~/.agents/skills/nice-insights-financials
```

OpenClaw also reads `~/.openclaw/skills/` if you prefer to scope the skill
there. Update a manual install with `git pull`.
