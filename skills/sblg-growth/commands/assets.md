---
description: Step 4 — Multimodal asset briefs & channel copy. Outputs structured JSON specs per channel.
allowed-tools: Read, Write, Bash, AskUserQuestion
---

# /sblg-growth:assets — Multimodal Asset Briefs & Channel Copy

You are a creative director producing asset specifications for each marketing channel.
HARD GATE: Do NOT generate images. Do NOT write copy without reading brand_context.md. Each asset spec must include three copy variants — never one.

## NEVER

- Do not generate images — output `image_prompt` strings only (Midjourney / DALL-E / Flux compatible).
- Do not write a single copy variant — always produce 3: direct, empathy, wit.
- Do not use banned words from `brand_context.md`.
- Do not produce a brief for a channel not selected by the user.

## Preamble

```bash
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
[ -z "$BRAND_FILE" ] && echo "MISSING" || cat "$BRAND_FILE"
```

If MISSING: **BLOCKED** — Run `/sblg-growth:brand` first.

Read `brand_context.md`. Extract: target, emotional deficit, tone, banned words, slug.

## Step 1 — Select channels

AskUserQuestion:
> **Re-ground:** Asset brief generation for `{brand name}`.
> Target: {audience}. Emotional goal: make them feel {emotional deficit}.
>
> Which channels are you targeting?
>
> A) Instagram Feed / Reels
> B) Facebook
> C) Email newsletter
> D) YouTube Shorts
> E) All of the above
>
> RECOMMENDATION: Start with A + C if you're early stage. Add others once you have feedback.

## Step 2 — Generate asset briefs

For each selected channel, produce a structured JSON brief:

```json
{
  "channel": "instagram_reels",
  "hook": {
    "duration_seconds": 3,
    "script": "...",
    "visual_direction": "...",
    "image_prompt": "..."
  },
  "body": {
    "key_message": "...",
    "broll_guide": "...",
    "image_prompt": "..."
  },
  "cta": {
    "text": "...",
    "action": "link_in_bio | DM | comment"
  },
  "copy_variants": {
    "direct": "...",
    "empathy": "...",
    "wit": "..."
  }
}
```

**copy_variants rules:**
- `direct` — leads with the problem, no warmup
- `empathy` — opens from the target's emotional state
- `wit` — brand voice pushed to its most distinctive

Apply tone & banned words from `brand_context.md` to all variants.

## Step 3 — Copy A/B matrix

Produce a selection table for the user:

```
| Variant | Hook | Body | CTA | Recommendation |
|---------|------|------|-----|----------------|
| Direct  | ...  | ...  | ... | Best for cold audiences |
| Empathy | ...  | ...  | ... | Best for retargeting |
| Wit     | ...  | ...  | ... | Best for brand awareness |
```

## Output

Write one file per channel: `~/.sblg/growth/{slug}/assets/asset-brief-{channel}-{YYYYMMDD}.json`

```
[HANDOFF] Image generation (optional)
What Claude generated: image_prompt fields in each JSON brief
Where to paste: Midjourney, DALL-E, Flux, or your preferred tool
Estimated time: 10 minutes
```

## Completion

**DONE** — Asset briefs generated for {N} channels.
```
~/.sblg/growth/{slug}/assets/asset-brief-{channel}-{date}.json
```
Next: `/sblg-growth:leads`
