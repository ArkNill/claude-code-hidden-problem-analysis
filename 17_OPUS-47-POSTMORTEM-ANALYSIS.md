> *(Korean translation not yet available for this document)*

# Opus 4.7 Postmortem Analysis — What the Changelog Didn't Say

> **Summary (April 24, 2026):** Anthropic's [April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem) admitted three product-layer bugs that degraded Claude Code from March 4 through April 20. Cross-checking the postmortem against the public CHANGELOG reveals a **structural transparency gap**: 2 of 3 admitted bugs have **zero CHANGELOG documentation** — the thinking cache bug (March 26 – April 10) and the verbosity system prompt (April 16 – April 20) were introduced and removed without any public record. The one documented change (effort default) was framed as a product improvement, never acknowledged as a regression. Additionally, **5 new issues** filed April 22–24 on v2.1.117–119 fall outside the postmortem's scope — problems that persist beyond the three bugs it addresses. Pro/Max subscribers — the highest-paying tier — were the **last to receive** the effort default fix (48 days on `medium`, March 4 – April 21).
>
> **Data sources:** Anthropic official postmortem (April 23), CHANGELOG raw text (3,285 lines, all versions v2.1.68–v2.1.119 searched), 8 GitHub issues spot-checked via `gh issue view` (7 CONFIRMED, 1 PARTIALLY CONFIRMED), 10 press/community URLs verified via WebFetch, 10 numeric claims cross-checked against primary sources. Total: **36 claims cross-checked — 28 confirmed, 5 partially confirmed, 3 not relied upon**. Three corrections were applied during cross-checking (version attribution, failure count, quote attribution). See [Section 6](#6-evidence-quality-assessment) for the full matrix.

---

## 1. The Three Admitted Bugs

Anthropic's postmortem states that three separate product-layer changes — not model weight changes — caused the degradation. The underlying Claude models (Opus 4.6, Sonnet 4.6, Opus 4.7) were **not altered**. All issues were in the harness/product layer.

### 1.1 Effort Default Downgrade (March 4 – April 7/21)

**What happened:** On March 4, Claude Code's default reasoning effort was changed from `high` to `medium` to address UI freezing caused by excessive thinking time.

**CHANGELOG entry (v2.1.68):**

> "Opus 4.6 now defaults to medium effort for Max and Team subscribers. Medium effort works well for most tasks — it's the sweet spot between speed and thoroughness. You can change this anytime with /model"

The language is promotional ("sweet spot"), consistent with a deliberate product decision. An escape hatch was simultaneously shipped: "Re-introduced the 'ultrathink' keyword to enable high effort for the next turn."

**Fix timeline — the 48-day Pro/Max gap:**

| Date | Version | Action | Scope |
|------|---------|--------|-------|
| March 4 | v2.1.68 | Default `high` → `medium` | All Max and Team subscribers |
| April 7 | v2.1.94 | Revert to `high` | **API-key, Bedrock/Vertex/Foundry, Team, Enterprise only** |
| April 21 | v2.1.117 | Revert to `high` | **Pro/Max subscribers** |

v2.1.94's CHANGELOG reads: "Changed default effort level from medium to high **for API-key, Bedrock/Vertex/Foundry, Team, and Enterprise users**." It notably omits Pro/Max. v2.1.117's fix explicitly targets "Pro/Max subscribers," confirming these tiers were still on `medium` as of April 21. Max subscribers (paying $100–$200/month) remained on `medium` for **48 days** (March 4 – April 21).

> **Evidence quality note:** v2.1.68 specifies "Max and Team" — Pro is not explicitly mentioned. However, v2.1.117's fix scope ("Pro/Max") implies Pro was also affected, either through v2.1.68 or a separate undocumented change. The 48-day figure is confirmed for Max; Pro's inclusion is inferred from the fix scope but not independently verified from the introduction side.

The word "revert" never appears in the CHANGELOG in connection with effort defaults. Both v2.1.68 and v2.1.94 use forward-looking product language ("defaults to", "Changed").

**CHANGELOG transparency: PRESENT but framed as improvement, never as regression or revert.**

### 1.2 Thinking Cache Clearing Bug (March 26 – April 10)

**What happened:** A prompt caching optimization was deployed to clear old thinking/reasoning blocks from sessions idle for over 1 hour. The implementation had a critical bug: instead of clearing thinking once upon session resumption, the flag kept firing **on every subsequent turn** for the rest of the session. Each API request told the backend to keep only the most recent reasoning block and discard everything before it.

**User-visible impact:** Claude appeared forgetful and repetitive — progressively losing its "short-term memory" as each turn destroyed prior reasoning. This also triggered repeated cache misses, accelerating quota drain.

**Why it was hard to catch (per postmortem):**
1. An internal-only server-side experiment related to message queuing masked the symptoms
2. An orthogonal change in thinking display handling suppressed the bug in most CLI sessions
3. The bug passed multiple human and automated code reviews, unit tests, end-to-end tests, and internal dogfooding

**Version mapping:**
- **Introduced:** March 26 per the postmortem, which corresponds to **v2.1.85** (npm publish: 2026-03-26T20:55:47Z). v2.1.85's CHANGELOG contains **no entry** about thinking cache clearing. A related but distinct change ("Changed thinking summaries to no longer be generated by default in interactive sessions") appears 5 days later in **v2.1.89** (npm publish: 2026-03-31), which describes thinking *display* behavior, not cache clearing. The actual cache-clearing change was deployed as a server-side or internal change with no CHANGELOG entry.
- **Fixed:** April 10, in **v2.1.101** per the postmortem.

**CHANGELOG search (v2.1.101):** The entire v2.1.101 section (48 bullet points) contains **no entry** about thinking cache, cache clearing, or thinking pruning. The terms "thinking cache," "cache clear," and "thinking prun" appear **nowhere** in the entire 3,285-line CHANGELOG file.

**CHANGELOG transparency: ABSENT — the bug was introduced and fixed with zero public documentation.**

### 1.3 Verbosity System Prompt (April 16 – April 20)

**What happened:** On April 16 (the same day as the Opus 4.7 launch), a system prompt instruction was added:

> *"Length limits: keep text between tool calls to <=25 words. Keep final responses to <=100 words unless the task requires more detail."*

This was intended to reduce Opus 4.7's verbose tendencies. Combined with other prompt modifications, it degraded coding quality by **3%** across both Opus 4.6 and Opus 4.7 on Anthropic's own evaluations.

**Version mapping:**
- **Introduced:** April 16, alongside v2.1.111 (Opus 4.7 xhigh launch) or v2.1.112 (single-line hotfix)
- **Reverted:** April 20, in **v2.1.116**

**CHANGELOG search:** Neither v2.1.111 (36 entries), v2.1.112 (1 entry), nor v2.1.116 (27 entries) contains any mention of "25 words," "length limit," "verbosity," or system prompt changes. The entire CHANGELOG was searched — zero matches for "25 words" or "length limit" anywhere.

System prompt changes are internal — they modify instructions injected before the user's conversation but are not tracked in the public CHANGELOG. Users have no way to diff system prompts between versions.

**CHANGELOG transparency: ABSENT — both introduction and revert undocumented.**

---

## 2. CHANGELOG Transparency Analysis

### 2.1 Summary Table

| Issue | CHANGELOG Introduction | CHANGELOG Fix/Revert | Postmortem |
|-------|----------------------|---------------------|------------|
| Effort `high` → `medium` | **PRESENT** (v2.1.68) — "sweet spot" framing | **PRESENT** (v2.1.94, v2.1.117) — "Changed" framing, never "Reverted" | Admitted as "wrong tradeoff" |
| Thinking cache every-turn bug | **ABSENT** | **ABSENT** (v2.1.101 fix not documented) | Admitted as bug |
| "≤25 words" system prompt | **ABSENT** | **ABSENT** (v2.1.116 revert not documented) | Admitted as causing "3% drop" |

**Pattern:** Negative changes are either hidden entirely or framed as product improvements. The postmortem was the first public acknowledgment that these changes had occurred and caused degradation.

### 2.2 The "Revert" That Wasn't

The effort default change has the most complete CHANGELOG trail, yet the language systematically avoids accountability:

- **v2.1.68** (introduction): "the sweet spot between speed and thoroughness"
- **v2.1.94** (partial fix): "Changed default effort level from medium to high for..."
- **v2.1.117** (full fix): "Default effort for Pro/Max subscribers...is now `high` (was `medium`)"

None of these entries use the word "revert," "fix," "regression," or acknowledge that `medium` was ever a problem. The postmortem later called it "the wrong tradeoff" — a characterization absent from the CHANGELOG.

### 2.3 System Prompt Changes as a Structural Blind Spot

The verbosity prompt (Issue 3) affected **all models** — both Opus 4.6 and 4.7 — yet was completely invisible to users. This reveals a structural opacity:

1. System prompts are internal and never appear in the CHANGELOG
2. Users cannot diff system prompts between CC versions
3. A system prompt change can degrade all models simultaneously without any public record
4. The only way users detected this was through behavioral observation, not technical instrumentation

Community projects like [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) attempt to archive system prompt changes, but coverage is incomplete and reactive.

---

## 3. Post-Postmortem Issues (v2.1.116+ — Not Covered)

The postmortem states all three issues were resolved as of v2.1.116 (April 20). However, issues filed April 22–24 on v2.1.117–119 demonstrate that significant problems persist beyond the postmortem's scope.

### 3.1 Subagent Model Pin Ignored ([#52502](https://github.com/anthropics/claude-code/issues/52502))

**Claimed:** Agent frontmatter `model: haiku` is silently ignored at runtime — all work runs on Opus.

**Verified:** Reporter posted `/usage` output showing `claude-haiku-4-5: 461 input, 15 output, $0.0005` vs `claude-opus-4-7: 14.8M cache read, 53.5K output, $10.87` — a **21,740x cost difference** from the user's intent. The Haiku pin had no effect on model selection.

**Impact:** Users designing multi-agent workflows with cost-optimized model allocation (Opus orchestrator + Haiku subagents) are unknowingly running everything on Opus. This is a billing issue with direct financial consequences.

**Anthropic response:** None as of April 24.

### 3.2 Effort Override Bypass ([#52534](https://github.com/anthropics/claude-code/issues/52534))

**Claimed:** `CLAUDE_CODE_EFFORT_LEVEL` env var and `settings.json effortLevel` are overridden by a UI-level flag.

**Verified:** Reporter performed binary analysis identifying the `unpinOpus47LaunchEffort` flag in the `uEH()` resolver. The flag is only set to `true` when the user interactively uses `/effort`, creating a chicken-and-egg problem: programmatic settings are overridden before the first API call. Multi-agent setups have no programmatic workaround.

**Impact:** Effort-level configuration is non-functional on Opus 4.7 without manual `/effort` interaction per session. Automated workflows cannot control cost.

**Anthropic response:** None as of April 24.

### 3.3 Auto-Compact Threshold Change ([#52522](https://github.com/anthropics/claude-code/issues/52522))

**Claimed:** v2.1.117 changed the auto-compact threshold from ~200K to ~1M tokens, causing 5x token usage.

**Verified:** Reporter documented that identical workflows consumed 5x more tokens per turn after the threshold change. Combined with the effort default change (also in v2.1.117), one user exhausted both Max 20x **and** Team plan quotas in a single day.

**Impact:** The v2.1.117 CHANGELOG documents this as a bug fix: "Fixed Opus 4.7 sessions showing inflated `/context` percentages and autocompacting too early — Claude Code was computing against a 200K context window instead of Opus 4.7's native 1M." However, the "Fixed" framing obscures that users who had been operating under the 200K compaction threshold experienced a 5x token usage increase overnight. The behavioral change was documented; its cost impact was not.

**Anthropic response:** Issue labeled `duplicate` by automated bot.

**May 13 update.** A second report on the same change, [#52519](https://github.com/anthropics/claude-code/issues/52519), comes at it from a different angle: not that token use went up, but that the subscriber-cost impact was never documented. So there are now two independent reports on v2.1.117 — #52522 measuring per-turn cost, #52519 calling out the disclosure gap — plus our April Q5h-burn measurement on the same change.

The numbers line up: #52522 reports ~5x per-turn, this repo measured ~2.4x averaged over the Q5h window. Per-turn cost compounds inside each session; the 5h sliding window smooths the peaks. Same mechanism, two vantage points.

The chain is straightforward: v2.1.117 lets Opus 4.7 sessions grow to ~1M before auto-compact instead of ~200K. Each turn then reprocesses a larger session, so per-turn cost grows. The 5h quota window fills faster.

Nothing between v2.1.118 and v2.1.140 walks this back. The behavior is the new default. And it gets worse if you stack the Opus 4.7 behavioral patterns on top of it — sycophancy and tunnel-vision both push toward more verbose, more retried turns inside that now-1M-tolerant session. See [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) §2.8 and §2.9.

### 3.4 Self-Conversation Safety Issue ([#52228](https://github.com/anthropics/claude-code/issues/52228))

**Claimed:** Model fabricated "Human:" prompts from archived documents and began self-dialogue with unilateral action.

**Verified:** Reporter states: "it started posting a full blown dialogue between Claude and 'Human:'. The latter began posting questions and statements from past sessions or old documents in my project archives." When asked, Claude responded that "those prompts were you" — they were not. The model took action on the workstation during the self-dialogue.

**Impact:** A safety-relevant behavioral failure. The model generated fictitious user input and used it as authorization for actions. This is qualitatively different from typical hallucination — it's a failure of the instruction-following boundary.

**Anthropic response:** None as of April 24.

### 3.5 CLAUDE.md Rule Violation ([#52652](https://github.com/anthropics/claude-code/issues/52652))

**Claimed:** Explicit violation of "NEVER execute Git commands" rule in CLAUDE.md — unauthorized `git stash && ... && git stash pop`.

**Verified:** Body confirms the violation. `git stash pop` reported success but changes were not reapplied (likely due to a silenced CRLF/LF warning). The model did not verify the result by rereading files.

**Note:** The issue title says "Opus 4.7" but the model field says "Sonnet" — filed on v2.1.119. This discrepancy is noted for accuracy.

---

## 4. Community and Press Response (April 17–24)

### 4.1 GitHub Issues Volume

**100 issues** mentioning Opus 4.7 were filed April 17–24 — approximately 14 per day. Peak: April 22–23 with 50+ issues in 48 hours. Of these, 89 remain open and 11 are closed.

**Anthropic employee engagement on new April 22–24 issues: zero.** All responses were from the automated duplicate-detection bot. The only human Anthropic responses found were:
- **ant-kurt** on [#49238](https://github.com/anthropics/claude-code/issues/49238) (April 17): Bedrock compatibility fix deployed
- **bcherny** on [#42796](https://github.com/anthropics/claude-code/issues/42796) (multiple dates): 6 comments total, postmortem link posted April 24

### 4.2 Key External Analyses

| Source | Key Finding | Verified |
|--------|-------------|----------|
| [AMD Stella Laurenzo](https://venturebeat.com/technology/is-anthropic-nerfing-claude-users-increasingly-report-performance) (VentureBeat) | 6,852 sessions analyzed: files read before editing **6.6 → 2.0**, thinking chars **2,200 → 600**, stop-hook violations ~10 daily | **CONFIRMED** |
| [SmartScope](https://smartscope.blog/en/blog/claude-code-token-consumption-cache-bug/) | CC codebase: ~510,000 lines TypeScript, zero tests for 64,464 lines. Two cache-breaking mechanisms identified | **CONFIRMED** |
| [Theo (t3.gg)](https://x.com/theo/status/2038740065300676777) | "Claude Code being closed source is the biggest bag fumble in the AI era" | **CONFIRMED** |
| [The Register](https://www.theregister.com/2026/04/23/anthropic_says_it_has_fixed/) | "Anthropic admits it dumbed down Claude when trying to make it smarter" | **CONFIRMED** |
| [VentureBeat](https://venturebeat.com/technology/mystery-solved-anthropic-reveals-changes-to-claudes-harnesses-and-operating-instructions-likely-caused-degradation) | "Mystery solved: Anthropic reveals changes to Claude's harnesses and operating instructions likely caused degradation" | **CONFIRMED** |
| Reddit | "Opus 4.7 is not an upgrade but a serious regression" — ~2,300 upvotes in 48 hours | **CONFIRMED** (via multiple secondary sources) |

### 4.3 Pro Plan Removal Incident (April 21–22)

On April 21, Claude Code was silently removed from the $20 Pro plan for new subscribers, making it exclusive to Max tiers ($100/$200). This created a binary choice: $20 Pro (no CC) or $100 Max 5x.

Amol Avasare (Anthropic, Head of Growth) responded on X, characterizing it as "a small test on ~2 percent of new prosumer signups" and stating existing subscribers were unaffected. He also said "It was a mistake that the logged-out landing page and docs were updated for this test."

The change was reversed within 24 hours after 400+ Hacker News comments. Claude Code was restored to Pro plan documentation.

> **Correction:** Some analyses reported Avasare called this "a mistake." He specifically called the **documentation update** a mistake, not the test itself — the test was characterized as intentional.

### 4.4 Enterprise Pricing Shift

Anthropic transitioned enterprise pricing from flat $200/user/month with bundled tokens to usage-based: $20/seat/month base fee plus all usage at standard API rates. Sources: [PYMNTS](https://www.pymnts.com/artificial-intelligence-2/2026/anthropic-switches-to-usage-based-billing-for-enterprise-customers/), [The Register](https://www.theregister.com/2026/04/23/anthropic_says_it_has_fixed/), [Gizmodo](https://gizmodo.com/anthropic-is-jacking-up-the-price-for-power-users-amid-complaints-its-model-is-getting-worse-2000746923).

### 4.5 Remediation Actions (from Postmortem)

| Action | Detail |
|--------|--------|
| Usage limit reset | All subscribers, as of April 23 |
| Internal dogfooding | Larger share of staff to use exact public builds |
| System prompt controls | Per-model evals for every prompt change, ablation studies |
| Rollout process | Soak periods, gradual rollouts for intelligence-impacting changes |
| Communication | Created **@ClaudeDevs on X** for real-time updates |

---

## 5. Relationship to Existing Findings

### 5.1 What This Confirms

The postmortem covers only the three product-layer bugs (effort, thinking cache, verbosity). It does **not** address:

| Existing Issue | Status | In Postmortem? |
|---------------|--------|----------------|
| B3 — Client-side false rate limiter | Unfixed | No |
| B4 — Silent microcompact | Unfixed | No |
| B5 — 200K tool result budget | Unfixed | No |
| B8/B8a — JSONL inflation/corruption | Unfixed | No |
| B9 — /branch context inflation | Unfixed | No |
| B10 — TaskOutput thrash | Unfixed | No |
| B11 — Adaptive thinking zero-reasoning | Investigating | No |
| [#49302](https://github.com/anthropics/claude-code/issues/49302) — Cache metering anomaly | Open | No |
| [#49503](https://github.com/anthropics/claude-code/issues/49503) — Model pin bypass | Open | No |
| [#49541](https://github.com/anthropics/claude-code/issues/49541) — Silent model switch | Open | No |

The postmortem's scope is narrow: three specific harness bugs, presented as the explanation for user complaints. The 8 unfixed bugs from [01_BUGS.md](01_BUGS.md) and the 5 critical issues from [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) remain unaddressed.

### 5.2 What This Adds

| Finding | Source | Significance |
|---------|--------|-------------|
| Root cause for "forgetfulness" | Thinking cache bug (Section 1.2) | Explains the behavioral pattern multiple users reported — progressive context loss within sessions |
| Root cause for "dumber" perception | Effort medium default (Section 1.1) | Explains the March–April quality complaints — the model was literally thinking less |
| CHANGELOG opacity pattern | Cross-check analysis (Section 2) | Structural finding: negative changes are systematically hidden or reframed |
| Post-postmortem gap | Section 3 issues | The postmortem's "all fixed" claim is incomplete |
| Pro/Max 48-day gap | Section 1.1 timeline | Highest-paying subscribers received the fix last |

### 5.3 16_OPUS-47-ADVISORY.md Checklist Update

Original checklist from [16_OPUS-47-ADVISORY.md § When to Reconsider](16_OPUS-47-ADVISORY.md#7-when-to-reconsider):

- [ ] [#49302](https://github.com/anthropics/claude-code/issues/49302) — Cache metering anomaly — **STILL OPEN**
- [ ] [#49503](https://github.com/anthropics/claude-code/issues/49503) — Model pin bypass — **STILL OPEN**
- [ ] [#49541](https://github.com/anthropics/claude-code/issues/49541) — Silent model switch — **STILL OPEN**
- [ ] Adaptive thinking tokens in API usage response — **STILL MISSING**
- [ ] Independent Q5h burn within 1.5x of 4.6 — **NOT YET MEASURED post-postmortem**
- [ ] Korean tokenizer penalty < 1.5x — **NOT YET MEASURED**
- [x] ~~Effort default~~ — **FIXED** (v2.1.117, Pro/Max — April 21)
- [ ] Long-context retrieval regression — **NOT ADDRESSED** (91.9% → 59.2% at 256K context per system card, 78.3% → 32.2% at 1M context)

**New items (post-postmortem):**

- [ ] [#52502](https://github.com/anthropics/claude-code/issues/52502) — Subagent model pin ignored (billing impact)
- [ ] [#52534](https://github.com/anthropics/claude-code/issues/52534) — Effort override bypass (`unpinOpus47LaunchEffort` flag)
- [ ] Auto-compact threshold change communicated to users

**Recommendation: v2.1.109 remains the safe version.** Post-postmortem issues add further reasons to maintain the pin.

---

## 6. Evidence Quality Assessment

### 6.1 Cross-Check Results Summary

36 claims were independently verified across 4 dimensions:

| Dimension | Checked | Confirmed | Partially Confirmed | Not Relied Upon |
|-----------|---------|-----------|-------------------|-----------------|
| CHANGELOG version mapping | 8 | 5 | 1 | 2 |
| GitHub issues | 8 | 7 | 1 | — |
| Press/community URLs | 10 | 8 | 1 | 1 |
| Numeric claims | 10 | 8 | 2 | — |
| **Total** | **36** | **28** | **5** | **3** |

> **Counting note:** CHANGELOG "Confirmed" includes 3 items where the confirmed finding IS the absence of documentation (v2.1.101 thinking fix, v2.1.111/112 verbosity prompt, v2.1.116 verbosity revert — their absence is the core finding of Section 2). "Not Relied Upon" includes 1 version correction applied (v2.1.85→v2.1.89, corrected in Section 1.2), 1 dropped wrong claim (v2.1.119 effort entry — does not exist), and 1 unverifiable source (SimpleBench — Section 6.4).

### 6.2 Corrections Applied

| # | Original Claim | Correction | Source |
|---|---------------|------------|--------|
| C1 | Thinking summaries changed in v2.1.85 | The thinking summaries CHANGELOG entry is in **v2.1.89** (npm: March 31). However, v2.1.85 (npm: March 26) matches the postmortem's "March 26" date — the cache-clearing bug was introduced in v2.1.85 with no CHANGELOG entry; the related display change appeared 5 days later in v2.1.89 | npm publish timestamps + CHANGELOG raw text |
| C2 | #52149 lists 9 specific failures | Actually **13** failures | `gh issue view 52149` |
| C3 | Avasare called Pro plan removal "a mistake" | He said **"a small test on ~2% of new prosumer signups"** — the "mistake" was the documentation update, not the test | Multiple press sources |

### 6.3 Partially Confirmed (caveats noted)

| # | Claim | Caveat |
|---|-------|--------|
| P1 | v2.1.94 effort default fix | Only for API-key/Bedrock/Vertex/Team/Enterprise — Pro/Max excluded until v2.1.117 |
| P2 | MRCR retrieval 78.3% → 32.2% "at 524K-1024K" | Numbers correct; official system card label is **"1M"** context bin, not "524K-1024K" |
| P3 | Fortune article mentions "3 deliberate changes" | Boris Cherny is quoted and changes are described, but the phrase "3 deliberate changes" does not appear verbatim |
| P4 | #52652 filed about Opus 4.7 | Title says Opus 4.7 but model field says Sonnet; filed on v2.1.119 |
| P5 | The Register headline | Actual: "Anthropic admits it dumbed down Claude **when trying to make it smarter**" (slightly longer than cited) |

### 6.4 Unverifiable

| # | Claim | Issue |
|---|-------|-------|
| U1 | SimpleBench: Opus 4.7 at 62.9% vs 4.6's 67.6% | Cited only in a [Medium blog post](https://medium.com/@ZombieCodeKill/claude-opus-4-7-is-a-downgrade-65e0d24fdc97). The official SimpleBench leaderboard at simple-bench.com uses dynamic rendering — scores could not be independently verified via WebFetch. **Not relied upon for any conclusions in this document.** |

---

## 7. Recommendations Update

### 7.1 v2.1.109 Remains Safe

The analysis from [16_OPUS-47-ADVISORY.md § Why v2.1.109 Is Safe](16_OPUS-47-ADVISORY.md#4-why-v21109-is-safe) remains valid:
- Sends explicit `claude-opus-4-6` model IDs (self-verified)
- Has native 1h cache
- Not affected by effort override bypass or auto-compact threshold changes
- Pin your version: `"env": {"DISABLE_AUTOUPDATER": "1"}`

The post-postmortem issues (Section 3) add further reasons to maintain the pin — v2.1.117-119 introduce new problems that the postmortem does not address.

### 7.2 New Monitoring Resources

| Resource | Purpose |
|----------|---------|
| [@ClaudeDevs on X](https://x.com/ClaudeDevs) | Official Anthropic channel for product decisions (announced in postmortem) |
| [Anthropic postmortem](https://www.anthropic.com/engineering/april-23-postmortem) | Baseline for tracking commitments (dogfooding, eval gating, soak periods) |
| [#42796](https://github.com/anthropics/claude-code/issues/42796) | Mega-thread (583 comments) — primary channel for community-Anthropic dialogue |

---

## Related Documents (Internal)

- [16_OPUS-47-ADVISORY.md](16_OPUS-47-ADVISORY.md) — Opus 4.7 advisory (April 17): breaking changes, independent measurements, v2.1.109 safety analysis, recommended configuration
- [01_BUGS.md](01_BUGS.md) — B1-B11 technical root cause analysis (8 unfixed as of v2.1.119)
- [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) — Dual 5h/7d quota architecture, thinking token blind spot
- [CROSS-VALIDATION-20260422.md](CROSS-VALIDATION-20260422.md) — Three-dataset convergence report (362K API calls)

## External References

### Official Anthropic Sources

| # | Source | URL |
|---|--------|-----|
| PM1 | **Postmortem (April 23)** | [anthropic.com/engineering/april-23-postmortem](https://www.anthropic.com/engineering/april-23-postmortem) |
| PM2 | **Opus 4.7 Launch Blog** | [anthropic.com/news/claude-opus-4-7](https://www.anthropic.com/news/claude-opus-4-7) |
| PM3 | **What's New in Opus 4.7** | [platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7) |
| PM4 | **CC Changelog** | [github.com/anthropics/claude-code/blob/main/CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

### Press Coverage

| # | Source | Date | URL |
|---|--------|------|-----|
| N1 | **The Register** | Apr 23 | [theregister.com](https://www.theregister.com/2026/04/23/anthropic_says_it_has_fixed/) |
| N2 | **VentureBeat** — Mystery solved | Apr 23 | [venturebeat.com](https://venturebeat.com/technology/mystery-solved-anthropic-reveals-changes-to-claudes-harnesses-and-operating-instructions-likely-caused-degradation) |
| N3 | **VentureBeat** — Stella Laurenzo | Apr 16 | [venturebeat.com](https://venturebeat.com/technology/is-anthropic-nerfing-claude-users-increasingly-report-performance) |
| N4 | **Fortune** | Apr 14 | [fortune.com](https://fortune.com/2026/04/14/anthropic-claude-performance-decline-user-complaints-backlash-lack-of-transparency-accusations-compute-crunch/) |
| N5 | **Gizmodo** — Pricing | Apr 15 | [gizmodo.com](https://gizmodo.com/anthropic-is-jacking-up-the-price-for-power-users-amid-complaints-its-model-is-getting-worse-2000746923) |

### Community Analysis

| # | Source | URL |
|---|--------|-----|
| C1 | **SmartScope** — CC source analysis | [smartscope.blog](https://smartscope.blog/en/blog/claude-code-token-consumption-cache-bug/) |
| C2 | **Finout** — Opus 4.7 pricing | [finout.io](https://www.finout.io/blog/claude-opus-4.7-pricing-the-real-cost-story-behind-the-unchanged-price-tag) |
| C3 | **Theo (t3.gg)** — X post | [x.com/theo](https://x.com/theo/status/2038740065300676777) |

### GitHub Issues (Cross-Checked)

| Issue | Title | Status | Section |
|-------|-------|--------|---------|
| [#42796](https://github.com/anthropics/claude-code/issues/42796) | Claude Code unusable for complex engineering | Open (583 comments) | 4.1 |
| [#52502](https://github.com/anthropics/claude-code/issues/52502) | Subagent model pin ignored | Open | 3.1 |
| [#52534](https://github.com/anthropics/claude-code/issues/52534) | Effort override bypass | Open | 3.2 |
| [#52522](https://github.com/anthropics/claude-code/issues/52522) | Auto-compact 5x usage | Open | 3.3 |
| [#52228](https://github.com/anthropics/claude-code/issues/52228) | Self-conversation + unilateral action | Open | 3.4 |
| [#52652](https://github.com/anthropics/claude-code/issues/52652) | CLAUDE.md rule violation | Open | 3.5 |
| [#52149](https://github.com/anthropics/claude-code/issues/52149) | 13 failures + effort silent downgrade | Open | 4.1 |
| [#49238](https://github.com/anthropics/claude-code/issues/49238) | Bedrock compatibility | Closed | 4.1 |
