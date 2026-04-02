# Claude Code Cache Bug Analysis

Measured analysis of cache bugs in Claude Code that caused **10-20x token inflation** on paid plans (Max 5/20). Includes controlled benchmarks comparing npm vs standalone binary installations.

> **Last updated:** April 2, 2026

---

## Current Status: v2.1.90 — Stabilized

**v2.1.90 has largely resolved the cache issue.** Both npm and standalone binary installations now achieve **86%+ overall cache read ratio** and **95-99% in stable sessions**. This is a dramatic improvement from v2.1.89, where standalone binary sessions sustained 4-34% cache read.

| Version | Installation | Cache Read (stable) | Sub-agent Cold Start | Verdict |
|---------|-------------|--------------------|--------------------|---------|
| **v2.1.90** | **npm (Node.js)** | **95-99.8%** | **79-87%** | **Optimal** |
| **v2.1.90** | **Standalone (ELF)** | **95-99.7%** | **47-67%** (recovers to 94-99%) | **Good** |
| v2.1.89 | Standalone (ELF) | 90-99% | **4-17%** (never recovers) | **Avoid** |
| v2.1.68 | npm | Normal | Normal | Safe but outdated |

**If you're on v2.1.89 or earlier standalone, update to v2.1.90 immediately.** The single most impactful action you can take.

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

Two client-side cache bugs, identified through community reverse engineering ([Reddit analysis](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)):

### Bug 1 — Sentinel Replacement (standalone binary only)

**GitHub Issue:** [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524)

The standalone binary's embedded Bun fork contains a `cch=00000` sentinel replacement mechanism. Under certain conditions, the sentinel in `messages` gets incorrectly substituted — breaking the cache prefix and forcing a full rebuild.

- **v2.1.89:** Catastrophic — cache read drops to 4-17%, never recovers
- **v2.1.90:** Partially mitigated — cold start still affected (47-67%), but recovers to 94-99% after warming
- **npm:** Not affected — the JavaScript bundle does not contain this logic

### Bug 2 — Resume Cache Breakage (v2.1.69+)

**GitHub Issue:** [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629)

`deferred_tools_delta` (introduced in v2.1.69) causes the first message's structure on `--resume` to not match the server's cached version — resulting in a complete cache miss. On a 500K token conversation, a single resume costs ~$0.15 in quota.

**Applies to both npm and standalone.** Avoid `--resume` entirely.

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

### Server-Side Factor (Unresolved)

Even with cache working perfectly (91-99%), multiple users report faster quota drain compared to 2-3 weeks ago. This suggests a **server-side change** in rate limit calculation — not fixable client-side.

**Org-level quota sharing:** Accounts on the same billing method share rate limit pools ([#41881](https://github.com/anthropics/claude-code/issues/41881)). Source code confirms `passesEligibilityCache` and `overageCreditGrantCache` are keyed by `organizationUuid`, not `accountUuid`.

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
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — Conversation history invalidated (Bug 1: sentinel)
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume cache regression (Bug 2: deferred_tools_delta)
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= billing hash substitution

### Token Inflation Mechanisms
- [#41663](https://github.com/anthropics/claude-code/issues/41663) — Prompt cache causes excessive token consumption
- [#41607](https://github.com/anthropics/claude-code/issues/41607) — Duplicate compaction subagents (5x identical work)
- [#41767](https://github.com/anthropics/claude-code/issues/41767) — Auto-compact loops in v2.1.89
- [#42260](https://github.com/anthropics/claude-code/issues/42260) — Resume replays thinking signatures as input tokens
- [#42256](https://github.com/anthropics/claude-code/issues/42256) — Read tool re-sends oversized images every message

### Rate Limit Reports (major threads)
- [#16157](https://github.com/anthropics/claude-code/issues/16157) — Instantly hitting usage limits (1400+ comments)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — Session limits exhausted abnormally fast (300+ comments)
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — My original report (Max 20, 100% in ~70 min)

### Community Engagement

As of April 2, 2026: **100+ comments on 80 unique issues**. Anthropic official response count: **zero** (2+ months of silence).

## Community References

- [Reddit: Reverse engineering analysis of Claude Code cache bugs](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — Community cache patch + test toolkit
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy-based Claude Code traffic analysis
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Transparent proxy for Claude Code

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
- **Date:** April 2, 2026

---

*This analysis is based on community research and personal measurement. It is not endorsed by Anthropic. All workarounds use only official tools and documented features.*
