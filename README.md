# Claude Code Cache Bug Analysis

Measured analysis of two cache bugs in Claude Code that cause **10-20x token inflation**, leading to rapid rate limit exhaustion on paid plans (Max 5/20).

## TL;DR

Two client-side cache bugs cause the Anthropic API server to miss cached conversation prefixes, forcing a full rebuild on every turn. This inflates token consumption by 10-20x, exhausting rate limits in minutes instead of hours. Workarounds using only official tools are documented below.

---

## Root Cause

Identified through community reverse engineering ([Reddit analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)):

### Bug 1 — Sentinel Replacement (standalone binary only)

**GitHub Issue:** [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524)

The standalone binary ships with a custom Bun fork that contains a `cch=00000` sentinel replacement mechanism. When conversation content includes certain internal strings, the sentinel in `messages` gets incorrectly substituted — breaking the cache prefix.

**Result:** Full cache rebuild on every API call.

**Scope:** Standalone binary only. The npm version (`npx @anthropic-ai/claude-code`) does not contain this logic.

### Bug 2 — Resume Cache Breakage (v2.1.69+)

**GitHub Issue:** [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629)

Starting from v2.1.69, `deferred_tools_delta` was introduced in the message structure. When resuming a session (`--resume`), the first message's structure doesn't match what the server cached — resulting in a complete cache miss.

**Impact:** On a 500K token conversation, a single resume costs ~$0.15 in quota.

### Combined Effect

When both bugs are active, the cache hit rate drops to near 0%. Every token on every turn is billed at full price — which is why rate limits are exhausted in minutes instead of hours.

---

## Measured Data

### Methodology

I set up a transparent local monitoring proxy using [`ANTHROPIC_BASE_URL`](https://docs.anthropic.com/en/docs/claude-code/settings#environment-variables) (official environment variable) to log `cache_creation_input_tokens` and `cache_read_input_tokens` from each API response.

This is a pass-through proxy that does not modify requests or responses — it only reads the usage metadata from API responses for logging. `ANTHROPIC_BASE_URL` is documented by Anthropic for proxy/gateway routing.

### Before Workarounds — Affected Sessions (v2.1.89)

Audited via session JSONL files using `cache_creation_input_tokens` / `cache_read_input_tokens`:

| Session | Turns | Cache Read Ratio | Status |
|---------|-------|-----------------|--------|
| Session A | 168 | **4.3%** | poor |
| Session B | 89 | **22.6%** | poor |
| Session C | 233 | **34.6%** | poor |

At 4.3% cache read ratio, nearly every token is billed at full price — roughly **20x the expected cost per turn**. These "poor" sessions were the primary cause of rate limit exhaustion.

### After Workarounds — Per-Request Proxy Log

| Request | Cache Creation | Cache Read | Read Ratio |
|---------|---------------|------------|------------|
| 1 (cold start) | 13,535 | 21,125 | 60.9% |
| 2 | 3,827 | 34,660 | 90.1% |
| 3 | 693 | 38,487 | 98.2% |
| 4 | 4,839 | 39,180 | 89.0% |
| 5 | 1,270 | 44,019 | 97.2% |
| 6 | 247 | 45,289 | 99.5% |
| 7 | 443 | 45,536 | 99.0% |
| 8 | 257 | 45,979 | 99.4% |
| 9 | 1,025 | 46,236 | 97.8% |
| 10 | 659 | 47,261 | 98.6% |
| 11 | 655 | 47,920 | 98.7% |
| 12 | 1,335 | 48,575 | 97.3% |
| 13 | 1,417 | 49,910 | 97.2% |
| 14 | 2,467 | 51,327 | 95.4% |
| 15 | 2,538 | 53,794 | **95.5%** |

After cold start warmup, cache read ratio stabilizes at **95-99%**. This is normal behavior — the server caches the conversation prefix and only bills the delta on each turn.

### Summary

| Metric | Before (affected) | After (workarounds) |
|--------|-------------------|---------------------|
| Cache read ratio | 4.3% - 34.6% | 89% - 99.5% |
| Effective token cost per turn | ~10-20x inflated | ~1x (normal) |
| Rate limit (Max 20) | 100% in ~70 min | Stable — 14% after extended session |

---

## Usage Precautions (Updated April 2, 2026)

Beyond the cache bugs, several Claude Code behaviors significantly accelerate token consumption. These apply regardless of whether the cache fix is in place.

### Behaviors to Avoid

| Behavior | Why | Measured Impact |
|----------|-----|-----------------|
| `--resume` | Replays **entire** conversation history as billable input tokens. Opaque thinking block signatures (base64) are included in the replay. | 500K+ tokens burned on a single resume of a long session ([#42260](https://github.com/anthropics/claude-code/issues/42260)) |
| `/dream` | Triggers background API calls that consume tokens without visible output | Silent drain, difficult to detect |
| `/insights` | Same as `/dream` — hidden background token consumption | Reported to cause "insane token usage" ([#40438](https://github.com/anthropics/claude-code/issues/40438)) |
| v2.1.89 (latest) | Cache prefix bug still present + terminal content rendering regression on Linux/IntelliJ | All token inflation issues persist + broken UI ([#42244](https://github.com/anthropics/claude-code/issues/42244)) |

### Behaviors to Use with Caution

| Behavior | Why | Measured Data |
|----------|-----|---------------|
| Sub-agents (Agent tool, Haiku) | Each Haiku sub-agent call creates a **fresh context** with **0% cache read**. No cache sharing with parent session. | **317K input tokens across 31 sub-agent calls** (measured via proxy) |
| Multiple terminals | Each terminal is an independent session with its own context. No shared quota pacing. | ~2x drain rate with 2 active terminals |
| Large CLAUDE.md / context files | Sent as input on **every single turn**. With broken cache, billed at full price each time. | 30KB CLAUDE.md = 30KB × N turns fully billed |
| Slash commands that rewrite files | Trigger large context rebuilds mid-session | 20-27% of session budget per invocation reported |
| Session start / compaction | `cache_creation` spikes are structural and unavoidable at these boundaries | Normal — budget for it |

### Server-Side Factor

Even with cache working perfectly (91-99% read ratio), multiple users report faster quota drain compared to 2-3 weeks ago. This suggests a **server-side change** in rate limit calculation (pool size, cost weighting, or both) — not fixable client-side. The precautions above help stretch whatever budget remains.

**Related:** Shared billing methods can share rate limit pools — if another account uses the same payment method, their usage counts against yours ([#41881](https://github.com/anthropics/claude-code/issues/41881)).

---

## Safe Workarounds

These use only official tools and releases — no binary modification required.

### 1. Use the npm version (avoids Bug 1)

```bash
npx @anthropic-ai/claude-code
```

The npm package does not contain the sentinel replacement logic that causes Bug 1. This is the simplest workaround.

### 2. Downgrade to v2.1.68 (avoids both bugs)

```bash
npm install -g @anthropic-ai/claude-code@2.1.68
```

v2.1.68 predates the `deferred_tools_delta` change (Bug 2). You lose recent features but avoid both cache regressions.

### 3. Avoid session resume — always start fresh

Start fresh sessions instead of using `--resume`. Bug 2 triggers specifically on session resume where the message structure mismatch causes a full cache miss. Use a structured CLAUDE.md to restore context instead.

### 4. Monitor your cache efficiency

Set up a local transparent proxy using `ANTHROPIC_BASE_URL` ([official env var](https://docs.anthropic.com/en/docs/claude-code/settings#environment-variables)) to log cache metrics from API responses. This lets you verify whether your sessions are healthy or affected.

Example: route through `http://localhost:8080` and parse `cache_creation_input_tokens` / `cache_read_input_tokens` from the response `usage` object.

### 5. Keep conversations short, context lean

Shorter conversations = smaller cache prefix = less damage when cache misses occur. If you notice rapid usage consumption, end the session and start fresh. Keep CLAUDE.md small — every byte is resent every turn.

### 6. Minimize sub-agents and parallel terminals

Use sub-agents only when essential (each one is a fresh uncached context). Run one terminal at a time to avoid parallel quota drain.

---

## How to Check If You're Affected

The session JSONL files in `~/.claude/projects/` contain usage data for each turn. Look for the `cache_creation_input_tokens` and `cache_read_input_tokens` fields:

- **Healthy session:** `cache_read` >> `cache_creation` (read ratio > 80%)
- **Affected session:** `cache_creation` >> `cache_read` (read ratio < 40%)

If most of your sessions show low read ratios, you are likely affected by one or both bugs.

---

## Related Issues

### Root Cause Bugs
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — Conversation history invalidated (Bug 1: sentinel)
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume cache regression (Bug 2: deferred_tools_delta)
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= billing hash substitution

### Token Inflation Mechanisms
- [#41663](https://github.com/anthropics/claude-code/issues/41663) — Prompt cache causes excessive token consumption
- [#41607](https://github.com/anthropics/claude-code/issues/41607) — Duplicate compaction subagents (5x identical work)
- [#41767](https://github.com/anthropics/claude-code/issues/41767) — Auto-compact loops in v2.1.89
- [#41750](https://github.com/anthropics/claude-code/issues/41750) — Context management fires on every turn
- [#42260](https://github.com/anthropics/claude-code/issues/42260) — Resume replays thinking signatures as input tokens
- [#42256](https://github.com/anthropics/claude-code/issues/42256) — Read tool re-sends oversized images every message
- [#42290](https://github.com/anthropics/claude-code/issues/42290) — /export truncates + /resume delivers incomplete context

### Regressions (v2.1.89)
- [#42244](https://github.com/anthropics/claude-code/issues/42244) — Terminal content disappearing (Linux/IntelliJ)
- [#42240](https://github.com/anthropics/claude-code/issues/42240) — Slash command autocomplete broken

### Rate Limit Reports (major threads)
- [#16157](https://github.com/anthropics/claude-code/issues/16157) — Instantly hitting usage limits (1400+ comments)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — Session limits exhausted abnormally fast (300+ comments)
- [#41930](https://github.com/anthropics/claude-code/issues/41930) — Critical: Widespread abnormal usage drain — multiple root causes
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — My original report (Max 20, 100% in ~70 min)

### Community Engagement

As of April 2, 2026, I've posted root cause analysis + precautions across **96 comments on 80+ issues**. Anthropic official response count: **zero** (2+ months of silence across all rate-limit issues).

## Community References

- [Reddit: Reverse engineering analysis of Claude Code cache bugs](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — Community-developed cache patch + test toolkit (patches both Bug 1 and Bug 2)
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy-based Claude Code traffic analysis
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Transparent proxy for Claude Code

---

## Environment

- **Plan:** Max 20 ($200/mo)
- **OS:** Linux (Ubuntu)
- **Versions tested:** v2.1.89 (affected), v2.1.81 (with workarounds), v2.1.68 (not affected)
- **Date:** April 2, 2026 (updated)

---

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
