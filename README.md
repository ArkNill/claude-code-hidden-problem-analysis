# Claude Code Rate Limit Crisis — Root Cause Analysis

> **TL;DR:** Claude Code has **7 confirmed bugs across 5 layers** that drain usage faster than expected. The cache bugs (1-2) are fixed in v2.1.91. Five others — silent tool result truncation, false rate limiting, context stripping, log inflation, and server-side changes — remain unfixed as of v2.1.91 (April 3, 2026). All findings below are backed by proxy-measured data from controlled tests.

This repo documents the investigation, from the initial 70-minute drain on a $200/mo Max plan to a systematic 6-layer bug model tested across v2.1.89 through v2.1.91. The analysis is independent and community-driven — Anthropic has not responded on any of the 91+ related GitHub issues.

> **Last updated:** April 3, 2026 — v2.1.91 tested, [CHANGELOG.md ref](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

---

## Background

On April 1, 2026, my Max 20 plan ($200/mo) hit 100% usage in ~70 minutes during normal coding. JSONL analysis showed the affected session averaging **36.1% cache read** (min 21.1%) where it should have been 90%+ — every token billed at full price. An earlier session that day had already degraded to **47.0% average** (233 entries, min 8.3%).

Immediate workaround was downgrading from v2.1.89 to v2.1.68 (npm, pre-regression baseline). Cache immediately recovered to **97.6% average** (119 entries) on the downgraded version — confirming the regression was v2.1.89-specific. I then set up a transparent monitoring proxy (cc-relay) using the official `ANTHROPIC_BASE_URL` env var to capture per-request data going forward.

What began as a personal debugging session turned into a broader investigation as dozens of other users reported the same issue across [91+ GitHub issues](#related-issues). The community — [@Sn3th](https://github.com/Sn3th), [@rwp65](https://github.com/rwp65), [@dbrunet73](https://github.com/dbrunet73), [@luongnv89](https://github.com/luongnv89), and [12 others](#contributors--acknowledgments) — independently found different pieces of the puzzle.

This repo consolidates those findings into a structured analysis with measured data.

### Anthropic's Position (April 2)

Lydia Hallie (Anthropic) [posted on X](https://x.com/lydiahallie/status/2039800715607187906):

> *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."*

She [recommended](https://x.com/lydiahallie/status/2039800718371307603) using Sonnet as default, lowering effort level, starting fresh instead of resuming, and capping context with `CLAUDE_CODE_AUTO_COMPACT_WINDOW=200000`.

Our measured data agrees that the cache bugs are fixed, but identifies five additional mechanisms that Anthropic's statement doesn't address — see the [bug table below](#whats-happening-april-3-2026).

### Cache TTL (not a bug)

Separately, [@luongnv89](https://github.com/luongnv89) [documented](https://github.com/luongnv89/cc-context-stats/blob/main/context-stats-cache-misses.md) that idle gaps of 13+ hours cause a full 350K-token cache rebuild when resuming (cache write costs $3.75/M vs read at $0.30/M — a 12.5x difference). Our data shows shorter gaps (5-26 minutes) maintain 96%+ cache, so this only affects long idle periods. It's by design (5-minute TTL), not a bug — but worth knowing about.

### What Changed in v2.1.91 ([full changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md))

The most relevant v2.1.91 changes for this analysis:
- `_meta["anthropic/maxResultSizeChars"]` annotation (up to 500K) — MCP tool results can now opt out of truncation. **Built-in tools are NOT affected.**
- `--resume` transcript chain break fix — additional stability for session resumption
- `Edit` tool uses shorter `old_string` anchors — reduces output tokens
- `stripAnsi` routed through `Bun.stripANSI` — performance improvement that appears to have closed the Sentinel gap between npm and standalone

---

## What's Happening (April 3, 2026)

The original cache regression (v2.1.89 standalone draining 100% in 70 minutes) is **fixed** in v2.1.90-91. Cache read ratios are back to 95-99% in stable sessions, and npm/standalone now perform identically on v2.1.91.

But cache was only part of the story. Proxy-based testing revealed five additional mechanisms that remain active:

| Bug | What It Does | Impact | Status (v2.1.91) | Evidence |
|-----|-------------|--------|-------------------|----------|
| **B1** Sentinel | Standalone binary corrupts cache prefix | 4-17% cache read (v2.1.89) | **Fixed** — npm=standalone now | [BENCHMARK.md](BENCHMARK.md) |
| **B2** Resume | `--resume` replays full context uncached | 20x cost per resume | **Fixed** (avoid `--resume` anyway) | [#34629](https://github.com/anthropics/claude-code/issues/34629) |
| **B3** False RL | Client blocks API calls with fake error | Instant "Rate limit reached", 0 API calls | **Unfixed** — 151 entries / 65 sessions | [#40584](https://github.com/anthropics/claude-code/issues/40584) |
| **B4** Microcompact | Tool results silently cleared mid-session | Context quality degrades (cache unaffected) | **Unfixed** — 327 events detected | [#42542](https://github.com/anthropics/claude-code/issues/42542) |
| **B5** Budget cap | 200K aggregate limit on tool results | Older results truncated to 1-41 chars | **Unfixed** (MCP override only) | [MICROCOMPACT.md](MICROCOMPACT.md) |
| **B8** Log inflation | Extended thinking duplicates JSONL entries | 2.87x local token inflation | **Unfixed** | [#41346](https://github.com/anthropics/claude-code/issues/41346) |
| **Server** | Peak-hour limits tightened + 1M billing bug | Reduced effective quota | **By design** (Anthropic confirmed) | [#42616](https://github.com/anthropics/claude-code/issues/42616) |

### What You Can Do

1. **Update to v2.1.91** — this fixes the cache regression that caused the worst drain
2. **npm or standalone — either is fine on v2.1.91** (Sentinel gap closed, [84.7% identical cold start](#measured-data))
3. **Don't use `--resume` or `--continue`** — replays full context as billable input
4. **Start fresh sessions periodically** — the 200K tool result cap (B5) means older file reads get silently truncated after ~15-20 tool uses
5. **Avoid `/dream` and `/insights`** — background API calls that drain silently

```jsonc
// ~/.claude/settings.json — disable auto-update
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

---

## npm vs Standalone Binary

**Short answer: on v2.1.91, it doesn't matter.** Both achieve identical 84.7% cold-start cache read and 97-99% in stable sessions. The Sentinel gap that existed in v2.1.90 (47-67% vs 79-87% on sub-agent cold starts) has been closed.

If you want the details, Claude Code ships in two forms:

### Standalone Binary (ELF)

- Installed via `curl -fsSL https://claude.ai/install.sh | bash`
- Ships as a **single ELF 64-bit executable** (~228MB) with embedded Bun runtime
- Contains the Sentinel replacement mechanism (`cch=00000`) that can corrupt cache prefixes
- **v2.1.91 status:** Cold start varies (27.8-84.7% depending on workspace), but recovers to 99%+ in 1-2 requests. Sub-agents start at 0% but warm quickly to 91%+

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

**On v2.1.90:** npm had a 12-40pp advantage on sub-agent cold starts (79-87% vs 47-67%). Both reached 94-99% once warmed (3-5 requests). **On v2.1.91:** this gap is closed — both hit 84.7% on cold start. See **[BENCHMARK.md](BENCHMARK.md)** for per-request data.

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

Below are the technical details for each bug. Bugs 1-2 (cache layer) are fixed. Bugs 3-5 and 8 are the ones still causing problems — if you're experiencing drain on v2.1.91, these are why.

Bugs 1-2 were identified through community reverse engineering ([Reddit analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)). Bugs 3-5 and 8 were discovered through proxy-based testing on April 2-3, 2026:

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

**Note:** `--continue` has the same cache invalidation behavior ([#42338](https://github.com/anthropics/claude-code/issues/42338) confirmed). While v2.1.90-91 changelogs address `--resume`, we recommend avoiding both `--resume` and `--continue` until fully verified — start fresh sessions instead.

### Bug 3 — Client-Side False Rate Limiter (all versions)

**GitHub Issue:** [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584)

The local rate limiter generates **synthetic "Rate limit reached" errors** without ever calling the Anthropic API. These errors are identifiable in session logs by:

```json
{
  "model": "<synthetic>",
  "usage": { "input_tokens": 0, "output_tokens": 0 }
}
```

Triggered by large transcripts and concurrent sub-agent spawns. [@rwp65](https://github.com/rwp65) observed it with a ~74MB transcript in [#40584](https://github.com/anthropics/claude-code/issues/40584). The rate limiter appears to multiply `context_size × concurrent_requests`, so multi-agent workflows get blocked even when each individual request is small.

- **Discovery:** [@rwp65](https://github.com/rwp65) in [#40584](https://github.com/anthropics/claude-code/issues/40584) (March 29, 2026)
- **Cross-referenced by:** [@marlvinvu](https://github.com/marlvinvu) across [#40438](https://github.com/anthropics/claude-code/issues/40438), [#39938](https://github.com/anthropics/claude-code/issues/39938), [#38239](https://github.com/anthropics/claude-code/issues/38239)
- **Status:** **Unfixed** — present in all versions through v2.1.91
- **Impact:** Users see "Rate limit reached" immediately, even after hours of inactivity when the budget should have fully reset. No API call is made, so the error is entirely client-generated.

### Bug 4 — Silent Microcompact → Context Quality Degradation (v2.1.89+)

**GitHub Issue:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)

Three compaction mechanisms in `src/services/compact/` run **silently on every API call**, stripping old tool results without user notification.

| Mechanism | Source | Trigger | Control |
|-----------|--------|---------|---------|
| **Time-based microcompact** | `microCompact.ts:422` | Gap since last assistant message exceeds threshold | GrowthBook: `getTimeBasedMCConfig()` |
| **Cached microcompact** | `microCompact.ts:305` | Count-based trigger, uses `cache_edits` API to delete old tool results | GrowthBook: `getCachedMCConfig()` |
| **Session memory compact** | `sessionMemoryCompact.ts:57` | Runs before autocompact | GrowthBook flag |

**Key findings:**
- All three bypass `DISABLE_AUTO_COMPACT` and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`
- Controlled by **server-side GrowthBook A/B testing flags** — Anthropic can change behavior without a client update
- Tool results silently replaced with `[Old tool result content cleared]` — no compaction notification shown
- **327 clearing events across all tested sessions** detected via proxy across multiple sessions
- All cleared indices are **even-numbered** → targets tool_use/tool_result pairs specifically
- Cleared indices **expand over time** as conversation grows

**Cache impact (updated April 3 — measured):**

Our proxy-based testing revealed a **correction** to the initial hypothesis: microcompact does NOT cause sustained cache invalidation in main sessions.

| Context | Cache ratio during clearing |
|---------|---------------------------|
| Main session | **99%+** — no impact (stable substitution preserves prefix) |
| Sub-agent cold start | **0-39%** — drops observed at clearing moments |
| Sub-agent warmed | **94-99%** — recovers normally |

Cache ratio stays high because the same `[Old tool result content cleared]` marker is substituted consistently, preserving the prompt prefix between calls. But the model can no longer see the original file contents or command outputs — it only sees the placeholder. In practice, this means the agent can't accurately quote earlier tool results and may retry approaches it already tried. [@Sn3th](https://github.com/Sn3th) reports effective context dropping to ~40-80K tokens in sessions with 50+ tool uses despite the 1M window — a 95-96% reduction in usable context.

- **Update (April 3):** GrowthBook flag survey across 4 machines / 4 accounts shows **all gates disabled** — yet context is still being stripped (a compaction code path independent of the three documented GrowthBook-gated mechanisms). See [MICROCOMPACT.md](MICROCOMPACT.md) for full analysis.
- **Discovery:** [@Sn3th](https://github.com/Sn3th) in [#42542](https://github.com/anthropics/claude-code/issues/42542) (April 2, 2026)
- **Status:** **Unfixed** in v2.1.91. 14 events detected in v2.1.91 test sessions, identical pattern.

### Bug 5 — Tool Result Budget Enforcement (all versions, NEW)

**Discovered:** April 3, 2026 (via cc-relay proxy enhancement)
**Source:** [@Sn3th](https://github.com/Sn3th) identified the GrowthBook flags; we confirmed behavioral activation.

A **separate pre-request pipeline** (`applyToolResultBudget()`) truncates tool results based on server-controlled thresholds. This runs BEFORE microcompact (Bug 4) and is independent of it.

**Active GrowthBook flags (confirmed in `~/.claude.json`):**
```
tengu_hawthorn_window:       200,000  (aggregate tool result cap across all messages)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: true    (system prompt tells model to expect clearing)
```

**Measured impact:**
- **261 budget events** detected in a single session when total tool result chars exceeded 200K
- Tool results reduced to **1-41 characters** (from original thousands+)
- Budget threshold exceeded at **242,094 chars** (> 200K cap)
- After ~15-20 file reads, older results are silently truncated

**v2.1.91:** Added `_meta["anthropic/maxResultSizeChars"]` (up to 500K) — but this only applies to **MCP tool results**. Built-in tools (Read, Bash, Grep, Glob, Edit) are **not affected** by this override. The 200K aggregate cap remains for normal usage.

**No env var override exists.** `DISABLE_AUTO_COMPACT`, `DISABLE_COMPACT`, and all other known environment variables do not touch this code path.

### Bug 8 — JSONL Log Duplication (all versions)

**GitHub Issue:** [anthropics/claude-code#41346](https://github.com/anthropics/claude-code/issues/41346)

Extended thinking generates **2-5 PRELIM entries** per API call in session JSONL files, with identical `cache_read_input_tokens` and `cache_creation_input_tokens` as the FINAL entry. This inflates local token accounting.

**Measured (April 3):**

| Session Type | PRELIM | FINAL | Ratio | Token Inflation |
|-------------|--------|-------|-------|----------------|
| Main session | 79 | 82 | 0.96x | **2.87x** |
| Sub-agent | 39 | 20 | 1.95x | — |
| Sub-agent | 12 | 7 | 1.71x | — |
| Previous session | 16 | 6 | **2.67x** | — |

**Open question:** Does the server-side rate limiter count PRELIM entries? If yes, extended thinking sessions are charged 2-3x more against the rate limit than the actual API usage.

---

## Measured Data

### Methodology

Transparent local monitoring proxy using [`ANTHROPIC_BASE_URL`](https://docs.anthropic.com/en/docs/claude-code/settings#environment-variables) (official environment variable). The proxy logs `cache_creation_input_tokens` and `cache_read_input_tokens` from each API response without modifying requests or responses (source-audited).

### v2.1.89 Standalone — Before Fix (Broken, JSONL data)

| Session | Entries | Avg Cache Read | Min | Status |
|---------|---------|---------------|-----|--------|
| `64d42269` (4/1 16:33) | 233 | **47.0%** | 8.3% | drain starting |
| `9100c2d2` (4/1 18:04) | 89 | **36.1%** | 21.1% | worst drain — triggered investigation |
| Session A (earlier JSONL) | 168 | **4.3%** | — | poor — 20x cost inflation |

After downgrading to v2.1.68 (npm): `892388f6` recovered to **97.6% average** (119 entries, min 60.9%).

### v2.1.90 — After Fix (Both Installations)

| Metric | npm (Node.js) | Standalone (ELF) |
|--------|--------------|-----------------|
| Scenarios completed | 7 (incl. 79-report parallel agent read) | 4 (forge, browsegrab, feedkit 5-turn, 3-project parallel) |
| Usage consumed | 28% → 35% (**7%**) | 35% → 40% (**5%**) |
| Overall cache read | **86.4%** | **86.2%** |
| Stable session read | **95-99.8%** | **95-99.7%** |

Full per-request data and warming curves: **[BENCHMARK.md](BENCHMARK.md)**

### Version Comparison Summary (Full Benchmark, Same Scenarios)

| Metric | v2.1.89 Standalone | v2.1.90 npm | v2.1.90 standalone | v2.1.91 npm | v2.1.91 standalone |
|--------|-------------------|------------|-------------------|------------|-------------------|
| Cold start | **4-17%** | 63-80% | **14-47%** | **84.5%** | **27.8%** |
| Recovery to 95%+ | Never | 3-5 reqs | 3-5 reqs | **2 reqs** | **1 req** |
| Sub-agent cold | — | 54-80% | 14-47% | **54%** | **0%** |
| Sub-agent stable | — | 87-94% | 94-99% | **93-99%** | **91-99%** |
| Stable session | 90-99% | **95-99.8%** | **95-99.7%** | **98-99.6%** | **94-99%** |
| Overall | ~20% | 86.4% | 86.2% | **88.4%** | **84.1%** |
| Verdict | **Avoid** | Good | Good | **Best** | **Good** |

v2.1.91 standalone cold start varies by workspace (27.8% in full benchmark vs 84.7% in single-prompt test), but recovery is dramatically faster than v2.1.90 (1 request vs 3-5). Both installations converge to 94-99% once warmed. See **[BENCHMARK.md](BENCHMARK.md)** for per-request data.

---

## Usage Precautions

### Behaviors to Avoid

| Behavior | Why | Measured Impact |
|----------|-----|-----------------|
| `--resume` | Replays entire conversation history as billable input including opaque thinking block signatures | 500K+ tokens per resume ([#42260](https://github.com/anthropics/claude-code/issues/42260)) |
| `/dream`, `/insights` | Background API calls consume tokens without visible output | Silent drain ([#40438](https://github.com/anthropics/claude-code/issues/40438)) |
| v2.1.89 or earlier standalone | Sentinel bug causes sustained 4-17% cache read | 3-4x token waste, never recovers |
| Enabling auto-update | Future versions may reintroduce regressions | Pin v2.1.91 until Bugs 3-5 are fixed |

### Behaviors to Use with Caution

| Behavior | Why | Recommendation |
|----------|-----|----------------|
| Parallel sub-agents (single terminal) | Each agent starts with fresh context, but warms up and shares billing context | **Safe** — agents warm up to 94-99% after 3-5 requests |
| Multiple terminals simultaneously | Each terminal is a fully independent session — no cache sharing, parallel quota drain | **Limit to one active terminal** |
| Large CLAUDE.md / context files | Sent on every turn — with broken cache, billed at full price each time | Keep lean; less critical on v2.1.90 with working cache |
| Session start / compaction | `cache_creation` spikes are structural and unavoidable | Normal — budget for it |

### Server-Side Factors (Unresolved)

Even with cache at 95-99%, drain persists. As of April 3, the [#38335 mega-thread](https://github.com/anthropics/claude-code/issues/38335) has 365+ comments and [#16157](https://github.com/anthropics/claude-code/issues/16157) has 1,400+ — both still active with new reports on v2.1.90-91. At least three server-side issues contribute:

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
claude --version   # should show 2.1.91 or later
file $(which claude)   # should show symbolic link to cli.js
```

### For Existing Standalone Users

```bash
# 1. Update to v2.1.91
claude update

# 2. Disable auto-update (add to existing settings.json)
# Add "DISABLE_AUTOUPDATER": "1" to the "env" section

# 3. Verify
claude --version   # should show 2.1.91 or later
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

If most sessions show low read ratios, you're likely on an affected version. Update to v2.1.91.

---

## Related Issues

### Root Cause Bugs
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — Conversation history invalidated (Bug 1: sentinel) — **improved in v2.1.89-91**
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume cache regression (Bug 2: deferred_tools_delta) — **improved in v2.1.90-91**
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= billing hash substitution
- [#40584](https://github.com/anthropics/claude-code/issues/40584) — **Client-side false rate limiter** (Bug 3: 151 synthetic entries confirmed) — **unfixed**
- [#42542](https://github.com/anthropics/claude-code/issues/42542) — **Silent microcompact → context degradation** (Bug 4: 327 events, cache unaffected) — **unfixed**
- Bug 5: **Tool result budget enforcement** (200K aggregate cap, discovered via GrowthBook flags) — **unfixed** (v2.1.91 MCP override only)
- [#41346](https://github.com/anthropics/claude-code/issues/41346) — **JSONL log duplication** (Bug 8: 2.87x PRELIM inflation) — **unfixed**

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

**GitHub:** Zero responses across 91+ rate-limit issues (2+ months of silence).

**April 2, 2026 — Lydia Hallie (Anthropic, Product) posted on X:**

> *"Peak-hour limits are tighter and 1M-context sessions got bigger, that's most of what you're feeling. We fixed a few bugs along the way, but none were over-charging you."*

**Our measured data raises questions about this assessment:**
- **Bug 5 (200K cap):** Tool results silently truncated to 1-41 chars after the aggregate 200K threshold. Users paying for 1M context effectively have a 200K tool result budget for built-in tools — the rest is silently discarded.
- **Bug 3 (synthetic RL):** 151 `<synthetic>` entries across 65 sessions on our setup alone. The client blocks API calls without server involvement — users see "Rate limit reached" with zero actual API consumption.
- **Bug 8 (PRELIM duplication):** Extended thinking sessions log 2-3x more token entries than actual API calls. Whether the server-side rate limiter counts these remains an open question.

| Who | Platform | What | Link |
|-----|----------|------|------|
| **Lydia Hallie** | X | Full statement on rate limits (thread start) | [Post 1](https://x.com/lydiahallie/status/2039800715607187906) |
| **Lydia Hallie** | X | Follow-up with usage tips (thread end) | [Post 2](https://x.com/lydiahallie/status/2039800718371307603) |
| **Lydia Hallie** | X | *"We shipped some fixes that should help"* (earlier) | [Post](https://x.com/lydiahallie/status/2039107775314428189) |
| **Thariq Shihipar** | X | *"Bug with prompt caching... hotfixed"* (earlier incident) | [Post](https://x.com/trq212/status/2027232172810416493) |
| **Official Changelog** | GitHub | v2.1.89-91 fix entries | [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

### Community Engagement

As of April 3, 2026: **180+ comments on 91 unique issues** (including v2.1.90-91 benchmark updates + Bug 3-5 cross-references). Anthropic official response on GitHub: **zero**.

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

## Community References

### Analysis & Tools
- [Reddit: Reverse engineering analysis of Claude Code cache bugs](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — Community cache patch + test toolkit
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy-based Claude Code traffic analysis
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Transparent proxy for Claude Code
- [CUStats](https://custats.info) — Real-time usage tracking and visualization
- [context-stats](https://github.com/luongnv89/cc-context-stats) — Per-interaction cache metrics export and analysis (by [@luongnv89](https://github.com/luongnv89))
- [BudMon](https://github.com/weilhalt/budmon) — Desktop dashboard for rate-limit header monitoring

### Token Optimization Tools (complementary, not bug fixes)
- [rtk](https://github.com/rtk-ai/rtk) — Tool output compression (trims CLI/test results post-execution, reduces input token volume)
- [tokenlean](https://github.com/edimuj/tokenlean) — 54 CLI tools for agents (extracts symbols/snippets instead of full file reads, reduces base token count)

---

## Files in This Repo

| File | Description |
|------|-------------|
| [README.md](README.md) | This file — overview, current status, and recommendations |
| [BENCHMARK.md](BENCHMARK.md) | Controlled npm vs standalone benchmark with raw per-request data (v2.1.90) |
| [MICROCOMPACT.md](MICROCOMPACT.md) | Deep dive on silent context stripping (Bug 4) + tool result budget (Bug 5) |
| [TIMELINE.md](TIMELINE.md) | 14-month chronicle of rate limit issues (Phase 1-9, 50+ issues) |
| [TEST-RESULTS-0403.md](TEST-RESULTS-0403.md) | April 3 integrated test results — all 6 bugs verified with measured data |

## Environment

- **Plan:** Max 20 ($200/mo)
- **OS:** Linux (Ubuntu), HP ZBook Ultra G1a
- **Versions tested:** v2.1.91 (npm + standalone head-to-head), v2.1.90 (npm + standalone benchmark), v2.1.89 (affected), v2.1.68 (pre-bug baseline)
- **Monitoring:** cc-relay v2 transparent proxy — microcompact detection, budget enforcement scanning, session ID support, JSONL analyzer
- **Date:** April 3, 2026

---

## Contributors & Acknowledgments

This analysis builds on work by many community members who independently investigated and measured these issues:

| Who | Contribution |
|-----|-------------|
| [@Sn3th](https://github.com/Sn3th) | Discovered and documented the three microcompact mechanisms (Bug 4), identified GrowthBook flag extraction method, found the additional `applyToolResultBudget()` pipeline (Bug 5) and per-tool caps, confirmed server-side context mutation across multiple machines |
| [@rwp65](https://github.com/rwp65) | Discovered the client-side false rate limiter (Bug 3) with detailed log evidence showing `<synthetic>` model entries |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | Provided Windows GrowthBook flag dumps confirming cross-platform consistency, tested with telemetry disabled |
| [@dbrunet73](https://github.com/dbrunet73) | Published real-world OTel comparison data (v2.1.88 vs v2.1.90) confirming cache improvement |
| [@maiarowsky](https://github.com/maiarowsky) | Confirmed Bug 3 on v2.1.90 with 26 synthetic entries across 13 sessions |
| [@luongnv89](https://github.com/luongnv89) | Analyzed cache TTL behavior with per-interaction granularity, built [CUStats](https://custats.info) and [context-stats](https://github.com/luongnv89/cc-context-stats) |
| [@edimuj](https://github.com/edimuj) | Measured grep/file-read token waste (3.5M tokens across 1800+ calls), built [tokenlean](https://github.com/edimuj/tokenlean) |
| [@amicicixp](https://github.com/amicicixp) | Verified v2.1.90 cache improvement with before/after testing |
| [@simpolism](https://github.com/simpolism) | Identified v2.1.90 changelog correlation with `--resume` cache fix |
| [@weilhalt](https://github.com/weilhalt) | Built [BudMon](https://github.com/weilhalt/budmon) for real-time rate-limit header monitoring |
| [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) | Reported old Docker versions (v2.1.74/86) draining — key server-side evidence ([#37394](https://github.com/anthropics/claude-code/issues/37394)) |
| [@SC7639](https://github.com/SC7639) | Provided additional regression data confirming the mid-March timeline |
| Reddit community | [Reverse engineering analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) of cache sentinel mechanism |

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
