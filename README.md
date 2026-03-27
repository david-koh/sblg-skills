# sblg-skills

First-principles thinking skills for AI agents. Built on the [Agent Skills](https://agentskills.io) open standard.

Works with Claude Code, OpenAI Codex, Cursor, Gemini CLI, and [30+ other agents](https://agentskills.io).

## Installation

```bash
npx skills add david-koh/sblg-skills
```

Install a specific skill:

```bash
npx skills add david-koh/sblg-skills --skill sblg-decide
```

### Claude Code plugin

```bash
/plugin marketplace add david-koh/sblg-skills
/plugin install sblg-skills@sblg-skills-marketplace
```

## Skills

| Skill | Use when | Description |
|-------|----------|-------------|
| **sblg-decide** | "A vs B", tradeoff analysis | Expert panel (pro/con/neutral) → atomic decomposition → cross-domain precedent mapping → Decision Record |
| **sblg-solve** | Stuck on a problem, need a breakthrough | Expert panel diagnosis → first-principles decomposition → cross-domain pattern mapping → 3 solution tiers |
| **sblg-learn** | Learning a new domain from scratch | Expert panel → atomic fundamentals → cross-domain connections → roadmap → interactive study sessions |

All three skills share the same pipeline: **Expert Input → First-Principles Decomposition → Cross-Domain Mapping → Transfer Learning**.

## How it works

1. **Expert Panel** — Auto-generates 3-5 top-tier expert personas who independently analyze your input
2. **First-Principles Decomposition** — Strips emotion, convention, and marketing to extract real constraints (constants) vs removable assumptions (variables)
3. **Cross-Domain Mapping** — Maps structural similarities from domains you already know
4. **Transfer Learning** — Applies patterns from other domains to produce actionable output

## License

[MIT](LICENSE)
