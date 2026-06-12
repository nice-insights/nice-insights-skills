# nice-insights-skills

Shared agent skills for Nice Insights clients and the internal team.

These skills follow the open [Agent Skills](https://agentskills.io) standard, so the
same `SKILL.md` works across **Claude Code**, **OpenAI Codex**, and **OpenClaw** — no
runtime-specific copies.

## Skills

| Skill | Description |
|-------|-------------|
| [`nice-insights-metrics`](./nice-insights-metrics) | Query ecommerce metrics — ad spend, sales, orders, margins, CAC, retention/LTV, and email activity |

> **Source of truth:** `nice-insights-metrics/SKILL.md` is published from the
> `nice-chat` backend (`backend/chat/mcp/skills/`). Don't edit it here directly —
> change it in the backend and run `backend/scripts/publish_skill.sh`.

## Install

### Claude Code

Install with [`npx skills`](https://github.com/obra/skills) (recommended):

```bash
npx skills add nice-insights/nice-insights-skills -g -a claude-code -y
```

Update later with `npx skills update`. Or symlink manually:

```bash
git clone https://github.com/nice-insights/nice-insights-skills ~/src/nice-insights-skills
ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.claude/skills/nice-insights-metrics
```

### Codex / OpenClaw

Both read skills from `~/.agents/skills/` (the shared open-standard location):

```bash
git clone https://github.com/nice-insights/nice-insights-skills ~/src/nice-insights-skills
ln -s ~/src/nice-insights-skills/nice-insights-metrics ~/.agents/skills/nice-insights-metrics
```

OpenClaw also reads `~/.openclaw/skills/` if you'd rather scope it there. Update with `git pull`.
