> **🇰🇷 [한국어 버전](ko/08_UPDATE-LOG.md)**

# Investigation Log — Daily Progress

> Date-by-date record of what was investigated, discovered, and published. Each entry documents the day's focus, methodology, and findings. Event counts in each entry reflect that day's data — for cumulative totals, see [13_PROXY-DATA.md](13_PROXY-DATA.md).

---

## April 27, 2026 — Factual Corrections Across 3 Documents

**Focus:** Cross-check all published documents against CHANGELOG v2.1.110–v2.1.119 and current GitHub issue statuses. Correct factual errors and stale references.

**What was done:**
- Fetched full CHANGELOG (v2.1.110–v2.1.119, 10 releases) and cross-referenced against all tracked issues from 16_ and 17_ chapters
- Verified current GitHub status of 12 tracked issues via `gh issue view`: #49302, #49503, #49541, #49555, #49585, #49593, #49618, #49747, #52228, #52502, #52522, #52534
- Full read-through of 16_, 17_, README, CROSS-VALIDATION, DATASET-ARKNILL, 09_QUICKSTART, 01_BUGS for factual accuracy

**Corrections applied (3 commits):**

1. **17_ Section 3.3 factual error** (amend to `4a8e964`): Claimed #52522 auto-compact threshold change was "Not mentioned in the v2.1.117 CHANGELOG." This is incorrect — v2.1.117 CHANGELOG explicitly documents: "Fixed Opus 4.7 sessions showing inflated `/context` percentages and autocompacting too early — Claude Code was computing against a 200K context window instead of Opus 4.7's native 1M." Corrected to acknowledge the CHANGELOG entry while noting the "Fixed" framing obscures the 5x cost impact.

2. **README stale version references** (`8bc38f8`): "v2.1.112 (latest)" → "v2.1.119" in 2 locations (Current Status body text + Environment section). Also capped the "Update to v2.1.91+" advice with explicit v2.1.109 upper bound and 4.7 advisory link, resolving inconsistency with the page-top warning.

3. **CROSS-VALIDATION + DATASET metadata** (`09fc28a`): Removed stale "Status: Working document (not committed)" headers from both documents — they were committed in `5500c2e` (April 22).

**Issue status snapshot (April 27):**

| Issue | Apr 24 status | Apr 27 status | Changed? |
|-------|---------------|---------------|----------|
| #49302 (cache metering) | OPEN | OPEN | No |
| #49503 (model pin bypass) | OPEN | OPEN | No |
| #49585 (smoosh cache) | OPEN | OPEN | No |
| #49618 (bash classifier) | OPEN | **CLOSED** | ✅ |
| #49593 (context bloat) | OPEN | **CLOSED** | ✅ |
| #49747 (XML/JSON mixing) | OPEN | OPEN | No |
| #52228 (self-conversation) | OPEN | OPEN | No |
| #52502 (subagent model pin) | OPEN | OPEN | No |
| #52534 (effort override) | OPEN | OPEN | No |

Two issues closed since April 24, but neither changes the upgrade recommendation — 9 checklist items remain unresolved.

**No new chapter or content additions.** All changes are corrections to existing documents.

**Published:**
- [17_OPUS-47-POSTMORTEM-ANALYSIS.md](17_OPUS-47-POSTMORTEM-ANALYSIS.md) — Section 3.3 factual correction
- [README.md](README.md) — stale version references + advisory consistency
- [CROSS-VALIDATION-20260422.md](CROSS-VALIDATION-20260422.md) — metadata fix
- [DATASET-ARKNILL-20260422.md](DATASET-ARKNILL-20260422.md) — metadata fix

---

## April 24, 2026 — Postmortem Analysis: What the Changelog Didn't Say

**Focus:** Cross-check Anthropic's [April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem) against the full public CHANGELOG (3,285 lines, all versions v2.1.68–v2.1.119). Identify post-postmortem issues on v2.1.117–119.

**What was done:**
- Downloaded and searched the complete CHANGELOG for all terms related to the 3 admitted bugs (effort default, thinking cache, verbosity prompt)
- Mapped each admitted bug to its introduction version, fix version, and CHANGELOG presence/absence
- Spot-checked 8 GitHub issues via `gh issue view` (7 confirmed, 1 partially confirmed)
- Verified 10 press/community URLs via WebFetch
- Cross-checked 10 numeric claims against primary sources
- Applied 3 corrections during the cross-check process (version attribution C1, failure count C2, quote attribution C3)
- Identified 5 new issues on v2.1.117–119 that fall outside the postmortem's scope

**Findings:**
- **CHANGELOG transparency gap:** 2 of 3 admitted bugs have **zero CHANGELOG documentation** — the thinking cache clearing bug (v2.1.85 → v2.1.101 fix) and the verbosity system prompt (v2.1.111 → v2.1.116 revert) were introduced and removed without any public record
- **Effort default framing:** The one documented change (effort `high` → `medium`) was framed as a product improvement ("sweet spot"), never acknowledged as a regression. The word "revert" never appears in any CHANGELOG entry related to effort defaults
- **Pro/Max 48-day gap:** Max subscribers (paying $100–$200/month) remained on `medium` effort for 48 days (March 4 – April 21) — the longest of any tier. v2.1.94 (April 7) fixed API-key/Bedrock/Vertex/Team/Enterprise; Pro/Max waited until v2.1.117 (April 21)
- **Post-postmortem issues (5 new):** #52502 (subagent model pin ignored — $10.87 vs $0.0005), #52534 (effort override bypass via `unpinOpus47LaunchEffort` flag), #52522 (auto-compact 5x usage), #52228 (self-conversation + unilateral action), #52652 (CLAUDE.md rule violation)
- **36 claims cross-checked:** 28 confirmed, 5 partially confirmed, 3 not relied upon

**Published:**
- [17_OPUS-47-POSTMORTEM-ANALYSIS.md](17_OPUS-47-POSTMORTEM-ANALYSIS.md) (new) — full postmortem cross-check with 7 sections
- [README.md](README.md) — April 24 latest update section, TL;DR updated with postmortem link
- [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) — Section 7 checklist updated with effort FIXED + 2 new post-postmortem items (#52502, #52534), pointer to 17_ added at document top

---

## April 22, 2026 — Three-Dataset Cross-Validation + Community Response

**Focus:** Cross-validate cache_read weight change findings across three independent datasets (ArkNill 45.8K, seanGSISG 215K, cnighswonger 101K). Respond to Issue #3 and #4 community contributions. Send first email response to cnighswonger.

**What was done:**
- Expanded proxy dataset to **45,884 requests** / 320 sessions / April 1–22 (from 38,996 / 272 / April 1–16)
- Model substitution check expanded to **41,306 requests** with both request and response model fields — **zero mismatches** (from 36,956 on April 19)
- Computed ArkNill April quota multiplier: total visible 5.22B / visible-without-cache_read 159M = **32.9x** (falls between seanGSISG 14.7x and cnighswonger 38.6x)
- New Q7d utilization analysis: **13.5% of Opus requests in 80–100% bucket** (vs 0.6% for Q5h) — the 7-day window binds harder than Q5h for sustained heavy users
- Cross-validated seanGSISG's Scripts v2: sliding window confirms higher peaks (Mar 140.9%, Apr 195.7%), counterfactual properly leads, tool_use estimate improved (mean=171 vs fixed 80)
- Validated seanGSISG's 4.7 self-correction: per-call multipliers confounded by workload variance within 4.6 window (avg input varies 2–93 tokens by day). Retraction clean. Structural assessment: 4.7 cost is architectural/pricing, not client-fixable
- Confirmed iterations = compounding factor (30.9pp gap), not independent driver
- Confirmed three-dataset CacheRead per 1% convergence: ArkNill 1.5–2.1M, seanGSISG 1.62–1.72M, cnighswonger 1.67–1.77M
- fallback-percentage 0.5 confirmed invariant: ArkNill 37,363/37,363, cnighswonger 14,000+/14,000+
- Sent cnighswonger email reply: fgrosswig assessment (technically solid, Gitea mirror + product pivot), 4.7 structural diagnosis, llm-relay introduction (multi-provider diversification)
- Posted Issue #3 comment with Q5h/Q7d bucket tables, 32.9x multiplier, three-dataset convergence declaration

**Findings:**
- **Three-dataset convergence** is the headline: CacheRead per 1% lands in the same 1.5–2.1M range across 3 independent datasets with different plans, geographies, and collection methods
- **Q7d is the overlooked constraint**: 13.5% of requests in Q7d 80–100% vs only 0.6% in Q5h 80–100%. For sustained users, the 7-day window accumulates pressure that the 5-hour window resets away
- **Before-data limitation resolved**: seanGSISG's Dec 2025 – Mar 2026 data + cnighswonger's January baseline (474 calls, >20x multiplier) close the gap. 02_RATELIMIT-HEADERS.md updated accordingly
- **Opus 4.7 is structural, not fixable**: Invisible thinking tokens + tokenizer inflation + metering changes are architectural/pricing decisions. DISABLE_ADAPTIVE_THINKING is the only workaround (3.3x reduction per cnighswonger), but it degrades reasoning depth

**Published:**
- [CROSS-VALIDATION-20260422.md](CROSS-VALIDATION-20260422.md) (new) — three-dataset convergence report
- [DATASET-ARKNILL-20260422.md](DATASET-ARKNILL-20260422.md) (new) — ArkNill-primary dataset analysis
- DATASET.md — removed (content absorbed into CROSS-VALIDATION and DATASET-ARKNILL documents above)
- [README.md](README.md) — April 22 latest update, contributors expanded, environment updated to 45,884/320
- [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) — before-data "partially resolved" → "resolved", header count 37,363
- [13_PROXY-DATA.md](13_PROXY-DATA.md) — totals updated to April 22, model check 41,306
- [15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md) — model check 41,306
- [14_DATA-SOURCES.md](14_DATA-SOURCES.md) — proxy requests 45,884
- Issue #3 comment — [cross-validation response](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3#issuecomment-4294208182)

---

## April 17, 2026 — Opus 4.7 Advisory Published

**Focus:** Synthesize Opus 4.7 risk data from three independent sources into a formal advisory recommending v2.1.109 as the safe version.

**What was done:**
- Collected and cross-referenced 80 Gmail notifications (April 15–17) spanning 73+ comments on [#42796](https://github.com/anthropics/claude-code/issues/42796), 39+ on [#38335](https://github.com/anthropics/claude-code/issues/38335), 23 on [fgrosswig/claude-gateway#1](https://github.com/fgrosswig/claude-gateway/issues/1), 7 on [cache-fix Discussion #25](https://github.com/cnighswonger/claude-code-cache-fix/discussions/25), and 5 on [#49585](https://github.com/anthropics/claude-code/issues/49585)
- Integrated self-measured benchmark data (3 effort levels × 3 hard tasks × 20-turn session isolation, n=3, v2.1.112/Opus 4.7 on hmj PC)
- Cross-validated cnighswonger's 2.4x Q5h measurement against fgrosswig's independent 2.6x gateway data — convergence at 2.4-2.6x confirms a real effect
- Documented fgrosswig's live A/B test: 12.5x cold-start, 50x worst-case single call
- Verified v2.1.109 safety: `claude -p --output-format json` shows explicit `claude-opus-4-6[1m]` model ID, `cache_creation_1h=43139 / cache_creation_5m=0` (native 1h cache)
- Conducted evidence quality assessment: logical consistency review (4 items), numerical rigor review (4 items), evidence gap analysis (4 items)
- Published [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) with 7 sections covering breaking changes, 5 critical unresolved issues, 4 independent measurements, v2.1.109 risk matrix, recommended configuration, full gap analysis, and upgrade reconsideration checklist

**Findings:**
- **2.4x averaged Q5h burn** on Opus 4.7 (cnighswonger, 71 calls / 38 min) — composed of tokenizer +35% and invisible adaptive thinking overhead ~105%. Cross-validated at 2.6x by fgrosswig
- **12.5x cold-start multiplier** (fgrosswig A/B, same account, same minute): 4.7 shows 301K cache_read vs 4.6 at 821K, yet burns 12.5x more quota. Hypothesis: thinking tokens partially replace cache reads
- **Model pin bypass** ([#49503](https://github.com/anthropics/claude-code/issues/49503)): `settings.json "model": "claude-opus-4-6"` ignored by v2.1.111+. Does not affect v2.1.109
- **Smoosh pipeline** ([#49585](https://github.com/anthropics/claude-code/issues/49585)): deafsquad identified per-turn smoosh fold of dynamic `<system-reminder>` text into `tool_result.content` — causes byte drift, breaking cache prefixes. cache-fix v2.0.0-beta.4 (smoosh_split) reduces `cache_creation` by 99.8% in production
- **Self-benchmark caveat**: Our controlled test shows stock ≈ intercept (~1x) because it used `--effort high`, print mode, English prompts, and 20-turn isolated sessions. This does not contradict the 2.4x — it confirms the overhead is conditional on session characteristics
- **Community sentiment**: Multiple subscription cancellations reported (mhbosch, vrathore18, MrRobot701, madeumendes), legal action discussed (dsmoz), media coverage emerging

**Published:**
- [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) (new) — Opus 4.7 advisory with v2.1.109 recommendation
- [README.md](README.md) — April 17 latest update, TL;DR updated with advisory link
- [09_QUICKSTART.md](09_QUICKSTART.md) — Opus 4.7 warning banner, v2.1.109 Option C, model pin configuration
- [08_UPDATE-LOG.md](08_UPDATE-LOG.md) — this entry

---

## April 16, 2026 — Data Source Re-Audit & DW Consolidation

**Focus:** Re-audit the labeling of previously published figures after noticing that the descriptor "single machine, 1,735 JSONL files" was imprecise.

**What was done:**
- Enumerated all local data assets: ubuntu-1 `~/.claude/projects/` (stock, 2,098 JSONL, 911 MB), ubuntu-1 an isolated override environment` (override env, 2,324 JSONL, 948 MB), win-1 `.claude/projects/` (Max 5x, 171 JSONL, 5.7 MB). Total: 4,593 JSONL / 512,149 messages / ~1.9 GB
- Re-loaded the cc-relay SQLite proxy database — 38,996 requests across 272 sessions, April 1–16 (previously reported as 35,554 / 251 / April 1–15)
- Created a new schema in an internal Postgres database with five tables and indexed every file/message/request with explicit environment labels (machine, tier, cc_mode, cc_version, proxy_stack)
- Verified the 96.9% cache_read dominance at the proxy layer on the fresh data (matches the previously published 97.3% figure, and independently cross-validates fgrosswig's gateway forensics at 97.3% of 98.7M tokens)
- Published [14_DATA-SOURCES.md](14_DATA-SOURCES.md) as the authoritative label matrix and reconciliation table
- Corrected the environment descriptors in [README.md](README.md), [13_PROXY-DATA.md](13_PROXY-DATA.md), [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md), and [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md): "single machine" is replaced with the explicit dataset label `ubuntu-1-stock` (CC stock mode) and a pointer to the parallel `ubuntu-1-override` dataset

**Findings:**
- The published analysis is entirely computed from `ubuntu-1-stock` and remains reproducible from the same sessions. Historical snapshot figures (17,610 requests on April 1–8; 532 JSONL files scanned on April 1–8) are kept as-is and annotated as historical
- Post-April 10 cache_read ratio under the overridden environment (`ubuntu-1-override`) is **97.08%** on 24,694 assistant turns vs **96.00%** on `ubuntu-1-stock` over the same period — consistent with the 1h TTL being preserved under the override
- The win-1 Max 5x dataset is deliberately excluded from the main published analysis to preserve the single-plan (Max 20x) control for all figures in this repo

**Published:**
- [14_DATA-SOURCES.md](14_DATA-SOURCES.md) (new) — data label matrix and reconciliation
- [15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md) (new) — per-environment cache_read, pre/post April 10 shift, Max 20x vs Max 5x model dispatch
- Label / figure updates to [README.md](README.md), [13_PROXY-DATA.md](13_PROXY-DATA.md), [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md), [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)
- Draft branch: `data-label-correction-20260416`

**Key findings from [15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md):**
- Post-April 10 cache_read ratio: `ubuntu-1-override` 97.08% (24,694 turns) vs `ubuntu-1-stock` 96.00% (4,357 residual turns) — the +1.08 pp gap is consistent with the 1h TTL being preserved under the override
- Daily trend: `ubuntu-1-override` reaches 98.16% cache_read on April 15 (8,175 assistant turns)
- Model dispatch differs by plan tier: Haiku accounts for ~21% of assistant turns on both Max 20x datasets but only **0.11%** on the Max 5x dataset (1 turn out of 895). Consistent with third-party observations of tier-dependent dispatcher behaviour (fgrosswig [#38335](https://github.com/anthropics/claude-code/issues/38335), cnighswonger Max 5x zero-mismatch)

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
- **Bug 5 (Budget Cap)** discovered: `applyToolResultBudget()` enforces 200K aggregate cap via `tengu_hawthorn_window` GrowthBook flag. **261 budget events** measured — tool results truncated to 1-41 chars (April 3 session; full-week max: 49 chars). v2.1.91 `maxResultSizeChars` override is MCP-only, built-in tools unaffected.
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

## April 9, 2026 — Community-Wide Fact-Check, 9 New Findings

**Focus:** Systematic collection and verification of all new issues/comments from April 6-9 (500+ new issues, 168 comments on #42796 alone, 110+ comments on 8 core issues). Full fact-check with evidence strength ratings.

**What was done:**
- Collected ALL new issues (500+) and comments across 9 core issues from April 6-9
- Classified each finding by evidence strength: STRONG / MODERATE / WEAK
- Cross-checked claims against independent sources, verified numerical consistency, identified logical gaps and alternative explanations
- Corrected factual error in our own Anthropic response bias analysis (see below)

**New bugs added (STRONG — 5 confirmed):**
- **Bug 8a (JSONL corruption):** Non-atomic writes during concurrent tool execution drop `tool_result` entries → permanent session corruption. 3 independent issues (#45286, #31328, #21321), 10+ duplicates in meta-issue.
- **Bug 9 (/branch inflation):** `/branch` duplicates message history, inflating 6%→73% context in one message. 3 duplicate issues confirm.
- **Bug 10 (TaskOutput thrashing):** Deprecation message causes 21x context injection (87K vs 4K) → triple autocompact → fatal error. JSONL log evidence.
- **Bug 11 (Adaptive thinking zero-reasoning):** bcherny (Anthropic) acknowledged on HN that adaptive thinking can emit zero reasoning → fabrication. Specific examples: stripe API version, git SHA suffix, apt package list. Workaround: `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`.
- **Bug 2a (SendMessage cache miss):** Agent SDK `SendMessage` resume produces complete cache miss (`cache_read=0`) including system prompt. Different from CLI resume bug. cnighswonger independently confirmed.

**Preliminary findings added (MODERATE — 4 conditional):**
- **P1/P2 (Cache TTL dual tiers):** Two triggers for 1h→5m TTL downgrade — (A) `DISABLE_TELEMETRY=1` (Anthropic `has repro`, n=1) and (B) quota 100% crossing (cnighswonger interceptor data). Likely one server-side mechanism with multiple disqualifying conditions.
- **P3 (Output efficiency prompt):** v2.1.64 (Mar 3) added "Try the simplest approach first" to system prompt. Confirmed via [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts). Likely aggravating factor, not sole cause.
- **P4 (Third-party detection gap):** Raw SDK calls with OAuth bill to plan instead of extra usage. HTTP header evidence. 42+ misclassification issues.

**Key quantitative data from community (with caveats):**
- **@wpank**: 47,810 requests, $10,700 spend. v2.1.63 vs v2.1.96 = 3.3x cost. *Caveat: unequal session durations (5.15h vs 76min) inflate headline number.*
- **@cnighswonger**: 4,700 calls, 98.3% cache hit with interceptor. 52-73% of calls needed block relocation, 98% had non-deterministic tool ordering. *Caveat: no controlled before/after baseline.*
- **v2.1.63 downgrade**: 4 independent confirmations of improvement. *Caveat: breno-ribeiro706 notes "still faster than a month ago" — server-side issues persist regardless of CLI version.*

**Anthropic response (April 6-9):**
- **bcherny**: 6 comments on #42796 (April 6 only), then complete silence April 7-8
- Acknowledged: thinking redaction = UI-only, adaptive thinking medium=85 default, duplicate-recording bug fix
- Did NOT address: system prompt change, v2.1.63 downgrade data, billing routing bug
- **HN (separate)**: bcherny acknowledged adaptive thinking zero-reasoning bug, model team investigating
- **All other issues (#38335, #41930, #42542, etc.):** 0 Anthropic responses

**★ Correction — Anthropic response bias:**
Previous analysis stated bcherny responded "to" stellaraccident (AMD director) within 4 hours. **Fact-check found this is incorrect.** bcherny's first response (Apr 6, 17:55 UTC) came **5 hours before** stellaraccident's first comment (Apr 6, 22:54 UTC). The trigger was HN virality (58 comments/day spike), not corporate affiliation. The actual bias is toward **visibility/virality**, not toward specific individuals. The broader silence pattern remains real: #38335 (478 comments, 15 days, 0 responses), #41930 (49 comments, 8+ days, 0 responses).

**New community tools & repos:**
- [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) — `NODE_OPTIONS` fetch interceptor fixing block position + tool ordering + image carry-forward (by [@cnighswonger](https://github.com/cnighswonger))
- [cc-trace](https://github.com/alexfazio/cc-trace) — mitmproxy-based CC API interception + analysis (★152) (by [@alexfazio](https://github.com/alexfazio))
- [X-Ray-Claude-Code-Interceptor](https://github.com/Renvect/X-Ray-Claude-Code-Interceptor) — Node.js proxy with payload analysis + smart stripping (by [@Renvect](https://github.com/Renvect))
- [claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) — Version-tracked CC system prompt diffs (by Piebald-AI)
- [openwolf](https://github.com/cytostack/openwolf) — Token usage stabilization (by cytostack)

**Published:** Bug updates to 01_BUGS.md, 07_TIMELINE.md, 08_UPDATE-LOG.md, 10_ISSUES.md.

### Changelog Cross-Reference (v2.1.92–v2.1.97)

**Focus:** Systematic cross-reference of [official changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) against all 9 unfixed bugs + 4 preliminary findings.

**What was done:**
- Reviewed complete changelog entries for v2.1.92, v2.1.94, v2.1.96, and v2.1.97 (v2.1.93 and v2.1.95 do not exist — skipped versions)
- Mapped every fix/feature against Bug Matrix (B3–B11, B2a, P1–P4)
- Identified false positives: v2.1.94 "429 rate-limit handling" and v2.1.97 "exponential backoff" fix **server 429 response** handling, not B3's **client-side synthetic** rate limiter (different code path). v2.1.92 "Write tool diff 60% faster" is diff computation speed, not B5 budget enforcement.

**Findings:**
- **0 of 9 unfixed bugs addressed** across 6 releases (8 days of development)
- **B11 symptoms reduced** (v2.1.94 effort default medium→high, v2.1.92 whitespace-only thinking crash fix) — root cause "investigating" per bcherny with no follow-up
- **B8 possibly partial** — v2.1.92 transcript accuracy fix may affect PRELIM duplication, but JSONL file-level verification needed
- **P3 confirmed still active** — "Output Efficiency" system prompt section (added v2.1.64, Mar 3) present verbatim in v2.1.97 system prompt, unchanged through 33 releases
- **B10 arguably worsened** — TaskOutput deprecation message more prominently embedded in v2.1.97 system prompt
- Development priorities in v2.1.92–97: Bedrock wizard, Cedar syntax highlighting, focus view toggle, `/tag` removal, footer layout, MCP buffer fixes, OAuth improvements — UI/infrastructure polish, not core accounting or context integrity

**Published:** New section added to [01_BUGS.md — Changelog Cross-Reference](01_BUGS.md#changelog-cross-reference-v2192v2197), README.md status/environment updated to reflect v2.1.97 verification.

---

## April 10-12 — No investigation (personal leave)

Offline for health reasons. cc-relay proxy continued collecting data unattended (ubuntu-1 stayed on). No analysis, no community engagement, no commits.

---

## April 13, 2026 — Catch-up: v2.1.101, P3 verified gone, proxy data extended

**Focus:** Returned from 3-day break. Caught up on community activity (Gmail notifications + GitHub API), then ran self-verification against local data before publishing anything.

**What was done:**

Reviewed ~200 GitHub notification emails and fetched recent issue/comment activity across 11 tracked threads. Two new CC versions shipped while I was away: v2.1.98 (security hardening) and v2.1.101 (resume/MCP fixes). v2.1.99 and v2.1.100 don't exist in the public changelog — skipped.

Cross-referenced both changelogs against the bug matrix, same methodology as the April 9 pass. Result: still zero fixes for B3–B11. The only meaningful change is B2a — v2.1.101 fixed CLI `--resume` cache misses for deferred tools/MCP/custom agents, which is the general category B2a falls into. But B2a's specific code path (Agent SDK `SendMessage` orchestrator) wasn't mentioned, so upgraded to POSSIBLY FIXED rather than FIXED.

**P3 self-verification:**
Scanned all 353 local JSONL session files for the exact "Output efficiency" text strings ("straight to the point", "do not overdo"). Found a clear boundary:
- April 8: 1 session PRESENT, rest ABSENT
- April 9: 5 sessions PRESENT, ~20 ABSENT (mixed — likely claudeGt on v2.1.91 vs auto-updated stock)
- **April 10 onward: 0 occurrences across ~30 sessions**

The text is gone. Can't pin the exact version (v2.1.99/100 don't exist), and the changelog doesn't mention it. First spotted by @wjordan (external, not Anthropic) via the Piebald-AI system prompt archive. Updated P3 status from PRELIMINARY to OBSERVED REMOVED.

**Proxy data extended:**
cc-relay kept running during my absence and continues collecting. Queried usage.db: **27,708 total requests** across **218 sessions** (April 1–13, 13 days). `fallback-percentage` = 0.5 on all requests with headers — same as the initial 3,702-request sample, just 7x more data. Also noted community cross-account data from #41930 (cnighswonger 11,502 calls Max 5x, 0xNightDev Max 5x EU) — included as reference with explicit caveat that the field's meaning is undocumented.

**First-turn cache miss measurement:**
Queried usage.db for first-turn `cache_read` across sessions with ≥3 requests: **113/143 (79%) start with cache_read=0**. This explains why users still complain about first-turn costs on v2.1.91+ even though B1/B2 are fixed. Community analysis (#47098 by @wadabum) identified the structural cause: skills and CLAUDE.md land in `messages[0]` instead of `system[]`, breaking prefix-based caching. Newer versions are improving this (community data shows ~29% zero-read on v2.1.104), but we measured 79% across our mixed-version dataset.

**Community context (observed, not investigated):**
The 3 days I was away were intense. #42796 saw a wave of subscription cancellations and competitor migration reports (Codex, GLM 5.1, Kimi 2.5). @0xNightDev filed EU consumer protection documentation. @cnighswonger and @fgrosswig shipped multiple tool versions (cache-fix v1.7.1, usage-dashboard v1.6.0). Several safety incidents reported (#46947 blockchain transfer, #46971 model self-injecting prompt injection). None of this was independently verified — just noted for context.

**Published:** Updates to 01_BUGS.md (changelog cross-reference v2.1.98–101, P3 status, first-turn cache note), 02_RATELIMIT-HEADERS.md (fallback-percentage extended data), README.md (April 13 section, status table, environment).

---

## April 15, 2026 — Data refresh, verified through v2.1.108

**Focus:** Proxy data refresh + changelog verification through v2.1.108.

**Data updated:**
- Proxy dataset expanded to **35,554 requests** across **251 sessions** (April 1–15). Further extended on April 16 to **38,996 / 272 / April 1–16**; see the April 16 entry below
- Post-barrier requests: **9,996** (April 10 14:25 – April 15), still zero B4/B5 events — 5 days of clean data
- Overall cache efficiency: **98.3%**
- Changelog verified through **v2.1.108** — still zero fixes for B3–B11

**Published:** Data refresh across README.md, 01_BUGS.md, 02_RATELIMIT-HEADERS.md, 05_MICROCOMPACT.md, 13_PROXY-DATA.md.

---

## Planned (as of April 15)

Carried forward from April 9, with updates:

- ~~Continue rate limit header data collection through April 10~~ ✅ Done (38,996 requests through April 16, dataset `ubuntu-1-stock`)
- ~~Verify P3 "Output efficiency" prompt~~ ✅ Done (OBSERVED REMOVED, 353 JSONL scan)
- **Thinking token isolation test**: still pending. Run sessions with `alwaysThinkingEnabled: false` and compare per-1% utilization cost
- **v2.1.92+ JSONL verification:** Check if B8 PRELIM duplication is reduced in transcript
- **B2a verification on v2.1.101:** Test Agent SDK `SendMessage` resume to confirm POSSIBLY FIXED → FIXED
- **`fallback-percentage` monitoring:** Track whether the value changes over time
- **Monitor #47098 (cache structure):** Track whether Anthropic moves skills/CLAUDE.md to `system[]` in future versions

---

*This log is updated with each significant investigation activity. See [07_TIMELINE.md](07_TIMELINE.md) for the broader 14-month history of the rate limit crisis.*
