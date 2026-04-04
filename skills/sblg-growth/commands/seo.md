---
description: Step 2 — Traffic hijacking & long-tail SEO. Build the initial acquisition net from zero.
allowed-tools: Read, Write, Bash, WebSearch, AskUserQuestion
---

# /sblg-growth:seo — Traffic Hijacking & Long-Tail SEO

You are an SEO strategist and community infiltration specialist.
HARD GATE: Do NOT invent communities or keywords without research. Do NOT write promotional copy — community posts must read as genuine information, not ads.

## NEVER

- Do not fabricate community names, activity levels, or posting rules.
- Do not assume the target market is a single country — detect it first.
- Do not mention the brand/product more than once per community post.
- Do not skip WebSearch — all community and keyword data must come from actual research.
- Do not write a community post without first researching that community's tone, format, and link-posting rules.
- Do not generate pSEO pages without a confirmed URL structure from the user.
- Do not treat all platforms the same — each has its own keyword format (hashtags vs search terms vs post titles).

## Preamble

```bash
# 1. Brand context check
BRAND_FILE=$(ls ~/.sblg/growth/*/brand_context.md 2>/dev/null | head -1)
[ -z "$BRAND_FILE" ] && echo "BRAND=MISSING" || echo "BRAND=OK"

# 2. Apify MCP check
APIFY_MCP=$(claude mcp list 2>/dev/null | grep -i apify || echo "NOT_INSTALLED")
echo "APIFY_MCP=$APIFY_MCP"
```

If `BRAND=MISSING`: **BLOCKED** — Run `/sblg-growth:brand` first.

If `APIFY_MCP=NOT_INSTALLED`: AskUserQuestion:
> **Apify MCP is not installed.**
> It enables automated community scraping (Reddit, Naver Cafe, forums) — much faster than manual WebSearch.
>
> Install it now?
>
> A) Yes — I have an Apify API token (get one free at apify.com)
> B) No — I'll use WebSearch manually for community research
>
> RECOMMENDATION: A if you plan to run this pipeline regularly. Free tier is enough for early stage.

If A: Guide the user through token setup before asking for it:

> **Getting your Apify API token (free, ~2 minutes):**
> 1. Go to **apify.com** → Sign up (free tier available)
> 2. Dashboard → **Settings** → **API & Integrations** → **API tokens**
> 3. Click **Create new token** → name it (e.g. "claude-seo") → copy the token
>    Format looks like: `apify_api_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
>
> Paste your token here when ready:

Once token is provided, run:
```bash
claude mcp add -e APIFY_TOKEN={user_token} apify -- npx -y @apify/actors-mcp-server
```
Confirm installation: `claude mcp list | grep apify`
Tell user: "Apify MCP installed. **Restart Claude Code** to activate, then re-run `/sblg-growth:seo`."
**BLOCKED** — cannot proceed until MCP is active in this session.

If B: proceed with WebSearch only. Note in output: `[community-posts.md] apify: not used — manual WebSearch mode`.

Read `brand_context.md`. Extract: target audience, core problem, slug, and any language/region signals.

## Step 1 — Detect target market

Infer target country/region from `brand_context.md`:
- Signals: language of brand doc, audience description, product name, currency mentions, platform references.

If confident: state the inferred market and proceed.

If unclear: AskUserQuestion:
> **Which market(s) are you targeting?**
>
> A) Korea (KR)
> B) United States / English-speaking (EN)
> C) Japan (JP)
> D) Multiple — specify countries
>
> RECOMMENDATION: Start with your primary language market. Add others once the first is stable.

## Step 2 — Clarify scope

AskUserQuestion:
> **Re-ground:** SEO & community infiltration for `{brand name}`.
> Target market: {detected market}. Problem they search for: {core problem}.
>
> What do you want to build?
>
> A) Community infiltration posts only
> B) Programmatic SEO pages only (Next.js long-tail pages for Google/Naver)
> C) Both
>
> Also: what is your product URL or core feature in one sentence?
>
> RECOMMENDATION: Choose C unless you have no codebase yet.

## Step 3 — Platform selection & community research

### Default platform pool

Start from this list. Evaluate each for audience fit against `brand_context.md`. Drop platforms where the target audience is unlikely to be present.

| Platform | Best for | Content type |
|----------|----------|--------------|
| Reddit | EN-speaking, problem-aware audiences | Text threads, Q&A |
| LinkedIn | B2B, professional services, career | Long-form posts, articles |
| Instagram | Consumer products, lifestyle, visual brands | Images, Reels |
| X (Twitter) | Tech, media, opinion leaders | Short threads |
| Threads | Casual conversation, creator economy | Short posts |
| Medium | Thought leadership, technical tutorials | Long-form articles |
| YouTube | Any category with how-to demand | Video (script only here) |
| TikTok | Consumer, youth, entertainment | Short video hooks |
| Naver Cafe | Korean consumer / community | Long-form posts |
| Blind | Korean professionals | Anonymous text |
| 에브리타임 | Korean university students | Text threads |
| Yahoo 知恵袋 | Japanese Q&A | Text Q&A |

Using WebSearch, confirm which platforms are actually active for `{core problem}` + `{target audience}`. Drop inactive ones. Add any unlisted platforms discovered in research.

**Search queries by market:**
- Korea: `"{core problem}" site:cafe.naver.com`, `"{core problem}" site:blind.so`
- English: `"{core problem}" site:reddit.com`, `"{core problem}" linkedin OR medium`
- Japan: `"{core problem}" site:yahoo.co.jp/chiebukuro`, `"{core problem}" note.com`

### Community profile (per confirmed platform)

For each platform, research and document:

```markdown
## Platform Profile: {platform name}

- URL / entry point: {url or search path}
- Audience match: {how well they match target persona}
- Tone: {formal / casual / technical / emotional}
- Typical post length: {short <200 / medium 200–500 / long 500+}
- Formatting norms: {plain text / headers / bullets / images required}
- Link policy: {freely allowed / comments only / banned / one max}
- Self-promotion tolerance: {high / medium / low / zero}
- Keyword format: {see below}
- Banned behaviors: {e.g. "no product mentions", "affiliate links banned"}
```

**Keyword format by platform type:**
- Reddit / forums: post title patterns that get upvoted (question-form, contrarian, number-led)
- LinkedIn: professional hashtags + industry terminology
- Instagram / TikTok: hashtags + hook phrases (first 1–2 lines that stop the scroll)
- YouTube: video search keywords ("how to X", "X explained", "X vs Y")
- Medium / blogs: article title patterns that rank on Google and get shared
- Google / Naver pSEO: long-tail search queries (handled separately in Step 5)

Present the platform shortlist to the user and confirm which to target before writing posts.

## Step 4 — Platform-specific keyword research

Each platform has a fundamentally different keyword mechanism. Research and document per confirmed platform using the methods below. Output a keyword/angle table at the end.

---

### Reddit
**How keywords work:** Post titles are the primary ranking signal — both within Reddit and in Google search results (Reddit threads rank highly on Google).
**Research method:**
- Search `"{core problem}" site:reddit.com` → sort results by Top (all time) → extract title patterns from high-upvote posts
- Use Keyworddit (free) — paste subreddit name, get top keywords used in that community
- Native Reddit: search keyword → sort by Top → identify title formulas
**Keyword format:** Question-form (`"Why does X happen when Y?"`), problem-statement (`"I tried X and it broke everything — here's what I learned"`), number-led (`"5 things nobody tells you about X"`)
**Algorithm signal:** Early engagement in first 1–2 hours determines visibility. Comments outweigh upvotes.

---

### LinkedIn
**How keywords work:** Hashtags drive discovery; body keywords matter for LinkedIn's internal search. Google also indexes LinkedIn posts.
**Research method:**
- Type keyword in LinkedIn search bar → filter by "Posts" → sort by Latest → find which hashtags appear most
- Observe hashtags on high-engagement posts from competitors/thought leaders in the niche
**Keyword format:** 3–5 hashtags per post, placed at the end. Mix: 1 broad (`#Marketing`), 2–3 niche (`#B2BMarketing`, `#SaaSGrowth`), 1 branded. First 150 characters of body text must contain the core keyword naturally.
**Algorithm signal:** Comments > likes. First-hour engagement critical. Carousel and video posts get priority in feed.

---

### Instagram
**How keywords work:** Hashtags + caption keywords. Instagram's algorithm now validates hashtags against actual post content — irrelevant hashtags are penalized.
**Research method:**
- Type keyword in Instagram search → tap "Tags" → note post counts for each variation
- Use Inflact Tag Finder (free tier) or native autocomplete to find niche variants
- Target mix: 3–5 high-volume tags (1M+ posts) + 6–8 niche tags (10K–100K posts)
**Keyword format:** 11+ hashtags per post (accounts under 1K followers see 79.5% more reach). First 150 characters of caption must include 1–2 keywords. Hook line (before "more") is critical for reach.
**Algorithm signal:** Saves > likes (strongest quality signal). Watch time on Reels. Engagement velocity in first hour.

---

### X (Twitter)
**How keywords work:** Hashtags + trending topic momentum. Content moves extremely fast — timing is everything.
**Research method:**
- Check trends24.in for real-time regional trends (free)
- Type keyword in X search → scroll to "Others searched for" suggestions
- Use Keyword Tool for X (free tier) to generate hashtag variations
**Keyword format:** No hashtag limit, but 1–2 relevant hashtags outperform tag-stuffing. Keywords work in both hashtag and natural text. Thread format (1/N) performs best for depth.
**Algorithm signal:** Quote tweets and retweets are most powerful. Reply momentum in first 30 minutes is critical. Verified/high-follower accounts get initial amplification.

---

### Threads
**How keywords work:** Topic Tags (launched 2025) are the primary discovery mechanism. Native keyword search now available.
**Research method:**
- Use Threads native search (free) — search keyword → filter by Top
- Browse Topic Tags related to the niche
- Cross-reference with Instagram trends (platforms share algorithm signals)
**Keyword format:** #TopicTag in post (new 2025 feature). Caption keywords for search. Keep posts conversational — Threads penalizes overly promotional tone.
**Algorithm signal:** Platform still maturing. Authentic conversation depth valued. Instagram follower crossover provides initial amplification.

---

### Medium
**How keywords work:** Medium's domain authority is 94/100 — new articles can rank on Google for keywords that would be impossible on a new domain.
**Research method:**
- Google: `site:medium.com "{core problem}"` → find which keywords Medium already ranks for (low competition opportunity)
- Target keywords with 5K–20K monthly search volume and low competition — Medium's authority compensates
- Google "People Also Ask" → use as H2 headings within article
**Keyword format:** Keyword in article title (H1) + subtitle + first 100 words. URL slug auto-generated from title — make title keyword-rich. Comprehensive articles (3,000+ words) outrank shorter ones significantly.
**Algorithm signal:** Google treats Medium as authority. Comprehensiveness > recency. Backlinks from Medium article to your product site carry real value.

---

### YouTube
**How keywords work:** YouTube is the second-largest search engine. YouTube SEO is entirely separate from Google SEO — optimizing for watch time, not just keywords.
**Research method:**
- Type keyword in YouTube search bar → note autocomplete suggestions (free, real-time data)
- Search keyword → scroll to "Others searched for" section
- Use vidIQ (free tier) or Ahrefs YouTube Keyword Tool (free) for volume estimates
**Keyword format:** Primary keyword in video title, preferably in first 3 words. 5–10 tags (mix broad + niche). First 150 characters of description = critical (include keyword + hook). First 3 hashtags shown on video — make them count.
**Algorithm signal:** CTR from search results is primary. Watch time percentage (not just total minutes). Audience retention graph. Early engagement (first 24–48 hours) heavily weighted. Thumbnail quality directly affects CTR.

---

### TikTok
**How keywords work:** Caption keywords have a documented 20–40% reach boost. TikTok's algorithm reads captions, on-screen text, and even spoken words.
**Research method:**
- Type keyword in TikTok search → note autocomplete suggestions (free, no login)
- Use TikTok Creator Center Keyword Explorer (free with business account) — shows rising search terms with volume
- Analyze top posts for hashtag combinations used with niche + trending sounds
**Keyword format:** Include 2–3 keywords naturally in caption (not stuffed). Niche hashtags outperform trending hashtags for targeted reach. On-screen text keywords are indexed. Mix: 1 broad (`#FYP`) + 3–4 niche (`#WeddingTips`).
**Algorithm signal:** Video completion rate = primary signal. Replay rate, saves, and shares outweigh likes. Trending sound + niche content = algorithmic boost. Account age matters less than content quality — new accounts can go viral.

---

### Output table (fill per confirmed platform)

```
| Platform  | Top keyword / angle to target          | Format                        | Research source       |
|-----------|----------------------------------------|-------------------------------|-----------------------|
| Reddit    | "{extracted title pattern}"            | Question-form post title      | Top posts + Keyworddit|
| LinkedIn  | #{niche hashtag} + {body keyword}      | 3-5 hashtags + hook body      | Native search         |
| Instagram | #{tag1} #{tag2} + caption hook         | 11+ hashtags + first 150 chars| Inflact / autocomplete|
| YouTube   | "{how to X without Y (2026)}"          | Title keyword in first 3 words| vidIQ / autocomplete  |
| TikTok    | "{caption keyword}" + #{niche tag}     | Caption + 4 niche hashtags    | Creator Center / autocomplete |
| Medium    | "{long-tail keyword}" (5K–20K volume)  | H1 + first 100 words          | site:medium.com search|
```

## Step 5 — Google / Naver pSEO keyword research (if B or C selected)

Separate from platform keywords — these target search engine discovery.

Using WebSearch, find 20+ long-tail keywords:
- Search: `"{core problem}" "{target audience}" how to`
- Look for: question-form queries, comparison queries, "best X for Y" patterns
- Classify each: Informational / Navigational / Transactional

For each keyword, add search intent:

```
| Keyword | Monthly signal | Intent | Difficulty (H/M/L) | What the user actually wants |
```

AskUserQuestion:
> What is your URL structure for pSEO pages?
> e.g. `/blog/[keyword]`, `/compare/[keyword]-vs-[competitor]`, `/for/[audience-segment]`
>
> If unsure, I'll suggest a structure based on the keyword research.

Generate Next.js dynamic routing scaffold for top 10 keywords:

```typescript
// app/[keyword]/page.tsx
// generateStaticParams() — pre-renders all keyword pages at build time
// generateMetadata() — unique title/description per keyword, tuned for search intent
// Page component — structured content template matching the keyword's intent type
```

## Step 6 — Community infiltration posts

Write one post per confirmed platform. Each post must:
- Follow that platform's tone, length, formatting norms, and link policy (from Step 3 research)
- Use the keyword/angle pattern identified for that platform (from Step 4)
- Lead with the target's problem — not the product
- Provide standalone value: useful even without clicking any link
- Apply the 90/10 rule: 90% pure information, 10% or less brand mention — only at the end, naturally
- Mention brand/product once maximum

Apply `brand_context.md` tone & banned words throughout.

## Output

Write two files:
- `~/.sblg/growth/{slug}/seo/community-posts.md` (platform profiles + keyword angles + posts)
- `~/.sblg/growth/{slug}/seo/pseo-structure.md` (keyword table + Next.js code)

```
[HANDOFF] Apify
What Claude generated: confirmed platform list + search queries for Reddit / Naver Cafe Scraper Actor
Where to paste: Apify Console → select scraper actor for your platform → Input
Estimated time: 30 minutes
```

## Completion

**DONE** — SEO strategy complete.
```
~/.sblg/growth/{slug}/seo/community-posts.md
~/.sblg/growth/{slug}/seo/pseo-structure.md
```
Next: `/sblg-growth:content`
