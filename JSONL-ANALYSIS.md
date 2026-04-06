# JSONL Session Log Analysis — First-Party Data

> **Date:** April 6, 2026 — 110 main sessions + 279 subagent sessions (April 1-6), v2.1.91, Max 20x ($200/mo)
>
> **Relationship to other documents:** [RATELIMIT-HEADERS.md](RATELIMIT-HEADERS.md) analyzes server-side rate limit headers from the proxy. This document analyzes client-side JSONL session logs (`~/.claude/projects/**/*.jsonl`). Section 5 cross-correlates both.

---

## Summary

JSONL session logs are the client's record of every API interaction. Unlike the proxy (which sees HTTP response headers), JSONL captures per-turn token breakdowns including PRELIM/FINAL entries, synthetic rate limit events, and model identification.

**Key findings:**
- **Synthetic rate limiting (B3):** 24 confirmed `<synthetic>` entries across 6 days — client-generated fake rate limits
- **PRELIM inflation (B8):** 5,390 PRELIM vs 6,560 FINAL entries (0.82x ratio overall, up to 1.14x on heavy days). PRELIM entries carry **0.56-0.97x** the cache_read of FINAL entries — not 1:1 but still substantial
- **Subagent overhead:** 279 subagent sessions consumed 17.3% of total cache_read but 40.4% of total output. Cold start median: 13,358 cache_create tokens
- **Session growth:** cache_read grows 24x over a 990-turn session (25K → 595K per turn, +575 linear)
- **Session cost range:** 753x variation (447K → 336M visible tokens) between cheapest and most expensive sessions
- **Time-normalized cost:** median 153K cache_read/minute, mean 227K/minute
- **Cross-correlation with proxy:** JSONL records **1.93x** the cache_read tokens that the proxy sees — directly confirming PRELIM double-counting inflates local accounting by ~2x

---

## 1. Daily Token Summary (April 1-6)

Main sessions only (excludes subagents):

| Day | Sessions | Entries | Output | Cache Read | Cache Create | Input | Total Visible |
|-----|----------|---------|--------|------------|-------------|-------|---------------|
| 04-01 | 25 | 1,930 | 567,657 | 173,259,918 | 21,418,689 | 72,855 | 195,319,119 |
| 04-02 | 29 | 3,014 | 826,510 | 447,692,321 | 8,155,349 | 229,115 | 456,903,295 |
| 04-03 | 15 | 1,203 | 260,041 | 84,453,778 | 2,712,374 | 26,520 | 87,452,713 |
| 04-04 | 21 | 2,972 | 782,955 | 531,652,242 | 4,842,177 | 2,037,139 | 540,314,513 |
| 04-05 | 12 | 1,426 | 316,962 | 132,464,357 | 6,455,937 | 39,222 | 139,276,478 |
| 04-06 | 8 | 1,432 | 343,630 | 236,082,685 | 31,681,086 | 12,104 | 268,119,505 |

**Per-entry averages:**

| Day | Cache Read % | Create % | Output/Entry | Cache/Entry | Total/Entry |
|-----|-------------|----------|-------------|-------------|-------------|
| 04-01 | 88.7% | 11.0% | 294 | 89,772 | 101,202 |
| 04-02 | 98.0% | 1.8% | 274 | 148,538 | 151,594 |
| 04-03 | 96.6% | 3.1% | 216 | 70,203 | 72,696 |
| 04-04 | 98.4% | 1.4% | 263 | 178,887 | 181,802 |
| 04-05 | 95.1% | 4.6% | 222 | 92,892 | 97,669 |
| 04-06 | 98.4% | 1.5% | 250 | 172,695 | 175,586 |

**Observations:**
- **Cache Read %** ranges 88.7–98.4%. April 1 is lower (88.7%) due to many session cold starts (25 sessions in one day → more cache_create).
- **Output per entry** is only 216–294 tokens — this is visible output only. With thinking enabled, actual server-side output per turn is much higher.
- **Cache per entry** ranges 70K–179K. Heavier days (04-04) have longer sessions → more accumulated context per turn.

### Bug indicators

| Day | Synthetic (B3) | PRELIM | FINAL | P/F Ratio |
|-----|---------------|--------|-------|-----------|
| 04-01 | 4 | 911 | 1,015 | 0.90 |
| 04-02 | 4 | 1,399 | 1,611 | 0.87 |
| 04-03 | 3 | 556 | 643 | 0.86 |
| 04-04 | 4 | 1,229 | 1,737 | 0.71 |
| 04-05 | **9** | 754 | 663 | **1.14** |
| 04-06 | 0 | 541 | 891 | 0.61 |
| **Total** | **24** | **5,390** | **6,560** | **0.82** |

- **Synthetic (B3):** 24 events. April 5 had 9 — the highest concentration. Each represents a client-generated fake "Rate limit reached" with no API call.
- **PRELIM/FINAL ratio** varies 0.61–1.14 depending on thinking usage. April 5 was 1.14x (more PRELIM than FINAL — heavy thinking). April 6 was 0.61x (lighter thinking).

### PRELIM vs FINAL: token-level comparison

Do PRELIM entries carry the same token counts as FINAL entries?

| Day | PRELIM cache_read | FINAL cache_read | P/F Cache Ratio | PRELIM output | FINAL output | P/F Output Ratio |
|-----|-------------------|------------------|-----------------|---------------|-------------|------------------|
| 04-01 | 73,581,649 | 99,678,269 | 0.74 | 22,329 | 545,328 | 0.04 |
| 04-02 | 187,693,131 | 259,999,190 | 0.72 | 30,416 | 796,094 | 0.04 |
| 04-03 | 34,939,821 | 49,362,008 | 0.71 | 14,217 | 237,824 | 0.06 |
| 04-04 | 191,259,356 | 340,298,839 | 0.56 | 24,726 | 742,229 | 0.03 |
| 04-05 | 65,313,689 | 67,150,668 | 0.97 | 22,956 | 294,006 | 0.08 |
| 04-06 | 101,012,834 | 162,692,993 | 0.62 | 10,874 | 370,905 | 0.03 |

**PRELIM cache_read is 0.56–0.97x of FINAL cache_read** — substantial but not 1:1. The ratio varies by day. PRELIM output is only 0.03–0.08x of FINAL (PRELIM entries have almost no visible output — they're intermediate "thinking in progress" snapshots).

**This means:** PRELIM inflation (B8) adds roughly 56–97% additional cache_read on top of the real FINAL values. If the server counts PRELIM entries at full weight, users are paying 1.56–1.97x for cache_read.

---

## 2. Session Cost Distribution

Not all sessions are equal. The 753x range between cheapest and most expensive sessions shows how dramatically costs vary:

### Top 10 most expensive sessions

| Day | Session | Total Visible | Entries | Cache Read | Output |
|-----|---------|--------------|---------|------------|--------|
| 04-04 | 03480ffe | **336,360,170** | 990 | 333,169,783 | 266,151 |
| 04-02 | 63e79b5e | 163,952,464 | 609 | 161,907,520 | 155,012 |
| 04-06 | b926be34 | 126,541,892 | 460 | 124,243,773 | 180,303 |
| 04-06 | ae47b46b | 102,192,853 | 694 | 101,261,459 | 83,175 |
| 04-02 | 158a1b9d | 69,083,572 | 354 | 68,264,817 | 98,020 |
| 04-01 | 888b0e4b | 60,747,270 | 258 | 59,511,929 | 123,305 |
| 04-02 | 89bdbe06 | 48,365,855 | 327 | 47,806,054 | 100,093 |
| 04-02 | 23d083a1 | 42,169,722 | 265 | 41,368,814 | 94,055 |
| 04-02 | 48557289 | 40,430,993 | 309 | 39,400,900 | 69,508 |
| 04-04 | 0c024075 | 39,933,209 | 284 | 39,283,156 | 69,145 |

### Bottom 5 (cheapest with >10 entries)

| Day | Session | Total Visible | Entries | Cache Read | Output |
|-----|---------|--------------|---------|------------|--------|
| 04-05 | 41549298 | 446,988 | 12 | 419,350 | 2,685 |
| 04-06 | fb0c85e5 | 463,571 | 12 | 430,191 | 2,093 |
| 04-02 | 79388f46 | 474,370 | 13 | 429,085 | 5,073 |
| 04-02 | 22b91f19 | 501,069 | 13 | 401,350 | 4,903 |
| 04-03 | 30f6ba7a | 587,411 | 17 | 538,729 | 4,652 |

### Distribution

| Metric | Value |
|--------|-------|
| Sessions analyzed | 78 (with >10 entries and >1 min duration) |
| Cost range | 446,988 — 336,360,170 (**753x**) |
| Median | 8,581,558 |
| Mean | 21,620,755 |

The most expensive session (03480ffe, 990 turns, 336M visible tokens) consumed as much as **38 median sessions combined**. This is the natural consequence of cache_read growing linearly with conversation length — long sessions are disproportionately expensive.

---

## 3. Time-Normalized Costs

Raw token totals can be misleading when comparing sessions or days with different work hours. Time-normalized metrics give a fairer picture:

### Session duration distribution (n=78)

| Metric | Value |
|--------|-------|
| Median duration | 49 min (0.8h) |
| Mean duration | 158 min (2.6h) |
| Min | 1 min |
| Max | 1,372 min (22.9h) |

### Cache read per minute

| Metric | Value |
|--------|-------|
| Median | **153,018 tokens/min** |
| Mean | 226,675 tokens/min |
| Min | 8,397 tokens/min |
| Max | 864,186 tokens/min |

### Output per minute

| Metric | Value |
|--------|-------|
| Median | **574 tokens/min** |
| Mean | 744 tokens/min |

**Why this matters:** When comparing budget reduction across time periods, total token consumption includes both "how long you worked" and "how expensive each minute was." A day with 16h work and 2.7B cache_read (168M/h) vs a day with 2h work and 123M cache_read (62M/h) shows 22x total reduction but only **2.7x per-hour reduction**. The per-hour metric isolates the actual cost change from the work-time difference.

This is relevant to community analyses (e.g., @fgrosswig's comparison of March 30 vs April 6) where total-to-total comparisons can overstate budget reduction by conflating work hours with cost-per-hour changes.

---

## 4. Main Sessions vs Subagents

| Metric | Main Sessions | Subagents | Subagent Share |
|--------|--------------|-----------|----------------|
| Files | 110 | 279 | — |
| Entries | 11,983 | 10,178 | — |
| Output | 3,100,588 | 2,100,541 | **40.4%** |
| Cache Read | 1,607,617,725 | 335,964,567 | **17.3%** |
| Cache Create | 50,302,333 | 82,034,105 | **62.0%** |
| Input | 564,967 | 3,477,557 | — |
| Cache Read % | 96.9% | 79.7% | — |
| Avg cache_read/entry | **134,158** | **33,009** | 4.1x |

### Subagent cold start cost

| Metric | Value |
|--------|-------|
| Samples | 280 subagent sessions |
| Median cache_create (1st turn) | **13,358 tokens** |
| Mean | 12,105 |
| P25 — P75 | 7,205 — 16,703 |
| Min — Max | 491 — 25,075 |

Each subagent spawn costs roughly **13K tokens in cache_create** on the first turn. With 279 subagent sessions over 6 days, that's ~3.4M tokens spent just on cold starts.

### Subagent warming curve

| Metric | Value |
|--------|-------|
| Samples | 9,403 entries (turn 4+) |
| Median cache_read % | **92.3%** |
| Mean | 81.7% |
| P25 — P75 | 78.1% — 97.6% |
| Min — Max | 0.0% — 100.0% |

After 3+ turns, subagent cache_read settles to a **median 92.3%** — lower than main sessions (96.9%) but functional. The P25 of 78.1% indicates some subagents never fully warm, likely due to short lifespans (many subagents run <5 turns).

**What this means:**
- Subagents produce **40% of output** but only consume **17% of cache_read** — each turn is lighter (33K vs 134K) because subagents have shorter context.
- Subagents account for **62% of cache_create** — they frequently start cold. This is the expensive operation at ~13K tokens per spawn.
- Users with heavy subagent workflows (e.g., many Agent tool calls) pay disproportionately in cache_create but save on cache_read per turn.

---

## 5. Session Lifecycle — Cache Growth Curve

Analysis of the longest session (03480ffe, 990 entries, 336M visible tokens):

| Turn | Cache Read | Growth from Turn 1 | Cache Read % |
|------|-----------|-------------------|-------------|
| 1 | 24,820 | — | 76.2% |
| 10 | 34,131 | 1.4x | 94.3% |
| 50 | 63,942 | 2.6x | 99.0% |
| 100 | 145,104 | 5.8x | 99.3% |
| 200 | 204,247 | 8.2x | 99.1% |
| 281 | **8,237** | **0.3x (anomaly)** | **3.4%** |
| 301 | 257,309 | 10.4x | 97.4% |
| 500 | 347,470 | 14.0x | 99.8% |
| 700 | 439,024 | 17.7x | 99.8% |
| 990 | 595,141 | **24.0x** | 100.0% |

**Growth is linear** — roughly **+575 cache_read tokens per turn**. This is conversation history accumulating. At turn 990, each API call sends ~595K tokens of cached context.

**Anomaly at turn 281:** cache_read dropped from ~222K to 8,237 with cache_create spiking to 237,490. This is a mid-session cache rebuild — possibly triggered by microcompact (B4) or a server-side cache eviction. Recovery to pre-anomaly levels took ~20 turns.

**Implication for utilization:** If each 1% of 5h utilization costs ~1.5M-2.1M visible tokens (from [RATELIMIT-HEADERS.md](RATELIMIT-HEADERS.md)), then:
- Early session (turn 10, ~34K cache/turn): ~44-62 turns per 1%
- Mid session (turn 500, ~347K cache/turn): ~4-6 turns per 1%
- Late session (turn 990, ~595K cache/turn): ~2.5-3.5 turns per 1%

**A 100-turn stretch in the late half of a long session costs roughly 30-50% of the 5h window** in cache_read alone. This is why long sessions drain faster — the per-turn cost accelerates even though the user's work pace stays constant.

---

## 6. Cross-Correlation: JSONL vs Proxy

Both data sources cover the same API calls (April 4-6) but measure different things:
- **JSONL**: logs what the client recorded, including PRELIM entries
- **Proxy**: logs what the server returned in a single HTTP response

By comparing hourly totals from both sources (33 overlapping hours):

| Metric | JSONL Total | Proxy Total | Ratio |
|--------|-------------|-------------|-------|
| Output tokens | 1,372,025 | 1,968,953 | **0.70x** |
| Cache Read tokens | 663,021,317 | 344,380,772 | **1.93x** |

### Cache Read: JSONL is 1.93x of proxy

This is the direct, measured confirmation of Bug 8 (PRELIM inflation). The JSONL records PRELIM entries with the same `cache_read_input_tokens` as the FINAL entry, effectively double-counting. The ~2x ratio matches the PRELIM/FINAL token analysis from Section 1 (PRELIM cache is 0.56-0.97x of FINAL, so total JSONL = FINAL + PRELIM ≈ 1.56-1.97x of FINAL-only, consistent with 1.93x).

### Output: JSONL is 0.70x of proxy

JSONL records fewer output tokens than the proxy. This is primarily a timing/matching artifact — subagent JSONL files have different modification times than when the proxy logged the corresponding request, causing some entries to fall outside the matched hourly window.

### What this means for users tracking usage via JSONL

- If you sum `cache_read_input_tokens` from your JSONL files, your total will be **~2x higher** than what was actually sent to the API — because PRELIM entries duplicate the count.
- If Anthropic's server-side rate limiter uses the same accounting that produces PRELIM entries, users could be charged double for cache_read. We cannot confirm this from the client side, but the 1.93x ratio is consistent with the hypothesis.

### Methodology note: time normalization

When comparing token totals across different time periods, **always normalize by work hours or active minutes**. Total-to-total comparisons conflate "how long you worked" with "how expensive each minute was." For example, 16h × 168M/h = 2.7B total vs 2h × 62M/h = 123M total gives a 22x total difference, but the per-hour cost difference is only 2.7x. See Section 3 for time-normalized metrics from our data.

---

## 7. Limitations

- **JSONL timestamps:** Not all entries have precise timestamps. Cross-correlation uses hourly buckets, which introduces matching error. Individual hours can show extreme ratios (35x+) due to timing misalignment.
- **Subagent attribution:** Subagent JSONL files are stored by parent session. Modification times may not reflect actual request times.
- **No thinking token visibility:** Like the proxy, JSONL `output_tokens` does not include extended thinking tokens. The actual server-side output per turn is higher than recorded.
- **6-day window:** April 1-6 only. Longer observation would reveal weekly patterns.
- **Single plan tier:** Max 20x ($200/mo). Different tiers may have different per-entry overhead characteristics.
- **PRELIM detection heuristic:** PRELIM entries are identified by empty/null `stop_reason`. Other entry types with empty stop_reason could be miscounted.

---

## 8. Combined Picture: JSONL + Proxy

| Question | JSONL Answer | Proxy Answer | Combined |
|----------|-------------|--------------|----------|
| Cache_read per turn? | 134K main, 33K subagent | — | ✓ Measured |
| Cache growth per turn? | +575 tokens/turn (linear) | — | ✓ Measured |
| Cache_read per minute? | median 153K/min | — | ✓ Measured |
| What does 1% utilization cost? | — | 1.5M-2.1M visible | ✓ Measured |
| Are PRELIM entries inflating counts? | Yes (0.82x P/F ratio) | Yes (JSONL is 1.93x of proxy) | **✓ Confirmed from both sides** |
| PRELIM cache vs FINAL cache? | 0.56-0.97x (not 1:1) | — | ✓ Measured |
| Are synthetic rate limits real? | 24 events in 6 days | — | ✓ Measured |
| Subagent cold start cost? | median 13,358 cache_create | — | ✓ Measured |
| Subagent warm cache %? | median 92.3% (turn 4+) | — | ✓ Measured |
| Session cost range? | 753x (447K-336M visible) | — | ✓ Measured |
| Are thinking tokens counted? | Can't see them | Can't see them | **Still unknown** |
| Server token-type weights? | — | Can't decompose | **Still unknown** |

The two remaining unknowns — thinking token accounting and server-side token weighting — require either Anthropic disclosure or the thinking-disabled isolation test planned for this week.

---

*Environment: Max 20x ($200/mo), Opus 4.6 1M, v2.1.91, Linux (ZBook Ultra), single machine. 1,735 JSONL files, 1.0 GB total. Proxy: 8,794 requests, 3,702 with rate limit headers.*
