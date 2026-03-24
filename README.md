# nice-insights-skills

Shared Claude Code skills for the Nice Insights clients and internal team. Clone this repo and symlink skills into your `~/.claude/skills/` directory.

## Skills

| Skill | Description |
|-------|-------------|
| `nice-insights-metrics` | Query ecommerce metrics — ad spend, sales, orders, margins, CAC, and more |

## Setup

1. Clone this repo:
   ```bash
   git clone <repo-url> ~/src/nice-insights-skills
   ```

2. Symlink the skill(s) you want into `~/.claude/skills/`:
   ```bash
   ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.claude/skills/nice-insights-metrics
   ```

Skills will then appear in Claude Code's `/` autocomplete in any project.
