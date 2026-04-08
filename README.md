> **🇰🇷 [한국어 버전](ko/README.md)**

# Claude Code Hidden Problem Analysis

> **TL;DR:** Claude Code has **6 confirmed client-side bugs across 4 layers** that drain usage faster than expected. Cache bugs (B1-B2) are fixed in v2.1.91. Four remain unfixed (B3, B4, B5, B8). Additionally, proxy-captured rate limit headers reveal a **dual 5h/7d window quota system** with a significant **thinking token blind spot** — visible output explains less than half the observed utilization. All findings are backed by proxy-measured data.
>
> **Last updated:** April 6, 2026 — README restructured, rate limit header analysis added

---

## Latest Update (April 6)

### Rate limit header analysis — [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)

Transparent proxy (cc-relay) captured `anthropic-ratelimit-unified-*` headers across **3,702 requests** (April 4-6), revealing the server-side quota architecture:

**Dual sliding window system:**
- Two independent counters: **5-hour** (`5h-utilization`) and **7-day** (`7d-utilization`)
- `representative-claim` = `five_hour` in **100%** of requests — the 5h window is always the bottleneck
- 5h windows reset on roughly 5-hour intervals; 7d resets weekly (April 10, 12:00 KST for this account)

**Per-1% utilization cost** (measured across 5 active windows on Max 20x / $200/mo):

| Metric | Range | Note |
|--------|-------|------|
| Output per 1% | 9K-16K | Visible output only (thinking excluded) |
| Cache Read per 1% | 1.5M-2.1M | 96-99% of visible token volume |
| Total Visible per 1% | 1.5M-2.1M | Output + Cache Read + Input |
| 7d accumulation ratio | 0.12-0.17 | 7d_delta relative to 5h_peak |

**Thinking token blind spot:** Extended thinking tokens are **not included** in the `output_tokens` field from the API. At 9K-16K visible output per 1%, a full 5h window (100%) = only 0.9M-1.6M visible output tokens — low for several hours of Opus work. The gap is consistent with thinking tokens being counted against the quota, but the exact mechanism can't be confirmed from the client side. Thinking-disabled isolation test planned for the week of April 6.

**Community cross-validation:**
- [@fgrosswig](https://github.com/fgrosswig): [64x budget reduction](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) — dual-machine 18-day JSONL forensics (Mar 26: 3.2B tokens no limit → Apr 5: 88M at 90%)
- [@Commandershadow9](https://github.com/Commandershadow9): [34-143x capacity reduction](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296) — cache fix confirmed, capacity drop independent of cache bug, thinking token hypothesis

**v2.1.89 separation:** The cache regression (Mar 28 - Apr 1) is a separate, resolved issue. The capacity reduction exists independently — clean comparison: golden period (Mar 23-27, cache 98-99%) vs post-fix (Apr 2+, cache 84-97%), both with healthy cache. Data collection ongoing through April 10 (full 7d cycle).

---

## Current Status (April 6, 2026)

Cache regression (v2.1.89) is **fixed** in v2.1.90-91. Four additional client-side bugs and server-side quota changes remain active:

| Bug | What It Does | Impact | Status (v2.1.91) | Details |
|-----|-------------|--------|-------------------|---------|
| **B1** Sentinel | Standalone binary corrupts cache prefix | 4-17% cache read (v2.1.89) | **Fixed** | [01_BUGS.md](01_BUGS.md#bug-1--sentinel-replacement-standalone-binary-only) |
| **B2** Resume | `--resume` replays full context uncached | Full cache miss per resume | **Fixed** | [01_BUGS.md](01_BUGS.md#bug-2--resume-cache-breakage-v2169) |
| **B3** False RL | Client blocks API calls with fake error | Instant "Rate limit reached" | **Unfixed** | [01_BUGS.md](01_BUGS.md#bug-3--client-side-false-rate-limiter-all-versions) |
| **B4** Microcompact | Tool results silently cleared mid-session | Context quality degrades | **Unfixed** | [01_BUGS.md](01_BUGS.md#bug-4--silent-microcompact--context-quality-degradation-all-versions-server-controlled), [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| **B5** Budget cap | 200K aggregate limit on tool results | Older results truncated to 1-41 chars | **Unfixed** | [01_BUGS.md](01_BUGS.md#bug-5--tool-result-budget-enforcement-all-versions), [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| **B8** Log inflation | Extended thinking duplicates JSONL entries | 2.87x local token inflation | **Unfixed** | [01_BUGS.md](01_BUGS.md#bug-8--jsonl-log-duplication-all-versions) |
| **Server** | Quota architecture + thinking token accounting | Reduced effective capacity | **By design** | [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |

### What You Can Do

1. **Update to v2.1.91** — fixes the cache regression (worst drain)
2. **npm or standalone — both fine on v2.1.91** (Sentinel gap closed)
3. **Don't use `--resume` or `--continue`** — replays full context as billable input
4. **Start fresh sessions periodically** — the 200K tool result cap (B5) silently truncates older results
5. **Avoid `/dream` and `/insights`** — background API calls that drain silently

See [09_QUICKSTART.md](09_QUICKSTART.md) for setup guide and self-diagnosis.

---

## Server-Side Factors (Unresolved)

Even with cache at 95-99%, drain persists. At least four server-side issues contribute:

**1. Server-side accounting change:** Old Docker versions (v2.1.74, v2.1.86 — never updated) started draining fast recently, proving the issue isn't purely client-side ([#37394](https://github.com/anthropics/claude-code/issues/37394)).

**2. 1M context billing regression:** A late-March regression causes the server to incorrectly classify Max plan 1M context requests as "extra usage." Debug logs show a 429 error at only ~23K tokens ([#42616](https://github.com/anthropics/claude-code/issues/42616)).

**3. Dual-window quota architecture (April 6):** 5h + 7d independent windows. Each 1% of 5h costs ~1.5M-2.1M visible tokens (96-99% cache_read). See [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md).

**3a. Thinking token blind spot (April 6):** Visible output is only 9K-16K per 1% — consistent with thinking tokens being counted against the quota but invisible to clients. Community reports by [@Commandershadow9](https://github.com/Commandershadow9) and [@fgrosswig](https://github.com/fgrosswig) independently reached the same conclusion through JSONL analysis. See [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md).

**4. Org-level quota sharing:** Accounts under the same organization share rate limit pools. `passesEligibilityCache` and `overageCreditGrantCache` are keyed by `organizationUuid`, not `accountUuid`. Originally discovered by [@dancinlife](https://github.com/dancinlife) through client-side analysis of the obfuscated JavaScript bundle.

---

## Usage Precautions

| Behavior | Why | Recommendation |
|----------|-----|----------------|
| `--resume` / `--continue` | Replays entire history as billable input | **Avoid** — start fresh |
| `/dream`, `/insights` | Background API calls, silent drain | **Avoid** |
| Multiple terminals | No cache sharing, parallel drain | **Limit to one** |
| Large CLAUDE.md / context files | Sent every turn, scales with cache_read | **Keep lean** |
| v2.1.89 or earlier standalone | Sentinel bug, sustained 4-17% cache | **Update to v2.1.91** |

---

## Background

### How this started

On April 1, 2026, my Max 20 plan ($200/mo) hit 100% usage in ~70 minutes during normal coding. JSONL analysis showed the session averaging **36.1% cache read** (min 21.1%) where it should have been 90%+. Every token was being billed at full price.

Downgrading from v2.1.89 to v2.1.68 immediately recovered cache to **97.6%** — confirming the regression was version-specific. I set up a transparent monitoring proxy (cc-relay) to capture per-request data going forward.

What started as personal debugging quickly expanded. Dozens of users were reporting the same symptoms across what became [91+ GitHub issues](10_ISSUES.md). Community members — [@Sn3th](https://github.com/Sn3th), [@rwp65](https://github.com/rwp65), [@fgrosswig](https://github.com/fgrosswig), [@Commandershadow9](https://github.com/Commandershadow9), and [12 others](10_ISSUES.md#contributors--acknowledgments) — independently found different pieces of the puzzle.

**The investigation timeline:**

| Date | What happened |
|------|--------------|
| Apr 1 | 70-minute 100% drain → v2.1.89 regression confirmed, proxy setup |
| Apr 2 | Bugs 3-4 discovered (false rate limiter, silent microcompact). Anthropic's Lydia Hallie posts on X |
| Apr 3 | Bug 5 discovered (200K budget cap). v2.1.91 benchmark: cache fixed, 4 other active bugs persist (B3-B5, B8). [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md) |
| Apr 4-6 | cc-relay captures 3,702 requests with rate limit headers. Community analysis continues |
| Apr 6 | Dual-window quota analysis published. Community cross-validation (fgrosswig 64x, Commandershadow9 34-143x). [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |

Full 14-month chronicle (Feb 2025 – Apr 2026): [07_TIMELINE.md](07_TIMELINE.md)

### Anthropic's Position (April 2)

Lydia Hallie (Anthropic, Product) [posted on X](https://x.com/lydiahallie/status/2039800715607187906):

> *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."*

She [recommended](https://x.com/lydiahallie/status/2039800718371307603) using Sonnet as default, lowering effort level, starting fresh instead of resuming, and capping context with `CLAUDE_CODE_AUTO_COMPACT_WINDOW=200000`.

**Where our data diverges from this assessment:**

- **"None were over-charging you"** — Bug 5 silently truncates tool results to 1-41 chars after a 200K aggregate threshold. Users paying for 1M context effectively have a 200K tool result budget for built-in tools. 261 truncation events measured in a single session.
- **"We fixed a few bugs"** — Cache bugs (B1-B2) are fixed, but Bugs 3-5 and B8 remain active in v2.1.91. Client-side false rate limiter (B3) generated 151 synthetic "Rate limit reached" errors across 65 sessions on our setup — zero API calls made.
- **"Peak-hour limits are tighter"** — Our April 6 proxy data shows the bottleneck is always the 5h window (`representative-claim` = `five_hour` in 100% of 3,702 requests), regardless of time of day. Weekend and off-peak data shows the same pattern.
- **Thinking token accounting** — Extended thinking tokens don't appear in `output_tokens` from the API, yet visible output alone explains less than half the observed utilization cost. If thinking tokens are counted against quota at output-token rate, this is a significant invisible cost that users have no way to monitor or control.

**GitHub response:** Zero across 91+ rate-limit issues (2+ months of silence). All official communication has been via personal X posts and the changelog. See [10_ISSUES.md](10_ISSUES.md#anthropic-official-response) for full statement history.

### Cache TTL (not a bug)

[@luongnv89](https://github.com/luongnv89) [documented](https://github.com/luongnv89/cc-context-stats/blob/main/context-stats-cache-misses.md) that idle gaps of 13+ hours cause a full cache rebuild. Anthropic documents a 5-minute TTL, though our data shows 5-26 minute gaps sometimes maintaining 96%+ cache — the actual TTL may be longer in practice. Not a bug, but worth knowing about.

---

## Documents

| File | What | Date |
|------|------|------|
| **[README.md](README.md)** | This file — overview, latest updates, current status | Apr 6 |
| **[02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)** | Dual 5h/7d window architecture, per-1% cost, thinking token blind spot | Apr 6 |
| **[03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md)** | Session log analysis: PRELIM inflation, subagent costs, lifecycle curve, proxy cross-validation | Apr 6 |
| **[01_BUGS.md](01_BUGS.md)** | Bug 1-5, 8 technical details + measured data | Apr 3 |
| **[05_MICROCOMPACT.md](05_MICROCOMPACT.md)** | Deep dive: silent context stripping (Bug 4) + tool result budget (Bug 5) | Apr 3 |
| **[04_BENCHMARK.md](04_BENCHMARK.md)** | npm vs standalone benchmark with raw per-request data | Apr 3 |
| **[06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md)** | April 3 integrated test results — all bugs verified | Apr 3 |
| **[07_TIMELINE.md](07_TIMELINE.md)** | 14-month chronicle of rate limit issues (Phase 1-9) | Apr 6 |
| **[09_QUICKSTART.md](09_QUICKSTART.md)** | Setup guide + self-diagnosis | Apr 3 |
| **[08_UPDATE-LOG.md](08_UPDATE-LOG.md)** | Daily investigation log — what was found, when, how | Apr 1-6 |
| **[10_ISSUES.md](10_ISSUES.md)** | 91+ related issues + community tools + contributors | Apr 6 |
| **[11_USAGE-GUIDE.md](11_USAGE-GUIDE.md)** | Essential usage guide — sessions, context, CLAUDE.md, token-saving | Apr 8 |
| **[12_ADVANCED-GUIDE.md](12_ADVANCED-GUIDE.md)** | Power user guide — hooks, subagents, monitoring, rate limit tactics | Apr 8 |

## Environment

- **Plan:** Max 20 ($200/mo)
- **OS:** Linux (Ubuntu), HP ZBook Ultra G1a
- **Versions tested:** v2.1.91, v2.1.90, v2.1.89, v2.1.68
- **Monitoring:** cc-relay v2 transparent proxy (8,794 requests, 3,702 with rate limit headers)
- **Date:** April 6, 2026

---

## Contributors

This analysis builds on work by many community members. Full details in [10_ISSUES.md](10_ISSUES.md#contributors--acknowledgments).

| Who | Key Contribution |
|-----|-----------------|
| [@Sn3th](https://github.com/Sn3th) | Discovered microcompact mechanisms (Bug 4), GrowthBook flags, budget pipeline (Bug 5), confirmed server-side context mutation across multiple machines |
| [@rwp65](https://github.com/rwp65) | Discovered client-side false rate limiter (Bug 3) with detailed log evidence |
| [@fgrosswig](https://github.com/fgrosswig) | 64x budget reduction forensics — dual-machine 18-day JSONL before/after comparison ([#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)) |
| [@Commandershadow9](https://github.com/Commandershadow9) | 34-143x capacity reduction analysis, thinking token hypothesis ([#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)) |
| [@kolkov](https://github.com/kolkov) | Built [ccdiag](https://github.com/kolkov/ccdiag), identified three v2.1.91 `--resume` regressions ([#43044](https://github.com/anthropics/claude-code/issues/43044)) |
| [@simpolism](https://github.com/simpolism) | v2.1.90 changelog correlation, resume cache fix patch (99.7-99.9% hit) |
| [@luongnv89](https://github.com/luongnv89) | Cache TTL analysis, built [CUStats](https://custats.info) and [context-stats](https://github.com/luongnv89/cc-context-stats) |
| [@dancinlife](https://github.com/dancinlife) | organizationUuid quota pooling and oauthAccount cross-contamination bug |
| [@weilhalt](https://github.com/weilhalt) | Built [BudMon](https://github.com/weilhalt/budmon) for rate-limit header monitoring |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | Windows GrowthBook flag dumps confirming cross-platform consistency |
| [@dbrunet73](https://github.com/dbrunet73) | Real-world OTel comparison data (v2.1.88 vs v2.1.90) confirming cache improvement |
| [@maiarowsky](https://github.com/maiarowsky) | Confirmed Bug 3 on v2.1.90 with 26 synthetic entries across 13 sessions |
| [@edimuj](https://github.com/edimuj) | Measured grep/file-read token waste (3.5M tokens / 1800+ calls), built [tokenlean](https://github.com/edimuj/tokenlean) |
| [@amicicixp](https://github.com/amicicixp) | Verified v2.1.90 cache improvement with before/after testing |
| [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) | Reported old Docker versions draining — key server-side evidence ([#37394](https://github.com/anthropics/claude-code/issues/37394)) |
| [@SC7639](https://github.com/SC7639) | Additional regression data confirming mid-March timeline |
| Reddit community | [Reverse engineering analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) of cache sentinel mechanism |

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
