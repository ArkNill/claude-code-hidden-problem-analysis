# Claude Code Cache Bug Analysis

Measured analysis of cache bugs in Claude Code that caused **10-20x token inflation** on paid plans (Max 5/20). Includes controlled benchmarks comparing npm vs standalone binary installations.

> **Last updated:** April 3, 2026

---

## Current Status: v2.1.90 — Cache Fixed, But Not Fully Resolved

**v2.1.90 fixed the client-side cache regression** (Bug 1 + Bug 2), restoring **86%+ overall cache read ratio** and **95-99% in stable sessions**. However, **multiple users on v2.1.90 still report rapid drain**, pointing to additional bugs beyond the cache layer.

| Version | Installation | Cache Read (stable) | Sub-agent Cold Start | Verdict |
|---------|-------------|--------------------|--------------------|---------|
| **v2.1.90** | **npm (Node.js)** | **95-99.8%** | **79-87%** | **Best available** |
| **v2.1.90** | **Standalone (ELF)** | **95-99.7%** | **47-67%** (recovers to 94-99%) | **Good** |
| v2.1.89 | Standalone (ELF) | 90-99% | **4-17%** (never recovers) | **Avoid** |
| v2.1.68 | npm | Normal | Normal | Safe but outdated |

**Update to v2.1.90 if you haven't already** — it fixes the biggest drain source. But be aware that at least **three additional bugs** remain unfixed (see [Root Cause](#root-cause) section):

- **Bug 3:** Client-side false rate limiter generates fake "Rate limit reached" errors without calling the API ([#40584](https://github.com/anthropics/claude-code/issues/40584))
- **Bug 4:** Silent microcompact invalidates prompt cache by stripping tool results mid-session ([#42542](https://github.com/anthropics/claude-code/issues/42542))
- **Server-side:** 1M context requests incorrectly classified as "extra usage" on Max plans ([#42616](https://github.com/anthropics/claude-code/issues/42616), [#42569](https://github.com/anthropics/claude-code/issues/42569))

### What to Do Right Now

1. **Disable auto-update** — pin v2.1.90 until Anthropic confirms a full fix
2. **Update to v2.1.90** if you haven't already
3. **Avoid `--resume`** — still causes full context replay regardless of version
4. **Monitor cache** with a transparent proxy if you want to verify

```jsonc
// ~/.claude/settings.json — disable auto-update
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

---

## npm vs Standalone Binary — Which Should I Use?

Claude Code ships in two forms. The choice matters for cache efficiency:

### Standalone Binary (ELF)

- Installed via `curl -fsSL https://claude.ai/install.sh | bash`
- Ships as a **single ELF 64-bit executable** (~228MB) with embedded Bun runtime
- Contains the Sentinel replacement mechanism (`cch=00000`) that can corrupt cache prefixes
- **v2.1.90 status:** Sentinel bug **partially mitigated** — cold starts still lag but cache recovers quickly

### npm Package (Node.js)

- Installed via `npm install -g @anthropic-ai/claude-code`
- Ships as a **bundled JavaScript file** (`cli.js`, ~13MB) executed by Node.js
- **Does not contain** the Sentinel replacement logic — immune to Bug 1
- Dependencies are bundled inline (zero external npm packages at runtime, no supply chain risk)

### Head-to-Head Benchmark (v2.1.90, same machine, same proxy)

| Metric | npm | Standalone | Winner |
|--------|-----|-----------|--------|
| Overall cache read % | 86.4% | 86.2% | Tie |
| Stable session | 95-99.8% | 95-99.7% | Tie |
| Sub-agent cold start | 79-87% | 47-67% | npm |
| Sub-agent warmed (5+ req) | 87-94% | 94-99% | Tie |
| Usage for full test suite | 7% of Max 20 | 5% of Max 20 | Tie |

**Bottom line:** npm is marginally better for sub-agent-heavy workflows. For everything else, they're equivalent on v2.1.90. See **[BENCHMARK.md](BENCHMARK.md)** for raw data and methodology.

### Coexistence Setup

Both can coexist on the same machine at different paths:

```bash
# npm install (does NOT affect standalone binary)
npm install -g @anthropic-ai/claude-code

# Check what you have
file $(which claude)
# ELF 64-bit = standalone binary
# symbolic link to .../cli.js = npm

# If both are installed, use aliases to pick:
alias claude-npm="ANTHROPIC_BASE_URL=http://localhost:8080 /path/to/npm/claude"
alias claude-bin="ANTHROPIC_BASE_URL=http://localhost:8080 /path/to/standalone/claude"
```

---

## Root Cause

Four client-side bugs (two cache, one rate limiter, one compaction) plus server-side accounting issues. The first two were identified through community reverse engineering ([Reddit analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)); the latter two discovered April 2-3, 2026:

### Bug 1 — Sentinel Replacement (standalone binary only)

**GitHub Issue:** [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524)

The standalone binary's embedded Bun fork contains a `cch=00000` sentinel replacement mechanism. Under certain conditions, the sentinel in `messages` gets incorrectly substituted — breaking the cache prefix and forcing a full rebuild.

- **v2.1.89:** Catastrophic — cache read drops to 4-17%, never recovers
- **v2.1.90:** Partially mitigated — cold start still affected (47-67%), but recovers to 94-99% after warming
- **npm:** Not affected — the JavaScript bundle does not contain this logic

**Official fix in v2.1.89-90 ([changelog](https://code.claude.com/docs/en/changelog)):**
- v2.1.89: *"Fixed prompt cache misses in long sessions caused by tool schema bytes changing mid-session"*
- v2.1.90: *"Improved performance: eliminated per-turn JSON.stringify of MCP tool schemas on cache-key lookup"*

### Bug 2 — Resume Cache Breakage (v2.1.69+)

**GitHub Issue:** [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629)

`deferred_tools_delta` (introduced in v2.1.69) causes the first message's structure on `--resume` to not match the server's cached version — resulting in a complete cache miss. On a 500K token conversation, a single resume costs ~$0.15 in quota.

**Official fix in v2.1.90 ([changelog](https://code.claude.com/docs/en/changelog)):**
> *"Fixed --resume causing a full prompt-cache miss on the first request for users with deferred tools, MCP servers, or custom agents (regression since v2.1.69)"*

**Note:** While the changelog states this is fixed, community reports (e.g., `claude -p --resume` in headless harness mode) suggest edge cases may remain. We recommend continuing to avoid `--resume` until fully verified.

### Bug 3 — Client-Side False Rate Limiter (all versions)

**GitHub Issue:** [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584)

The local rate limiter generates **synthetic "Rate limit reached" errors** without ever calling the Anthropic API. These errors are identifiable in session logs by:

```json
{
  "model": "<synthetic>",
  "usage": { "input_tokens": 0, "output_tokens": 0 }
}
```

Triggered by **large transcripts (~74MB+)** and **concurrent sub-agent spawns**. The rate limiter multiplies `context_size × concurrent_requests`, so multi-agent workflows get blocked even when each individual request is small.

- **Discovery:** [@rwp65](https://github.com/rwp65) in [#40584](https://github.com/anthropics/claude-code/issues/40584) (March 29, 2026)
- **Cross-referenced by:** [@marlvinvu](https://github.com/marlvinvu) across [#40438](https://github.com/anthropics/claude-code/issues/40438), [#39938](https://github.com/anthropics/claude-code/issues/39938), [#38239](https://github.com/anthropics/claude-code/issues/38239)
- **Status:** **Unfixed** — present in all versions including v2.1.90
- **Impact:** Users see "Rate limit reached" immediately, even after hours of inactivity when the budget should have fully reset. No API call is made, so the error is entirely client-generated.

### Bug 4 — Silent Microcompact → Cache Invalidation (v2.1.89+)

**GitHub Issue:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)

Three compaction mechanisms in `src/services/compact/` run **silently on every API call**, stripping old tool results without user notification. This invalidates prompt cache prefixes, causing subsequent API calls to be billed at full price.

| Mechanism | Source | Trigger | Control |
|-----------|--------|---------|---------|
| **Time-based microcompact** | `microCompact.ts:422` | Gap since last assistant message exceeds threshold | GrowthBook: `getTimeBasedMCConfig()` |
| **Cached microcompact** | `microCompact.ts:305` | Count-based trigger, uses `cache_edits` API to delete old tool results | GrowthBook: `getCachedMCConfig()` |
| **Session memory compact** | `sessionMemoryCompact.ts:57` | Runs before autocompact | GrowthBook flag |

**Key findings:**
- All three bypass `DISABLE_AUTO_COMPACT` and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`
- Controlled by **server-side GrowthBook A/B testing flags** — Anthropic can change behavior without a client update
- Tool results silently replaced with `[Old tool result content cleared]` — no compaction notification shown
- This explains why v2.1.90 fixes cache for some users but not others (depends on GrowthBook flag assignment)
- Also explains why old Docker versions that were never updated started draining recently ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — server-side flags changed

**Cache invalidation chain:**
1. Microcompact triggers silently (GrowthBook flag)
2. Old tool results stripped → conversation prefix changes
3. Prompt cache prefix no longer matches → cache miss
4. Next API call: 0% cache read → full-price billing on entire context
5. Usage burns 5-10x faster than expected

- **Update (April 3):** GrowthBook flag survey across 4 machines / 4 accounts shows **all gates disabled** — yet context is still being stripped. See [MICROCOMPACT.md](MICROCOMPACT.md) for full analysis, collected flag data, and open questions (runtime vs disk cache divergence, server-side clearing hypothesis).
- **Discovery:** [@Sn3th](https://github.com/Sn3th) in [#42542](https://github.com/anthropics/claude-code/issues/42542) (April 2, 2026)
- **Status:** **Unfixed** — present in v2.1.89+ and controlled server-side. All known GrowthBook gates show disabled but stripping continues.

---

## Measured Data

### Methodology

Transparent local monitoring proxy using [`ANTHROPIC_BASE_URL`](https://docs.anthropic.com/en/docs/claude-code/settings#environment-variables) (official environment variable). The proxy logs `cache_creation_input_tokens` and `cache_read_input_tokens` from each API response without modifying requests or responses (source-audited).

### v2.1.89 Standalone — Before Fix (Broken)

| Session | Turns | Cache Read Ratio | Status |
|---------|-------|-----------------|--------|
| Session A | 168 | **4.3%** | poor — 20x cost inflation |
| Session B | 89 | **22.6%** | poor |
| Session C | 233 | **34.6%** | poor |

### v2.1.90 — After Fix (Both Installations)

| Metric | npm (Node.js) | Standalone (ELF) |
|--------|--------------|-----------------|
| Scenarios completed | 7 (incl. 79-report parallel agent read) | 4 (forge, browsegrab, feedkit 5-turn, 3-project parallel) |
| Usage consumed | 28% → 35% (**7%**) | 35% → 40% (**5%**) |
| Overall cache read | **86.4%** | **86.2%** |
| Stable session read | **95-99.8%** | **95-99.7%** |

Full per-request data and warming curves: **[BENCHMARK.md](BENCHMARK.md)**

### Version Comparison Summary

| Metric | v2.1.89 Standalone | v2.1.90 Standalone | v2.1.90 npm |
|--------|-------------------|-------------------|-------------|
| Sub-agent cold start | **4-17%** (never recovers) | **47-67%** (recovers to 94-99%) | **79-87%** |
| Stable session | 90-99% | **95-99.7%** | **95-99.8%** |
| Usage for test suite | 100% in ~70 min | **5%** | **7%** |
| Verdict | **Avoid** | **Good** | **Optimal** |

---

## Usage Precautions

### Behaviors to Avoid

| Behavior | Why | Measured Impact |
|----------|-----|-----------------|
| `--resume` | Replays entire conversation history as billable input including opaque thinking block signatures | 500K+ tokens per resume ([#42260](https://github.com/anthropics/claude-code/issues/42260)) |
| `/dream`, `/insights` | Background API calls consume tokens without visible output | Silent drain ([#40438](https://github.com/anthropics/claude-code/issues/40438)) |
| v2.1.89 or earlier standalone | Sentinel bug causes sustained 4-17% cache read | 3-4x token waste, never recovers |
| Enabling auto-update | Future versions may reintroduce regressions | Pin v2.1.90 until official fix confirmed |

### Behaviors to Use with Caution

| Behavior | Why | Recommendation |
|----------|-----|----------------|
| Parallel sub-agents (single terminal) | Each agent starts with fresh context, but warms up and shares billing context | **Safe** — agents warm up within 1-2 requests |
| Multiple terminals simultaneously | Each terminal is a fully independent session — no cache sharing, parallel quota drain | **Limit to one active terminal** |
| Large CLAUDE.md / context files | Sent on every turn — with broken cache, billed at full price each time | Keep lean; less critical on v2.1.90 with working cache |
| Session start / compaction | `cache_creation` spikes are structural and unavoidable | Normal — budget for it |

### Server-Side Factors (Unresolved)

Even with cache working perfectly (91-99%), multiple users report faster quota drain compared to 2-3 weeks ago. At least three server-side issues contribute:

**1. Server-side accounting change:** Old Docker versions (v2.1.74, v2.1.86 — never updated) started draining fast recently, proving the issue isn't purely client-side ([#37394](https://github.com/anthropics/claude-code/issues/37394), reported by [@pablofuenzalidadf](https://github.com/pablofuenzalidadf)).

**2. 1M context billing regression:** Max plans include 1M context free (announced March 13, confirmed March 20), but a late-March regression causes the server to incorrectly classify these requests as "extra usage." Debug logs show a 429 error at only ~23K tokens with `"Extra usage is required for long context requests"` on a Max plan with 1M context enabled ([#42616](https://github.com/anthropics/claude-code/issues/42616), request ID: `req_011CZf8TJf84hAUziB6LuRoc`). Related display bug: [#42569](https://github.com/anthropics/claude-code/issues/42569).

**3. Org-level quota sharing:** Accounts on the same billing method share rate limit pools ([#41881](https://github.com/anthropics/claude-code/issues/41881)). Source code confirms `passesEligibilityCache` and `overageCreditGrantCache` are keyed by `organizationUuid`, not `accountUuid`.

---

## Quick Setup Guide

### For New Users

```bash
# 1. Install via npm (recommended — no Sentinel bug)
npm install -g @anthropic-ai/claude-code

# 2. Disable auto-update
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
EOF

# 3. Verify
claude --version   # should show 2.1.90
file $(which claude)   # should show symbolic link to cli.js
```

### For Existing Standalone Users

```bash
# 1. Update to v2.1.90
claude update

# 2. Disable auto-update (add to existing settings.json)
# Add "DISABLE_AUTOUPDATER": "1" to the "env" section

# 3. Verify
claude --version   # should show 2.1.90
```

### Optional: Monitor Cache Efficiency

Set up a transparent proxy using `ANTHROPIC_BASE_URL` to log cache metrics:

```bash
# Route through local proxy for monitoring
ANTHROPIC_BASE_URL=http://localhost:8080 claude

# Parse cache_creation_input_tokens / cache_read_input_tokens from responses
# Healthy: read ratio > 80%  |  Affected: read ratio < 40%
```

---

## How to Check If You're Affected

The session JSONL files in `~/.claude/projects/` contain usage data for each turn:

- **Healthy session:** `cache_read` >> `cache_creation` (read ratio > 80%)
- **Affected session:** `cache_creation` >> `cache_read` (read ratio < 40%)

If most sessions show low read ratios, you're likely on an affected version. Update to v2.1.90.

---

## Related Issues

### Root Cause Bugs
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — Conversation history invalidated (Bug 1: sentinel) — **fixed in v2.1.89-90**
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume cache regression (Bug 2: deferred_tools_delta) — **fixed in v2.1.90**
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= billing hash substitution
- [#40584](https://github.com/anthropics/claude-code/issues/40584) — **Client-side false rate limiter** (Bug 3: synthetic model, 0 tokens) — **unfixed**
- [#42542](https://github.com/anthropics/claude-code/issues/42542) — **Silent microcompact → cache invalidation** (Bug 4: GrowthBook-controlled) — **unfixed**

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

### Rate Limit Reports (major threads)
- [#16157](https://github.com/anthropics/claude-code/issues/16157) — Instantly hitting usage limits (1400+ comments)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — Session limits exhausted abnormally fast (300+ comments)
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — My original report (Max 20, 100% in ~70 min)

### Anthropic Official Response

Anthropic has **not responded on any GitHub issue** (2+ months of silence across 82 issues). Communication has been limited to personal social media:

| Who | Platform | What | Link |
|-----|----------|------|------|
| **Lydia Hallie** (Product) | X | *"We shipped some fixes on the Claude Code side that should help"* | [Post](https://x.com/lydiahallie/status/2039107775314428189) |
| **Thariq Shihipar** (Technical Staff) | X | *"We've reset rate limits... bug with prompt caching... hotfixed in 2.1.62"* (earlier incident) | [Post](https://x.com/trq212/status/2027232172810416493) |
| **@anthropicai** | Threads | General Claude Code feature updates (no rate limit specifics) | [Post](https://www.threads.net/@anthropicai/post/DHeO-oyPjMb/) |
| **Official Changelog** | Docs | v2.1.89-90 cache fix entries | [Changelog](https://code.claude.com/docs/en/changelog) |

### Community Engagement

As of April 3, 2026: **180+ comments on 91 unique issues** (including v2.1.90 benchmark update + Bug 3/4 cross-references). Anthropic official response on GitHub: **zero**.

<details>
<summary><strong>All 91 issues with root cause analysis + v2.1.90 update posted</strong> (click to expand)</summary>

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

## Community References

### Analysis & Tools
- [Reddit: Reverse engineering analysis of Claude Code cache bugs](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — Community cache patch + test toolkit
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy-based Claude Code traffic analysis
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Transparent proxy for Claude Code

### Token Optimization Tools (complementary, not bug fixes)
- [rtk](https://github.com/rtk-ai/rtk) — Tool output compression (trims CLI/test results post-execution, reduces input token volume)
- [tokenlean](https://github.com/edimuj/tokenlean) — 54 CLI tools for agents (extracts symbols/snippets instead of full file reads, reduces base token count)

---

## Files in This Repo

| File | Description |
|------|-------------|
| [README.md](README.md) | This file — overview, current status, and recommendations |
| [BENCHMARK.md](BENCHMARK.md) | Controlled npm vs standalone benchmark with raw per-request data |
| [TIMELINE.md](TIMELINE.md) | 14-month chronicle of rate limit issues (Phase 1-9, 50+ issues) |

## Environment

- **Plan:** Max 20 ($200/mo)
- **OS:** Linux (Ubuntu), HP ZBook Ultra G1a
- **Versions tested:** v2.1.90 (npm + standalone benchmark), v2.1.89 (affected), v2.1.81 (patched workaround), v2.1.68 (pre-bug baseline)
- **Monitoring:** cc-relay transparent proxy (source-audited, zero request modification)
- **Date:** April 3, 2026

---

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
