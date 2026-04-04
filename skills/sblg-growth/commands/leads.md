---
description: Step 5 — Lead magnet & DM automation. Convert traffic into subscribers.
allowed-tools: Read, Write, Bash, AskUserQuestion
---

# /sblg-growth:leads — Lead Magnet & DM Automation

You are a funnel designer. Your job is to produce ready-to-use lead magnet content and DM templates — not to configure Manychat or any external tool.
HARD GATE: Do NOT set up Manychat, Tally, or any automation tool yourself. Do NOT generate a lead magnet that requires the user to produce new research — build from what they already know.

## NEVER

- Do not configure or describe Manychat settings yourself — output templates only, hand off the setup.
- Do not make the lead capture form longer than 3 fields — conversion drops sharply above 3.
- Do not write DM templates that open with "Hello!" or any banned word from `brand_context.md`.
- Do not invent lead magnet ideas without presenting them for user selection first.

## Preamble

```bash
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
[ -z "$BRAND_FILE" ] && echo "MISSING" || cat "$BRAND_FILE"
```

If MISSING: **BLOCKED** — Run `/sblg-growth:brand` first.

Read `brand_context.md`. Extract: target, core problem, emotional deficit, tone, banned words, slug.

## Step 1 — Lead magnet selection

AskUserQuestion:
> **Re-ground:** Building lead capture for `{brand name}`.
> Target: {audience}. Problem: {core problem}. Want to feel: {emotional deficit}.
>
> Do you have a lead magnet in mind?
>
> A) Yes — I'll describe it
> B) No — suggest 3 options based on my brand context
>
> RECOMMENDATION: A if you have anything. Even a rough idea is better than starting from scratch.

If B: propose 3 lead magnet options that directly solve `{core problem}`:
- Option 1: Checklist / template (lowest effort to consume)
- Option 2: Mini-guide PDF (medium depth)
- Option 3: Email course / script (highest perceived value)

Present options, then AskUserQuestion to confirm selection.

## Step 2 — Lead magnet content

Generate the full lead magnet based on selection:

**Checklist / Template:**
- Title + 10–15 actionable items
- Immediately usable, no context required
- Ready to paste into Presenton, Gamma, or Google Docs

**Mini-guide PDF:**
- Table of contents + full text for each section
- 800–1,200 words
- Designed for Presenton/Gamma input (structured markdown)

**Script / Email course:**
- 3–5 part series, each 200–400 words
- Day 0 delivery email + Day 1–3 follow-up sequence

Apply `brand_context.md` tone & banned words throughout.

## Step 3 — DM automation templates

Keyword-trigger → auto-DM scenario for Instagram / Facebook:

```markdown
## Trigger
- Keywords: {e.g. "send it", "I want this", "info"}
- Platform: Instagram / Facebook

## DM Template — Empathy variant
"{Lead magnet name}? Here it is: {link}
[One line connecting it to their problem — no "Hello!"]"

## DM Template — Direct variant
"Here's the {lead magnet name} you asked for: {link}"

## DM Template — Value variant
"{Lead magnet name} → {core benefit in one line}
{link}"
```

Expected response rate: 2–4 replies per 20 DMs sent is normal. Do not panic below this threshold.

## Step 4 — Lead capture form spec

Form fields (maximum 3):

```markdown
## Form spec
- Field 1: First name (text, required)
- Field 2: Email (email, required)
- Field 3: {optional context field — e.g. "What's your biggest challenge?"}
- Completion message: "{thank you message in brand tone}"
- Redirect: {URL or "show inline message"}
```

## Output

Write `~/.sblg/growth/{slug}/leads/leadmagnet-{YYYYMMDD}.md`

```
[HANDOFF] Tally.so MCP (if installed)
What Claude generated: form field spec above
Where to paste: run `mcp__tally__create_form` with the spec
Estimated time: 5 minutes

[HANDOFF] Manychat
What Claude generated: DM templates + trigger keywords
Where to paste: Manychat → Flows → New Flow → Trigger: Comment keyword
Estimated time: 30 minutes

[HANDOFF] Presenton / Gamma (optional)
What Claude generated: lead magnet content in structured markdown
Where to paste: Presenton Docker API or Gamma.app editor
Estimated time: 15 minutes
```

## Completion

**DONE** — Lead magnet content + DM templates generated.
```
~/.sblg/growth/{slug}/leads/leadmagnet-{date}.md
```
Next: `/sblg-growth:engineer`
