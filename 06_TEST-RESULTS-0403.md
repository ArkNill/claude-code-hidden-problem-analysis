> **🇰🇷 [한국어 버전](ko/06_TEST-RESULTS-0403.md)**

# Test Results — April 3, 2026

> Consolidated test results from the integrated testing session.
> Machine: ZBook Ultra G1a (Linux x86-64), Plan: Max 20 ($200/mo)
> Proxy: cc-relay v2 (budget detection + session ID support)

---

## Test Environment

| Component | Version | Path |
|-----------|---------|------|
| npm (Node.js) | v2.1.91 | `~/.nvm/versions/node/v22.22.1/bin/claude` |
| Standalone (ELF) | v2.1.91 | `~/.local/bin/claude` |
| Proxy | cc-relay v2 | `~/GitHub/cc-relay/` on `:8080` |
| DB | SQLite | `~/.cc-relay/usage.db` |

---

## Bug Verification Results

### B3: Synthetic Rate Limiter — CONFIRMED (151 entries)

**Method:** Searched all existing JSONL files for `<synthetic>` model entries.

**Result:**
- **151 `<synthetic>` entries** across **65 session files**
- 25 in `acompact` sub-agents, 126 in regular sessions
- Signature: `model=<synthetic> input=0 output=0 stop=stop_sequence`
- Text: `"No response requested."`

**Interpretation:** Client-side rate limiter fires WITHOUT calling the API. Confirmed on our setup (v2.1.90). Not a rare edge case — 65 sessions affected.

---

### B4: Microcompact — CONFIRMED (327 events total, pattern analyzed)

**Method:** cc-relay `_scan_microcompact()` detects `[Old tool result content cleared]` in outgoing request bodies.

**Results (v2.1.90, from existing data):**

| Session | Events | Cleared Indices | Msg Range | Tool Results |
|---------|--------|-----------------|-----------|--------------|
| Session 1 (02:29-02:54) | 67 | [20,22,38,40,44,46,162,166,172,174,206] | 121→251 | 53→112 |
| Session 2 (09:23-09:47) | 71 | [2,4,6,8,10,12,58,60,68,70,74,76] | 59→135 | 38→81 |

**Key Findings:**

1. **All cleared indices are even-numbered** → targets tool_use/tool_result pairs specifically
2. **Indices expand over time** — starts with early messages, progressively includes newer ones
3. **Main session cache NOT affected** — ratio stays 99%+ during clearing (stable substitution preserves prefix)
4. **Sub-agent cold starts DO show cache drops** — 0%, 39%, 82% observed at clearing moments
5. **All GrowthBook gates disabled** yet clearing occurs → indicates a compaction path independent of the three documented GrowthBook gates

**v2.1.91 Verification:**
- 14 microcompact events in v2.1.91 test sessions
- Same index pattern `[58, 60, 68, 70, 74, 76]`
- **No change from v2.1.90** — v2.1.91 does not fix microcompact

**Cache Impact:**

| Context | During Clearing | Without Clearing |
|---------|----------------|-----------------|
| Main session | **99%+** (no impact) | 99%+ |
| Sub-agent cold start | **0-39%** | 47-87% |
| Sub-agent warmed | **94-99%** | 94-99% |

**Revised Assessment:** Microcompact does NOT cause cache invalidation in main sessions. The real harm is **context quality degradation** — the agent loses access to earlier tool results and cannot accurately quote or reference them.

---

### B5: Tool Result Budget Enforcement — CONFIRMED (active, 200K cap)

**Method:** cc-relay `_scan_budget_enforcement()` measures tool result sizes in outgoing requests.

**GrowthBook Flags (confirmed active):**
```
tengu_hawthorn_window:       200,000 (aggregate cap)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: True
```

**Results:**
- Budget exceeded detected at **242,094 chars** (> 200K cap)
- **261 budget events** with `suspiciously_small` marker
- Tool results reduced to **1-41 chars** (originally thousands+)
- Examples: `msg[32] 1 char`, `msg[42] 2 chars`, `msg[172] 22 chars` (x33 occurrences)

**v2.1.91 Verification:**
- 71 budget events in v2.1.91 test session
- Same patterns — `maxResultSizeChars` override is MCP-only, built-in tools unaffected

**Impact:** Paid-for 1M context effectively has a **200K tool result ceiling** for built-in tools. Beyond this, older results are silently truncated.

---

### B8: JSONL Log Duplication — CONFIRMED (2.87x inflation)

**Method:** `tools/jsonl_analyzer.py` classifies PRELIM vs FINAL entries.

**Results across sessions:**

| Session | Size | PRELIM | FINAL | Ratio | Inflation |
|---------|------|--------|-------|-------|-----------|
| Current main | 1115KB | 79 | 82 | 0.96x | 2.87x |
| Previous session | 118KB | 16 | 6 | 2.67x | — |
| Sub-agent | 326KB | 39 | 20 | 1.95x | — |
| Sub-agent | 186KB | 12 | 7 | 1.71x | — |

**Key:** The "inflation" factor measures total input tokens (all entries) vs FINAL-only tokens = **2.87x** for current session. Sub-agents consistently show higher PRELIM ratios.

**Open Question:** Does the server-side rate limiter count PRELIM entries? If yes, thinking sessions are charged 2-3x more against the rate limit.

---

### B2: Resume Cache — NOT RE-TESTED (v2.1.90+91 fixes acknowledged)

v2.1.90 changelog explicitly fixes `--resume` cache miss regression (since v2.1.69).
v2.1.91 adds transcript chain break fix.

**Indirect evidence from existing data:**
- 45 requests after 5+ min idle gap analyzed
- Average cache_creation% after >5min gap: **18.2%** (vs 6.7% for <30s gap)
- Most 5-min gaps showed **96%+** cache read — TTL appears longer than 5 min in practice

---

### Cache TTL (Layer 0, from Luong NGUYEN analysis)

**External finding:** 13-hour idle → 350K token cache rebuild = 9% session budget.
**Our data:** 5-26 min idle → **96%+ cache maintained**. TTL expiration requires much longer gaps.

**Assessment:** Cache TTL is a design choice, not a bug. Impacts long-idle scenarios (hours), not normal work patterns.

---

## v2.1.90 vs v2.1.91 Comparison

| Metric | v2.1.90 | v2.1.91 |
|--------|---------|---------|
| Cache warming (cold→97%) | ~5 requests | ~3 requests |
| Stable session | 95-99% | 97-99% |
| Microcompact | Active | **Active (unchanged)** |
| Budget enforcement | Active | **Active (MCP override only)** |
| Session ID header | Not captured | **Working** |
| Synthetic RL (B3) | Present | Not re-tested (no fix in changelog) |
| JSONL duplication (B8) | 2.87x | Not re-tested (no fix in changelog) |

---

## v2.1.91 npm vs Standalone (Head-to-Head)

**Setup:** Same prompt, same proxy, same machine, same version (v2.1.91), run in parallel.

| Metric | npm (Node.js) | Standalone (ELF) |
|--------|--------------|-----------------|
| Requests | 7 | 7 |
| Duration | 60s | 69s |
| **Overall cache** | **82.8%** | **82.8%** |
| Cold start | 84.7% | 84.7% |
| Stable range | 88-97% | 77-89% |
| Max body size | 190KB | 210KB |
| Microcompact events | 4 | 4 |
| Budget events | 0 | 0 |

**Key finding: v2.1.91 closes the Sentinel gap.** On v2.1.90, standalone cold start was 47-67% vs npm's 79-87%. On v2.1.91, both start at **identical 84.7%**. The `stripAnsi` Bun optimization and other v2.1.91 changes appear to have neutralized the Sentinel issue.

### Per-Request Data: npm (0523f810)

| Time | Cache Create | Cache Read | Read % | Body KB | Note |
|------|-------------|------------|--------|---------|------|
| 10:14:36 | 4,628 | 25,576 | **84.7%** | 109 | cold start |
| 10:14:37 | 0 | 0 | **0.0%** | 14 | sub-agent init |
| 10:14:49 | 13,970 | 25,576 | **64.7%** | 139 | context growth |
| 10:14:56 | 1,055 | 39,546 | **97.4%** | 143 | stabilizing |
| 10:14:56 | 0 | 0 | **0.0%** | 13 | sub-agent init |
| 10:15:04 | 5,376 | 40,601 | **88.3%** | 160 | warming |
| 10:15:36 | 11,903 | 45,977 | **79.4%** | 190 | new context burst |

### Per-Request Data: Standalone (e437f406)

| Time | Cache Create | Cache Read | Read % | Body KB | Note |
|------|-------------|------------|--------|---------|------|
| 10:14:35 | 4,628 | 25,576 | **84.7%** | 109 | cold start (identical to npm) |
| 10:14:35 | 0 | 0 | **0.0%** | 14 | sub-agent init |
| 10:14:45 | 5,625 | 30,204 | **84.3%** | 137 | context growth |
| 10:14:53 | 4,318 | 35,829 | **89.2%** | 142 | warming |
| 10:15:10 | 11,807 | 40,147 | **77.3%** | 173 | new context burst |
| 10:15:10 | 0 | 0 | **0.0%** | 13 | sub-agent init |
| 10:15:44 | 11,761 | 51,954 | **81.5%** | 210 | heavy read |

### Per-Request Data: Extended Heavy Session (40e5bd8b)

17 files read + 4 large grep/find commands. This session was designed to push past the 200K budget threshold.

| Time | Cache Create | Cache Read | Read % | Body KB | Note |
|------|-------------|------------|--------|---------|------|
| 10:07:26 | 11,762 | 18,727 | **61.4%** | 110 | cold start |
| 10:07:26 | 0 | 0 | **0.0%** | 14 | sub-agent |
| 10:07:36 | 5,806 | 30,489 | **84.0%** | 147 | warming |
| 10:07:45 | 6,093 | 36,295 | **85.6%** | 149 | warming |
| 10:07:54 | 1,060 | 42,388 | **97.6%** | 154 | **stable** |
| 10:08:00 | 1,449 | 43,448 | **96.8%** | 158 | stable |
| 10:08:15 | 747 | 44,897 | **98.4%** | 160 | stable |
| 10:08:22 | 1,469 | 45,644 | **96.9%** | 166 | stable |
| 10:08:38 | 702 | 47,113 | **98.5%** | 168 | stable |
| 10:08:38 | 0 | 0 | **0.0%** | 13 | sub-agent |
| 10:08:56 | 6,018 | 47,815 | **88.8%** | 224 | heavy context growth |
| 10:08:56 | 0 | 0 | **0.0%** | 18 | sub-agent |
| 10:09:10 | 48,218 | 25,576 | **34.7%** | 277 | **cache rebuild** after context shift |
| 10:10:06 | 13,615 | 73,794 | **84.4%** | 294 | recovering |

10 microcompact events detected during this session. 0 budget events (body stayed under 300KB — threshold likely not reached in token terms despite 294KB body).

### Microcompact Correlation (v2.1.90 main session, 800KB+ body)

This data shows what happens in a long session where body size exceeds 500KB and microcompact is actively clearing:

| Time | Create | Read | Read % | Body KB | MC Active? |
|------|--------|------|--------|---------|-----------|
| 10:20:29 | 237,490 | 8,237 | **3.4%** | 746 | YES |
| 10:20:38 | 801 | 245,727 | **99.7%** | 749 | YES |
| 10:20:52 | 159 | 246,528 | **99.9%** | 749 | YES |
| 10:20:57 | 206 | 246,687 | **99.9%** | 750 | YES |
| 10:21:14 | 5,986 | 246,893 | **97.6%** | 769 | |
| 10:21:35 | 330 | 252,879 | **99.9%** | 770 | YES |
| 10:22:15 | 442 | 253,209 | **99.8%** | 771 | YES |
| 10:22:27 | 2,345 | 253,651 | **99.1%** | 779 | YES |
| 10:22:33 | 641 | 255,996 | **99.8%** | 781 | YES |
| 10:23:15 | 6,896 | 257,309 | **97.4%** | 803 | YES |

Key observation: even at 800KB+ body with active microcompact clearing, cache read stays **97-99.9%** — confirming that microcompact's stable substitution preserves the prompt prefix. The one outlier (3.4% at 10:20:29) is a cold-start/context-shift event, not caused by microcompact itself.

### Cross-Version Comparison

| Metric | v2.1.90 npm | v2.1.90 standalone | v2.1.91 npm | v2.1.91 standalone |
|--------|------------|-------------------|------------|-------------------|
| Overall | 86.4% | 86.2% | 82.8% | 82.8% |
| Cold start | 79-87% | **47-67%** | 84.7% | **84.7%** |
| Stable | 95-99.8% | 95-99.7% | 88-97% | 77-89% |
| Sub-agent cold | 79-87% | 14-47% | — | — |

**Note:** v2.1.91 "overall" appears lower (82.8% vs 86%) because the test session was shorter (7 requests vs 11+ scenarios in v2.1.90 benchmark). Stable-phase readings on longer v2.1.91 sessions show 97-99%, consistent with v2.1.90.

**Recommendation update:** On v2.1.91, **either installation is fine**. npm retains a theoretical advantage (no Sentinel code path) but the practical difference is now negligible.

---

## Tools Used

- **cc-relay v2** — transparent proxy with microcompact detection + budget enforcement scanning + session ID support
- **tools/jsonl_analyzer.py** — PRELIM/FINAL classification, synthetic detection, duplicate tool call detection
- **tools/gb_watcher.py** — GrowthBook feature flag monitoring (11 keys)
- **cc-relay SQLite DB** — 3,500+ logged requests across multiple sessions

---

## Changelog Cross-Reference

### v2.1.91 (Apr 2)
- `_meta["anthropic/maxResultSizeChars"]` up to 500K — **B5 MCP-only workaround**
- `--resume` transcript chain break fix — **B2 additional fix**
- `Edit shorter old_string anchors` — output token reduction

### v2.1.90 (Apr 1)
- `--resume` full cache miss fix (since v2.1.69) — **B2 main fix**
- `per-turn JSON.stringify elimination` — **B1 improvement**
- `rate-limit dialog infinite loop` — UX fix
- `autocompact thrash loop detection` — actually in v2.1.89

### v2.1.89 (Apr 1)
- `autocompact thrash loop` 3-retry stop — **B7 partial fix**
- `tool schema bytes changing mid-session` — **B1 partial fix**
- `nested CLAUDE.md re-injection` — context bloat fix
- `StructuredOutput schema cache ~50% failure` — separate cache bug
- `misleading "Rate limit reached" message` — **B3 message-only fix**

### v2.1.86 (Mar 27)
- `X-Claude-Code-Session-Id` header — session tracking for proxies
- `Read tool compact format + deduplication` — token reduction

### v2.1.84 (Mar 25)
- `idle-return 75min prompt` — Cache TTL UX mitigation
- `MCP tool descriptions 2KB cap` — MCP overhead reduction
- `"Improved p90 prompt cache rate"` — unspecified cache improvement
