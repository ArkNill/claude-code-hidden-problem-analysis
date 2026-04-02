# npm vs Standalone Binary — Cache Efficiency Benchmark

> Controlled comparison on the **same machine, same version (v2.1.90), same proxy**, differing only in installation method.

**Date:** April 2, 2026

---

## Environment

| Parameter | Value |
|-----------|-------|
| **Machine** | HP ZBook Ultra G1a 14" (Linux, x86-64) |
| **Plan** | Max 20 ($200/mo) |
| **Version** | v2.1.90 (both npm and standalone) |
| **Proxy** | cc-relay (transparent pass-through, logging only) |
| **Monitoring** | `cc-relay watch` + `cc-relay stats` + `cc-relay recent` |

### Installation Paths

| Alias | Path | Binary Type |
|-------|------|-------------|
| `claude-npm` | `~/.nvm/versions/node/v22.22.1/bin/claude` → `cli.js` | **Node.js** (symlink to npm module) |
| `claude` | `~/.local/bin/claude` → `~/.local/share/claude/versions/2.1.90` | **ELF 64-bit** (standalone binary) |

Both routed through `ANTHROPIC_BASE_URL=http://localhost:8080` (cc-relay proxy). Proxy source code confirmed **zero request/response modification** — only usage logging.

---

## Test Design

### npm Session — Scenarios

| # | Scenario | Purpose |
|---|----------|---------|
| 1 | Simple greeting ("현재 시간 알려줘") | Baseline — CLAUDE.md + MEMORY.md cold load |
| 2 | Single file read + summarize (markgrab README) | Read tool 1x, minimal context |
| 3 | Multi-file code exploration (feedkit entry point + 3 classes) | Glob + Read multiple, context accumulation |
| 4 | Follow-up on previous context (feedkit error handling) | Cache reuse test — prior turn context |
| 5 | Parallel comparison (markgrab vs feedkit dependencies) | Sub-agent potential, multi-project |
| 6 | Full project status read (12+ memory files) | Heavy read, broad context |
| 7 | **79 analysis reports** — 3 parallel sub-agents | Maximum load: 3 agents, 79 files, full summarization |

### Standalone Binary Session — Scenarios

| # | Scenario | Purpose |
|---|----------|---------|
| A | Large codebase exploration (forge pipeline) | New context forced — not in prior session |
| B | Multi-project architecture comparison (markgrab vs browsegrab) | Multi-file read, context growth |
| C | 5-turn sequential follow-up (feedkit deep dive) | Cache reuse across consecutive turns |
| D | 3-project parallel sub-agents (embgrep vs diffgrab vs snapgrab) | Maximum sub-agent load |

---

## Results

### npm v2.1.90 — Session Summary

| Metric | Value |
|--------|-------|
| **Usage consumed** | 28% → 35% (**7% for entire session**) |
| **Turns** | 1,753 (including sub-agent turns) |
| **Overall cache read ratio** | **86.4%** |
| **Status** | healthy |

#### npm — Per-Request Cache Read % (representative sample)

| Time | Input | Output | Cache Create | Cache Read | **Read %** | Context |
|------|-------|--------|-------------|------------|-----------|---------|
| 13:12 | 0 | 0 | 0 | 0 | 0.0% | Session init |
| 13:12 | 1 | 480 | 24,401 | 42,469 | **63.5%** | Cold start (CLAUDE.md load) |
| 13:12 | 14,368 | 162 | 11,910 | 49,029 | **80.5%** | First real query |
| 13:12 | 3,913 | 485 | 3,350 | 66,870 | **95.2%** | Stable |
| 13:12 | 460 | 434 | 10,651 | 70,220 | **86.8%** | Sub-agent init |
| 13:14 | 4,707 | 4,009 | 15,393 | 60,939 | **79.8%** | Sub-agent work |
| 13:15 | 1 | 6,872 | 6,044 | 95,449 | **94.0%** | Heavy read |
| 13:16 | 1 | 7,140 | 16,195 | 80,871 | **83.3%** | 79-report agent |
| 13:17 | 4,112 | 4,150 | 15,355 | 103,1xx | **87.0%** | Agent merge |
| 13:17 | 3 | 174 | 8,387 | 118,5xx | **93.4%** | Post-agent |
| 13:18 | 3 | 567 | 253 | 126,9xx | **99.8%** | Stable |
| 13:20 | 3 | 660 | 612 | 127,1xx | **99.5%** | Stable |
| 13:20 | 3 | 513 | 709 | 127,7xx | **99.4%** | Stable |
| 13:21 | 1 | 341 | 769 | 128,5xx | **99.4%** | Stable |
| 13:21 | 1 | 208 | 3,529 | 129,6xx | **97.4%** | Stable |
| 13:21 | 1 | 65 | 270 | 133,1xx | **99.8%** | Stable |
| 13:21 | 3 | 232 | 8,831 | 61,148 | **87.4%** | Sub-agent |
| 13:21 | 1 | 119 | 736 | 69,979 | **99.0%** | Stable |
| 13:21 | 1 | 92 | 211 | 70,715 | **99.7%** | Stable |

**npm sub-agent cold start: 79-87% cache read.**

---

### Standalone Binary v2.1.90 — Per-Request Cache Read %

| Time | Input | Output | Cache Create | Cache Read | **Read %** | Context |
|------|-------|--------|-------------|------------|-----------|---------|
| 13:29 | 0 | 0 | 0 | 0 | 0.0% | Session init |
| 13:29 | 0 | 0 | 0 | 0 | 0.0% | Session init |
| 13:29 | 3 | 1,815 | 9,576 | 8,114 | **45.9%** | Sub-agent init |
| 13:29 | 3 | 2,102 | 9,716 | 8,114 | **45.5%** | Sub-agent init |
| 13:29 | 6,499 | 1,076 | 18,699 | 9,753 | **34.3%** | Sub-agent work |
| 13:29 | 3 | 195 | 288 | 67,792 | **99.6%** | Main session stable |
| 13:29 | 40 | 197 | 2,969 | 68,080 | **95.8%** | Stable |
| 13:29 | 449 | 171 | 7,601 | 71,049 | **90.3%** | Stable |
| 13:29 | 1 | 237 | 1,287 | 78,650 | **98.4%** | Stable |
| 13:29 | 2,629 | 317 | 12,454 | 79,937 | **86.5%** | Context growth |
| 13:29 | 3,316 | 2,000 | 47,385 | 9,753 | **17.1%** | Sub-agent heavy |
| 13:30 | 2,266 | 2,658 | 56,857 | 9,753 | **14.6%** | Sub-agent heavy |
| 13:30 | 1 | 2,631 | 48,030 | 9,753 | **16.9%** | Sub-agent heavy |
| 13:30 | 1 | 2,350 | 7,894 | 92,391 | **92.1%** | Recovery |
| 13:30 | 3 | 452 | 889 | 74,716 | **98.8%** | Stable |
| 13:30 | 1 | 97 | 472 | 75,605 | **99.4%** | Stable |
| 13:30 | 1 | 100 | 115 | 76,077 | **99.8%** | Stable |
| 13:30 | 1 | 73 | 910 | 76,077 | **98.8%** | Stable |
| 13:30 | 1 | 412 | 3,905 | 76,192 | **95.1%** | Stable |
| 13:30 | 1 | 97 | 807 | 80,097 | **99.0%** | Stable |

#### Standalone — Scenarios C & D (continued, 13:35~13:36)

| Time | Input | Output | Cache Create | Cache Read | **Read %** | Context |
|------|-------|--------|-------------|------------|-----------|---------|
| 13:35 | 3 | 212 | 9,068 | 8,114 | **47.2%** | Sub-agent D init |
| 13:35 | 3 | 212 | 9,068 | 8,114 | **47.2%** | Sub-agent D init |
| 13:35 | 3 | 155 | 1,741 | 143,098 | **98.8%** | Main stable |
| 13:35 | 1 | 275 | 6,407 | 17,182 | **72.8%** | Sub-agent work |
| 13:35 | 1 | 374 | 6,757 | 17,182 | **71.8%** | Sub-agent work |
| 13:35 | 1 | 434 | 2,959 | 17,182 | **85.3%** | Sub-agent warming |
| 13:35 | 1 | 110 | 3,045 | 23,589 | **88.6%** | Sub-agent warming |
| 13:35 | 1 | 108 | 11,676 | 23,939 | **67.2%** | New context (forge) |
| 13:35 | 1 | 255 | 225 | 26,634 | **99.2%** | Stable |
| 13:35 | 1 | 345 | 4,359 | 94,645 | **95.6%** | Stable |
| 13:35 | 1 | 173 | 123 | 35,615 | **99.7%** | Stable |
| 13:35 | 1 | 73 | 284 | 35,738 | **99.2%** | Stable |
| 13:35 | 1 | 225 | 9,627 | 26,859 | **73.6%** | Sub-agent new context |
| 13:35 | 1 | 245 | 403 | 99,004 | **99.6%** | Main stable |
| 13:35 | 1 | 560 | 284 | 99,407 | **99.7%** | Main stable |
| 13:35 | 1 | 355 | 710 | 99,691 | **99.3%** | Main stable |
| 13:35 | 1 | 177 | 394 | 100,401 | **99.6%** | Main stable |
| 13:35 | 1 | 2,884 | 8,710 | 20,141 | **69.8%** | Sub-agent heavy |
| 13:36 | 3 | 36 | 3,299 | 144,839 | **97.8%** | Main stable |
| 13:36 | 3 | 258 | 272 | 100,795 | **99.7%** | Main stable |

**Standalone sub-agent cache read (complete A-D): 14.6-47.2% cold start → 67-88% warming → 95-99.7% stable.**

#### Standalone — Scenarios C & D continued (13:36~13:38, stable phase)

| Time | Input | Output | Cache Create | Cache Read | **Read %** | Context |
|------|-------|--------|-------------|------------|-----------|---------|
| 13:36 | 3 | 36 | 3,299 | 144,839 | **97.8%** | Main stable |
| 13:36 | 3 | 258 | 272 | 100,795 | **99.7%** | Main stable |
| 13:36 | 846 | 2,628 | 312 | 36,486 | **99.2%** | Sub-agent work |
| 13:36 | 1 | 2,863 | 2,012 | 35,738 | **94.7%** | Sub-agent work |
| 13:36 | 3 | 22 | 2,953 | 148,138 | **98.0%** | Main stable |
| 13:36 | 1 | 630 | 2,027 | 101,067 | **98.0%** | Stable |
| 13:36 | 1 | 282 | 651 | 103,094 | **99.4%** | Stable |
| 13:36 | 1 | 343 | 2,142 | 103,745 | **98.0%** | Stable |
| 13:36 | 3 | 1,828 | 6,117 | 148,138 | **96.0%** | Main + agent merge |
| 13:36 | 1 | 1,091 | 402 | 105,887 | **99.6%** | Stable |
| 13:37 | 1 | 1,522 | 1,242 | 106,289 | **98.8%** | Stable |
| 13:37 | 1 | 634 | 1,562 | 107,531 | **98.6%** | Stable |
| 13:37 | 1 | 440 | 674 | 109,093 | **99.4%** | Stable |
| 13:37 | 1 | 344 | 480 | 109,767 | **99.6%** | Stable |
| 13:37 | 1 | 634 | 383 | 110,247 | **99.7%** | Stable |
| 13:37 | 1 | 291 | 673 | 110,630 | **99.4%** | Stable |
| 13:38 | 1 | 291 | 330 | 111,303 | **99.7%** | Stable |
| 13:38 | 3 | 433 | 16,412 | 111,633 | **87.2%** | New context burst |

**After warming: standalone achieves 94-99.7% consistently — virtually identical to npm.**

---

## Comparison

### Overall

| Metric | npm (Node.js) | Standalone (ELF) |
|--------|--------------|-----------------|
| Version | v2.1.90 | v2.1.90 |
| Scenarios completed | 1-7 (incl. 79-report parallel read) | A-D (forge, browsegrab, feedkit 5-turn, 3-project parallel) |
| **Usage consumed** | 28% → 35% (**7%**) | 35% → 40% (**5%**) |
| Overall cache read % | **86.4%** | **86.2%** |
| Sub-agent cold start cache % | **79-87%** | **47-67%** |
| Sub-agent warmed cache % | 87-94% | **94-99%** |
| Stable session cache % | 95-99.8% | **95-99.7%** |

### Sub-Agent Cache Behavior (Key Difference)

Both installations are identical v2.1.90, yet sub-agent context initialization shows different cache efficiency. The gap narrows as sub-agents warm up:

| Sub-Agent Phase | npm Read % | Standalone Read % | Delta |
|-----------------|-----------|-------------------|-------|
| Cold start (first 1-2 requests) | 63.5-80% | **14.6-47.2%** | **-16 to -66pp** |
| Warming (3-5 requests) | 79-87% | 67-88% | -0 to -20pp |
| Warmed (5+ requests) | 87-94% | **94-99%** | **~0pp or better** |
| Stable main session | 97-99.8% | **95-99.7%** | **~0pp** |

### Key Finding: v2.1.90 Standalone Dramatically Improved vs v2.1.89

| Metric | v2.1.89 Standalone | v2.1.90 Standalone | v2.1.90 npm |
|--------|-------------------|-------------------|-------------|
| Sub-agent cold start | **4-34%** | **14-47%** | **63-87%** |
| Sub-agent warmed | N/A (never recovered) | **94-99%** | **87-94%** |
| Stable session | 90-99% | **95-99.7%** | 95-99.8% |
| Overall efficiency | Poor — rapid drain | **Good** | **Optimal** |
| Usage for full test suite | 100% in ~70 min | **5% for scenarios A-D** | **7% for scenarios 1-7** |

**v2.1.90 standalone is a major improvement.** The Sentinel bug's impact is now limited to sub-agent cold starts (first 1-2 requests). Once warmed, standalone achieves **94-99% cache read — matching or exceeding npm's warming phase.** The critical difference from v2.1.89 is that v2.1.90 **recovers** from initial cache misses, while v2.1.89 never did.

### Token Cost Implication (Updated with Full Data)

Total usage across both sessions: **28% → 40% = 12% of Max 20 plan** for:
- 7 npm scenarios (including 79 analysis reports via 3 parallel agents)
- 4 standalone scenarios (forge exploration, architecture comparison, 5-turn sequential, 3-project parallel agents)

| Metric | npm | Standalone (v2.1.90) | Standalone (v2.1.89) |
|--------|-----|---------------------|---------------------|
| Sub-agent cold start cost | Low | Moderate (2-3x on first 1-2 requests) | Severe (10-20x sustained) |
| Sub-agent warmed cost | ~1x | **~1x** | N/A (never warms) |
| Effective session cost | 1x | **~1.1-1.3x** | **~3-4x** |
| Practical impact | Optimal | **Acceptable for all workflows** | Avoid |

v2.1.90 standalone's overhead is now marginal — limited to the first 1-2 sub-agent requests before cache warms up. For most real-world usage, the difference between npm and standalone v2.1.90 is negligible.

### Operational Note: Parallel Agents vs Parallel Terminals

This benchmark was conducted with **parallel agents within a single terminal** (recommended). This is different from running multiple terminal sessions simultaneously:

| Pattern | Cache Behavior | Recommendation |
|---------|---------------|----------------|
| Single terminal, parallel agents | Agents share billing context, main session cache intact | **Safe** |
| Multiple terminals simultaneously | Each terminal = independent session, no cache sharing, parallel quota drain | **Use with caution** |

Parallel agents within one session benefit from the main session's warmed cache. Multiple terminals each start cold and compete for the same rate limit pool.

---

## Why the Difference?

### Sentinel Bug (Bug 1) — Partially Mitigated in v2.1.90, Not Fully Resolved

The standalone binary ships as an ELF executable built with a custom Bun fork. This fork contains a `cch=00000` sentinel replacement mechanism that can corrupt cache prefixes under certain conditions.

**Evidence from this benchmark:**
- Same version (v2.1.90), same machine, same proxy, same API key
- Only variable: binary type (ELF vs Node.js)
- Main session cache is **nearly identical** (~95-99%) — both installations perform equally in stable conversation
- Sub-agent cold starts still show a gap (47% vs 80%), but v2.1.90 is significantly better than v2.1.89 (47% vs 14%)
- The gap closes as sub-agents warm up (72-88% after a few requests)

**Interpretation:** Anthropic appears to have partially addressed the Sentinel bug in v2.1.90. The cache prefix corruption is less severe — it no longer produces catastrophic 4-17% read ratios on every sub-agent turn. However, fresh context initialization still performs ~20-30pp worse than the npm version, suggesting the underlying mechanism is still present but attenuated.

### Proxy Ruled Out

cc-relay source code audited: pure pass-through. No request/response body modification. Only reads `usage` metadata from responses for logging. The cache behavior difference is entirely between the Claude Code binary and the Anthropic API server.

---

## Recommendation

| Use Case | v2.1.90 npm | v2.1.90 Standalone | v2.1.89 Standalone |
|----------|------------|-------------------|-------------------|
| General coding (few sub-agents) | **Best** | **Good** | Avoid |
| Heavy sub-agent workloads | **Best** | **Good** (cold start overhead only) | Avoid (3-4x sustained) |
| Parallel agent research/analysis | **Best** | **Good** (agents warm up quickly) | Avoid |
| Maximum token efficiency | **Best** | **Close to optimal** | Avoid |

**Bottom line:** v2.1.90 standalone is a major improvement — the gap with npm is now marginal (limited to sub-agent cold starts that warm up within 1-2 requests). Both v2.1.90 installations consumed comparable quota (7% npm vs 5% standalone for equivalent workloads). **The real enemy is v2.1.89 and earlier**, where cache never recovered. Users on older standalone versions should update to v2.1.90 immediately regardless of installation method.

### Disable Auto-Update First

**Critical:** Before testing, disable auto-update to prevent version drift during benchmarking — and keep it off until Anthropic confirms a full cache fix. A surprise update could reintroduce regressions.

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

This applies to all Claude Code invocations (standalone, npm, patched) since they all read `~/.claude/settings.json`. npm installs have no auto-update mechanism regardless, but setting this ensures standalone binary stays pinned.

### Installation

```bash
# npm global install
npm install -g @anthropic-ai/claude-code

# Verify it's Node.js based
file $(which claude)
# Should show: symbolic link to .../cli.js (NOT ELF executable)

# If standalone binary takes PATH priority, use alias:
alias claude-npm="~/.nvm/versions/node/v22.22.1/bin/claude"
```

### Monitoring

```bash
# Set up transparent proxy for cache monitoring
# Use ANTHROPIC_BASE_URL (official env var) to route through your proxy
ANTHROPIC_BASE_URL=http://localhost:8080 claude
```

---

## Reproduction

To reproduce this benchmark:

1. Install both npm and standalone versions (they coexist at different paths)
2. Set up a transparent logging proxy using `ANTHROPIC_BASE_URL`
3. Run identical workloads through each installation
4. Compare `cache_creation_input_tokens` vs `cache_read_input_tokens` in proxy logs
5. Focus on **sub-agent requests** — that's where the difference is most dramatic

The key insight: **stable main-session cache looks similar between both installations**. The bug primarily manifests during **fresh context creation** (sub-agents, new sessions, post-compaction). If you only test single-session chat, you may miss the difference entirely.

---

*All data collected via cc-relay transparent proxy on April 2, 2026. No request/response modification. Raw proxy logs available on request.*
