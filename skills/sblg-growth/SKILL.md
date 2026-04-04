---
name: sblg-growth
description: >-
  AI-powered marketing & growth hacking pipeline — brand context injection,
  SEO traffic hijacking, OSMU content factory, multimodal asset briefs,
  lead magnet automation, growth engineering, performance ads, autonomous engine.
  Use when: "marketing", "growth", "content", "ads", "SEO", "sblg-growth".
license: MIT
metadata:
  author: sblg
  version: "1.0.0"
---

# sblg-growth — AI Marketing & Growth Pipeline

Every step reads `brand_context.md` first. Without it, every command produces AI Slop.

## HARD GATE — Applied to ALL commands

Before executing any step:

```bash
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
echo "${BRAND_FILE:-NOT_FOUND}"
```

If `NOT_FOUND`: Stop immediately. Output:
> "brand_context.md not found. Run `/sblg-growth:brand` first."

## NEVER (across all commands)

- Do not invent target personas. Read them from `brand_context.md` only.
- Do not use banned words from `brand_context.md` in any generated copy.
- Do not claim Claude can execute external tool setup (Manychat, Meta Ads, etc.) — generate artifacts and hand off.
- Do not skip steps without explicit user confirmation.
- Do not proceed with empty required fields — block and ask.

## Handoff Format

When external tooling is required, always use this exact format:

```
[HANDOFF] {Tool name}
What Claude generated: {artifact description}
Where to paste: {exact UI location}
Estimated time: {N minutes}
```

## Completion Status

Every command ends with exactly one of:
- **DONE** — completed, list output paths
- **DONE_WITH_CONCERNS** — completed, state the concern
- **BLOCKED** — cannot proceed, state what is missing and the exact command to fix it
- **NEEDS_CONTEXT** — missing required input, state exactly what is needed

## Output Structure

```
~/.sblg/growth/{project-slug}/
├── _status.json
├── brand_context.md
├── system-prompt.md
├── seo/
├── content/
├── assets/
├── leads/
├── ab-tests/
├── ads/
└── engine/
```

## _status.json Format

```json
{
  "project": "",
  "slug": "",
  "started_at": "",
  "last_updated": "",
  "completed_steps": [],
  "current_step": 0,
  "next_action": ""
}
```

## Commands

| Command | Step | Description |
|---------|------|-------------|
| `/sblg-growth:brand` | 1 | Brand context setup — generates brand_context.md + system-prompt.md |
| `/sblg-growth:seo` | 2 | Traffic hijacking & long-tail SEO |
| `/sblg-growth:content` | 3 | OSMU content factory |
| `/sblg-growth:assets` | 4 | Multimodal asset briefs & channel copy |
| `/sblg-growth:leads` | 5 | Lead magnet & DM automation |
| `/sblg-growth:engineer` | 6 | Growth engineering & A/B testing |
| `/sblg-growth:ads` | 7 | Performance marketing |
| `/sblg-growth:engine` | 8 | Autonomous growth engine |
| `/sblg-growth` | — | Full pipeline orchestrator |
