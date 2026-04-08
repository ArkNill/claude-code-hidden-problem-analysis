> **🇰🇷 [한국어 버전](ko/10_ISSUES.md)**

# Related Issues, Community Tools & Contributors

> As of April 6, 2026: **180+ comments on 91+ unique issues**. Anthropic official response on GitHub: **zero**.

---

## Issue Index by Category

### Root Cause Bugs
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — Conversation history invalidated (Bug 1: sentinel) — **improved in v2.1.89-91**
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume cache regression (Bug 2: deferred_tools_delta) — **improved in v2.1.90-91**
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= billing hash substitution
- [#40584](https://github.com/anthropics/claude-code/issues/40584) — **Client-side false rate limiter** (Bug 3: 151 `<synthetic>` entries across 65 session files (all-time; 24 entries in the April 1-6 analysis window — see [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md))) — **unfixed**
- [#42542](https://github.com/anthropics/claude-code/issues/42542) — **Silent microcompact → context degradation** (Bug 4: 3,782 events, 15,998 items cleared — see [13_PROXY-DATA.md](13_PROXY-DATA.md)) — **unfixed**
- Bug 5: **Tool result budget enforcement** (200K aggregate cap, 72,839 events, 100% truncation — see [13_PROXY-DATA.md](13_PROXY-DATA.md)) — **unfixed** (v2.1.91 MCP override only)
- [#41346](https://github.com/anthropics/claude-code/issues/41346) — **JSONL log duplication** (Bug 8: 2.37x avg inflation across 532 files, max 4.42x — see [13_PROXY-DATA.md](13_PROXY-DATA.md)) — **unfixed**

### Server-Side Billing Bugs
- [#42616](https://github.com/anthropics/claude-code/issues/42616) — Spurious 429 "Extra usage required" at 23K tokens on Max plan with 1M context
- [#42569](https://github.com/anthropics/claude-code/issues/42569) — 1M context incorrectly shown as extra billable usage on Max plan
- [#37394](https://github.com/anthropics/claude-code/issues/37394) — Old Docker versions (never updated) drain fast — server-side accounting change

### Token Inflation Mechanisms
- [#41663](https://github.com/anthropics/claude-code/issues/41663) — Prompt cache causes excessive token consumption
- [#41607](https://github.com/anthropics/claude-code/issues/41607) — Duplicate compaction subagents (5x identical work)
- [#41767](https://github.com/anthropics/claude-code/issues/41767) — Auto-compact loops in v2.1.89
- [#42260](https://github.com/anthropics/claude-code/issues/42260) — Resume replays thinking signatures as input tokens
- [#42256](https://github.com/anthropics/claude-code/issues/42256) — Read tool re-sends oversized images every message
- [#42590](https://github.com/anthropics/claude-code/issues/42590) — Context compaction too aggressive on 1M context window

### Session Loading / JSONL Pipeline
- [#43044](https://github.com/anthropics/claude-code/issues/43044) — **`--resume` loads 0% context on v2.1.91** — three regressions in session loading pipeline (by [@kolkov](https://github.com/kolkov))

### Rate Limit Reports (major threads)
- [#16157](https://github.com/anthropics/claude-code/issues/16157) — Instantly hitting usage limits (1400+ comments)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — Session limits exhausted abnormally fast (300+ comments)
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — My original report (Max 20, 100% in ~70 min)

---

## Anthropic Official Response

**GitHub:** Zero responses across 91+ rate-limit issues (2+ months of silence).

**April 2, 2026 — Lydia Hallie (Anthropic, Product) posted on X:**

> *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."*

**Our measured data raises questions about this assessment:**
- **Bug 5 (200K cap):** Tool results silently truncated to 1-41 chars after the aggregate 200K threshold. Users paying for 1M context effectively have a 200K tool result budget for built-in tools — the rest is silently discarded.
- **Bug 3 (synthetic RL):** 151 `<synthetic>` entries across 65 session files (all-time; 24 entries in the April 1-6 analysis window — see [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md)). The client blocks API calls without server involvement — users see "Rate limit reached" with zero actual API consumption.
- **Bug 8 (PRELIM duplication):** Extended thinking sessions log 2-3x more token entries than actual API calls. Whether the server-side rate limiter counts these remains an open question.

| Who | Platform | What | Link |
|-----|----------|------|------|
| **Lydia Hallie** | X | Full statement on rate limits (thread start) | [Post 1](https://x.com/lydiahallie/status/2039800715607187906) |
| **Lydia Hallie** | X | Follow-up with usage tips (thread end) | [Post 2](https://x.com/lydiahallie/status/2039800718371307603) |
| **Lydia Hallie** | X | *"We shipped some fixes that should help"* (earlier) | [Post](https://x.com/lydiahallie/status/2039107775314428189) |
| **Thariq Shihipar** | X | *"Bug with prompt caching... hotfixed"* (earlier incident) | [Post](https://x.com/trq212/status/2027232172810416493) |
| **Official Changelog** | GitHub | v2.1.89-91 fix entries | [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

---

## Community Analysis & Tools

### Analysis Tools
- [Reddit: Reverse engineering analysis of Claude Code cache bugs](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — Community cache patch + test toolkit
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy-based Claude Code traffic analysis
- [ccdiag](https://github.com/kolkov/ccdiag) — Go-based JSONL session log recovery and DAG analysis tool (by [@kolkov](https://github.com/kolkov))
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Transparent proxy for Claude Code
- [CUStats](https://custats.info) — Real-time usage tracking and visualization
- [context-stats](https://github.com/luongnv89/cc-context-stats) — Per-interaction cache metrics export and analysis (by [@luongnv89](https://github.com/luongnv89))
- [BudMon](https://github.com/weilhalt/budmon) — Desktop dashboard for rate-limit header monitoring
- [claude-usage-dashboard](https://github.com/fgrosswig/claude-usage-dashboard) — Standalone Node.js JSONL dashboard with forensic analysis, multi-host aggregation, and implicit budget estimation (by [@fgrosswig](https://github.com/fgrosswig))
- [Resume cache fix patch](https://gist.github.com/simpolism/302621e661f462f3e78684d96bf307ba) — Fixes two remaining `--resume` cache misses on v2.1.91 (by [@simpolism](https://github.com/simpolism))

### Token Optimization Tools
- [rtk](https://github.com/rtk-ai/rtk) — Tool output compression
- [tokenlean](https://github.com/edimuj/tokenlean) — 54 CLI tools for agents (extracts symbols/snippets instead of full file reads)

### Related Session Loading Analysis

[@kolkov](https://github.com/kolkov) independently analyzed v2.1.91 `--resume` regressions at the **JSONL DAG / session loading layer** — a different level from the API-level cache bugs.

**Issue:** [anthropics/claude-code#43044](https://github.com/anthropics/claude-code/issues/43044)

| # | Regression | Effect |
|---|-----------|--------|
| 1 | `walkChainBeforeParse` removed | Fork pruning gone for JSONL files >5MB |
| 2 | `ExY` timestamp fallback bridges fork boundaries | Forks merged across timestamp boundaries |
| 3 | Missing `leafUuids` check | Last-session detection picks wrong log file |

**Tool:** [ccdiag](https://github.com/kolkov/ccdiag). This covers the **pre-API pipeline** (how JSONL logs are reconstructed), distinct from the API-level analysis in this repo.

---

<details>
<summary><strong>All 91 issues with root cause analysis + v2.1.91 update posted</strong> (click to expand)</summary>

| # | Issue | Title |
|---|-------|-------|
| 1 | [#6457](https://github.com/anthropics/claude-code/issues/6457) | 5-hour limit reached in less than 1h30 |
| 2 | [#16157](https://github.com/anthropics/claude-code/issues/16157) | Instantly hitting usage limits with Max subscription |
| 3 | [#16856](https://github.com/anthropics/claude-code/issues/16856) | Excessive token usage in Claude Code 2.1.1 — 4x+ faster rate consumption |
| 4 | [#17016](https://github.com/anthropics/claude-code/issues/17016) | Claude hitting usage limits in no time |
| 5 | [#22435](https://github.com/anthropics/claude-code/issues/22435) | Inconsistent and Undisclosed Quota Accounting Changes |
| 6 | [#22876](https://github.com/anthropics/claude-code/issues/22876) | Rate limit 429 errors despite dashboard showing available quota |
| 7 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | Opus 4.6 token consumption significantly higher than 4.5 |
| 8 | [#34410](https://github.com/anthropics/claude-code/issues/34410) | Max x20 plan — 5-hour quota consumed in ~10 prompts |
| 9 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Claude Code Usage for Max Plan hitting limits extremely fast |
| 10 | [#38239](https://github.com/anthropics/claude-code/issues/38239) | Extremely rapid token consumption |
| 11 | [#38335](https://github.com/anthropics/claude-code/issues/38335) | Session limits exhausted abnormally fast since March 23 |
| 12 | [#38345](https://github.com/anthropics/claude-code/issues/38345) | Abnormal rate limit consumption — 1-2 prompts exhausting 5-hour window |
| 13 | [#38350](https://github.com/anthropics/claude-code/issues/38350) | Abnormal / inflated rate limit / session usage |
| 14 | [#38896](https://github.com/anthropics/claude-code/issues/38896) | Rate limit reached despite 0% usage |
| 15 | [#39938](https://github.com/anthropics/claude-code/issues/39938) | 93% rate limit consumed in ~3 minutes on fresh session |
| 16 | [#39966](https://github.com/anthropics/claude-code/issues/39966) | Abnormal token consumption rate on Opus 4.5 — 100% usage in ~1 hour |
| 17 | [#40438](https://github.com/anthropics/claude-code/issues/40438) | Rate Limit Reached without typing anything, /insights causes insane token usage |
| 18 | [#40524](https://github.com/anthropics/claude-code/issues/40524) | Conversation history invalidated on subsequent turns **(Root Cause: Sentinel)** |
| 19 | [#40535](https://github.com/anthropics/claude-code/issues/40535) | ClaudeMax — Session Throttling in Daily Limit |
| 20 | [#40652](https://github.com/anthropics/claude-code/issues/40652) | CLI mutates historical tool results via cch= billing hash substitution **(Root Cause)** |
| 21 | [#40790](https://github.com/anthropics/claude-code/issues/40790) | Excessive token consumption spike since March 23rd |
| 22 | [#40851](https://github.com/anthropics/claude-code/issues/40851) | Opus 4.6 (Max $100) — Quota reaches 93% after minimal prompting |
| 23 | [#40871](https://github.com/anthropics/claude-code/issues/40871) | Make one prompt and jump to 20% on Max x5 |
| 24 | [#40895](https://github.com/anthropics/claude-code/issues/40895) | Opus 4.6 on Max plan: usage seems disproportionately high per prompt |
| 25 | [#40903](https://github.com/anthropics/claude-code/issues/40903) | Max Plan usage limits significantly reduced compared to 2 weeks ago |
| 26 | [#40956](https://github.com/anthropics/claude-code/issues/40956) | Rate limit error on MAX+ plan when invoking slash commands |
| 27 | [#41001](https://github.com/anthropics/claude-code/issues/41001) | Team Premium seat weekly limit stuck at 100% after only 102M tokens |
| 28 | [#41055](https://github.com/anthropics/claude-code/issues/41055) | Weekly usage reset failed + incorrect token burn rate on $200 Max plan |
| 29 | [#41076](https://github.com/anthropics/claude-code/issues/41076) | Max5 plan hitting session limits daily (including off-peak hours) |
| 30 | [#41084](https://github.com/anthropics/claude-code/issues/41084) | Usage Limits Show 0% Daily But 41% Weekly — Phantom Usage |
| 31 | [#41158](https://github.com/anthropics/claude-code/issues/41158) | Excessive token consumption |
| 32 | [#41173](https://github.com/anthropics/claude-code/issues/41173) | Rate limit reached at 16% session usage on Max $200 plan |
| 33 | [#41174](https://github.com/anthropics/claude-code/issues/41174) | Usage limit reached in 10 minutes |
| 34 | [#41212](https://github.com/anthropics/claude-code/issues/41212) | Rate limit reached on ALL surfaces — Max 20x, 18% usage, non-peak |
| 35 | [#41249](https://github.com/anthropics/claude-code/issues/41249) | Excessive token consumption rate — usage depleting faster than expected |
| 36 | [#41346](https://github.com/anthropics/claude-code/issues/41346) | Extended thinking generates duplicate .jsonl log entries (~2-3x token inflation) |
| 37 | [#41377](https://github.com/anthropics/claude-code/issues/41377) | Status line usage % doesn't match website session usage |
| 38 | [#41424](https://github.com/anthropics/claude-code/issues/41424) | Claude Max + Claude Code rate limits exhausting abnormally fast |
| 39 | [#41504](https://github.com/anthropics/claude-code/issues/41504) | Max plan: usage exhausted abnormally fast, twice in one day |
| 40 | [#41506](https://github.com/anthropics/claude-code/issues/41506) | Token usage increased ~3-5x without any configuration change |
| 41 | [#41508](https://github.com/anthropics/claude-code/issues/41508) | 20x Max Plan User Hitting "Extra Usage" Limits After Single Session |
| 42 | [#41521](https://github.com/anthropics/claude-code/issues/41521) | Max Plan 5x Phantom Consumption and Large Token Usage Increased |
| 43 | [#41550](https://github.com/anthropics/claude-code/issues/41550) | Usage extremely high after 5-10 minutes into planning a feature |
| 44 | [#41567](https://github.com/anthropics/claude-code/issues/41567) | Not even doing real work and we're at 40% (Max x5) |
| 45 | [#41583](https://github.com/anthropics/claude-code/issues/41583) | Rate limit errors on Pro Plan at 26% usage |
| 46 | [#41590](https://github.com/anthropics/claude-code/issues/41590) | New account immediately hits rate limit with zero usage |
| 47 | [#41605](https://github.com/anthropics/claude-code/issues/41605) | /fast toggle reports "extra usage credits exhausted" on Pro Max |
| 48 | [#41607](https://github.com/anthropics/claude-code/issues/41607) | Duplicate compaction subagents spawned (5x identical work) |
| 49 | [#41617](https://github.com/anthropics/claude-code/issues/41617) | Excessive token consumption after recent updates (Max plan) |
| 50 | [#41640](https://github.com/anthropics/claude-code/issues/41640) | Excessive API usage consumption and degraded response performance |
| 51 | [#41651](https://github.com/anthropics/claude-code/issues/41651) | Frequent 529 Overloaded errors on Claude Max plan |
| 52 | [#41663](https://github.com/anthropics/claude-code/issues/41663) | Prompt Cache causes excessive token consumption (10-20x inflation) |
| 53 | [#41666](https://github.com/anthropics/claude-code/issues/41666) | v2.1.88 yanked release caused wasted session + significant token loss |
| 54 | [#41674](https://github.com/anthropics/claude-code/issues/41674) | Unexpectedly High API Usage and Rate Limit Hits |
| 55 | [#41728](https://github.com/anthropics/claude-code/issues/41728) | You've hit your limit · resets 5am (Europe/Istanbul) |
| 56 | [#41749](https://github.com/anthropics/claude-code/issues/41749) | Unexpected excessive API usage consumption |
| 57 | [#41750](https://github.com/anthropics/claude-code/issues/41750) | Context management fires on every turn with empty applied_edits |
| 58 | [#41767](https://github.com/anthropics/claude-code/issues/41767) | Auto-Compact loops continuously in v2.1.89 |
| 59 | [#41776](https://github.com/anthropics/claude-code/issues/41776) | API Error / Rate limit reached |
| 60 | [#41779](https://github.com/anthropics/claude-code/issues/41779) | Excessive token consumption compared to previous versions |
| 61 | [#41781](https://github.com/anthropics/claude-code/issues/41781) | You've hit your limit · resets 12am (America/Los_Angeles) |
| 62 | [#41788](https://github.com/anthropics/claude-code/issues/41788) | **Max 20 plan: 100% exhausted in ~70 min (my original report)** |
| 63 | [#41802](https://github.com/anthropics/claude-code/issues/41802) | Session hangs with excessive token consumption and no output |
| 64 | [#41812](https://github.com/anthropics/claude-code/issues/41812) | Excessive token usage due to startup overhead |
| 65 | [#41853](https://github.com/anthropics/claude-code/issues/41853) | Excessive token consumption without corresponding output |
| 66 | [#41854](https://github.com/anthropics/claude-code/issues/41854) | Token usage rate calculation may exceed weekly limits unexpectedly |
| 67 | [#41859](https://github.com/anthropics/claude-code/issues/41859) | Excessive token consumption causing unusable API costs |
| 68 | [#41866](https://github.com/anthropics/claude-code/issues/41866) | Extreme Token Burn with Claude Code CLI |
| 69 | [#41891](https://github.com/anthropics/claude-code/issues/41891) | Rate limit consumed without API calls |
| 70 | [#41928](https://github.com/anthropics/claude-code/issues/41928) | Accelerated restrictions and token usage |
| 71 | [#41930](https://github.com/anthropics/claude-code/issues/41930) | Critical: Widespread abnormal usage drain — multiple root causes |
| 72 | [#41944](https://github.com/anthropics/claude-code/issues/41944) | Usage limits not displaying after responses in CLI mode (2.1.89) |
| 73 | [#42003](https://github.com/anthropics/claude-code/issues/42003) | You've hit your limit · resets 12pm (America/Sao_Paulo) |
| 74 | [#42244](https://github.com/anthropics/claude-code/issues/42244) | 2.1.89 — terminal content disappearing |
| 75 | [#42247](https://github.com/anthropics/claude-code/issues/42247) | Flag to limit amount of turn history sent |
| 76 | [#42249](https://github.com/anthropics/claude-code/issues/42249) | Extreme token consumption — quota depleted in minutes |
| 77 | [#42256](https://github.com/anthropics/claude-code/issues/42256) | Read tool sends oversized images on every subsequent message |
| 78 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | Resume loads disproportionate tokens from opaque thinking signatures |
| 79 | [#42261](https://github.com/anthropics/claude-code/issues/42261) | You've hit your limit · resets 1am (Europe/London) |
| 80 | [#42272](https://github.com/anthropics/claude-code/issues/42272) | Excessive token consumption since 2.1.88 — 66% usage in 2 questions |
| 81 | [#42277](https://github.com/anthropics/claude-code/issues/42277) | New session doesn't reset usage limits correctly |
| 82 | [#42290](https://github.com/anthropics/claude-code/issues/42290) | /export truncates + /resume delivers incomplete context |
| — | [#42338](https://github.com/anthropics/claude-code/issues/42338) | Session resume invalidates entire prompt cache |
| 83 | [#42390](https://github.com/anthropics/claude-code/issues/42390) | Rate limit triggered despite 0% usage in /usage |
| 84 | [#42409](https://github.com/anthropics/claude-code/issues/42409) | Excessive API usage consumption during active session |
| 85 | [#42542](https://github.com/anthropics/claude-code/issues/42542) | **Silent context degradation — 3 microcompact mechanisms (Bug 4: Root Cause)** |
| 86 | [#42569](https://github.com/anthropics/claude-code/issues/42569) | 1M context window incorrectly shown as extra billable usage on Max plan |
| 87 | [#42583](https://github.com/anthropics/claude-code/issues/42583) | You've hit your limit — 1M actual vs 120-160K expected |
| 88 | [#42592](https://github.com/anthropics/claude-code/issues/42592) | Token consumption 100x faster after v2.1.88 — 21 min to 5-hour limit |
| 89 | [#42609](https://github.com/anthropics/claude-code/issues/42609) | Reached limit session in under 5 minutes (resume-triggered) |
| 90 | [#42616](https://github.com/anthropics/claude-code/issues/42616) | Spurious 429 "Extra usage required" at 23K tokens on Max plan with 1M context |
| 91 | [#42590](https://github.com/anthropics/claude-code/issues/42590) | Context compaction too aggressive on 1M context window (Opus 4.6) |

</details>

---

## Contributors & Acknowledgments

This analysis builds on work by many community members who independently investigated and measured these issues:

| Who | Contribution |
|-----|-------------|
| [@Sn3th](https://github.com/Sn3th) | Discovered the three microcompact mechanisms (Bug 4), GrowthBook flag extraction, `applyToolResultBudget()` pipeline (Bug 5), confirmed server-side context mutation across multiple machines |
| [@rwp65](https://github.com/rwp65) | Discovered the client-side false rate limiter (Bug 3) with detailed log evidence |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | Windows GrowthBook flag dumps confirming cross-platform consistency |
| [@dbrunet73](https://github.com/dbrunet73) | Real-world OTel comparison data (v2.1.88 vs v2.1.90) confirming cache improvement |
| [@maiarowsky](https://github.com/maiarowsky) | Confirmed Bug 3 on v2.1.90 with 26 synthetic entries across 13 sessions |
| [@luongnv89](https://github.com/luongnv89) | Cache TTL analysis, built [CUStats](https://custats.info) and [context-stats](https://github.com/luongnv89/cc-context-stats) |
| [@edimuj](https://github.com/edimuj) | Measured grep/file-read token waste (3.5M tokens / 1800+ calls), built [tokenlean](https://github.com/edimuj/tokenlean) |
| [@amicicixp](https://github.com/amicicixp) | Verified v2.1.90 cache improvement with before/after testing |
| [@simpolism](https://github.com/simpolism) | Identified v2.1.90 changelog correlation, built resume cache fix patch (99.7-99.9% hit) |
| [@kolkov](https://github.com/kolkov) | Built [ccdiag](https://github.com/kolkov/ccdiag), identified three v2.1.91 `--resume` regressions ([#43044](https://github.com/anthropics/claude-code/issues/43044)) |
| [@weilhalt](https://github.com/weilhalt) | Built [BudMon](https://github.com/weilhalt/budmon) for real-time rate-limit header monitoring |
| [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) | Reported old Docker versions draining — key server-side evidence ([#37394](https://github.com/anthropics/claude-code/issues/37394)) |
| [@dancinlife](https://github.com/dancinlife) | Discovered `organizationUuid`-based quota pooling and `oauthAccount` cross-contamination bug |
| [@SC7639](https://github.com/SC7639) | Additional regression data confirming mid-March timeline |
| [@fgrosswig](https://github.com/fgrosswig) | 64x budget reduction forensics — dual-machine 18-day JSONL before/after comparison ([#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)) |
| [@Commandershadow9](https://github.com/Commandershadow9) | Confirmed cache fix, documented 34-143x capacity reduction, raised thinking token hypothesis ([#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)) |
| Reddit community | [Reverse engineering analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) of cache sentinel mechanism |

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
