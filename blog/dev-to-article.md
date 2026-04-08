---
title: "Claude Code's Thinking Tokens Count Against Your Quota — But You Can't See Them"
published: false
description: Proxy data from 17,610 API requests reveals that visible output explains less than half of Claude Code's quota consumption. Extended thinking tokens appear to be the missing cost — invisible to users, impossible to monitor.
tags: claudecode, ai, debugging, opensource
---

A full 5-hour quota window on Claude Code Max 20 ($200/month) produces only **0.9M-1.6M visible output tokens**. That's remarkably low for several hours of Opus work.

Where does the rest go? Extended thinking tokens are **not included** in the `output_tokens` field from the API response. You can't see them in JSONL session logs, you can't count them through a proxy, and Claude Code provides no way to monitor or control them. Yet the quota drains at a rate consistent with thinking tokens being counted at the output token rate.

This isn't speculation. Here's how we measured it.

## The Incident

On April 1, 2026, my Max 20 subscription hit 100% usage in **70 minutes** during normal coding. JSONL analysis showed cache read averaging **36.1%** where it should be 90%+. Downgrading from v2.1.89 to v2.1.68 recovered cache to **97.6%** immediately.

Instead of filing another "me too" issue, I built a transparent monitoring proxy.

## Methodology

[cc-relay](https://github.com/ArkNill/claude-code-hidden-problem-analysis) sits between Claude Code and the Anthropic API using the official `ANTHROPIC_BASE_URL` environment variable. It logs every request and response — including `anthropic-ratelimit-unified-*` headers — without modifying anything.

Over a full week: **17,610 requests** logged, **3,702** with rate limit headers captured. Source-audited, reproducible.

## The Numbers

The proxy revealed the server-side quota architecture: **dual sliding windows** — a 5-hour counter and a 7-day counter. In 100% of captured requests, the 5h window is the bottleneck.

**Per-1% of 5h utilization** (Max 20, $200/mo):

| Metric | Range |
|--------|-------|
| Visible output per 1% | 9K-16K tokens |
| Cache read per 1% | 1.5M-2.1M tokens |
| Total visible per 1% | 1.5M-2.1M tokens |

At 9K-16K visible output per 1%, using the full 5h window (100%) means only 0.9M-1.6M visible tokens of output. For context, that's roughly the output of 30-50 moderately sized code generations across an entire work session.

The gap between visible output and actual quota cost is consistent with **extended thinking tokens being the dominant cost factor** — but completely invisible to users.

## Independent Confirmation

Community members reached the same conclusion through separate investigations:

**[@fgrosswig](https://github.com/fgrosswig)** published a [64x budget reduction forensic](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) — 18-day dual-machine JSONL comparison showing March 26: 3.2B tokens with no limit → April 5: 88M at 90% utilization. The visible token count doesn't explain the consumption.

**[@Commandershadow9](https://github.com/Commandershadow9)** documented a [34-143x capacity reduction](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296), confirmed that cache fixes work, but showed the capacity drop is independent of any client-side bug. Thinking tokens were independently identified as the likely factor.

A thinking-disabled isolation test is planned for the week of April 6 to measure the exact impact.

## It Gets Worse: 5 More Bugs

The thinking token blind spot isn't the only problem. The investigation uncovered **6 confirmed client-side bugs across 4 layers**:

### Fixed (v2.1.91)

| Bug | What | Impact |
|-----|------|--------|
| **Sentinel** | Standalone binary breaks cache prefix | Cache drops to 4-17%, 20x cost ([#40524](https://github.com/anthropics/claude-code/issues/40524)) |
| **Resume** | `--resume` replays full context uncached | Full cache miss per resume on 500K conversation ([#34629](https://github.com/anthropics/claude-code/issues/34629)) |

**Update to v2.1.91 if you haven't already.** This fixes the single largest drain.

### Still Unfixed

| Bug | What | Impact |
|-----|------|--------|
| **Microcompact** | Server silently strips old tool results | 3,782 clearing events (15,998 items cleared) measured. Model loses earlier context ([#42542](https://github.com/anthropics/claude-code/issues/42542)) |
| **Budget cap** | 200K char aggregate cap on tool results | After ~15-20 file reads, older results truncated to 1-41 chars. 72,839 budget events (100% truncation rate) |
| **False rate limiter** | Client generates fake "Rate limit reached" | 151 synthetic errors across 65 sessions — zero API calls made ([#40584](https://github.com/anthropics/claude-code/issues/40584)) |
| **JSONL inflation** | Extended thinking duplicates log entries | 2.37x average token inflation (range 1.45x-4.42x, universal across 532 files) in local accounting ([#41346](https://github.com/anthropics/claude-code/issues/41346)) |

The microcompact and budget cap are particularly insidious: they're controlled by **server-side GrowthBook A/B testing flags**. Anthropic can change behavior without shipping a client update, and no environment variable overrides them. Users paying for 1M context effectively get a 200K tool result budget for built-in tools.

## Anthropic's Response

On April 2, Lydia Hallie (Anthropic, Product) [posted on X](https://x.com/lydiahallie/status/2039800715607187906):

> *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."*

Where our data diverges:

- **"None were over-charging you"** — Bug 5 truncates tool results to 1-41 chars after 200K. 72,839 budget events measured (100% truncation rate). Users paying for 1M context don't get 1M of tool results.
- **"Peak-hour limits are tighter"** — Proxy shows `representative-claim` = `five_hour` in 100% of 3,702 requests, regardless of time of day. Weekend data shows the same pattern.
- **Thinking tokens** — Not addressed. The dominant cost factor is invisible to users, with no way to monitor or control it.

Across **91+ GitHub issues** spanning 14 months, Anthropic has posted **zero official responses on GitHub**. All communication has been through personal X posts and the changelog.

## What You Can Do

1. **Update to v2.1.91** — fixes the cache regression
2. **Don't use `--resume` or `--continue`** — start fresh sessions instead
3. **Start new sessions periodically** — the 200K budget cap silently truncates older tool results
4. **Limit to one terminal** — multiple terminals don't share cache

Self-diagnosis guide: [09_QUICKSTART.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/09_QUICKSTART.md)

## This Was a Community Effort

This analysis builds on work from 17 community members who independently discovered different pieces of the puzzle:

- [@Sn3th](https://github.com/Sn3th) — discovered microcompact mechanisms (Bug 4), GrowthBook flags, and budget pipeline (Bug 5)
- [@rwp65](https://github.com/rwp65) — discovered the client-side false rate limiter (Bug 3) with detailed log evidence
- [@fgrosswig](https://github.com/fgrosswig) — 64x budget reduction forensics with dual-machine 18-day JSONL comparison
- [@Commandershadow9](https://github.com/Commandershadow9) — 34-143x capacity reduction analysis and thinking token hypothesis
- [@kolkov](https://github.com/kolkov) — built [ccdiag](https://github.com/kolkov/ccdiag), identified three v2.1.91 `--resume` regressions
- [@simpolism](https://github.com/simpolism) — resume cache fix patch (99.7-99.9% hit rate)
- [@luongnv89](https://github.com/luongnv89) — cache TTL analysis, built [CUStats](https://custats.info) and [context-stats](https://github.com/luongnv89/cc-context-stats)
- [@dancinlife](https://github.com/dancinlife) — organizationUuid quota pooling and oauthAccount cross-contamination bug
- [@weilhalt](https://github.com/weilhalt) — built [BudMon](https://github.com/weilhalt/budmon) for rate-limit header monitoring
- [@arizonawayfarer](https://github.com/arizonawayfarer) — Windows GrowthBook flag dumps confirming cross-platform consistency
- [@dbrunet73](https://github.com/dbrunet73) — real-world OTel comparison data (v2.1.88 vs v2.1.90) confirming cache improvement
- [@maiarowsky](https://github.com/maiarowsky) — confirmed Bug 3 on v2.1.90 with 26 synthetic entries across 13 sessions
- [@edimuj](https://github.com/edimuj) — measured grep/file-read token waste (3.5M tokens / 1,800+ calls), built [tokenlean](https://github.com/edimuj/tokenlean)
- [@amicicixp](https://github.com/amicicixp) — verified v2.1.90 cache improvement with before/after testing
- [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) — reported old Docker versions draining — key server-side evidence ([#37394](https://github.com/anthropics/claude-code/issues/37394))
- [@SC7639](https://github.com/SC7639) — additional regression data confirming mid-March timeline
- Reddit community — [reverse engineering analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) of the cache sentinel mechanism

## Full Analysis

**[github.com/ArkNill/claude-code-hidden-problem-analysis](https://github.com/ArkNill/claude-code-hidden-problem-analysis)**

11 documents: bug technical details, proxy-captured rate limit headers, JSONL forensics, benchmarks, 14-month timeline of 91+ issues, and community tools. April 8 full-week dataset, all data cross-verified.

---

*Community research and personal measurement. Not endorsed by Anthropic. All monitoring uses official tools and documented features.*
