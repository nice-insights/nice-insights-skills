# nice-insights-skills

Shared agent skills for the Nice Insights clients and internal team. Supports both Claude Code and OpenClaw.

## Skills

| Skill | Runtime | Description |
|-------|---------|-------------|
| `nice-insights-metrics` | Claude Code | Query ecommerce metrics — ad spend, sales, orders, margins, CAC, and more |
| `nice_insights_metrics` | OpenClaw | Same skill, tuned for the OpenClaw runtime (lives under `openclaw/`) |

## Setup — Claude Code

### Option 1: Install with npx skills (recommended)

```bash
npx skills add nice-insights/nice-insights-skills -g -a claude-code -y
```

This installs globally and creates symlinks from each skill into `~/.claude/skills/`.

To update to the latest version:

```bash
npx skills update
```

### Option 2: Clone and symlink manually

1. Clone this repo:
   ```bash
   git clone <repo-url> ~/src/nice-insights-skills
   ```

2. Symlink the skill(s) you want into `~/.claude/skills/`:
   ```bash
   ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.claude/skills/nice-insights-metrics
   ```

Skills will then appear in Claude Code's `/` autocomplete in any project.

## Setup — OpenClaw

Clone this repo and symlink the OpenClaw skill into `~/.openclaw/skills/`:

```bash
git clone <repo-url> ~/src/nice-insights-skills
ln -s ~/src/nice-insights-skills/openclaw/nice_insights_metrics ~/.openclaw/skills/nice_insights_metrics
```
