> **🇰🇷 [한국어 버전](ko/05_MICROCOMPACT.md)**

# Silent Context Mutation — Bugs 4 & 5

Two separate mechanisms silently modify your conversation before it reaches the API. Neither is controllable via environment variables, and both remain active on v2.1.91.

- **Bug 4 (Microcompact):** Earlier tool results replaced with `[Old tool result content cleared]`. 327 events measured (April 3 focused test). Doesn't hurt cache (99%+ maintained) but destroys context quality.
- **Bug 5 (Budget enforcement):** Tool results truncated after 200K aggregate chars. 261 events measured (April 3 focused test). Results reduced to 1-41 chars (April 3 session; full-week max: 49 chars).

> **Latest data (April 15):** Proxy data (35,554 requests, April 1–15) shows B4: **5,500 events** (18,858 items cleared), B5: **167,818 events** (100% truncation rate, 100% ≤50 chars) — all from the **unmodified baseline period** (April 1–10). A proxy-based GrowthBook flag override deployed April 10 eliminated both completely (167,818 → 0, 5,500 → 0 across 9,996 subsequent requests). See [01_BUGS.md](01_BUGS.md#growthbook-flag-override--controlled-elimination-test-april-1014) for the controlled test methodology. See [13_PROXY-DATA.md](13_PROXY-DATA.md) for the full dataset.

Both are controlled server-side via GrowthBook feature flags — Anthropic can change behavior without a client update. `/export` does NOT show the mutated version — it shows the full context, while the API receives the trimmed one.

> **Issue:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)
> **Discovery:** [@Sn3th](https://github.com/Sn3th), April 2, 2026
> **Status:** Unfixed in v2.1.91 | **Tested:** npm + standalone, April 3, 2026

---

## Bug 4: Microcompact — What's Happening

Tool results from earlier in a session are silently replaced with `[Old tool result content cleared]`. No compaction notification is shown, no hooks fire. The model continues operating without access to the original tool output — it can no longer quote file contents, reference specific grep results, or verify earlier command outputs.

In heavy tool-use sessions (50+ file reads, greps, bash commands), [@Sn3th](https://github.com/Sn3th) reports effective context dropping to ~40-80K despite the 1M window. Our proxy data supports this: in a 251-message session, 11 distinct message indices were cleared; in a 135-message session, 12 indices. Clearing begins within the first 60 messages and expands as the conversation grows.

## Three Compaction Mechanisms

These are controlled by [GrowthBook](https://www.growthbook.io/), a feature-flagging system. The Claude Code CLI fetches flags from Anthropic's servers and caches them in `~/.claude.json`. Users cannot disable these flags — Anthropic controls them server-side and can change behavior without a client update.

Source: `src/services/compact/` in the Claude Code binary (from the publicly available npm bundle, as documented in [#42542](https://github.com/anthropics/claude-code/issues/42542)).

| Mechanism | Source | Trigger | Control |
|-----------|--------|---------|---------|
| **Time-based microcompact** | `microCompact.ts:422` | Gap since last assistant message exceeds threshold | GrowthBook: `getTimeBasedMCConfig()` |
| **Cached microcompact** | `microCompact.ts:305` | Count-based trigger, uses `cache_edits` API to delete old tool results from server cache | GrowthBook: `getCachedMCConfig()` |
| **Session memory compact** | `sessionMemoryCompact.ts:57` | Runs before autocompact, keeps only ~40K tokens of recent messages | GrowthBook: `tengu_session_memory` / `tengu_sm_compact` |

All three bypass `DISABLE_AUTO_COMPACT` and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`. The only env var that touches SM compact is `DISABLE_CLAUDE_CODE_SM_COMPACT=true`, and even that doesn't cover microcompact.

## GrowthBook Feature Flag Survey

The CLI caches the GrowthBook payload to `~/.claude.json` under `cachedGrowthBookFeatures`. Extract with:

```bash
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
for k in ['tengu_slate_heron', 'tengu_session_memory', 'tengu_sm_compact',
          'tengu_sm_compact_config', 'tengu_cache_plum_violet']:
    print(f'{k}: {json.dumps(gb.get(k, \"NOT PRESENT\"), indent=2)}')
"
```

### Collected Data (April 2-3, 2026)

| Flag | Machine 1 (Sn3th, dev) | Machine 2 (Sn3th, prod) | Machine 3 (ArkNill, ubuntu-1) | Machine 4 (arizonawayfarer, Win) |
|------|------------------------|------------------------|---------------------------|-------------------------------|
| `tengu_slate_heron` | `{"enabled": false, ...}` | identical | identical | identical |
| `tengu_session_memory` | `false` | identical | identical | identical |
| `tengu_sm_compact` | `false` | identical | identical | identical |
| `tengu_sm_compact_config` | `{"minTokens": 2000, "maxTokens": 20000, ...}` | identical | identical | identical |
| `tengu_cache_plum_violet` | `true` | identical | identical | identical |

### Flag Glossary

| Flag | Purpose | Current Value |
|------|---------|---------------|
| `tengu_slate_heron` | Controls time-based microcompact (gap threshold, keep-recent count) | `enabled: false` |
| `tengu_session_memory` | Gates session memory compact | `false` |
| `tengu_sm_compact` | Gates SM compact (separate from session memory) | `false` |
| `tengu_sm_compact_config` | Thresholds for SM compact (min/max tokens, message count) | `{minTokens: 2000, maxTokens: 20000}` |
| `tengu_cache_plum_violet` | Unknown purpose — the only enabled flag across all surveyed machines | `true` |
| `tengu_hawthorn_window` | **Aggregate tool result budget** — total chars allowed across all tool results (Bug 5) | `200000` |
| `tengu_pewter_kestrel` | **Per-tool result size caps** — individual tool output limits (Bug 5) | `{global: 50000, Bash: 30000, Grep: 20000}` |
| `tengu_summarize_tool_results` | System prompt flag telling the model to expect cleared tool results | `true` |

Despite all microcompact gates showing disabled, context stripping persists — which points to the budget enforcement flags (`hawthorn_window`, `pewter_kestrel`) as the active mechanism, or an undocumented code path that doesn't check these gates.

**4 machines, 4 accounts (3 Linux, 1 Windows), all Max plan, v2.1.90 — every gate shows disabled. Context is still being stripped.**

## The SM Compact Threshold Problem

Even with the gate off, the remote config carries thresholds that are far more aggressive than the code defaults:

| Parameter | Remote Config | Code Default | Difference |
|-----------|--------------|-------------|------------|
| `minTokens` | **2,000** | 10,000 | **5x lower floor** |
| `maxTokens` | **20,000** | 40,000 | **2x lower ceiling** |
| `minTextBlockMessages` | 5 | — | — |

If this gate flips on via A/B test, sessions will retain 20K tokens max — a fraction of what users paying for 1M context expect.

## Cache Impact — Revised Understanding (April 3)

**Initial hypothesis (April 2):** We expected microcompact to invalidate prompt cache prefixes, causing 0% cache read and full-price billing.

**Measured result:** This hypothesis was **partially wrong.** Proxy data across 327 events (all sessions combined) shows the following. See [04_BENCHMARK.md](04_BENCHMARK.md) for detailed sub-agent cache measurements.

| Context | Cache ratio during clearing | Explanation |
|---------|---------------------------|-------------|
| Main session | **99%+** — no impact | Clearing substitutes the same marker consistently, so the prefix doesn't change between calls |
| Sub-agent cold start | **0-39%** — significant drops | Sub-agents build a new cache from the already-cleared context, hitting cold-start penalties |
| Sub-agent warmed (5+ req) | **94-99%** — normal | After warming, sub-agents stabilize regardless of cleared content |

**The real cost is context quality, not cache billing.** When tool results are cleared:
- The model cannot accurately reference earlier file contents, grep results, or command outputs
- This causes repeated failed approaches (the model can't see what it already tried)
- Long sessions degrade to ~40-80K effective context despite the 1M window (reported by [@Sn3th](https://github.com/Sn3th))

**Related observations:**
- Old Docker-pinned versions (v2.1.74/86, never updated) started draining recently ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — this confirms GrowthBook flags are changed server-side without requiring a client update
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` doesn't prevent clearing (microcompact bypasses it entirely)
- The drain appears intermittent across users — consistent with GrowthBook A/B testing controlling who gets hit and how aggressively

## Local Reproduction (In Progress)

### Setup

Proxy-based request body scanner (cc-relay + `ANTHROPIC_BASE_URL`) detects `[Old tool result content cleared]` in outgoing API requests before they leave the machine. GrowthBook file watcher diffs `~/.claude.json` every 10 seconds.

### Confirmed Results (April 3, 2026)

Systematic testing with enhanced cc-relay proxy across multiple sessions (3,500+ logged requests, 327 microcompact events total). See [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) for the full session log analysis.

**Clearing Pattern:**
- **327 events** detected total — 67 and 71 in two major v2.1.90 sessions, 14 in v2.1.91 test sessions, and the remaining 175 events from sub-agent sessions spawned during the v2.1.90 testing
- Clearing indices **expand over time**: starts at [20,22], grows to [20,22,38,40,44,46,162,166,172,174,206]
- **All cleared indices are even-numbered** → targets tool_use/tool_result pairs specifically
- Same indices cleared consistently on every subsequent API call (stable substitution)

**Cache Impact — KEY FINDING:**
- **Main session: NO cache impact.** Ratio stays 99%+ during active clearing. Because the same indices are consistently replaced with the same marker text, the prompt prefix doesn't change → cache stays valid.
- **Sub-agent cold starts: 0-39% cache drops** observed at moments where clearing is active. This is because sub-agents build a new cache from the already-cleared context.
- **Practical harm is context quality, not cache cost.** The agent loses access to earlier tool results and cannot accurately quote or reference them.

**Budget Enforcement (Bug 5) — NEW DISCOVERY:**

In addition to microcompact, a **separate pre-request pipeline** truncates tool results based on server-controlled GrowthBook thresholds:

```
tengu_hawthorn_window:       200,000 (aggregate tool result cap)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: true
```

This runs BEFORE microcompact via `applyToolResultBudget()`. Confirmed active:
- **261 budget events** detected in a single session
- Tool results reduced to **1-41 chars** in this session (originally thousands+; full-week max: 49 chars — see [13_PROXY-DATA.md](13_PROXY-DATA.md))
- Budget exceeded at **242,094 chars** (> 200K cap)
- v2.1.91 `maxResultSizeChars` override is **MCP-only** — built-in Read/Bash/Grep unaffected

**v2.1.91 Verification:**
- 14 microcompact events, 71 budget events in v2.1.91 test sessions
- **No change from v2.1.90** — both bugs persist identically

### Remaining Verification

1. **Binary code path identification** — search for `Old tool result content cleared` in decompiled binary
2. **GrowthBook network interception** — disk cache shows no changes during clearing; in-memory divergence suspected
3. **Budget truncation threshold** — exact token count (not char count) at which truncation begins

## Open Questions

### 1. GrowthBook Runtime vs Disk Cache Divergence

The disk cache in `~/.claude.json` is a snapshot of the last GrowthBook evaluation. But GrowthBook evaluates features using per-session attributes (user ID, plan tier, session ID, timestamp, etc.) at runtime. If the CLI re-evaluates mid-session with different attributes — after crossing a token threshold, or based on session duration — the disk cache would never reflect it.

Preliminary local testing shows no disk cache changes during active clearing, but this doesn't rule out in-memory divergence.

**Test:** Intercept GrowthBook SDK calls (or the GrowthBook API endpoint) during a session where stripping is actively happening, and compare runtime feature values against the disk cache.

### 2. Server-Side Content Clearing

When `[Old tool result content cleared]` appears, is it present in the raw API response from Anthropic, or does the CLI insert it locally?

Preliminary evidence points to **client-side** (marker found in outgoing request body), but needs verification with a longer session and confirmed first-clearing capture.

### 3. The Fourth Path

All three documented GrowthBook gates are off, yet clearing occurs. Confirmed locally. Candidates:
- A code path in the binary that doesn't check GrowthBook at all
- A compile-time flag active in certain builds
- A response header or metadata field that triggers client-side clearing
- The `cache_edits` API path (cached microcompact) operating independently of the GrowthBook gate

## Env Vars That Don't Help

| Env Var | What It Controls | Does It Block Microcompact? |
|---------|-----------------|---------------------------|
| `DISABLE_AUTO_COMPACT=true` | Autocompact only | **No** |
| `DISABLE_COMPACT=true` | All compaction including manual `/compact` | Probably, but kills manual compact too |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Autocompact threshold | **No** |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=true` | Session memory compact only | **No** (doesn't cover time-based or cached MC) |

**What's needed:** `DISABLE_MICROCOMPACT=true` — a dedicated env var to disable microcompact independently.

## Contributing Data

If you're experiencing silent context stripping, please share:

1. **GrowthBook flags** — run the extraction script above and post your values. If anyone shows `tengu_sm_compact: true` or `tengu_slate_heron.enabled: true`, that confirms the A/B angle.
2. **Proxy capture** — set `ANTHROPIC_BASE_URL` to a logging proxy and capture the raw API response when `[Old tool result content cleared]` appears. This determines whether it's client-side or server-side.
3. **Session details** — version, plan tier, OS, approximate session length and tool use volume when stripping was observed.

## References

- [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542) — Original issue with source code analysis
- [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524) — Bug 1: Sentinel cache corruption
- [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629) — Bug 2: Resume cache regression
- [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584) — Bug 3: Client-side false rate limiter
- [anthropics/claude-code#37394](https://github.com/anthropics/claude-code/issues/37394) — Docker-pinned old versions draining (server-side evidence)
- [anthropics/claude-code#42590](https://github.com/anthropics/claude-code/issues/42590) — Context compaction too aggressive on 1M
