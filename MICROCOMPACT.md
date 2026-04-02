# Silent Microcompact — Deep Dive (Bug 4)

> **GitHub Issue:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)
> **Discovery:** [@Sn3th](https://github.com/Sn3th), April 2, 2026
> **Status:** Unfixed — all known gates show "disabled", but context stripping still occurs

---

## What's Happening

Tool results from earlier in a session are silently replaced with `[Old tool result content cleared]`. No compaction notification is shown, no hooks fire. The agent continues operating on degraded context — making confident statements from internalized summaries because the source material is gone.

Sessions doing heavy tool use (50+ file reads, greps, bash commands) see effective context drop to ~40-80K despite the token counter showing far less than the 1M window.

## Three Compaction Mechanisms

Source: `src/services/compact/` in the Claude Code binary.

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

| Flag | Machine 1 (Sn3th, dev) | Machine 2 (Sn3th, prod) | Machine 3 (ArkNill, ZBook) | Machine 4 (arizonawayfarer, Win) |
|------|------------------------|------------------------|---------------------------|-------------------------------|
| `tengu_slate_heron` | `{"enabled": false, ...}` | identical | identical | identical |
| `tengu_session_memory` | `false` | identical | identical | identical |
| `tengu_sm_compact` | `false` | identical | identical | identical |
| `tengu_sm_compact_config` | `{"minTokens": 2000, "maxTokens": 20000, ...}` | identical | identical | identical |
| `tengu_cache_plum_violet` | `true` | identical | identical | identical |

All flags:
- `tengu_slate_heron.enabled`: **false** (time-based MC off)
- `tengu_session_memory`: **false** (session memory off)
- `tengu_sm_compact`: **false** (SM compact off)
- `tengu_cache_plum_violet`: **true** (purpose unclear — the only enabled flag)

**4 machines, 4 accounts (3 Linux, 1 Windows), all Max plan, v2.1.90 — every gate shows disabled. Context is still being stripped.**

## The SM Compact Threshold Problem

Even with the gate off, the remote config carries thresholds that are far more aggressive than the code defaults:

| Parameter | Remote Config | Code Default | Difference |
|-----------|--------------|-------------|------------|
| `minTokens` | **2,000** | 10,000 | **5x lower floor** |
| `maxTokens` | **20,000** | 40,000 | **2x lower ceiling** |
| `minTextBlockMessages` | 5 | — | — |

If this gate flips on via A/B test, sessions will retain 20K tokens max — a fraction of what users paying for 1M context expect.

## Why This Matters for Cache (Connection to Rate Limit Drain)

Anthropic's prompt caching works by matching exact token prefixes. If microcompact silently modifies or strips tool results mid-conversation:

1. Microcompact triggers silently (no notification)
2. Old tool results replaced with `[Old tool result content cleared]`
3. Conversation prefix changes → prompt cache prefix no longer matches
4. Next API call: **0% cache read** → full-price billing on entire context
5. Usage burns **5-10x** faster than expected

This chain explains several otherwise-puzzling observations:
- v2.1.90 fixed cache for some users but not others (depends on whether microcompact triggers)
- Old Docker-pinned versions that were never updated started draining ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — server-side flag change
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` doesn't help (microcompact bypasses it)
- The drain appears intermittent (GrowthBook A/B testing controls who gets hit)

## Open Questions

### 1. GrowthBook Runtime vs Disk Cache Divergence

The disk cache in `~/.claude.json` is a snapshot of the last GrowthBook evaluation. But GrowthBook evaluates features using per-session attributes (user ID, plan tier, session ID, timestamp, etc.) at runtime. If the CLI re-evaluates mid-session with different attributes — after crossing a token threshold, or based on session duration — the disk cache would never reflect it.

**Test:** Intercept GrowthBook SDK calls (or the remote eval API endpoint) during a session where stripping is actively happening, and compare runtime feature values against the disk cache.

### 2. Server-Side Content Clearing

When `[Old tool result content cleared]` appears, is it present in the raw API response from Anthropic, or does the CLI insert it locally? If the API response already has the content cleared, no client-side GrowthBook gate matters — the content never reaches the client.

**Test:** Run a transparent logging proxy (`ANTHROPIC_BASE_URL`) and capture the raw response body when stripping is observed. Check whether the tool result content is intact in the API response but cleared by the CLI, or already cleared server-side.

### 3. The Fourth Path

If runtime eval matches disk cache (all disabled) and the API response contains full content, there's a compaction path not gated by any of the three known GrowthBook flags. Candidates:
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
