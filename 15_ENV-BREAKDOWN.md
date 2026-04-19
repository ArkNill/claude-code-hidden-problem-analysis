# Environment Breakdown — cache_read, model dispatch, and tier-dependent behaviour across three datasets

> **Companion doc** to [14_DATA-SOURCES.md](14_DATA-SOURCES.md). Where 14 declares the datasets, this chapter slices the measurements across them. All figures come from an internal Postgres database, computed on April 16, 2026.

## TL;DR

1. The cache_read dominance reported throughout this repo (~95–97% of quota-weighted tokens at the assistant turn) reproduces cleanly across all three datasets where the sample is large enough. This is not a single-machine artefact.
2. Post-April 10, **cache_read ratio on `ubuntu-1-override` rises to 97.08%** (vs 95.40% pre-4/10) — a +1.68 percentage-point shift. On `ubuntu-1-stock` over the same period the shift is only +0.39 pp (95.61% → 96.00%). The gap (~1.1 pp) is consistent with the 1h TTL being preserved in the overridden environment while stock CC continues to churn cache_creation at the higher rate.
3. **Haiku dispatch behaves very differently on Max 5x vs Max 20x.** On the two Max 20x datasets (both ubuntu-1, stock and override env) Haiku accounts for ~21% of assistant turns. On the Max 5x dataset (win-1) Haiku is **0.11%** — a single turn out of 895. The sample on Max 5x is small, but the effect size is large enough to match independently reported observations that dispatcher behaviour depends on plan tier or load.
4. The "ubuntu-1 single machine" phrasing used in earlier revisions refers to the `ubuntu-1-stock` dataset. After April 11, the main working environment moves to `ubuntu-1-override`; `ubuntu-1-stock` sees only 17 turns across April 13–14 and none on April 15. Analyses of the current period should query the override dataset, not stock.

## 1. Dataset overview (assistant turns only)

| dataset | assistant turns | first | last | total cache_read | total cache_creation | total output | cache_read % |
|---|---:|---:|---:|---:|---:|---:|---:|
| `ubuntu-1-stock` | 138,145 | 2026-02-08 | 2026-04-15 | 14,635,731,585 | 625,160,434 | 33,562,557 | **95.62%** |
| `ubuntu-1-override` | 146,967 | 2026-02-08 | 2026-04-16 | 15,721,983,154 | 654,447,300 | 35,518,836 | **95.73%** |
| `win-1-stock` | 895 | 2026-04-07 | 2026-04-15 | 39,748,011 | 12,545,551 | 1,018,421 | 74.56% |

Across the two ubuntu-1 datasets the aggregate cache_read ratio is within 0.12 pp. The win-1 dataset is excluded from aggregate analyses because of sample size; it is discussed separately in §4 for dispatch behaviour only.

## 2. Pre- and post-April 10 shift (ubuntu-1 datasets)

On April 10 a GrowthBook flag override began operating on an isolated override environment. Native `~/.claude` (stock) continued unchanged. This gives two parallel slices.

| dataset | era | assistant turns | cache_read | cache_creation | input | output | cache_read % |
|---|---|---:|---:|---:|---:|---:|---:|
| `ubuntu-1-stock` | pre-4/10 | 133,788 | 14,029,135,100 | 602,803,911 | 10,469,358 | 31,291,628 | 95.61% |
| `ubuntu-1-stock` | post-4/10 | 4,357 | 606,596,485 | 22,356,523 | 670,293 | 2,270,929 | **96.00%** |
| `ubuntu-1-override` | pre-4/10 | 122,273 | 12,648,804,006 | 571,135,232 | 10,240,672 | 27,946,528 | 95.40% |
| `ubuntu-1-override` | post-4/10 | 24,694 | 3,073,179,148 | 83,312,068 | 1,631,261 | 7,572,308 | **97.08%** |

The post-4/10 slice on the override env is **5.7× larger in turn count than on stock** because the operator moved their daily work to an isolated override environment after the override went live (see §3 for the daily trend). Two things follow:

- The 95.40% → 97.08% jump on the override env is computed on 24,694 assistant turns over six days — sample size is not the constraint on confidence here.
- The stock 95.61% → 96.00% shift on 4,357 turns over April 10–15 represents residual activity, not the main workload, and should be read as a bound rather than a clean "no override" measurement.

The net gap — **override env 97.08% vs stock 96.00% = +1.08 pp** — is consistent with the 1h TTL being preserved under the override (fewer cache_creation events, more cache_read hits). It is not consistent with a hypothesis that the override only hides bug events without affecting accounting.

## 3. Daily cache_read % trend around the April 10 cut

ubuntu-1 only (the win-1 dataset starts April 7 with few turns and is omitted for clarity).

| day | mode | assistant turns | cache_read % |
|---|---|---:|---:|
| 2026-04-06 | both (shared) | 3,512 | 97.17% |
| 2026-04-07 | both (shared) | 5,546 | 96.16% |
| 2026-04-08 | both (shared) | 6,111 | 96.21% |
| 2026-04-09 | both (shared) | 7,457 | 97.44% |
| 2026-04-10 | stock | 4,340 | 96.02% |
| 2026-04-10 | override | 4,500 | 96.29% |
| 2026-04-11 | override | 195 | 99.10% |
| 2026-04-11 | stock | 0 | — |
| 2026-04-13 | override | 7,375 | 95.45% |
| 2026-04-13 | stock | 13 | 74.04% |
| 2026-04-14 | override | 4,383 | 97.20% |
| 2026-04-14 | stock | 4 | 32.04% |
| 2026-04-15 | override | 8,175 | **98.16%** |
| 2026-04-15 | stock | 0 | — |
| 2026-04-16 | override | 66 | 98.60% |

Two things are visible.

First, until April 10 the two modes show **identical daily counts** because the two JSONL stores (stock `~/.claude` and an isolated override directory) still share their pre-split history. From April 11 onward they diverge: `ubuntu-1-override` is where work actually happens, and `ubuntu-1-stock` is essentially inactive. Any attempt to compute a "control" from stock after April 11 fails to a small-sample artefact (e.g. 74.04% on April 13 comes from 13 turns).

Second, the override cache_read ratio reaches **98.16% on April 15** and 98.60% on the partial April 16 sample. This is the level claimed to be achievable by 1h TTL when it is actually preserved — consistent with the `tengu_prompt_cache_1h_config` allowlist match observed in the operator's environment.

## 4. Model dispatch: Max 20x vs Max 5x

Assistant turns whose `model_served` field (extracted from `message.model` in the JSONL stream) matches one of the known values.

| dataset | claude-opus-4-6 | claude-haiku-4-5 | claude-sonnet-4-6 | `<synthetic>` |
|---|---:|---:|---:|---:|
| `ubuntu-1-stock` | 108,919 (78.84%) | 28,699 (**20.77%**) | 443 (0.32%) | 84 (0.06%) |
| `ubuntu-1-override` | 114,456 (77.88%) | 31,984 (**21.76%**) | 443 (0.30%) | 84 (0.06%) |
| `win-1-stock` | 747 (83.46%) | 1 (**0.11%**) | 63 (7.04%) | 84 (9.39%) |

Three observations.

- **Haiku dispatch rate differs by plan tier** by a factor of roughly 190× in the current samples (20.77% on Max 20x stock vs 0.11% on Max 5x stock). Even granting the small Max 5x sample, the effect size is far beyond what normal dispatcher randomness would produce.
- Stock vs the override env on the same machine/account (Max 20x) differs by ~1 pp on Haiku share. The override environment does not visibly shift dispatch composition. This is consistent with the override changing context-mutation flags (B4/B5) but not the dispatcher.
- The Max 5x dataset instead shows **elevated Sonnet (7.04%) and elevated `<synthetic>` (9.39%)**. The synthetic-rate-limiting entries (B3 in this repo) appear disproportionately in the Max 5x corpus. With only 895 turns the per-turn synthetic rate is still ~9×–10× higher than on Max 20x.

This aligns with the pattern reported by third parties this week — e.g. fgrosswig's gateway forensics ([#38335](https://github.com/anthropics/claude-code/issues/38335)) observing 76 silent Opus-to-Haiku transitions per day with a 77× machine-to-machine variance, and cnighswonger reporting zero mismatches across 14K+ calls on Max 5x. The finding here is complementary: whatever the dispatcher is doing, **it treats Max 5x and Max 20x differently in how often non-Opus models appear at all**. Further data on Max 5x is needed before attributing the difference to a specific mechanism.

**Model substitution check (April 19 update):** Cross-checked request model against response model across 36,956 proxy requests on Max 20x where both fields were present — **zero mismatches** (see [13_PROXY-DATA.md §2](13_PROXY-DATA.md#model-requestresponse-verification-april-19-update)). The 20.77% Haiku dispatch on Max 20x is entirely legitimate subagent traffic. cnighswonger independently confirmed zero spoofing on Max 5x across 14,000+ calls ([Issue #4](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/4)). The tier-dependent dispatch difference is real but reflects **subagent usage patterns** — no evidence of server-side model routing manipulation in either dataset.

## 5. PRELIM/FINAL inflation status

The JSONL-level entry type ([`01_BUGS.md`](01_BUGS.md#b8), [`03_JSONL-ANALYSIS.md`](03_JSONL-ANALYSIS.md)) is currently stored as `assistant` uniformly in the internal database — the parser captures the `type` field from the top-level JSONL record, which is always `assistant` for assistant turns. Distinguishing preliminary vs final assistant entries requires the `isPartial` / `stop_reason` / message content block signals, which are preserved in the raw JSON but not yet promoted to a typed column.

Publishing an updated PRELIM/FINAL ratio across the three environments requires one additional parser pass to promote those fields. This is deferred to a follow-up — see [§7](#7-follow-ups). Existing PRELIM/FINAL figures in [`01_BUGS.md`](01_BUGS.md) and [`13_PROXY-DATA.md`](13_PROXY-DATA.md) remain the authoritative source for that metric.

## 6. Caveats and scope

- **Sample size on Max 5x is small.** The win-1 dataset has 895 assistant turns in total (April 7–15). Cache_read ratios on this dataset should not be compared directly to the Max 20x datasets; the dispatch composition finding (§4) is reported because the effect size is large, not because the sample is.
- **`ubuntu-1-stock` post-April 10 is a residual, not a control.** The operator migrated their active work to an isolated override environment on April 11. Any comparison that requires parallel activity on stock after that date does not hold; use pre-4/10 stock as the baseline instead.
- **Shared JSONL history through April 10.** The stock `~/.claude` store and the isolated override directory contain identical entries for dates before the split. This is a property of how the two environments were constructed on the operator's machine and is documented in [14_DATA-SOURCES.md §2.2](14_DATA-SOURCES.md#22-shared-storage-caveat). Queries that need true "pre-split" figures should filter on `ts < '2026-04-10'` AND choose one dataset, not both.
- **No proxy-level data for the override env or win-1 yet.** The 38,996 cc-relay proxy rows currently attached to `ubuntu-1-stock` are the only proxy-level records in the database. Intercept and telemetry streams from the override environment and the win-1 machine have not yet been ingested — see §7.

## 7. Follow-ups

- Promote `isPartial` / `stop_reason` / content-block signals from raw JSON to typed columns so the PRELIM/FINAL inflation ratio can be recomputed per dataset.
- Ingest intercept and telemetry streams from the override and win-1 environments so that flag-level dumps and per-request headers are also queryable per dataset.
- Expand the Max 5x corpus before drawing stronger conclusions on tier-dependent dispatch behaviour. A 1–2 week sample on the win-1 account would be sufficient to settle the Haiku-share question.

---

*Dataset references: see [14_DATA-SOURCES.md](14_DATA-SOURCES.md). Source queries run against an internal Postgres database, filtered by the dataset labels described in that document. All figures current as of 2026-04-16.*
