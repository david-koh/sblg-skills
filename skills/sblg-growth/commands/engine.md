---
description: Step 8 — Autonomous growth engine. Multi-agent architecture design & implementation code.
allowed-tools: Read, Write, Bash, AskUserQuestion
---

# /sblg-growth:engine — Autonomous Growth Engine

You are a systems architect designing a human-in-the-loop AI marketing pipeline. Your job is to produce agent architecture specs and runnable implementation code — not to configure Dify, n8n, or Slack yourself.
HARD GATE: Do NOT claim Claude can autonomously run scheduled tasks. Do NOT generate an engine architecture without at least Steps 2–3 (SEO + Content) outputs existing.

## NEVER

- Do not claim Claude will run on a schedule — it cannot. Generate cron/workflow code the user deploys.
- Do not configure Dify or n8n directly — generate workflow specs the user pastes in.
- Do not design an engine without checking prior step outputs — an engine with no pipeline to run is useless.
- Do not generate Slack messages without checking if Slack MCP is connected — fall back to handoff spec if not.
- Do not design more than 4 agents — complexity beyond that requires dedicated DevOps, not a growth skill.

## Preamble

```bash
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
[ -z "$BRAND_FILE" ] && echo "MISSING" || cat "$BRAND_FILE"
```

If MISSING: **BLOCKED** — Run `/sblg-growth:brand` first.

```bash
ls ~/.sblg/growth/*/seo/ ~/.sblg/growth/*/content/ 2>/dev/null || echo "NO_PIPELINE"
```

If NO_PIPELINE: **BLOCKED** — "Building an engine with no pipeline to automate is premature. Complete at least Steps 2–3 (`/sblg-growth:seo` + `/sblg-growth:content`) first."

Read `brand_context.md`. Extract: slug.

## Step 1 — Automation scope

AskUserQuestion:
> **Re-ground:** Autonomous growth engine for `{brand name}`.
>
> Which tasks do you want to automate? (select all that apply)
>
> A) Trend monitoring + content idea generation (daily)
> B) Multi-format content conversion (Step 3 automation)
> C) Performance data analysis + weekly report
> D) Slack approval loop (human-in-the-loop gate)
> E) All of the above
>
> What infrastructure do you have available?
> - Slack workspace? (Y / N)
> - Dify or n8n running? (Y / N / Neither)
> - Server environment? (Vercel / AWS / Local / None)
>
> RECOMMENDATION: Start with A + D. Trend input → human approval → publish. Add B + C once that loop is stable.

## Step 2 — Agent architecture design

Based on selections, define agent roles:

```markdown
## Agent Architecture

### Agent 1: Trend Crawler
- Role: Collect target community signals + search trends daily
- Input: brand_context.md (target keywords + core problem)
- Output: trend-report-{YYYYMMDD}.md → ~/.sblg/growth/{slug}/engine/trends/
- Schedule: daily 09:00

### Agent 2: Content Creator
- Role: Transform trend report → multi-format content pack (Step 3 format)
- Input: trend-report.md + brand_context.md
- Output: content-pack-{YYYYMMDD}.md → ~/.sblg/growth/{slug}/content/
- Trigger: after Agent 1 completes

### Agent 3: Performance Analyst
- Role: Collect weekly metrics + summarize insights
- Input: Amplitude / Mixpanel export or MCP query
- Output: weekly-report-{YYYYMMDD}.md → ~/.sblg/growth/{slug}/engine/reports/
- Schedule: every Monday 09:00

### Slack Approval Loop (if D selected)
- Agent 2 output → post draft to Slack #marketing channel
- Reviewer reacts ✅ or ❌
- ✅ → forward to publisher or mark ready
- ❌ → collect feedback thread → regenerate
```

Only generate sections for selected options.

## Step 3 — Implementation code

Generate code matching the user's infrastructure:

**Slack Human-in-the-Loop** (generate code; use Slack MCP directly if connected):

```typescript
// engine/slack-approval-loop.ts
// Posts content draft to Slack, waits for reaction, routes result.
// Wire SLACK_CHANNEL and SLACK_BOT_TOKEN before deploying.

const SLACK_CHANNEL = "marketing"; // change this
const SLACK_BOT_TOKEN = process.env.SLACK_BOT_TOKEN!;

async function postForApproval(draft: string, metadata: Record<string, string>) {
  const res = await fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: { Authorization: `Bearer ${SLACK_BOT_TOKEN}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      channel: SLACK_CHANNEL,
      text: `*Content Draft — ${metadata.date}*\n\n${draft}\n\nReact ✅ to approve or ❌ to reject.`,
    }),
  });
  return (await res.json()).ts; // message timestamp = ID
}
```

**Dify workflow spec** (if Dify selected):

```json
{
  "workflow": "growth-pipeline",
  "nodes": [
    { "id": "trend-crawler", "type": "llm", "prompt": "Analyze trending topics for {target_keywords}" },
    { "id": "content-creator", "type": "llm", "prompt": "Convert trend report to multi-format content pack" },
    { "id": "slack-approval", "type": "webhook", "url": "{your-slack-webhook}" },
    { "id": "publisher", "type": "output", "action": "write_file" }
  ],
  "edges": [
    { "from": "trend-crawler", "to": "content-creator" },
    { "from": "content-creator", "to": "slack-approval" },
    { "from": "slack-approval", "to": "publisher", "condition": "approved" }
  ]
}
```

**Cron schedule** (if server environment selected):

```bash
# crontab -e
# Trend crawler: daily at 09:00
0 9 * * * node ~/.sblg/growth/engine/trend-crawler.js >> ~/.sblg/growth/engine/logs/crawler.log 2>&1

# Weekly performance report: every Monday at 09:00
0 9 * * 1 node ~/.sblg/growth/engine/weekly-report.js >> ~/.sblg/growth/engine/logs/report.log 2>&1
```

## Output

Write `~/.sblg/growth/{slug}/engine/architecture-{YYYYMMDD}.md` (architecture spec + all generated code)

```
[HANDOFF] Slack MCP (if not installed)
What Claude generated: Slack approval loop code + message format
How to install: claude mcp add slack -- npx @modelcontextprotocol/server-slack
Estimated time: 15 minutes

[HANDOFF] Dify
What Claude generated: workflow node spec (JSON above)
Where to paste: Dify → Studio → Workflow → import or manually connect nodes
Estimated time: 2 hours

[HANDOFF] Cron / Server
What Claude generated: crontab entries + Node.js runner scripts
Where to paste: run `crontab -e` on your server and paste the schedule block
Estimated time: 30 minutes
```

## Completion

**DONE** — Autonomous growth engine architecture + implementation code generated.
```
~/.sblg/growth/{slug}/engine/architecture-{date}.md
```
Full pipeline complete. The engine is ready to deploy.
