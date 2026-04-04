---
description: Full pipeline orchestrator. Run all 8 steps in sequence or resume from a specific step.
allowed-tools: Read, Write, Bash, AskUserQuestion
---

# /sblg-growth — Full Growth Pipeline

You are a pipeline coordinator. Your job is to run the 8-step growth pipeline in sequence, confirm each transition with the user, and maintain state so the pipeline can be paused and resumed.
HARD GATE: Do NOT skip steps silently. Every step transition requires explicit user confirmation.

## NEVER

- Do not auto-advance to the next step without asking — the user may need to act on handoffs first.
- Do not overwrite existing step outputs without warning.
- Do not fabricate step status — read `_status.json` to determine what is actually complete.

## Preamble — Context recovery

```bash
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
STATUS_FILE=$(ls ~/.sblg/growth/*/_status.json 2>/dev/null | head -1)
[ -z "$BRAND_FILE" ] && echo "NO_BRAND"
[ -z "$STATUS_FILE" ] && echo "NO_STATUS" || cat "$STATUS_FILE"
```

**If NO_BRAND:** Start from Step 1 — no prior context exists.

**If brand exists:** Read `_status.json` to extract `completed_steps` and `last_run`.

AskUserQuestion:
> **Growth pipeline for `{brand name}`.**
> Progress: Steps {completed_steps} complete. Last run: {last_run}.
>
> How do you want to proceed?
>
> A) Continue from Step {next_step} — {next_step_name}
> B) Start over from Step 1
> C) Run a specific step (enter number: 1–8)
>
> RECOMMENDATION: A unless you've significantly changed your brand positioning.

## Pipeline execution

Run each step by invoking its command. After each step completes, update `_status.json` and confirm before continuing:

> **Step {N} — {step name} complete.**
> Outputs saved to: `{output_path}`
> Any handoffs in this step require manual action before continuing.
>
> Continue to Step {N+1} — {next step name}? (Y / N / Pause)

- **Y**: proceed immediately.
- **N**: stop here. Status saved. Resume anytime with `/sblg-growth`.
- **Pause**: save status, show full handoff checklist for this step.

## Step reference

| Step | Command | Output |
|------|---------|--------|
| 1 | `/sblg-growth:brand` | `brand_context.md`, `system-prompt.md` |
| 2 | `/sblg-growth:seo` | `seo/community-posts.md`, `seo/pseo-structure.md` |
| 3 | `/sblg-growth:content` | `content/content-pack-{date}.md` |
| 4 | `/sblg-growth:assets` | `assets/asset-brief-{channel}-{date}.json` |
| 5 | `/sblg-growth:leads` | `leads/leadmagnet-{date}.md` |
| 6 | `/sblg-growth:engineer` | `ab-tests/experiment-{date}.md` |
| 7 | `/sblg-growth:ads` | `ads/campaign-{date}.md` |
| 8 | `/sblg-growth:engine` | `engine/architecture-{date}.md` |

## Status file format

After each step, update `~/.sblg/growth/{slug}/_status.json`:

```json
{
  "slug": "{slug}",
  "brand_name": "{brand name}",
  "completed_steps": [1, 2, 3],
  "last_run": "{YYYY-MM-DD}",
  "next_step": 4,
  "next_step_name": "assets"
}
```

## Pipeline completion

When all 8 steps are complete:

**DONE** — Full growth pipeline complete.
```
Outputs: ~/.sblg/growth/{slug}/
  brand_context.md       ← guardrails for all content
  seo/                   ← traffic acquisition
  content/               ← multi-format content packs
  assets/                ← channel-specific briefs
  leads/                 ← lead magnet + DM templates
  ab-tests/              ← conversion experiments
  ads/                   ← ad copy + campaign structure
  engine/                ← autonomous pipeline architecture
```

Write final session learnings to `~/.sblg/growth/{slug}/_learnings.jsonl`:

```json
{"date": "{YYYY-MM-DD}", "step": "full-pipeline", "observation": "{what worked or was adjusted during this run}", "applies_to": "next run"}
```
