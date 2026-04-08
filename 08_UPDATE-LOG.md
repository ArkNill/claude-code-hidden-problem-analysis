> **🇰🇷 [한국어 버전](ko/08_UPDATE-LOG.md)**

# Investigation Log — Daily Progress

> Date-by-date record of what was investigated, discovered, and published. Each entry documents the day's focus, methodology, and findings. Event counts in each entry reflect that day's data — for cumulative totals, see [13_PROXY-DATA.md](13_PROXY-DATA.md).

---

## April 1, 2026 — Initial Discovery

**Trigger:** Max 20 plan ($200/mo) hit 100% usage in ~70 minutes during normal coding.

**What was done:**
- Analyzed session JSONL (`9100c2d2`): **36.1% avg cache read** (min 21.1%) — should be 90%+
- Analyzed earlier session (`64d42269`): **47.0% avg** (233 entries, min 8.3%) — already degrading
- Downgraded from v2.1.89 to v2.1.68 (npm): cache immediately recovered to **97.6%** (119 entries)
- Confirmed: regression is v2.1.89-specific, not account or server issue
- Set up cc-relay transparent proxy using `ANTHROPIC_BASE_URL` for per-request monitoring going forward

**Findings:**
- **Bug 1 (Sentinel)** identified — community reverse engineering ([Reddit](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)) traced the cause to `cch=00000` sentinel replacement in the standalone binary's embedded Bun fork
- v2.1.89 standalone: 4-17% cache read, never recovers. npm not affected.

**Published:** Nothing yet — data collection phase.

---

## April 2, 2026 — Bugs 3-4 Discovered, Anthropic Responds

**Focus:** Proxy-based testing on v2.1.90 (released this day) + community collaboration.

**What was done:**
- cc-relay v2 proxy operational — logging per-request `cache_creation`, `cache_read`, `status_code`, `latency_ms`
- v2.1.90 benchmark started: npm vs standalone head-to-head on same machine, same proxy
- GrowthBook feature flag extraction confirmed ([@Sn3th](https://github.com/Sn3th)) — `~/.claude.json` contains `cachedGrowthBookFeatures`

**Findings:**
- **Bug 3 (False Rate Limiter)** discovered by [@rwp65](https://github.com/rwp65) in [#40584](https://github.com/anthropics/claude-code/issues/40584): client generates synthetic "Rate limit reached" errors (`model: "<synthetic>"`) without calling API. Triggered by large transcripts + concurrent sub-agents.
- **Bug 4 (Microcompact)** discovered by [@Sn3th](https://github.com/Sn3th) in [#42542](https://github.com/anthropics/claude-code/issues/42542): three compaction mechanisms silently strip tool results, controlled by server-side GrowthBook A/B flags. Initial hypothesis: cache invalidation.
- **Anthropic response:** Lydia Hallie [posted on X](https://x.com/lydiahallie/status/2039800715607187906): *"Peak-hour limits are tighter and 1M-context sessions got bigger... We fixed a few bugs along the way, but none were over-charging you."*

**Published:** Nothing yet — still testing.

---

## April 3, 2026 — Bug 5 Discovered, v2.1.91 Benchmark, Repo Goes Public

**Focus:** v2.1.91 full benchmark + new bug discovery + comprehensive testing.

**What was done:**
- v2.1.91 released — ran full benchmark: npm vs standalone head-to-head
- Enhanced cc-relay with microcompact detection and budget enforcement scanning
- GrowthBook flag survey across 4 machines / 4 accounts
- Bug 4 cache impact **corrected**: proxy data shows main session cache stays 99%+ during clearing — impact is context quality, not cache invalidation
- Commented on 91+ GitHub issues with analysis cross-references

**Findings:**
- **Bug 5 (Budget Cap)** discovered: `applyToolResultBudget()` enforces 200K aggregate cap via `tengu_hawthorn_window` GrowthBook flag. **261 budget events** measured — tool results truncated to 1-41 chars. v2.1.91 `maxResultSizeChars` override is MCP-only, built-in tools unaffected.
- **Bug 8 (JSONL Duplication)** measured: extended thinking generates 2-5 PRELIM entries per API call. Main session: **2.87x** token inflation.
- **v2.1.91 benchmark results:** Sentinel gap closed — npm hit 84.5% cold start; standalone varied by workspace (27.8% in full benchmark, higher in preliminary tests). Cache recovers to 95%+ within a few requests. All bugs except B1-B2 persist.
- **Bug 3 confirmed:** 151 `<synthetic>` entries across 65 sessions on our setup.
- **Bug 4 event count:** 327 microcompact clearing events across the April 3 focused test sessions.

**Published:**
- Repository created: [claude-code-cache-analysis](https://github.com/ArkNill/claude-code-cache-analysis) (now renamed)
- [README.md](README.md), [04_BENCHMARK.md](04_BENCHMARK.md), [05_MICROCOMPACT.md](05_MICROCOMPACT.md), [07_TIMELINE.md](07_TIMELINE.md), [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md)
- Comments on 91+ GitHub issues

---

## April 4, 2026 — Rate Limit Header Capture Begins

**Focus:** cc-relay enhancement for rate limit header capture + operational hardening.

**What was done:**
- Enhanced cc-relay to capture `anthropic-ratelimit-unified-*` response headers → stored in `ratelimit_headers` column (JSON)
- cc-relay streaming hang fix: `client.request()` → `client.send(req, stream=True)` for true streaming proxy
- Added systemd watchdog (60s ping loop, `WatchdogSec=120`, `RestartSec=3`)
- `loginctl enable-linger` for pre-login service start

**Data collected:**
- **2,097 requests with rate limit headers** (first day of capture)
- Headers include: `5h-utilization`, `7d-utilization`, `representative-claim`, `fallback-percentage`, `overage-status`
- First observation: `representative-claim` = `five_hour` in 100% of requests

**Community:**
- [@dancinlife](https://github.com/dancinlife): discovered `organizationUuid`-based quota pooling — accounts under same org share rate limits. Added as contributor.

**Published:** dancinlife contributor addition to repo.

---

## April 5, 2026 — Data Accumulation, Community Analysis

**Focus:** Continued header data collection + community findings review.

**Data collected:**
- **1,333 additional requests with headers** (total now 3,430+)
- Total proxy DB: 8,794 requests, 1,245 microcompact events, 23,021 budget events. (327 events were from the April 3 focused test sessions; the 1,245 total includes all sessions captured by the proxy since April 1.)

**Community activity:**
- **[@fgrosswig](https://github.com/fgrosswig)** published [64x budget reduction forensics](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) in #38335: dual-machine 18-day JSONL analysis. Mar 26: 3.2B tokens no limit → Apr 5: 88M at 90%. Hypothesis: cache-read weight changed from ~0x to ~1x.
- **[@Commandershadow9](https://github.com/Commandershadow9)** published [34-143x capacity reduction](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296) in #41506: cache fix confirmed, but capacity dropped 34-143x independent of cache bug. Raised thinking token accounting hypothesis.
- **[@Sn3th](https://github.com/Sn3th)**: continued collaboration on #42542 findings.
- **[@marlvinvu](https://github.com/marlvinvu)** in #41346: correlated PRELIM 2-3x inflation with phantom usage reports across multiple issues. Suggested PRELIM + synthetic compound effect.

**Published:** kolkov resume regression reference added to repo.

---

## April 6, 2026 — Rate Limit Header Analysis, Repo Restructure

**Focus:** Analyze 48h of captured headers + publish findings + repo overhaul.

**Analysis performed:**
- Split 3,702 requests into **8 distinct 5h windows** based on `5h-reset` header timestamps
- Calculated **per-1% utilization cost**: 9K-16K visible output, 1.5M-2.1M cache_read per percentage point
- Identified **thinking token blind spot**: `output_tokens` excludes thinking tokens, visible output explains <50% of observed utilization
- Mapped **7d accumulation pattern**: ratio ~0.12-0.17 relative to 5h peak (6 data points, low precision)
- Established **v2.1.89 separation framework**: golden (Mar 23-27) / bug (Mar 28-Apr 1) / post-fix (Apr 2+) for clean capacity comparison
- Cross-validated against fgrosswig and Commandershadow9 data — consistent with cache-read weighting change and/or thinking token accounting change
- **JSONL session log analysis** (110 main + 279 subagent sessions, April 1-6):
  - 24 synthetic rate limit entries (B3) confirmed in own data
  - PRELIM/FINAL ratio: 0.82x overall, up to 1.14x on heavy days
  - Subagent analysis: 17.3% of cache_read, 40.4% of output, 62% of cache_create
  - Session lifecycle: cache_read grows 24x over 990 turns (25K→595K per turn, +575/turn linear)
  - **Proxy ↔ JSONL cross-correlation: JSONL cache_read is 1.93x of proxy** — directly confirms PRELIM double-counting (B8). Users tracking via JSONL see ~2x inflated numbers.

**Methodology notes:**
- Windows B (peak 0.04) and H (peak 0.02) excluded from per-1% analysis — low activity
- 5h boundaries approximately 5-hour spaced but not perfectly uniform (19:00→01:00 = 6h, 01:00→12:00 = 11h gap during inactive hours)
- Per-1% cache_read variance ~1.35x between windows — workload-dependent
- Plan-specific data (Max 20x/$200) — per-1% costs may differ on other tiers
- 48h / 8 windows is a small sample. Full 7-day cycle completes April 10.

**Comments posted:**
- [#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189807108) — proxy data shared with @fgrosswig, complementary analysis (their JSONL + our headers)
- [#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189847482) — proxy data shared with @Commandershadow9, methodology notes on their 34x calculation and thinking token hypothesis

**Repo changes:**
- Repository renamed: `claude-code-cache-analysis` → `claude-code-hidden-problem-analysis`
- **RATELIMIT-HEADERS.md** created: full header analysis with per-window data
- **README.md** restructured (653→175 lines): Latest Update at top, investigation timeline, Anthropic counterpoints, document index
- **BUGS.md** created: Bug 1-5, 8 details moved from README
- **QUICKSTART.md** created: setup guide + self-diagnosis moved from README
- **ISSUES.md** created: 91+ issues + community tools + contributors moved from README
- **UPDATE-LOG.md** created: this document
- Existing docs updated: BENCHMARK (issue count), TIMELINE (April 6 section, Bug 4 wording fix)
- Contributors added: @fgrosswig, @Commandershadow9, @kolkov, @simpolism (expanded)

**Published:** All of the above.

---

## Planned (as of April 6)

The following items were identified during the April 1-6 analysis cycle and remain pending:

- Continue rate limit header data collection through April 10 (7d window reset)
- **Thinking token isolation test**: run sessions with `alwaysThinkingEnabled: false` and compare per-1% utilization cost. If it drops significantly → thinking tokens are the main driver. If not → cache-read weighting is primary.
- Publish full 7-day cycle analysis with per-window utilization tracking
- Monitor community responses to #38335 and #41506 comments

---

*This log is updated with each significant investigation activity. See [07_TIMELINE.md](07_TIMELINE.md) for the broader 14-month history of the rate limit crisis.*
