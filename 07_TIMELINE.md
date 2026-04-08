> **🇰🇷 [한국어 버전](ko/07_TIMELINE.md)**

# Claude Code Rate Limit Crisis — Full Timeline

> 14-month chronicle of rate limit / token consumption issues in Claude Code.
> Data collected from GitHub Issues via API on 2026-04-02. Event counts reference their discovery-date baselines.
>
> **Latest data (April 8):** Full-week proxy dataset (17,610 requests, 129 sessions) with updated event counts in [13_PROXY-DATA.md](13_PROXY-DATA.md).

---

## Summary

| Metric | Value |
|--------|-------|
| **Duration** | 14 months (2025-02 ~ 2026-04-03, ongoing) |
| **Major escalation cycles** | Multiple across 9 phases (notably Phases 4, 6, 8, and 9) |
| **Largest issue** | #16157 — 1,422 comments, 647 thumbs_up |
| **Root causes identified** | 11 client-side + 6 server-side (compound) |
| **Anthropic official response** | X/Twitter only (Lydia Hallie, April 2-3). Zero GitHub responses across 91+ issues |

---

## Phase 1 — Early Signals (2025-02 ~ 2025-05)

Sporadic, individual reports. Treated as isolated user issues.

| Date | Issue | Title | Comments |
|------|-------|-------|----------|
| 2025-02-24 | [#16](https://github.com/anthropics/claude-code/issues/16) | Add ability to add per-key spend limits | 3 |
| 2025-02-28 | [#236](https://github.com/anthropics/claude-code/issues/236) | API Error on AWS bedrock (429) | 2 |
| 2025-03-03 | [#274](https://github.com/anthropics/claude-code/issues/274) | IPYNB Cell Output Consumes Significant Portion of Context Window | 2 |
| 2025-03-26 | [#624](https://github.com/anthropics/claude-code/issues/624) | Potential Token Consumption Anomaly in Anthropic API Interaction | 2 |
| 2025-05-17 | [#1138](https://github.com/anthropics/claude-code/issues/1138) | [BUG] Cost increased a lot | 6 |

---

## Phase 2 — Post Claude 4 Launch (2025-05-22 ~ 2025-06)

Claude 4 release triggered the first wave. Users report 10x faster consumption.

| Date | Issue | Title | Comments |
|------|-------|-------|----------|
| 2025-05-23 | [#1266](https://github.com/anthropics/claude-code/issues/1266) | **[BUG] Claude 4 is BURNING USAGE LIMITS x10 TIMES FASTER** | 9 |
| 2025-05-24 | [#1287](https://github.com/anthropics/claude-code/issues/1287) | Misleading Cost Command Output in Claude Code | 10 |
| 2025-05-26 | [#1328](https://github.com/anthropics/claude-code/issues/1328) | Claude AI usage limit reached | 9 |
| 2025-05-30 | [#1439](https://github.com/anthropics/claude-code/issues/1439) | Sonnet 4 is completely unusable because of 429s | 7 |
| 2025-06-02 | [#1492](https://github.com/anthropics/claude-code/issues/1492) | Hitting usage limits after 2 hours | 3 |
| 2025-06-09 | [#1836](https://github.com/anthropics/claude-code/issues/1836) | Claude AI usage limit reached | 4 |

---

## Phase 3 — Structural Defects Discovered (2025-07 ~ 2025-08)

Shift from individual complaints to recognition of architectural flaws.

| Date | Issue | Title | Comments | Key Finding |
|------|-------|-------|----------|-------------|
| 2025-07-10 | [#3300](https://github.com/anthropics/claude-code/issues/3300) | ADD LIMITS TO CLAUDE TOKEN CONSUMPTION | 4 | All-caps title = user frustration |
| 2025-07-12 | [#3406](https://github.com/anthropics/claude-code/issues/3406) | Built-in tools + MCP descriptions load on first message causing 10-20k token overhead | 7 | **Root cause: MCP overhead 10-20K per message** |
| 2025-07-15 | [#3572](https://github.com/anthropics/claude-code/issues/3572) | Anthropic API Overloaded Error with Repeated 529 Status Codes | 274 | Server-side overload |
| 2025-07-17 | [#3804](https://github.com/anthropics/claude-code/issues/3804) | Unexpected Token Consumption with No Code Generation | 3 | Token drain without code generation |
| 2025-07-17 | [#3843](https://github.com/anthropics/claude-code/issues/3843) | Excessive Token Consumption and Prolonged Processing in New Sessions | 4 | Even new sessions affected |
| 2025-07-21 | [#4095](https://github.com/anthropics/claude-code/issues/4095) | Massive Token Consumption — **1.67B tokens in 5 hours** | 5 | **1.67 billion tokens in 5 hours** |

---

## Phase 4 — Silent Quota Reduction + Mass Revolt (2025-09 ~ 2025-10)

Anthropic silently reduced weekly limits server-side. First major trust breach.

| Date | Issue | Title | Comments | Reactions |
|------|-------|-------|----------|-----------|
| 2025-10-07 | [#9094](https://github.com/anthropics/claude-code/issues/9094) | **[Meta] Unexpected change in Claude usage limits as of 2025-09-29** (30+ reports) | **121** | 60 |
| 2025-10-12 | [#9424](https://github.com/anthropics/claude-code/issues/9424) | **Weekly Usage Limits Making Claude Subscriptions Unusable** | **109** | **155** |

**Key facts:**
- Pro users: weekly usable hours dropped from 40-50h → 6-8h
- All tiers affected (Pro, Max 5x, Max 20x)
- "Bait and switch" accusations
- Subscription cancellations + competitor migration reported

---

## Phase 5 — OAuth Bug + Compaction Issues (2025-11 ~ 2025-12)

Server-side quota problems + client-side bugs compound.

| Date | Issue | Title | Comments | Key Finding |
|------|-------|-------|----------|-------------|
| 2025-11-01 | [#10784](https://github.com/anthropics/claude-code/issues/10784) | OAuth expiration causes retry storm consuming 100% session usage (3.75M wasted tokens) | 7 | **OAuth retry storm: 3.75M tokens wasted** |
| 2025-11-22 | [#12149](https://github.com/anthropics/claude-code/issues/12149) | Excessive weekly usage limit consumption (71% in 2 prompts) | 3 | 71% consumed in 2 prompts |
| 2025-12-01 | [#12786](https://github.com/anthropics/claude-code/issues/12786) | Rate limit incorrectly applied when switching between Max accounts | 3 | Cross-account rate limit leak |

---

## Phase 6 — The Mega Thread: Issue #16157 (2026-01)

The single largest issue in Claude Code history.

| Date | Issue | Title | Comments | Reactions |
|------|-------|-------|----------|-----------|
| **2026-01-03** | [**#16157**](https://github.com/anthropics/claude-code/issues/16157) | **[BUG] Instantly hitting usage limits with Max subscription** | **1,422** | **666** (647 thumbs_up) |
| 2026-01-04 | [#16270](https://github.com/anthropics/claude-code/issues/16270) | Usage limits bugged after double limits expired on new years | 40 | 71 |
| 2026-01-08 | [#16856](https://github.com/anthropics/claude-code/issues/16856) | Excessive token usage in Claude Code 2.1.1 — 4x+ faster rate consumption | **63** | 73 |
| 2026-01-09 | [#17016](https://github.com/anthropics/claude-code/issues/17016) | Claude hitting usage limits in no time, something broken for sure | 22 | 9 |
| 2026-01-09 | [#17084](https://github.com/anthropics/claude-code/issues/17084) | Opus 4.5 usage limits significantly reduced since January 2026 | **39** | 54 |
| 2026-01-21 | [#19673](https://github.com/anthropics/claude-code/issues/19673) | You've hit your limit — While usage is still at 84% | **99** | 74 |

**Key facts:**
- Max subscribers hitting limit immediately after reset
- Usage meter at 84% but "limit reached" — internal accounting mismatch
- #16157 became the mega-thread for all rate limit complaints

---

## Phase 7 — Opus 4.6 + Second Explosion (2026-02)

Silent auto-upgrade to Opus 4.6 + higher token consumption + compaction bugs.

| Date | Issue | Title | Comments | Reactions |
|------|-------|-------|----------|-----------|
| 2026-02-01 | [#22435](https://github.com/anthropics/claude-code/issues/22435) | Inconsistent and Undisclosed Quota Accounting Changes. **Legal liability claim ready.** | 11 | 17 |
| 2026-02-06 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | **Opus 4.6 token consumption significantly higher than 4.5** | **41** | 72 |
| 2026-02-08 | [#24179](https://github.com/anthropics/claude-code/issues/24179) | **Compaction death spiral — 211 compactions** consuming all tokens with zero progress | 8 | 2 |
| 2026-02-08 | [#24283](https://github.com/anthropics/claude-code/issues/24283) | Memory files loaded twice in git worktrees causing premature compaction | 3 | — |
| 2026-02-09 | [#24315](https://github.com/anthropics/claude-code/issues/24315) | Repeated context compaction drains weekly token budget — 9 consecutive compactions | 4 | — |
| 2026-02-14 | [#25769](https://github.com/anthropics/claude-code/issues/25769) | Misleading 'Rate limit reached' error during service outage | 2 | — |
| 2026-02-17 | [#26404](https://github.com/anthropics/claude-code/issues/26404) | Weekly token usage limit draining abnormally fast after Sonnet 4.6 update | 6 | 1 |
| 2026-02-23 | [#27869](https://github.com/anthropics/claude-code/issues/27869) | Chrome MCP screenshots accumulate — 17% of Max plan for 5 trivial turns | 6 | — |
| 2026-02-26 | [#28848](https://github.com/anthropics/claude-code/issues/28848) | **Max plan usage limits silently reduced since Claude 4.6 release** | **26** | 31 |
| 2026-02-28 | [#29579](https://github.com/anthropics/claude-code/issues/29579) | **Rate limit reached despite Max subscription and only 16% usage** | **139** | 79 |

**Key facts:**
- Opus 4.6 auto-upgraded without notice
- 211 consecutive compactions = infinite loop draining entire budget
- 16% usage showing "rate limit reached" = clear accounting error
- Legal liability mentioned for the first time

---

## Phase 8 — Cache Regression Confirmed + March Crisis (2026-03)

Root causes identified with measured data.

| Date | Issue | Title | Comments | Reactions | Significance |
|------|-------|-------|----------|-----------|-------------|
| 2026-03-15 | [**#34629**](https://github.com/anthropics/claude-code/issues/34629) | **Prompt cache regression since v2.1.69: cache_read never grows, ~20x cost increase** | 18 | 36 | **ROOT CAUSE #1 confirmed with data** |
| 2026-03-22 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Claude Code Usage for Max Plan hitting limits extremely fast | **59** | 35 | March 22 escalation point |
| 2026-03-24 | [**#38335**](https://github.com/anthropics/claude-code/issues/38335) | **Session limits exhausted abnormally fast since March 23** | **313** | **258** | **Third mega-thread** |
| 2026-03-24 | [#38357](https://github.com/anthropics/claude-code/issues/38357) | Max 20x: Usage meter climbing abnormally fast — 1-2% per simple message | 8 | 3 | Forensic analysis: cache 92.7% OK but meter still abnormal = server-side issue |
| 2026-03-29 | [**#40524**](https://github.com/anthropics/claude-code/issues/40524) | **Conversation history invalidated on subsequent turns** | **59** | **197** | **ROOT CAUSE #2: full cache rebuild on every turn** |
| 2026-03-31 | [#41249](https://github.com/anthropics/claude-code/issues/41249) | Excessive token consumption — usage depleting faster than expected | 9 | 16 | <1 hour to full depletion |
| 2026-03-31 | [#41506](https://github.com/anthropics/claude-code/issues/41506) | Token usage increased ~3-5x without configuration change (since ~March 28-29) | 9 | 9 | 57 plugins + 9 MCP = ~40K overhead/request |
| 2026-03-31 | [#41590](https://github.com/anthropics/claude-code/issues/41590) | **New account immediately hits rate limit with zero usage** | 14 | 0 | System-level defect |

---

## Phase 9 — April 2026: Ongoing (Current)

15+ new rate limit issues filed in a single day. Worldwide simultaneous reports.

| Date | Issue | Title | Comments |
|------|-------|-------|----------|
| 04-01 | [#41788](https://github.com/anthropics/claude-code/issues/41788) | Max 20: 100% in ~70 minutes (my report) | 11 |
| 04-01 | [#42052](https://github.com/anthropics/claude-code/issues/42052) | Max 20x: 100% usage after 2 hours of light work | 13 |
| 04-01 | [#42095](https://github.com/anthropics/claude-code/issues/42095) | 5 hour limit reached with just 4 prompts | — |
| 04-01 | [#42183](https://github.com/anthropics/claude-code/issues/42183) | Excessive token consumption without reading any files | — |
| 04-01 | [#42249](https://github.com/anthropics/claude-code/issues/42249) | Extreme token consumption — quota depleted in minutes | — |
| 04-01 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | Resume loads disproportionate tokens from opaque thinking signatures | — |
| 04-01 | [#42272](https://github.com/anthropics/claude-code/issues/42272) | Excessive consumption since 2.1.88 — 66% usage in 2 questions | — |
| 04-01 | [#42261](https://github.com/anthropics/claude-code/issues/42261), [#42259](https://github.com/anthropics/claude-code/issues/42259), [#42145](https://github.com/anthropics/claude-code/issues/42145), [#42132](https://github.com/anthropics/claude-code/issues/42132), [#42129](https://github.com/anthropics/claude-code/issues/42129), [#42104](https://github.com/anthropics/claude-code/issues/42104) | "You've hit your limit" — simultaneous reports (London, Berlin, Tokyo, Calcutta, Paris, Istanbul) | — |
| 04-02 | [#42338](https://github.com/anthropics/claude-code/issues/42338) | Session resume invalidates entire prompt cache | — |
| 04-02 | [#42390](https://github.com/anthropics/claude-code/issues/42390) | Rate limit triggered despite 0% usage in /usage | 3 |
| 04-02 | [#42409](https://github.com/anthropics/claude-code/issues/42409) | Excessive API usage consumption during active session | 4 |
| 04-02 | [**#42542**](https://github.com/anthropics/claude-code/issues/42542) | **Silent context degradation — 3 microcompact mechanisms strip tool results** | 2 | **ROOT CAUSE #3: GrowthBook-controlled compaction degrades context quality (cache 99%+ maintained)** |
| 04-02 | [**#42569**](https://github.com/anthropics/claude-code/issues/42569) | **1M context incorrectly shown as extra billable usage on Max plan** | 1 | **Server-side billing regression** |
| 04-02 | [#42583](https://github.com/anthropics/claude-code/issues/42583) | You've hit your limit — 1M actual vs 120-160K expected (v2.1.90) | 1 |
| 04-02 | [#42590](https://github.com/anthropics/claude-code/issues/42590) | Context compaction too aggressive on 1M context window (Opus 4.6) | 1 |
| 04-02 | [#42592](https://github.com/anthropics/claude-code/issues/42592) | Token consumption 100x faster after v2.1.88 — 21 min to 5-hour limit | 4 |
| 04-02 | [#42609](https://github.com/anthropics/claude-code/issues/42609) | Reached limit session in under 5 minutes (resume-triggered) | 1 |
| 04-02 | [**#42616**](https://github.com/anthropics/claude-code/issues/42616) | **Spurious 429 "Extra usage required" at 23K tokens on Max plan with 1M** | 1 | **Server-side: debug log proves API rejected valid request** |

### Rate Limit Header Analysis (April 6)

Transparent proxy (cc-relay) captured `anthropic-ratelimit-unified-*` response headers across 3,702 requests (April 4–6), revealing the server-side quota architecture:

- **Dual sliding window**: 5-hour + 7-day independent counters. `representative-claim` = `five_hour` in 100% of requests — the 5h window is always the bottleneck.
- **Per-1% cost**: 9K–16K visible output, 1.5M–2.1M cache_read per percentage point of 5h utilization (cache_read = 96–99% of visible cost).
- **Thinking token blind spot**: `output_tokens` in API response excludes thinking tokens. Visible output alone explains <50% of observed utilization — the remainder is likely thinking tokens, cache-read weighting, or both.
- **Community cross-validation**: @fgrosswig published [64x budget reduction forensics](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) (dual-machine 18-day JSONL: Mar 26 3.2B tokens no limit → Apr 5 88M at 90%). @Commandershadow9 published [34–143x capacity reduction analysis](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296) confirming cache fix but documenting capacity drop independent of cache bug. Both independently identified thinking tokens as a likely factor.
- **v2.1.89 separation**: Clean comparison framework established — golden period (Mar 23–27, cache 98–99%) vs v2.1.89 bug period (Mar 28–Apr 1, excluded as confounding variable) vs post-fix period (Apr 2+, cache 84–97%, capacity reduction visible).

Full analysis: [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)

### New Root Causes Discovered (April 2-3)

Two significant discoveries by community investigation:

**Bug 3 — Client-side false rate limiter** ([#40584](https://github.com/anthropics/claude-code/issues/40584), discovered by [@rwp65](https://github.com/rwp65)):
Local rate limiter generates synthetic "Rate limit reached" errors (`model: "<synthetic>"`, `input_tokens: 0`) without calling the API. Triggered by large transcripts + concurrent sub-agents. Cross-referenced by [@marlvinvu](https://github.com/marlvinvu) across [#40438](https://github.com/anthropics/claude-code/issues/40438), [#39938](https://github.com/anthropics/claude-code/issues/39938), [#38239](https://github.com/anthropics/claude-code/issues/38239). **Unfixed in v2.1.90.**

**Bug 4 — Silent microcompact → context quality degradation** ([#42542](https://github.com/anthropics/claude-code/issues/42542), discovered by [@Sn3th](https://github.com/Sn3th)):
Three compaction mechanisms (`microCompact.ts:422`, `microCompact.ts:305`, `sessionMemoryCompact.ts:57`) silently strip tool results on every API call, controlled by server-side GrowthBook A/B flags. Proxy testing (April 3) showed **cache ratio stays 99%+ in main sessions** during clearing — the stable substitution preserves the prompt prefix. The actual cost is **context quality degradation** (model loses access to earlier tool results). Also explains why old Docker versions started draining recently ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — server-side flags changed without client update. **Unfixed through v2.1.91, server-controlled.**

**Bug 5 — Tool result budget enforcement** (discovered by [@Sn3th](https://github.com/Sn3th), confirmed by proxy April 3):
`applyToolResultBudget()` runs before microcompact in the request pipeline, enforcing a 200K aggregate cap on tool results via `tengu_hawthorn_window` GrowthBook flag. Per-tool caps (Grep 20K, Bash 30K) via `tengu_pewter_kestrel`. 261 budget events measured in one session — tool results truncated to 1-41 chars. v2.1.91 `maxResultSizeChars` override is MCP-only. **Unfixed for built-in tools.**

**Server-side 1M billing regression** ([#42616](https://github.com/anthropics/claude-code/issues/42616), [#42569](https://github.com/anthropics/claude-code/issues/42569)):
Max plan 1M context (free since March 13) incorrectly classified as "extra usage." Debug log shows 429 at 23K tokens with request ID `req_011CZf8TJf84hAUziB6LuRoc`. **Server-side bug, unfixed.**

### Official Fixes Shipped (v2.1.89-90, April 1)

Anthropic shipped cache-related fixes in v2.1.89-90 without any GitHub issue response. Fixes were confirmed via [changelog](https://code.claude.com/docs/en/changelog) and personal X posts from Anthropic staff.

| Version | Fix | Addresses |
|---------|-----|-----------|
| v2.1.89 | Prompt cache misses from tool schema bytes changing mid-session | Bug 1 (partial) |
| v2.1.89 | StructuredOutput schema cache bug (~50% failure rate) | Cache stability |
| v2.1.89 | Autocompact thrash loop detection + stop | Token drain loop |
| v2.1.89 | Memory leak in LRU cache keys | Long session stability |
| v2.1.89 | Nested CLAUDE.md re-injection (dozens of times) | Context bloat |
| v2.1.90 | **`--resume` full prompt-cache miss (regression since v2.1.69)** | **Bug 2 (official fix)** |
| v2.1.90 | **Per-turn JSON.stringify of MCP tool schemas eliminated** | Cache key stability |
| v2.1.90 | Rate-limit options dialog infinite loop | Session crash |
| v2.1.90 | SSE large frame quadratic → linear | Performance |

**v2.1.91 (April 2-3):**

| Version | Fix | Addresses |
|---------|-----|-----------|
| v2.1.91 | `_meta["anthropic/maxResultSizeChars"]` up to 500K | Bug 10 — **MCP-only workaround** (built-in tools unaffected) |
| v2.1.91 | `--resume` transcript chain break fix | Bug 2 (additional fix) |
| v2.1.91 | Edit tool shorter `old_string` anchors | Output token reduction |

**Staff response (April 2-3):**
- [Lydia Hallie](https://x.com/lydiahallie/status/2039800715607187906) (thread): *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."* Community reaction was strongly negative — measured data from multiple independent investigators shows additional unfixed bugs (see [analysis repo](https://github.com/ArkNill/claude-code-cache-analysis)).
- [Lydia Hallie](https://x.com/lydiahallie/status/2039107775314428189) (earlier): *"We shipped some fixes on the Claude Code side that should help"*
- [Thariq Shihipar](https://x.com/trq212/status/2027232172810416493): Confirmed prompt caching bugs being investigated (earlier incident)

**Independent verification:** Controlled benchmarks confirm v2.1.90-91 achieve 82-86% overall cache read and 95-99% in stable sessions. v2.1.91 closes the npm/standalone gap (84.7% identical cold start). See [04_BENCHMARK.md](04_BENCHMARK.md).

---

## Identified Root Causes

These are compound — multiple bugs interact to produce the observed behavior.

Note: The numbering below (1-11) is a chronological discovery order for this timeline. For the actively tracked bugs, the mapping to the B-series identifiers used in [01_BUGS.md](01_BUGS.md) is: #1->B1, #2->B2, #8->B3, #9->B4, #10->B5, #11->B8. Items #3-7 were identified during the investigation but are tracked separately as they relate to older or less reproducible issues.

### Client-Side Bugs

| # | Root Cause | Issue | Version | Impact |
|---|-----------|-------|---------|--------|
| 1 | **Cache sentinel replacement** — standalone binary's Bun fork breaks cache prefix | [#40524](https://github.com/anthropics/claude-code/issues/40524) | All standalone | Full cache rebuild every turn. **Partially fixed in v2.1.89-90** ([changelog](https://code.claude.com/docs/en/changelog)) |
| 2 | **Resume cache regression** — `deferred_tools_delta` mismatch invalidates cache on resume | [#34629](https://github.com/anthropics/claude-code/issues/34629) | v2.1.69+ | **20x cost increase** (measured). **Fixed in v2.1.90** ([changelog](https://code.claude.com/docs/en/changelog)) |
| 3 | **Compaction infinite loop** — 211 consecutive compactions with zero progress | [#24179](https://github.com/anthropics/claude-code/issues/24179) | v2.1.x | Entire budget consumed. **Thrash loop fixed in v2.1.89** |
| 4 | **OAuth retry storm** — token expiration triggers retry cascade | [#10784](https://github.com/anthropics/claude-code/issues/10784) | — | 3.75M tokens wasted |
| 5 | **MCP tool description overhead** — loaded on every message | [#3406](https://github.com/anthropics/claude-code/issues/3406) | All | 10-20K tokens/message |
| 6 | **Memory file double-load** — in git worktrees | [#24283](https://github.com/anthropics/claude-code/issues/24283) | — | Premature compaction |
| 7 | **Thinking signature replay** — opaque base64 blocks resent on resume | [#42260](https://github.com/anthropics/claude-code/issues/42260) | — | 500K+ tokens per resume |
| 8 | **Client-side false rate limiter** — synthetic error blocks requests without API call | [#40584](https://github.com/anthropics/claude-code/issues/40584) | All | Instant "Rate limit reached" with `model: "<synthetic>"`, `input_tokens: 0`. **Unfixed** |
| 9 | **Silent microcompact → context degradation** — GrowthBook-controlled compaction strips tool results | [#42542](https://github.com/anthropics/claude-code/issues/42542) | v2.1.89+ | Context quality loss (cache stays 99%+ in main session). **Unfixed, server-controlled** |
| 10 | **Tool result budget enforcement** — `applyToolResultBudget()` caps tool results at 200K aggregate | GrowthBook flags | All | Tool results truncated to 1-41 chars after threshold. **Unfixed** (v2.1.91 MCP override only) |
| 11 | **JSONL log duplication** — extended thinking generates 2-5x PRELIM entries per API call | [#41346](https://github.com/anthropics/claude-code/issues/41346) | All | 2.87x token inflation in local logs. Server impact unknown. **Unfixed** |

### Server-Side Issues

| # | Root Cause | Evidence | Impact |
|---|-----------|----------|--------|
| 1 | **Silent quota reduction** — weekly/5h limits reduced without notice | [#9094](https://github.com/anthropics/claude-code/issues/9094), [#28848](https://github.com/anthropics/claude-code/issues/28848) | 5-8x reduction in usable hours |
| 2 | **Accounting mismatch** — meter shows 16-84% but "limit reached" | [#19673](https://github.com/anthropics/claude-code/issues/19673), [#29579](https://github.com/anthropics/claude-code/issues/29579) | Unpredictable cutoffs |
| 3 | **Org-level quota sharing** — accounts under same org share pool (`organizationUuid` keying) | Source code analysis | Unexpected cross-account drain |
| 4 | **Opus 4.6 auto-upgrade** — higher token consumption, no opt-out | [#23706](https://github.com/anthropics/claude-code/issues/23706) | Higher base cost per turn |
| 5 | **Server-side accounting change** — old Docker versions drain fast without client update | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Proves server-side cause independent of client bugs |
| 6 | **1M context billing regression** — Max plan 1M requests classified as "extra usage" | [#42616](https://github.com/anthropics/claude-code/issues/42616), [#42569](https://github.com/anthropics/claude-code/issues/42569) | Spurious 429 at 23K tokens, incorrect billing |

---

## Scale Indicators

Top issues by community engagement:

| Issue | Comments | Thumbs Up | Date | Title |
|-------|----------|-----------|------|-------|
| [#16157](https://github.com/anthropics/claude-code/issues/16157) | **1,422** | **647** | 2026-01-03 | Instantly hitting usage limits |
| [#38335](https://github.com/anthropics/claude-code/issues/38335) | **313** | **258** | 2026-03-24 | Session limits exhausted abnormally fast |
| [#3572](https://github.com/anthropics/claude-code/issues/3572) | **274** | — | 2025-07-15 | 529 Status Codes (server overload) |
| [#29579](https://github.com/anthropics/claude-code/issues/29579) | **139** | 79 | 2026-02-28 | Rate limit at 16% usage |
| [#9094](https://github.com/anthropics/claude-code/issues/9094) | **121** | 60 | 2025-10-07 | Silent quota change |
| [#9424](https://github.com/anthropics/claude-code/issues/9424) | **109** | **155** | 2025-10-12 | Subscriptions unusable |
| [#19673](https://github.com/anthropics/claude-code/issues/19673) | **99** | 74 | 2026-01-21 | Limit at 84% usage |
| [#16856](https://github.com/anthropics/claude-code/issues/16856) | **63** | 73 | 2026-01-08 | 4x+ faster consumption |

---

## Escalation Pattern

```
2025-02  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Sporadic (Phase 1)
2025-05  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Claude 4 launch (Phase 2)
2025-07  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Structural defects (Phase 3)
2025-09  █████████████░░░░░░░░░░░░░░░░░░░░░░░░  Silent quota cut (Phase 4)
2025-11  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  OAuth/compaction (Phase 5)
2026-01  ██████████████████████████████░░░░░░░░  MEGA THREAD #16157 (Phase 6)
2026-02  ████████████████████░░░░░░░░░░░░░░░░░  Opus 4.6 + legal (Phase 7)
2026-03  ████████████████████████████░░░░░░░░░  Cache regression confirmed (Phase 8)
2026-04  ████████████████████████████████████░░  Global simultaneous (Phase 9, ongoing)
```

Each new model release or version update has been a trigger for the next escalation cycle.

---

*Collected 2026-04-02, updated 2026-04-06 (rate limit header analysis, community cross-references, v2.1.89 separation framework). See [README.md](README.md) for root cause analysis and [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) for quota architecture analysis.*
