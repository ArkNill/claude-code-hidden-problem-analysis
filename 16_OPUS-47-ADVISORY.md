> *(Korean translation not yet available for this document)*
>
> **📌 April 24 update:** Anthropic published a [postmortem](https://www.anthropic.com/engineering/april-23-postmortem) admitting 3 product-layer bugs. CHANGELOG cross-check, post-postmortem issues, and updated checklist: **[17_OPUS-47-POSTMORTEM-ANALYSIS.md](17_OPUS-47-POSTMORTEM-ANALYSIS.md)**

# Opus 4.7 Advisory — Do Not Upgrade

> **Recommendation (April 17, 2026):** Do not upgrade Claude Code past **v2.1.109**. Opus 4.7 introduces tokenizer inflation (+35%), invisible adaptive thinking overhead (2.4x averaged Q5h burn), long-context retrieval regression (91.9% → 59.2%), and critical bugs including model pin bypass ([#49503](https://github.com/anthropics/claude-code/issues/49503)) and cache metering anomaly ([#49302](https://github.com/anthropics/claude-code/issues/49302)). v2.1.109 sends explicit `claude-opus-4-6` model IDs, has native 1h cache, and is unaffected by the April 23 API default switchover. Stay on v2.1.109 until all blockers are resolved.
>
> **Data sources:** Self-measured benchmark (3 effort levels × 3 hard tasks × 20-turn session isolation, n=3), two independent community interceptors ([@cnighswonger](https://github.com/cnighswonger) metered 71 API calls, [@fgrosswig](https://github.com/fgrosswig) gateway 2,085 timeline samples), community issue comments (73+ on [#42796](https://github.com/anthropics/claude-code/issues/42796), 39+ on [#38335](https://github.com/anthropics/claude-code/issues/38335)), and 5 critical GitHub issues filed April 16-17.

---

## 1. Breaking Changes (Official)

Source: [What's new in Claude Opus 4.7](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7)

### 1.1 Tokenizer Replacement (+35%)

> "This new tokenizer may use roughly **1x to 1.35x** as many tokens when processing text compared to previous models"

Per-token pricing is unchanged ($5/$25 input/output per MTok), so the effective cost increases by up to 35%. Code and non-English text (CJK, Korean) hit the upper bound (1.35x). For sessions with Korean `CLAUDE.md`, memory files, and Korean conversation — the +35% compounds on every turn's system prompt.

External analysis: [Finout](https://www.finout.io/blog/claude-opus-4.7-pricing-the-real-cost-story-behind-the-unchanged-price-tag) estimates coding workloads increase from ~$300 to ~$405/month (+35%).

### 1.2 Extended Thinking Removed — Adaptive Only

```python
# 4.6 (previous)
thinking = {"type": "enabled", "budget_tokens": 32000}

# 4.7 (current) — budget_tokens returns 400 error
thinking = {"type": "adaptive"}
```

- Sending `budget_tokens` on 4.7 → **400 error** (breaking)
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` and `MAX_THINKING_TOKENS` env vars **do not work on 4.7** ([#49555](https://github.com/anthropics/claude-code/issues/49555))
- CC versions ≤v2.1.110 send `thinking: {type: "enabled"}` → 400 error on 4.7
- Thinking content defaults to `display: "omitted"` instead of `"summarized"` — VS Code shows empty thinking blocks

### 1.3 Sampling Parameters Removed

`temperature`, `top_p`, `top_k` with non-default values → **400 error**. Any tool or wrapper using custom sampling breaks immediately.

### 1.4 Effort Default Changed

| Setting | Opus 4.6 | Opus 4.7 |
|---------|----------|----------|
| CC default effort | high | **xhigh** |

Self-measured impact on a simple implementation task (LRU cache, Round 1): xhigh produces +6% output tokens, +2% cost, +10% duration compared to high. On hard reasoning tasks (Round 2), the delta varies widely: −15% to +44% cost depending on task type, with no measurable quality improvement on bug detection or architecture tasks. Only mathematical proofs benefited from higher effort. See [Section 3.3](#33-self-measured-benchmark).

---

## 2. Critical Unresolved Issues

Five issues filed April 16-17, all unresolved as of this writing. **Anthropic staff response rate on 4.7 issues: 1 out of 30+** — only `brenden` acknowledged Bedrock breakage ([#49238](https://github.com/anthropics/claude-code/issues/49238)), coordinating with AWS. No staff responses on any quality, cost, thinking visibility, or model pinning issues. Boris Cherny was quoted on HN (not GitHub) regarding thinking summaries: "users don't read reasoning summaries anyway." See [External References](#additional-github-issues-47-specific-not-already-referenced) for the full issue landscape.

### 2.1 Cache Metering Anomaly ([#49302](https://github.com/anthropics/claude-code/issues/49302))

`cache_read_input_tokens` reported as being charged at full input-token rate. One reporter: 190M tokens on 4.6 did not exhaust 5h quota; 30M tokens on 4.7 exhausted 100% in 2 hours — reported as ~7x worse metering. Anthropic support acknowledged "inconsistency with documented behavior" but has not issued a fix.

> **Evidence quality note:** The "~7x" figure is derived from a single user's comparison of 4.6 vs 4.7 quota exhaustion rates. The exact multiplier depends on session characteristics (cache hit ratio, thinking overhead, context size). We report it as "reported as ~7x" rather than a confirmed measurement.

### 2.2 Silent Model Switch ([#49541](https://github.com/anthropics/claude-code/issues/49541))

On April 16 (launch day), active sessions were switched from 4.6 to 4.7 **without user notification**. The `claude-opus-4-7[1M]` variant does not auto-compact at 200K like 4.6 — cache can grow to 4-60M tokens unchecked. One user reported 5h quota exhausted in 30 minutes (391M cache-read).

### 2.3 Model Pin Bypass ([#49503](https://github.com/anthropics/claude-code/issues/49503))

`settings.json` with `"model": "claude-opus-4-6"` is **ignored** by CC v2.1.111. Sessions resume on Opus 4.7 regardless of configuration. The only defense is manually running `/model claude-opus-4-6` on every resume. Confirmed by [@cnighswonger](https://github.com/cnighswonger) with API-level interception.

**Scope:** This bypass is specific to v2.1.111+. CC versions ≤v2.1.109 send explicit model IDs and are not affected (self-verified — see [Section 4](#4-why-v21109-is-safe)).

### 2.4 Adaptive Under-Thinking ([#49555](https://github.com/anthropics/claude-code/issues/49555))

Multiple users report that adaptive thinking produces insufficient reasoning. The `effort` parameter only adjusts thinking depth — it does not restore the deterministic budget control that `budget_tokens` provided. Car wash litmus test (a simple logical reasoning prompt) fails ~50% of the time on 4.7 ([@gochax](https://github.com/gochax), tested on claude.ai).

> **Evidence quality note:** "4.6 no longer selectable in model picker" was reported by a single user ([@NtTestAlert](https://github.com/NtTestAlert), April 17). This may be temporary or account-specific and has not been independently verified.

### 2.5 Smoosh Pipeline Cache Breakage ([#49585](https://github.com/anthropics/claude-code/issues/49585))

`normalizeMessagesForAPI` runs a smoosh pass every turn that folds dynamic `<system-reminder>` text (containing `token_usage`, `output_token_usage`, `budget_usd`) into `tool_result.content` strings. Because these values change turn-over-turn, this produces byte drift that breaks prompt cache prefixes. Filed by [@deafsquad](https://github.com/deafsquad) with a minimal Python reproduction.

- **Impact measured:** ~204 fusion-attributable turns caused ~99M `cache_creation` tokens = 56% of total `cache_creation` spend
- **Mitigation:** [@cnighswonger](https://github.com/cnighswonger) released [claude-code-cache-fix v2.0.0-beta.4](https://github.com/cnighswonger/claude-code-cache-fix) with `smoosh_normalize` + `smoosh_split` — production-verified by deafsquad: `cache_creation` 940K → **1,704** (99.8% reduction)
- **Note:** This issue affects **all models**, not just Opus 4.7. It is included here because the 4.7 transition amplifies its impact (adaptive thinking tokens + tokenizer inflation compound on top of cache breakage).

### 2.6 Autocompact Fires at ~195K Despite 1M Context ([#53801](https://github.com/anthropics/claude-code/issues/53801))

> **Added:** May 4, 2026

Despite the v2.1.117 fix for [#52522](https://github.com/anthropics/claude-code/issues/52522) (auto-compact threshold change from 200K to 1M), auto-compaction continues to fire at **~195K tokens** on v2.1.120, using only **19.5% of the 1M context window**.

**Measured (v2.1.120, Opus 4.7 1M, `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=40`):**

| Trigger | preTokens | % of 1M |
|---------|-----------|---------|
| auto | 196,512 | 19.65% |
| auto | 194,360 | 19.44% |
| auto | 194,159 | 19.42% |
| auto | 195,854 | 19.59% |
| auto | 194,591 | 19.46% |

Data extracted from JSONL `compact_boundary` records. The tight band (4,353 token range, ~2.2% spread) indicates a **deterministic 200K internal threshold** that ignores both the model's 1M capacity and the user's 40% override setting.

- **Impact:** Users paying for 1M context effectively get ~195K before compaction destroys session state. After compaction, technical decisions and earlier reasoning are lost — the model cannot reference pre-compaction information.
- **Status:** #53801 closed as duplicate of #36381 and #52519 (both CLOSED). Whether the root cause is resolved remains unclear — the data was collected on v2.1.120, **post** the v2.1.117 fix.
- **Evidence strength:** **STRONG** — JSONL data extraction, precise timestamps, 6 consecutive reproductions in a single session.

### 2.7 xhigh Effort Regression — medium Currently Better ([#55301](https://github.com/anthropics/claude-code/issues/55301))

> **Added:** May 4, 2026

Proxy-measured evidence suggests `xhigh` effort on Opus 4.7 experienced a **quality regression on May 1, 2026** — with `medium` effort producing better results on the same model in the same session.

**Measured via [claude-hooks](https://github.com/mann1x/claude-hooks) transparent proxy (SQLite, stellaraccident canary phrases):**

| Date | Effort | Requests | ownership-dodging /1k |
|------|--------|----------|----------------------|
| Apr 28 | xhigh | 2,316 | 6.04 |
| Apr 29 | xhigh | 2,640 | 5.30 |
| Apr 30 | xhigh | 3,050 | 4.59 |
| **May 1** | **xhigh** | **309** | **29.13** (6x spike) |
| May 1 | medium | 486 | **2.06** |

Same model delivered (`claude-opus-4-7`), same service tier, same beta features, 98% cache hit rate. The quality regression was **effort-specific** — switching to `medium` in the same session cleared all symptoms immediately.

- **Evidence strength:** **MODERATE** — rigorous proxy methodology, but **n=1 reporter, single-day anomaly** (May 1 only). Could be a transient serving-side issue rather than a persistent regression. Needs independent verification before relying on this finding.
- **Interaction with Section 1.4:** If confirmed, this would invert the effort cost-benefit calculation: `xhigh` would be both more expensive and worse than `medium` on certain days.

---

## 3. Independent Measurements

### 3.1 cnighswonger: 2.4x Averaged Q5h Burn

**Source:** [claude-code-cache-fix Discussion #25](https://github.com/cnighswonger/claude-code-cache-fix/discussions/25) | [fgrosswig/claude-gateway#1](https://github.com/fgrosswig/claude-gateway/issues/1)

| Metric | Opus 4.6 | Opus 4.7 | Multiplier |
|--------|----------|----------|------------|
| Avg Q5h per turn | ~0.3% | ~0.73% | **2.4x** |
| Tokenizer contribution | — | ~35% | — |
| Remaining (adaptive thinking) | — | ~105% | — |

Measurement: 71 API calls over 38 minutes (CC v2.1.111, Max 5x US). The 2.4x combines tokenizer inflation (35%) and invisible adaptive thinking overhead (~105%). Thinking tokens are charged against quota but **not reported in the API usage response**.

Cross-validated by [@fgrosswig](https://github.com/fgrosswig) from gateway proxy data: **2.6x** (Q5h burn ~9%/hour on 4.6 vs ~24%/hour on 4.7). Two independent interceptors converging on 2.4-2.6x strengthens confidence.

Testing `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` on 4.7: a single call showed 0% Q5h delta vs 1-37% per call with it enabled — suggesting the 2.4x is entirely adaptive thinking overhead. However, fgrosswig reported this env var **crashes the gateway on 4.7** — single data point, unverified independently.

> **Evidence quality note:** The 2.4x is from a single session (n=1 session, 71 calls). While the call count provides reasonable statistical power, session-level factors (time of day, server load, cache state) are not controlled. The independent fgrosswig cross-validation at 2.6x mitigates this concern.

### 3.2 fgrosswig: 12.5x Cold-Start, 50x Worst-Case

**Source:** [fgrosswig/claude-gateway#1](https://github.com/fgrosswig/claude-gateway/issues/1)

Live A/B test on the same Max 5x account, same minute:

| Metric | Opus 4.6 | Opus 4.7 |
|--------|----------|----------|
| Q5h per call | ~0.1% | **~1.25%** |
| cache_read | 821K | 301K |
| Burn rate multiplier | 1x | **12.5x** |

At 12.5x, a Max 5x user would exhaust the 5h quota in **~24 minutes** instead of ~5 hours.

Worst observed single call: **5% Q5h consumed** in one request (fresh quota after 5h reset, 0 cache_read, 55s duration). This represents a **50x** multiplier — however, this is a single data point and should be treated as an extreme outlier rather than a representative measurement.

**Interpretation:** The 2.4x (averaged) → 12.5x (cold-start) → 50x (single-call) progression is consistent with **front-loaded adaptive thinking overhead**: the first few calls after a cache miss or session start incur disproportionate thinking cost, which amortizes over longer sessions. The averaged 2.4x is the most representative figure for typical usage; 12.5x represents the penalty for cold starts and short sessions; 50x is the theoretical worst case.

### 3.3 Self-Measured Benchmark

**Source:** `~/local/opus47-bench/` (v2.1.112, Opus 4.7, hmj PC Win11/Max5, `--effort high`)

#### Effort Comparison (3 hard tasks × 3 effort levels)

| Task | Effort | Duration | Output Tokens | Cost USD |
|------|--------|----------|---------------|----------|
| A (Bug detection, Go) | high | 89s | 6,844 | $0.227 |
| A | xhigh | 136s | 10,825 | $0.327 |
| A | max | 195s | 16,264 | $0.463 |
| B (Rate limiter arch) | high | 123s | 8,772 | $0.255 |
| B | xhigh | 101s | 7,301 | $0.218 |
| B | max | 214s | 16,368 | $0.462 |
| C (Median proof, math) | high | 62s | 4,352 | $0.144 |
| C | xhigh | 77s | 6,076 | $0.187 |
| C | max | 175s | 15,414 | $0.420 |

Quality was identical across effort levels for Tasks A and B. Only Task C (mathematical proof) showed a meaningful quality difference at `max` (additional Omega(log n) reduction proof).

#### 20-Turn Sequential Session (cache-fix isolation)

| Mode | Wall Time | API Cost | Output Tokens | cache_creation_1h | cache_creation_5m | 5h% Delta |
|------|-----------|----------|---------------|-------------------|-------------------|-----------|
| stock (plain claude) | 540s | $2.33 | 41,219 | 56,253 | **0** | +6%p |
| cache-fix only | 514s | $2.25 | 42,109 | 55,076 | **0** | +? |
| full stack (fixed + proxy) | 652s | $2.98 | 49,105 | 74,845 | **0** | +8%p |

**Key finding:** `cache_creation_5m=0` across all modes — v2.1.112 has **native 1h cache** (cache-fix TTL extension is redundant).

> **Why our benchmark did not reproduce worst-case scenarios:**
> 1. `--effort high` explicit — avoided xhigh default (+20% overhead)
> 2. `claude -p` print mode — non-interactive, no long-running session accumulation
> 3. English prompts — avoided tokenizer +35% Korean penalty
> 4. 20-turn isolated sessions — no 1M context infinite accumulation
>
> The community worst cases (2.4x–50x) occur in **long interactive sessions** with Korean/mixed-language content, `--resume` chains, and default effort settings. Our controlled benchmark intentionally isolated variables, which is why it shows `stock ≈ intercept` — this does not contradict the community findings but rather demonstrates that the overhead is **conditional on session characteristics**.

### 3.4 Cross-Validation Convergence

| Source | Measurement | Condition | Q5h Multiplier |
|--------|-------------|-----------|----------------|
| cnighswonger | 71-call average | Interactive, Max 5x | **2.4x** |
| fgrosswig (gateway) | Hourly rate | Long session, Max 5x | **2.6x** |
| fgrosswig (A/B) | Per-call, cold start | Same account, same minute | **12.5x** |
| fgrosswig (single call) | Single request, fresh quota | Worst observed | **50x** |
| Self-measured (hmj) | 20-turn print mode | Controlled, effort=high | **~1x** (no overhead) |

The ~1x self-measurement is not contradictory — it confirms that the overhead is **session-characteristic-dependent**, not a universal multiplier. The community's 2.4-2.6x averaged figure is the most representative for typical interactive usage.

### 3.5 Long-Context Retrieval Regression

From the Opus 4.7 model card discussion ([HN](https://news.ycombinator.com/item?id=47793546)):

| Context Range | Opus 4.6 | Opus 4.7 | Delta |
|---------------|----------|----------|-------|
| Overall retrieval | 91.9% | 59.2% | **−32.7pp** |
| 524K–1024K tokens | 78.3% | 32.2% | **−46.1pp** |

This is a separate dimension from cost — even if token overhead were resolved, long-context performance is substantially worse on 4.7. For workflows relying on large context windows (e.g., codebase-wide analysis, multi-file refactoring), this regression alone justifies staying on 4.6.

---

## 4. Why v2.1.109 Is Safe

### 4.1 Explicit Model ID Transmission (Verified)

Self-tested on ZBook v2.1.109 (`claude -p --output-format json "say exactly: ping"`):

```json
{
  "modelUsage": "claude-opus-4-6[1m]"
}
```

v2.1.109 sends `claude-opus-4-6` explicitly in every API request. This means the April 23 server-side default switchover (when the API default changes from 4.6 to 4.7) **does not affect v2.1.109** — the explicit model ID overrides the server default.

### 4.2 Native 1h Cache

Measured: `cache_creation_1h=43,139 / cache_creation_5m=0` on v2.1.109 — confirming native 1h cache support without cache-fix.

### 4.3 Model Pin Bypass Does Not Apply

The model pin bypass ([#49503](https://github.com/anthropics/claude-code/issues/49503)) is specific to **v2.1.111+** where the client's model resolution logic changed. v2.1.109 respects `settings.json` model configuration.

### 4.4 Risk Matrix

| Risk | v2.1.109 | v2.1.111+ |
|------|----------|-----------|
| Tokenizer +35% | Not exposed (uses 4.6) | **Exposed** |
| Cache metering 7x (#49302) | Not exposed | **Exposed** |
| Model pin bypass (#49503) | Not affected | **Affected** |
| Silent model switch (#49541) | Not affected | **Affected** |
| Adaptive thinking overhead | Not exposed (uses 4.6 thinking) | **2.4x averaged** |
| 4/23 default switchover | Safe (explicit model ID) | **Vulnerable** (pin ignored) |

> **Caveat:** v2.1.109 is safe **only if you do not upgrade CC**. Any CC update to v2.1.111+ will break this protection. Pin your version:
> ```json
> {"env": {"DISABLE_AUTOUPDATER": "1"}}
> ```

---

## 5. Recommended Configuration

### settings.json

```json
{
  "model": "claude-opus-4-6",
  "effortLevel": "high",
  "env": {
    "DISABLE_AUTOUPDATER": "1",
    "ANTHROPIC_MODEL": "claude-opus-4-6",
    "CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING": "1"
  }
}
```

### Behavioral Guidelines

| Do | Don't |
|----|-------|
| Stay on v2.1.109 (or any version ≤v2.1.110) | Upgrade to v2.1.111+ |
| Pin `claude-opus-4-6` explicitly | Rely on server defaults |
| Set `effortLevel: "high"` | Use default xhigh (4.7 default, +20% cost for no quality gain) |
| Start fresh sessions frequently | Use `--resume` / `--continue` (full context replay) |
| Monitor `/usage` each session | Assume quota is tracking correctly |

### If You Must Use v2.1.111+

If you have already upgraded and cannot downgrade:
1. Run `/model claude-opus-4-6` **on every session resume** — the model pin in settings.json is ignored
2. Set `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` — may reduce thinking overhead (single data point, [fgrosswig reports crash on 4.7](https://github.com/fgrosswig/claude-gateway/issues/1))
3. Install [claude-code-cache-fix v2.0.0-beta.4](https://github.com/cnighswonger/claude-code-cache-fix) — addresses smoosh pipeline cache breakage ([#49585](https://github.com/anthropics/claude-code/issues/49585))

---

## 6. Evidence Quality Assessment

This section documents known gaps in the evidence base. All claims in this document are annotated with their confidence level.

### 6.1 Logical Consistency

| # | Claim | Assessment |
|---|-------|------------|
| L1 | 2.4x vs 12.5x vs 50x are inconsistent | **Consistent.** These measure different conditions: session-averaged, cold-start per-call, and single-call worst-case. The progression is explained by front-loaded adaptive thinking overhead that amortizes over longer sessions. |
| L2 | Self-benchmark shows ~1x but community shows 2.4x+ | **Consistent.** Our benchmark used `--effort high`, print mode, English prompts, and 20-turn isolated sessions — conditions that avoid the compounding factors that produce 2.4x in interactive usage. This is not a contradiction but a confirmation that the overhead is conditional. |
| L3 | v2.1.109 is safe from 4/23 switchover | **Verified.** Self-tested: v2.1.109 sends explicit `claude-opus-4-6` model ID. Server defaults only apply when the client does not specify a model. The risk resumes if CC is upgraded. |
| L4 | Smoosh issue is model-independent but included in 4.7 advisory | **Justified.** While smoosh affects all models, the 4.7 transition amplifies its impact: adaptive thinking tokens + tokenizer inflation compound on top of cache breakage. Users upgrading to 4.7 face both issues simultaneously. |

### 6.2 Numerical Rigor

| # | Claim | Assessment |
|---|-------|------------|
| N1 | cnighswonger 2.4x from n=1 session | **Moderate confidence.** 71 API calls provide reasonable per-call statistics, but session-level factors (server load, time of day) are uncontrolled. Mitigated by fgrosswig's independent 2.6x cross-validation. |
| N2 | fgrosswig 50x worst-case | **Low confidence.** Single data point. Should be cited as "worst observed" not "typical." The 12.5x cold-start figure (from a controlled A/B test) is more reliable. |
| N3 | #49302 "~7x" cache metering | **Low confidence.** Derived from a single user's comparison (190M/5h vs 30M/2h). The exact multiplier is sensitive to session characteristics. We report it as "reported as ~7x" throughout. |
| N4 | Two independent sources converge at 2.4-2.6x | **High confidence.** cnighswonger (API-level metering) and fgrosswig (proxy-level gateway) used completely different interception architectures on different accounts. Convergence at 2.4-2.6x strongly suggests a real effect. |
| N5 | Long-context retrieval 91.9% → 59.2% | **Moderate confidence.** Sourced from HN discussion of the 221-page model card ([T8](#independent-technical-analysis)). The numbers appear to be extracted from the card's evaluation tables, but we have not independently verified them against the primary document. The 524K-1024K range regression (78.3% → 32.2%) is particularly severe and directly relevant to CC workflows with large codebases. |

### 6.3 Evidence Gaps

| # | Gap | Severity | Mitigation |
|---|-----|----------|------------|
| E1 | "Opus 4.6 no longer selectable" (NtTestAlert) | Low | Single report, may be temporary or account-specific. Not relied upon for recommendations. |
| E2 | `DISABLE_ADAPTIVE_THINKING` crashes on 4.7 (fgrosswig) | Moderate | Single report. Our benchmark did not test this on 4.7. Included as "reported, unverified independently." |
| E3 | No official Anthropic response to cost increase | High | Anthropic acknowledged #49302 inconsistency but has not addressed the 2.4x overhead, model pin bypass, or silent switch. We cannot confirm whether these are bugs or intended behavior changes. |
| E4 | No large-scale controlled experiment | High | All community measurements are observational (no randomized A/B with controlled workloads). Our self-benchmark is the closest to controlled but used non-representative conditions. A definitive answer requires Anthropic to publish metering methodology. |
| E5 | Long-context retrieval regression not independently verified | Moderate | The 91.9% → 59.2% figures are from HN discussion of the model card, not from our own testing. We have not run MRCR or equivalent retrieval benchmarks on 4.7. The model card itself is 221 pages and has not been independently audited. However, the regression is directionally consistent with community reports of degraded performance on large codebases. |
| E6 | Autocompact 195K threshold persists post-fix (#53801) | Moderate | Single reporter on v2.1.120 with JSONL data extraction (strong methodology). Issue closed as duplicate — unclear if truly fixed in later versions. Added May 4. |
| E7 | xhigh effort regression (#55301) | High | Single reporter, single-day anomaly (May 1). Could be transient serving-side issue. Rigorous proxy methodology but needs independent verification before acting on it. Added May 4. |

---

## 7. When to Reconsider

Upgrade to Opus 4.7 / newer CC versions when **all** of the following are confirmed in CC release notes:

- [ ] [#49302](https://github.com/anthropics/claude-code/issues/49302) — Cache metering anomaly resolved
- [ ] [#49503](https://github.com/anthropics/claude-code/issues/49503) — Model pin bypass fixed (`settings.json` respected)
- [ ] [#49541](https://github.com/anthropics/claude-code/issues/49541) — Silent model switch addressed (user notification or opt-in)
- [ ] Adaptive thinking tokens included in API usage response (transparency)
- [ ] Independent measurement confirms Q5h burn rate within 1.5x of 4.6 (accounting for tokenizer)
- [ ] Korean tokenizer penalty measured and acceptable (<1.5x on typical CLAUDE.md + conversation)
- [ ] Long-context retrieval regression addressed (91.9% → 59.2% at 256K context, 78.3% → 32.2% at 1M context per system card)
- [x] ~~Effort default~~ — **FIXED** (v2.1.117, April 21 — Pro/Max subscribers)
- [x] ~~Bash classifier hardcoded to 4.6~~ — [#49618](https://github.com/anthropics/claude-code/issues/49618) **CLOSED** (April 20) *(added May 4)*
- [x] ~~Context startup bloat~~ — [#49593](https://github.com/anthropics/claude-code/issues/49593) **CLOSED** (April 22) *(added May 4)*
- [ ] [#52502](https://github.com/anthropics/claude-code/issues/52502) — Subagent model pin respected (billing impact) *(added April 24)*
- [ ] [#52534](https://github.com/anthropics/claude-code/issues/52534) — Effort override bypass resolved (`unpinOpus47LaunchEffort`) *(added April 24)*
- [ ] [#53801](https://github.com/anthropics/claude-code/issues/53801) — Autocompact fires at ~195K despite 1M context (v2.1.120, post v2.1.117 fix) *(added May 4)*

**Monitoring resources:**
- [CC changelog](https://code.claude.com/docs/en/changelog) — check each release for #49302/#49503 fixes
- [@cnighswonger/claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) — community interceptor updates
- [fgrosswig/claude-gateway](https://github.com/fgrosswig/claude-gateway) — proxy forensics data
- [@ClaudeDevs on X](https://x.com/ClaudeDevs) — official Anthropic product updates (announced April 23)
- [17_OPUS-47-POSTMORTEM-ANALYSIS.md](17_OPUS-47-POSTMORTEM-ANALYSIS.md) — postmortem cross-check and post-postmortem issues

---

## Related Documents (Internal)

- [01_BUGS.md](01_BUGS.md) — B1-B11 technical root cause analysis
- [09_QUICKSTART.md](09_QUICKSTART.md) — Setup recommendations (updated with 4.7 warning)
- [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) — Quota architecture (5h/7d windows)
- [13_PROXY-DATA.md](13_PROXY-DATA.md) — Proxy dataset (45,884 requests)
- [15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md) — Per-environment cache_read breakdown

---

## External References

### Official Anthropic Sources

| # | Source | URL | Key Content |
|---|--------|-----|-------------|
| O1 | **Blog: Introducing Claude Opus 4.7** | [anthropic.com/news](https://www.anthropic.com/news/claude-opus-4-7) | Launch announcement. 13% coding lift over 4.6 on 93-task benchmark, 3x on Rakuten-SWE-Bench, xhigh effort, task budgets (beta). Pricing: $5/$25 MTok (unchanged) |
| O2 | **What's New in Claude Opus 4.7** | [platform.claude.com](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7) | Most detailed official reference. All 4 breaking changes, behavior changes, capability improvements |
| O3 | **Migration Guide (4.6 → 4.7)** | [docs.anthropic.com](https://docs.anthropic.com/en/docs/about-claude/models/migrating-to-claude-4) | 15-item migration checklist, code samples (Python/TS/cURL/Go/Java/C#/PHP/Ruby), `/claude-api migrate` skill |
| O4 | **Adaptive Thinking Documentation** | [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking) | Adaptive is the only supported thinking mode on 4.7. `budget_tokens` → 400 error. Thinking OFF by default. Summarized thinking billed on full internal tokens |
| O5 | **Effort Levels Documentation** | [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/effort) | 5 levels: max/xhigh/high/medium/low. "Start with xhigh for coding and agentic use cases." At xhigh/max, set max_tokens ≥ 64k |
| O6 | **Pricing** | [docs.anthropic.com](https://docs.anthropic.com/en/docs/about-claude/pricing) | Identical per-token rates 4.6/4.7. 1h cache writes $10/MTok. Note: "Opus 4.7 uses a new tokenizer... may use up to 35% more tokens for the same fixed text" |
| O7 | **API Release Notes (April 16)** | [platform.claude.com](https://platform.claude.com/docs/en/release-notes/api) | Confirms launch + breaking changes. Also: April 14 deprecation of Sonnet 4 / Opus 4 (retire June 15, 2026) |
| O8 | **Product Page** | [anthropic.com/claude/opus](https://www.anthropic.com/claude/opus) | Up to 90% savings with prompt caching, 50% with batch |

### Mainstream News Coverage

| # | Source | Date | URL | Key Content |
|---|--------|------|-----|-------------|
| N1 | **Fortune** | 4/14 | [fortune.com](https://fortune.com/2026/04/14/anthropic-claude-performance-decline-user-complaints-backlash-lack-of-transparency-accusations-compute-crunch/) | Boris Cherny (head of CC) cited 3 deliberate changes: adaptive thinking (Feb 9), effort dropped to medium (Mar 3), thinking redaction (Feb 12). Denied compute constraints |
| N2 | **VentureBeat** | 4/16 | [venturebeat.com](https://venturebeat.com/technology/is-anthropic-nerfing-claude-users-increasingly-report-performance) | AMD Senior Director Stella Laurenzo: "Claude has regressed to the point it cannot be trusted to perform complex engineering." 6,852 sessions analyzed |
| N3 | **Axios** | 4/16 | [axios.com](https://www.axios.com/2026/04/16/anthropic-claude-power-user-complaints) | "Anthropic's AI downgrade stings power users" — Anthropic confirmed changes but denied intentional degradation |
| N4 | **Axios** | 4/16 | [axios.com](https://www.axios.com/2026/04/16/anthropic-claude-opus-model-mythos) | "Anthropic concedes it trails unreleased Mythos" — rare self-admission of inferior release |
| N5 | **CNBC** | 4/15 | [cnbc.com](https://www.cnbc.com/2026/04/15/anthropic-outage-elevated-errors-claude-chatbot-code-api.html) | Day before 4.7 launch: elevated errors across all services, ~2,000 Downdetector reports |
| N6 | **CNBC** | 4/16 | [cnbc.com](https://www.cnbc.com/2026/04/16/anthropic-claude-opus-4-7-model-mythos.html) | Outperforms GPT-5.4 / Gemini 3.1 Pro on coding benchmarks |
| N7 | **Gizmodo** | 4/15 | [gizmodo.com](https://gizmodo.com/anthropic-is-jacking-up-the-price-for-power-users-amid-complaints-its-model-is-getting-worse-2000746923) | Enterprise pricing: flat $200/user → usage-based ($20 + compute). Could triple costs |
| N8 | **Gizmodo** | 4/16 | [gizmodo.com](https://gizmodo.com/anthropic-releases-claude-opus-4-7-to-remind-everyone-how-great-mythos-is-2000747469) | "Watered-down version of the product Anthropic really wants you to be thinking about" |
| N9 | **The Register** | 4/13 | [theregister.com](https://www.theregister.com/2026/04/13/claude_outage_quality_complaints/) | Claude analyzed its own issue tracker: quality complaints 3.5x baseline surge in April |
| N10 | **TechBooky** | 4/14 | [techbooky.com](https://www.techbooky.com/anthropic-faces-user-backlash-over-alleged-nerfing-of-claude-models/) | "AI shrinkflation" framing. Anthropic declined public inquiries |

### Independent Technical Analysis

| # | Source | URL | Key Finding |
|---|--------|-----|-------------|
| T1 | **Finout** — Opus 4.7 Pricing | [finout.io](https://www.finout.io/blog/claude-opus-4.7-pricing-the-real-cost-story-behind-the-unchanged-price-tag) | Coding workloads $300→$405/mo (+35%) |
| T2 | **llm-stats.com** — 4.7 vs 4.6 | [llm-stats.com](https://llm-stats.com/blog/research/claude-opus-4-7-vs-opus-4-6) | 4.7 wins 12/14 benchmarks. "Low-effort 4.7 matches medium-effort 4.6" — equivalent work costs less if effort is tuned |
| T3 | **Vellum AI** — Benchmarks | [vellum.ai](https://www.vellum.ai/blog/claude-opus-4-7-benchmarks-explained) | SWE-bench Verified 87.6% (+6.8), SWE-bench Pro 64.3% (+10.9). BrowseComp 79.3% (−4.4, one clear regression) |
| T4 | **CodeRabbit** — Code Review | [coderabbit.ai](https://www.coderabbit.ai/blog/claude-opus-4-7-for-ai-code-review) | 4.7 scored 68/100 (up from 55/100, +24%). Weakness: severity skews critical/major |
| T5 | **The Decoder** | [the-decoder.com](https://the-decoder.com/anthropics-claude-opus-4-7-makes-a-big-leap-in-coding-while-deliberately-scaling-back-cyber-capabilities/) | "Hidden cost structure" — identical text costs up to 1.35x more tokens |
| T6 | **DevToolPicks** | [devtoolpicks.com](https://devtoolpicks.com/blog/claude-opus-4-7-launch-review-2026) | "Benchmark your actual prompts before migrating." Notes Mythos outperforms, suggesting interim tech |
| T7 | **Artificial Analysis** | [artificialanalysis.ai](https://artificialanalysis.ai/models/claude-opus-4-7) | Opus 4.6 already used 30-60% more tokens than 4.5 on GDPval-AA benchmarks |
| T8 | **HN: Model Card** (167pts, 79cmt) | [news.ycombinator.com](https://news.ycombinator.com/item?id=47793546) | **Long-context retrieval 59.2% vs 4.6's 91.9%**. At 524k-1024k: 32.2% vs 78.3%. "Mythos" appears 331× in 221-page card |
| T9 | **HN: Launch Thread** | [news.ycombinator.com](https://news.ycombinator.com/item?id=47793411) | Users report 10-20x cost, "burns way more tokens." Open-weight models outperforming on reasoning |

### Community Deep Dives

| # | Source | URL | Key Finding |
|---|--------|-----|-------------|
| C1 | **Implicator.ai** — "Probably Wasn't Nerfed" | [implicator.ai](https://www.implicator.ai/claude-probably-wasnt-secretly-nerfed-anthropic-made-the-black-box-too-dark/) | Most balanced analysis. 119,866 API calls showing cache TTL shifts. "The model name stopped being the product." Calls for session-level telemetry |
| C2 | **dgtldept (Substack)** — Regression Fixes | [substack.com](https://dgtldept.substack.com/p/claude-opus-4-6-actually-did-get-dumber-regression-fixes) | 6 failure modes documented. Permission-asking 43×/day by Mar 18. Thinking chars 2,200→600. Actionable fixes included |
| C3 | **Perplexity AI Magazine** — 67% Reasoning Drop | [perplexityaimagazine.com](https://perplexityaimagazine.com/ai-news/anthropic-claude-opus-thinking-regression-2026/) | 73% thinking token reduction. Median counts 2,200→600 chars. Output inflation up to 64x |
| C4 | **Medium (Marianski)** — Cache Reverse Engineering | [medium.com](https://medium.com/@marianski.jacek/claude-code-cache-crisis-a-complete-reverse-engineering-analysis-9a6f4e03fae4) | MITM + Ghidra disassembly. Sentinel at `0x0374d610` causing 10-20x cost. 5,353 API requests analyzed |
| C5 | **SmartScope** — Source Code Leak Analysis | [smartscope.blog](https://smartscope.blog/en/blog/claude-code-token-consumption-cache-bug/) | 510K lines of TypeScript. Two cache-breaking mechanisms: attestation rotation + anti-distillation tool injection |
| C6 | **recca0120** — 95 Days of Cache Logs | [recca0120.github.io](https://recca0120.github.io/en/2026/04/14/claude-code-cache-ttl-audit/) | 3 TTL transitions over 95 days. Monthly costs inflated 15-53% |
| C7 | **CryptoNews** — "Token Eating Machine" | [cryptonews.net](https://cryptonews.net/news/other/32719928/) | Single test session depleted entire token quota for first time during evaluation |
| C8 | **Scortier (Substack)** — 6,852 Sessions | [substack.com](https://scortier.substack.com/p/claude-code-drama-6852-sessions-prove) | SWE-Bench-Pro independent testing: 6-point decline. 1M context degrades well before capacity |

### Additional GitHub Issues (4.7-specific, not already referenced)

| # | Issue | Date | Key Finding |
|---|-------|------|-------------|
| G1 | [#49268](https://github.com/anthropics/claude-code/issues/49268) | 4/16 | **Thinking summaries missing** — harness never sets `display: "summarized"`. Root cause for 4+ duplicate issues. Workaround: `--thinking-display summarized` |
| G2 | [#49238](https://github.com/anthropics/claude-code/issues/49238) | 4/16 | **Bedrock broken** — Anthropic staff `brenden` confirmed, working with AWS. Only confirmed staff response across all 4.7 issues |
| G3 | [#49689](https://github.com/anthropics/claude-code/issues/49689) | 4/17 | **4.6 removed from Desktop model picker** — users cannot select 4.6 via UI |
| G4 | [#49618](https://github.com/anthropics/claude-code/issues/49618) | 4/17 | **Bash classifier hardcoded to 4.6** — auto mode safety classifier uses `claude-opus-4-6-1m` model ID, breaks in 4.7 sessions |
| G5 | [#49747](https://github.com/anthropics/claude-code/issues/49747) | 4/17 | **XML/JSON tool call regression** — 4.7 emits mixed legacy XML format into JSON tool calls on long payloads |
| G6 | [#49751](https://github.com/anthropics/claude-code/issues/49751) | 4/17 | **Safety over-refusal** — flags standard computational structural biology as usage policy violation |
| G7 | [#49604](https://github.com/anthropics/claude-code/issues/49604) | 4/16 | **Car wash multi-turn regression** — new "commitment stickiness" failure mode on top of existing reasoning bugs |
| G8 | [#49219](https://github.com/anthropics/claude-code/issues/49219) | 4/16 | **/model 4.7 → 400 error** — CC sends `thinking.type: "enabled"`, 4.7 requires `"adaptive"`. Workaround: `CLAUDE_CODE_EXTRA_BODY` |
| G9 | [#48808](https://github.com/anthropics/claude-code/issues/48808) | 4/15 | **2-3x output token increase** since v2.1.96 — pre-4.7 but compounds with tokenizer inflation |
| G10 | [#49600](https://github.com/anthropics/claude-code/issues/49600) | 4/16 | **Team plan agents get 200K instead of 1M** context window on 4.7 |

### Community Tools

| Tool | Stars | URL | Relevance |
|------|-------|-----|-----------|
| **jserv/cjk-token-reducer** | 42 | [github.com](https://github.com/jserv/cjk-token-reducer) | Reduces CJK token usage 35-50% — directly counteracts tokenizer inflation |
| **phuryn/claude-usage** | 959 | [github.com](https://github.com/phuryn/claude-usage) | Local dashboard for token usage, costs, session history |
| **juliusbrussee/caveman** | ~5K | [github.com](https://github.com/juliusbrussee/caveman) | CC skill cutting ~65% output tokens via terse language. #1 trending on GitHub |
| **seifghazi/claude-code-proxy** | 443 | [github.com](https://github.com/seifghazi/claude-code-proxy) | Captures and visualizes in-flight CC API requests |

### Enterprise Pricing Context

| # | Source | URL | Key Finding |
|---|--------|-----|-------------|
| P1 | **Gizmodo** | [gizmodo.com](https://gizmodo.com/anthropic-is-jacking-up-the-price-for-power-users-amid-complaints-its-model-is-getting-worse-2000746923) | Flat $200/user → usage-based ($20 + compute). Could triple costs |
| P2 | **PYMNTS** | [pymnts.com](https://www.pymnts.com/artificial-intelligence-2/2026/anthropic-switches-to-usage-based-billing-for-enterprise-customers/) | Must pre-commit token spend. Legacy 10-15% enterprise discounts removed |
| P3 | **Dapta.ai** | [dapta.ai](https://dapta.ai/blog-posts/ai-news-week-14-anthropic-claude-pricing/) | Third-party agent frameworks no longer flat-rate. Single OpenClaw session: $1K-$5K. Up to 50x previous spending |
| P4 | **The New Stack** | [thenewstack.io](https://thenewstack.io/claude-million-token-pricing/) | (Positive) Eliminated long-context surcharge: 200K+ inputs dropped from $10 to $5/MTok |
| P5 | **GitHub Copilot** | [github.blog](https://github.blog/changelog/2026-04-16-claude-opus-4-7-is-generally-available/) | 4.7 on Copilot: 7.5x premium request multiplier (promotional through April 30) |
