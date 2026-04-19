> **🇰🇷 [한국어 버전](ko/02_RATELIMIT-HEADERS.md)**

# Rate Limit Header Analysis — Dual-Window Quota Architecture

> **Date:** April 6, 2026 — 48h proxy observation (April 4–6), v2.1.91, Max 20x ($200/mo)
>
> **Related comments:** [#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189807108), [#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189847482)

---

## Summary

Anthropic's API returns `anthropic-ratelimit-unified-*` headers on every response. Claude Code reads only `representative-claim` for the usage bar and discards the rest. Our cc-relay proxy captures all of them. From **23,374 requests with headers** (April 4–14), we identified a **dual sliding window** system — a 5-hour window and a 7-day window — and measured the per-1% utilization cost in tokens.

**Key findings:**
- The 5h window is the bottleneck in **77.4%** of requests (`representative-claim` = `five_hour`). **The 7d window can become the binding constraint** — observed on April 9–10 when 7d utilization reached 0.85–0.97 (see [§9](#9-seven_day-bottleneck-first-observation-april-914))
- Each 1% of 5h utilization costs roughly 1.5M–2.1M visible tokens (96–99% of which is cache_read)
- Visible output tokens per 1% are only 9K–16K — suggesting thinking tokens or cache-read weighting (or both) drive most of the quota cost
- The 7d counter accumulates proportionally to 5h usage (ratio ~0.12–0.17)
- 5h windows reset on roughly 5-hour intervals; 7d resets on a fixed weekly timestamp

---

## 1. Header Fields

Every API response includes these headers:

```
anthropic-ratelimit-unified-5h-utilization: 0.26
anthropic-ratelimit-unified-5h-reset: 1775455200
anthropic-ratelimit-unified-7d-utilization: 0.19
anthropic-ratelimit-unified-7d-reset: 1775973600
anthropic-ratelimit-unified-representative-claim: five_hour
anthropic-ratelimit-unified-fallback-percentage: 0.5
anthropic-ratelimit-unified-overage-status: allowed
anthropic-ratelimit-unified-overage-utilization: 0.0
```

| Header | Description |
|--------|-------------|
| `5h-utilization` | Current 5-hour window usage (0.0 = empty, 1.0 = exhausted) |
| `5h-reset` | Unix timestamp when this 5h window ends and resets |
| `7d-utilization` | Current 7-day window usage |
| `7d-reset` | Unix timestamp when this 7d window ends and resets |
| `representative-claim` | Which window is the tighter constraint — this is what the CLI displays |
| `fallback-percentage` | Capacity allocation ratio (consistently 0.5) |
| `overage-status` | Extra-usage billing state |
| `overage-utilization` | Extra-usage budget consumed |

### Aggregate observations (n=3,702 initial; n=23,374 as of April 14)

- `representative-claim` = `five_hour` in **77.4%** of requests (18,095/23,374); `seven_day` in **22.6%** (5,279/23,374). Initial 3,702-request sample (April 4–6) showed 100% five_hour — the 7d bottleneck only appeared when 7d utilization approached capacity (April 9–10). See [§9](#9-seven_day-bottleneck-first-observation-april-914).
- `fallback-percentage` = `0.5` in 100% of requests (23,374/23,374 — zero variance across 14 days)
- `overage-status` = `allowed` in 100% of requests; `overage-utilization` = `0.0` throughout
- 5h reset timestamps observed: 14:00, 19:00, 01:00, 12:00, 18:00, 23:00, 04:00, 09:00 KST — mostly 5h apart, with two exceptions: 19:00→01:00 is 6h, and 01:00→12:00 is 11h (likely an unobserved window boundary during inactive hours). Exact boundary pattern needs more data.
- 7d reset: April 10, 12:00 KST (fixed weekly, consistent across all responses)
- Max 5h utilization observed: **0.92**; max 7d: **0.99** (April 10)

---

## 2. Per-Window Data

Windows split by `5h-reset` header value (when the reset timestamp changes, that's a new window):

| # | Window ends (KST) | Reqs | Peak 5h | 7d start→end | Σ Output | Σ Cache Read | Σ Input |
|---|-------------------|------|---------|-------------|----------|-------------|---------|
| A | 4/4 14:00 | 1,332 | **0.39** | 0.06→0.11 | 617,750 | 69,413,540 | 2,098,077 |
| B | 4/4 19:00 | 132 | 0.04 | 0.11→0.11 | 33,723 | 10,716,011 | 4,169 |
| C | 4/5 01:00 | 633 | 0.27 | 0.11→0.15 | 360,205 | 55,546,393 | 569,512 |
| D | 4/5 12:00 | 51 | 0.08 | 0.15→0.16 | 17,518 | 2,360,057 | 5,590 |
| E | 4/5 18:00 | 616 | 0.26 | 0.16→0.19 | 299,586 | 45,397,643 | 545,389 |
| F | 4/5 23:00 | 466 | 0.21 | 0.19→0.22 | 192,424 | 32,005,226 | 437,827 |
| G | 4/6 04:00 | 368 | 0.18 | 0.22→0.25 | 190,076 | 34,083,583 | 112,763 |
| H | 4/6 09:00 | 104 | 0.02 | 0.25→0.25 | 25,901 | 11,009,102 | 886 |

Windows B and H had minimal activity (peak 0.04 and 0.02) — excluded from per-1% analysis.

All data is post-fix (v2.1.91, April 4 onward) and unaffected by the v2.1.89 cache regression.

---

## 3. Cost per 1% of 5h Utilization

Total tokens divided by peak utilization in percentage points. "Total Visible" = Output + Cache Read + Input (all measured token types; thinking tokens excluded since they're not in the API response):

| Window | Peak | Output per 1% | Cache Read per 1% | Total Visible per 1% |
|--------|------|---------------|-------------------|---------------------|
| A | 0.39 | 15,840 | 1,779,834 | 1,849,471 |
| C | 0.27 | 13,341 | 2,057,274 | 2,091,707 |
| E | 0.26 | 11,523 | 1,746,063 | 1,778,562 |
| F | 0.21 | 9,163 | 1,524,058 | 1,554,070 |
| G | 0.18 | 10,560 | 1,893,532 | 1,910,357 |

- Cache_read is 96–99% of visible tokens per utilization point
- Output per 1%: 9K–16K (variance ~1.73x)
- Cache Read per 1%: 1.5M–2.1M (variance ~1.35x)
- A full 5h window (100%) would correspond to only ~0.9M–1.6M visible output tokens

### Partially measured: thinking tokens

Extended thinking tokens are not included in `output_tokens` from the API. The actual per-1% cost is:

```
Visible output:    9K–16K       (measured)
Thinking output:   ~0.0–0.1%    (estimated from JSONL content blocks — see below)
Cache read:        1.5M–2.1M   (measured)
Input:             ~25K         (measured)
```

Independent JSONL-based analysis by [@seanGSISG](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3) (179K API calls, Dec 2025 – Apr 2026) estimated thinking block contribution at 0.0–0.1% of total quota from content block text. This is the first concrete measurement of this gap, though JSONL content blocks may represent displayed summaries rather than full server-side computation. The API's exclusion of thinking tokens from `output_tokens` means the full server-side cost remains unmeasurable from the client side.

---

## 4. 7d Accumulation Pattern

For windows with measurable 7d change (excluding B and H where 7d delta was 0.00). Note: Window D (peak 0.08) is included here despite being excluded from the per-1% table in Section 3 — the 7d accumulation is measurable even for light windows, while per-1% cost calculations become unreliable at very low peaks.

| Window | 5h Peak | 7d delta | Ratio (7d/5h) |
|--------|---------|----------|---------------|
| A | 0.39 | +0.05 | 0.128 |
| C | 0.27 | +0.04 | 0.148 |
| D | 0.08 | +0.01 | 0.125 |
| E | 0.26 | +0.03 | 0.115 |
| F | 0.21 | +0.03 | 0.143 |
| G | 0.18 | +0.03 | 0.167 |

Ratios cluster at 0.12–0.17. Directionally consistent but not precise — 7d values are 2-decimal, so +0.03 could be 0.025–0.035. Six data points is insufficient for a firm formula.

---

## 5. v2.1.89 Separation

The v2.1.89 cache regression (March 28 – April 1) is a **separate, resolved** issue from the capacity reduction. Clean comparison framework:

| Period | Cache health | Nature |
|--------|-------------|--------|
| Mar 23–27 ("golden") | 98–99% | Clean baseline |
| Mar 28–Apr 1 (v2.1.89 bug) | 86%, 29% broken | Exclude from capacity analysis |
| Apr 2–5 (v2.1.91+) | 84–97%, 0% broken | Post-fix — capacity reduction visible |

All proxy data in this document is from the post-fix period.

---

## 6. Limitations

- **Plan-specific.** Max 20x ($200/mo) data. Per-1% costs likely differ on other tiers.
- **No before-data — partially resolved.** Proxy started April 4 — no March baseline for comparison. [@seanGSISG's 179K-call dataset](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3) (Dec 2025 – Apr 2026) provides independent before-data confirming per-1% benchmarks hold across the transition period.
- **48h / 8 windows.** Consistent trends but small sample size. Full 7-day cycle completes April 10.
- **5h boundary timing approximate.** Eight reset timestamps observed, not enough to confirm exact pattern.
- **Thinking tokens partially measured.** JSONL content blocks suggest ~0.0–0.1% of quota ([@seanGSISG](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3)), but server-side computation may differ from displayed content.

---

## 7. Community Cross-References

### @fgrosswig — 64x Budget Reduction Forensics ([#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353))

- **Plan:** Max 5x ($100/mo), Opus 4.6 1M, dual-machine (Windows + Linux)
- **Method:** Parsed all local JSONL session logs across both machines, March 18 – April 5
- **Finding:** March 26 consumed 3.2B tokens (combined) with no limit hit; April 5 consumed 88M and hit 90% — ~64x effective budget reduction
- **Hypothesis:** Cache-read token weighting changed from ~0x to ~1x around March 23–28
- **Key data:** Per-day token tables, subagent cache analysis, hourly April 5 breakdown

### @Commandershadow9 — 34–143x Capacity Reduction ([#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296))

- **Plan:** Max $100/mo, Opus 4.6 1M, Debian 12, 15 plugins (reduced from 57), 9 MCP servers
- **Method:** Analyzed all local JSONL session transcripts, March 1 – April 5
- **Finding:** Cache fix confirmed (v2.1.91+), but 34–143x capacity reduction persists independent of cache bug
- **Hypothesis:** Thinking tokens now counted against quota at a different weight — invisible in local logs
- **Key data:** Golden period vs post-fix comparison, cross-tier table, plugin reduction test

### Relationship to our analysis

Both analyses measure **client-side token consumption** (what was sent/received). Our proxy data adds the **server-side utilization response** (how the server scored that consumption). Neither alone tells the full story — the combination gives both sides of the equation.

---

## 8. Extended `fallback-percentage` Data (April 14 update, self-measured)

> **Added:** April 13, 2026 — extended proxy data from our own cc-relay, plus community cross-account observations from [#41930](https://github.com/anthropics/claude-code/issues/41930). **Updated:** April 14 — expanded to 23,374 requests.

Section 1 documented `fallback-percentage` = `0.5` across our initial 3,702-request sample (April 4–6). Our proxy has now accumulated **23,374 requests with rate limit headers** (April 4–14, 11 days). The field remains **0.5 on every single request — zero variance across the entire dataset.**

| Metric | Value |
|--------|-------|
| Total requests with headers | **23,374** |
| `fallback-percentage` = 0.5 | **23,374 (100%)** |
| `overage-status` = allowed | **23,374 (100%)** |
| Date range | April 4 – April 14, 2026 |
| Plan | Max 20x ($200/mo) |
| Source | cc-relay proxy (our own) |

### Community cross-account observations (not our data — reference only)

Starting April 11, two independent researchers published cross-account data in [#41930](https://github.com/anthropics/claude-code/issues/41930):

| Observer | Plan | Region | Calls | Value | overage-status | Note |
|----------|------|--------|-------|-------|----------------|------|
| [@cnighswonger](https://github.com/cnighswonger) | Max 5x | US (IAD) | 11,502¹ | 0.5 | allowed | claude-code-cache-fix + claude-meter interceptor |
| [@0xNightDev](https://github.com/0xNightDev) | Max 5x | EU (AMS) | qualitative² | 0.5 | **rejected** (org_level_disabled) | claude-usage-dashboard |
| [#12829](https://github.com/anthropics/claude-code/issues/12829) | unknown | unknown | 1 | **0.2** | rejected (org_level_disabled) | Nov 2025, v2.0.50 |

¹ 11,502 of 11,505 rows contained the field; 3 were bootstrap calls where the field was not yet populated.
² 0xNightDev stated "every single request" but did not provide a numerical count.
³ cnighswonger subsequently confirmed 0.5 invariance across 14,000+ metered calls (April 4–16) in a separate controlled 4-session comparison on the same Max 5x account ([Issue #4](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/4)).

### What we know

1. **Consistency within accounts:** Our 38,996 requests over 16 days (dataset `ubuntu-1-stock`, April 1–16) and cnighswonger's 11,502 over 7 days both show zero variance. A subsequent controlled 4-session comparison on the same account confirmed this across 14,000+ metered calls³. The field is a **fixed per-account value**, not dynamically adjusted per request, time of day, or load.
2. **Variation across time:** 0.2 in November 2025 (#12829) vs 0.5 in April 2026 — the value can change over time.
3. **`overage-status` diverges across accounts on the same plan:** Our Max 20x and cnighswonger's Max 5x both show `allowed`, while 0xNightDev's Max 5x shows `rejected` + `org_level_disabled`.

### What we don't know

The **exact semantics** of `fallback-percentage` are undocumented by Anthropic. Community interpretations include: multiplicative quota cap (0.5 = 50% of theoretical max), fallback routing fraction, or post-exhaustion degradation threshold. [@cnighswonger cautioned](https://github.com/anthropics/claude-code/issues/41930#issuecomment-4230013175): *"Your three interpretations are all defensible and we can't distinguish them from the data we have."*

**We document the observed values without endorsing any interpretation.** Absent official documentation, treating any specific interpretation as fact would undermine the evidentiary standard of this repository. No Anthropic response to any comment in #41930 regarding this field as of April 13.

---

## 9. `seven_day` Bottleneck — First Observation (April 9–14)

> **Added:** April 14, 2026

Sections 1–7 reported `representative-claim` = `five_hour` in 100% of our initial 3,702 requests (April 4–6). With the expanded dataset (23,374 requests, April 4–14), the 7d window was observed as the binding constraint for the first time:

| Day | `five_hour` | `seven_day` | Avg 5h util | Avg 7d util |
|-----|-------------|-------------|-------------|-------------|
| Apr 4–8 | 12,072 | 0 | 0.10–0.30 | 0.10–0.63 |
| **Apr 9** | **141** | **4,148** | 0.037 | **0.848** |
| **Apr 10** | 1,366 | **1,131** | 0.229 | **0.966** |
| Apr 11–14 | 4,518 | 0 | 0.019–0.306 | 0.195–0.442 |

**What happened:** By April 9, cumulative 7d usage reached 0.85 (approaching the weekly cap). The 5h window was nearly empty after a reset (0.037), but the 7d window was the tighter constraint — so `representative-claim` flipped to `seven_day`. On April 10, 7d peaked at **0.966** (near-exhaustion). After the weekly reset (April 10, 12:00 KST), 7d dropped and `five_hour` resumed as the bottleneck.

**Implications:**
1. The 7d window is **not merely cosmetic** — it can become the active rate limiter during sustained heavy usage
2. Users who use CC intensively throughout the week (not just in bursts) may hit the 7d cap before 5h caps
3. The 5h utilization distribution shifts: with 7d binding, 5h stays low even during active work — the displayed usage bar may understate actual constraint

**Corrected finding:** The 5h window is the bottleneck in **77.4%** of requests (18,095/23,374), not 100% as previously reported. The 7d window was the bottleneck in **22.6%** (5,279/23,374), concentrated in April 9–10.

---

## 10. Data Interpretation Caveat — Environment Changes (April 14)

> **Added:** April 14, 2026

Our proxy data in dataset `ubuntu-1-stock` spans April 1–16 (**38,996 total requests**, 23,374 with rate limit headers as of the April 14 analysis). The measurement environment changed during this period:

| Period | Requests | Environment | Use for |
|--------|----------|-------------|---------|
| Apr 1 – Apr 10 14:25 | 25,558 | **Unmodified** CC — no flag overrides | Baseline measurements, bug event counts, per-1% cost analysis |
| Apr 10 14:25 – Apr 15 | 9,996 | GrowthBook flag override active (B4/B5 flags set to permissive values, see [01_BUGS.md](01_BUGS.md#growthbook-flag-override--controlled-elimination-test-april-1014)) | Override effect measurement only |

**Why this matters:**
- B4/B5 event counts (167,818 and 5,500) are entirely from the unmodified period
- Rate limit header analysis (§1–7) uses April 4–6 data (unmodified period) and is unaffected
- The `seven_day` bottleneck observation (§9) spans both periods but is driven by quota utilization levels, not flag overrides — flag overrides affect context mutation, not quota accounting
- `fallback-percentage` data (§8) spans both periods; the value is 0.5 in both, confirming the override does not affect this field

When citing data from this repository, note the measurement period to ensure the correct environmental context.

---

## 11. Next Steps

- **Thinking token isolation test:** Run sessions with thinking disabled and compare per-1% cost (planned but not yet executed — the thinking token hypothesis remains unverified)
- **Cross-tier comparison:** If community members on other plan tiers run proxies, we can compare per-1% costs across plans
- **`fallback-percentage` monitoring:** Track whether the value changes over time on our account

---

## Appendix: Hourly Proxy Data Grid (April 4-6)

Rate limit headers captured via cc-relay transparent proxy. Each row = one clock hour (KST). `5h util` and `7d util` are the maximum values observed within that hour.

| Hour (KST) | Reqs | Output | Cache Read | Input | 5h util | 7d util |
|-------------|------|--------|------------|-------|---------|---------|
| 04-04 09 | 74 | 34,368 | 2,523,601 | 244,743 | 4% | 6% |
| 04-04 10 | 304 | 154,768 | 12,224,810 | 1,092,266 | 16% | 8% |
| 04-04 11 | 410 | 179,010 | 27,317,171 | 48,757 | 23% | 9% |
| 04-04 12 | 388 | 158,247 | 23,770,875 | 62,670 | 32% | 10% |
| 04-04 13 | 156 | 91,357 | 3,577,083 | 649,641 | 39% | 11% |
| 04-04 14 | 77 | 16,938 | 4,778,807 | 4,100 | 2% | 11% |
| 04-04 15 | 39 | 10,282 | 4,129,802 | 49 | 2% | 11% |
| 04-04 18 | 16 | 6,503 | 1,807,402 | 20 | 4% | 11% |
| 04-04 20 | 93 | 39,232 | 9,215,178 | 69,206 | 4% | 12% |
| 04-04 21 | 89 | 61,574 | 3,441,361 | 435,855 | 9% | 12% |
| 04-04 22 | 80 | 46,656 | 5,023,044 | 31,703 | 11% | 13% |
| 04-04 23 | 371 | 212,743 | 37,866,810 | 32,748 | 27% | 15% |
| 04-05 07 | 13 | 4,484 | 317,744 | 4,990 | 5% | 15% |
| 04-05 08 | 38 | 13,034 | 2,042,313 | 600 | 8% | 16% |
| 04-05 13 | 19 | 9,367 | 905,905 | 26 | 1% | 16% |
| 04-05 14 | 375 | 161,991 | 29,579,110 | 179,609 | 17% | 18% |
| 04-05 15 | 198 | 126,249 | 12,448,681 | 365,714 | 26% | 19% |
| 04-05 16 | 24 | 1,979 | 2,463,947 | 40 | 26% | 19% |
| 04-05 18 | 165 | 65,864 | 9,575,191 | 162,835 | 7% | 20% |
| 04-05 19 | 167 | 73,519 | 10,659,315 | 251,812 | 15% | 21% |
| 04-05 21 | 90 | 34,033 | 8,511,367 | 23,111 | 20% | 22% |
| 04-05 22 | 44 | 19,008 | 3,259,353 | 69 | 21% | 22% |
| 04-05 23 | 200 | 115,995 | 12,659,554 | 112,454 | 13% | 24% |
| 04-06 00 | 112 | 69,549 | 13,537,745 | 213 | 18% | 25% |
| 04-06 01 | 18 | 1,335 | 2,288,596 | 29 | 18% | 25% |
| 04-06 02 | 18 | 1,386 | 2,673,247 | 33 | 18% | 25% |
| 04-06 03 | 20 | 1,811 | 2,924,441 | 34 | 18% | 25% |
| 04-06 04 | 18 | 1,510 | 2,007,438 | 28 | 0% | 25% |
| 04-06 05 | 18 | 1,538 | 2,750,357 | 32 | 0% | 25% |
| 04-06 06 | 25 | 7,601 | 3,291,379 | 453 | 1% | 25% |
| 04-06 07 | 43 | 15,252 | 2,959,928 | 373 | 2% | 25% |

---

*Environment: dataset `ubuntu-1-stock` — Max 20x ($200/mo), Opus 4.6 1M, v2.1.91 via cc-relay transparent proxy, Linux (ubuntu-1), native `~/.claude` (CC stock mode). 8,794 total proxy requests, 3,702 with rate limit headers (April 4–6 slice used for §1–7). Full current totals: 38,996 requests / 272 sessions (April 1–16). See [14_DATA-SOURCES.md](14_DATA-SOURCES.md) for the full label matrix.*

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All data captured using official tools and documented features.*
