# Claude Code Rate Limit Crisis — Full Timeline

> 14-month chronicle of rate limit / token consumption issues in Claude Code.
> Data collected from GitHub Issues via API on 2026-04-02.

---

## Summary

| Metric | Value |
|--------|-------|
| **Duration** | 14 months (2025-02 ~ 2026-04-02, ongoing) |
| **Major escalation cycles** | 4 |
| **Largest issue** | #16157 — 1,422 comments, 647 thumbs_up |
| **Root causes identified** | 7 (client + server, compound) |
| **Anthropic official response** | Zero (2+ months silence across all rate-limit issues) |

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
| 2026-03-24 | [**#38335**](https://github.com/anthropics/claude-code/issues/38335) | **Session limits exhausted abnormally fast since March 23** | **313** | **277** | **Third mega-thread** |
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

---

## Identified Root Causes

These are compound — multiple bugs interact to produce the observed behavior.

### Client-Side Bugs

| # | Root Cause | Issue | Version | Impact |
|---|-----------|-------|---------|--------|
| 1 | **Cache sentinel replacement** — standalone binary's Bun fork breaks cache prefix | [#40524](https://github.com/anthropics/claude-code/issues/40524) | All standalone | Full cache rebuild every turn |
| 2 | **Resume cache regression** — `deferred_tools_delta` mismatch invalidates cache on resume | [#34629](https://github.com/anthropics/claude-code/issues/34629) | v2.1.69+ | **20x cost increase** (measured) |
| 3 | **Compaction infinite loop** — 211 consecutive compactions with zero progress | [#24179](https://github.com/anthropics/claude-code/issues/24179) | v2.1.x | Entire budget consumed |
| 4 | **OAuth retry storm** — token expiration triggers retry cascade | [#10784](https://github.com/anthropics/claude-code/issues/10784) | — | 3.75M tokens wasted |
| 5 | **MCP tool description overhead** — loaded on every message | [#3406](https://github.com/anthropics/claude-code/issues/3406) | All | 10-20K tokens/message |
| 6 | **Memory file double-load** — in git worktrees | [#24283](https://github.com/anthropics/claude-code/issues/24283) | — | Premature compaction |
| 7 | **Thinking signature replay** — opaque base64 blocks resent on resume | [#42260](https://github.com/anthropics/claude-code/issues/42260) | — | 500K+ tokens per resume |

### Server-Side Issues

| # | Root Cause | Evidence | Impact |
|---|-----------|----------|--------|
| 1 | **Silent quota reduction** — weekly/5h limits reduced without notice | [#9094](https://github.com/anthropics/claude-code/issues/9094), [#28848](https://github.com/anthropics/claude-code/issues/28848) | 5-8x reduction in usable hours |
| 2 | **Accounting mismatch** — meter shows 16-84% but "limit reached" | [#19673](https://github.com/anthropics/claude-code/issues/19673), [#29579](https://github.com/anthropics/claude-code/issues/29579) | Unpredictable cutoffs |
| 3 | **Org-level quota sharing** — accounts with same billing share pool | [#41881](https://github.com/anthropics/claude-code/issues/41881) | Unexpected cross-account drain |
| 4 | **Opus 4.6 auto-upgrade** — higher token consumption, no opt-out | [#23706](https://github.com/anthropics/claude-code/issues/23706) | Higher base cost per turn |

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

*Collected 2026-04-02 via GitHub API. See [README.md](README.md) for root cause analysis and workarounds.*
