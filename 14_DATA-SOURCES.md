# Data Sources & Labeling

> **Why this doc exists.** Earlier revisions of this repo described the dataset as "ubuntu-1, single machine, 1,735 JSONL files." A re-audit on April 16 found that the actual data assets span three labeled environments, and the figures previously published correspond to a snapshot subset of the largest one. This document declares the full set of sources, the labels used for cross-analysis, and the relationship between current figures and historical snapshots.

## 1. Environment labels

Every dataset is tagged with the following dimensions. All downstream analyses are expected to filter on these labels explicitly.

| field | values |
|---|---|
| `machine` | `zbook` \| `win-1` \| (future: other lab machines) |
| `account_tier` | `max20` \| `max5` \| (future: `pro`, `api`) |
| `cc_mode` | `stock` (native `~/.claude`) \| `override` (isolated override environment with a GrowthBook flag override; additional components kept private) |
| `cc_version` | e.g. `v2.1.91`, `v2.1.109`, `v2.1.110` — representative version, actual value varies over the dataset timespan |
| `proxy_stack` | array of active proxy/instrumentation layers on the **public** side (e.g. `cc-relay`, `claude-code-cache-fix`). Any private components are recorded internally but not published. |

## 2. Current datasets (as of 2026-04-22)

| dataset | machine | tier | mode | version | proxy_stack | JSONL files | messages | proxy requests | size |
|---|---|---|---|---|---|---|---|---|---|
| `ubuntu-1-stock` | zbook | max20 | stock | v2.1.91 | cc-relay | 2,098 | 246,954 | 45,884 (cc-relay) | 911 MB |
| `ubuntu-1-override` | zbook | max20 | override | v2.1.109 | (private operational tooling — not disclosed) | 2,324 | 263,630 | (follow-up) | 948 MB |
| `win-1-stock` | win-1 | max5 | stock | v2.1.110 | transparent proxy + `claude-code-cache-fix` 1.10.0 (public) | 171 | 1,565 | (follow-up) | 5.7 MB |

**Totals: 4,593 JSONL files / 512,149 messages / 45,884 proxy requests / ~1.9 GB**

### 2.1 Notes per dataset

- **`ubuntu-1-stock`** — the historical baseline. All figures in earlier revisions of this repo originate from this dataset, filtered by date range (April 1–15 for proxy, selected session subsets for JSONL bulk scans).
- **`ubuntu-1-override`** — same machine and account, but an isolated override environment where a GrowthBook flag override has been active since April 10 (per [01_BUGS.md](01_BUGS.md#growthbook-flag-override--controlled-elimination-test-april-1014)). This environment also contains additional private components that are tracked internally for operational purposes but are out of scope for this repository. Used for the B4/B5 elimination test.
- **`win-1-stock`** — a separate Windows 11 machine running a distinct Max 5x account. Used for research and validation (version fingerprint checks, proxy compatibility, independent testing of public tools such as [`claude-code-cache-fix`](https://github.com/cnighswonger/claude-code-cache-fix)). **Intentionally excluded** from the main publishable body of this repo so that the Max 20x ubuntu-1 analysis remains controlled.

### 2.1.1 Environment update — May 13, 2026

`ubuntu-1-override` was upgraded on 2026-05-12 from v2.1.109 + claude-opus-4-6 to v2.1.139 + claude-opus-4-7, running through ~May 26 as a stability check against the [Opus 4.7 advisory](16_OPUS-47-ADVISORY.md). Earlier rows in the dataset table stay as they were — they record state at the time of each analysis, not current state. Sessions on `ubuntu-1-override` after 2026-05-12 carry the new version and model identifiers.

One adjacent thing worth recording: v2.1.132 added `CLAUDE_CODE_SESSION_ID`, which propagates the session ID through child processes. Cleaner cross-session join in any new proxy collection sweep.

### 2.2 Shared-storage caveat

Both `~/.claude/projects/` (stock) and `an isolated override environment` (override env) on ubuntu-1 contain JSONL entries dating back to **2026-02-08**. This is because the two directories were historically split from a shared history; the early entries exist in both. For environment-effect analyses (e.g. comparing pre/post override), filter by `ts >= '2026-04-10'` to isolate the period where the two environments diverged meaningfully.

## 3. Reconciliation with previously published figures

| figure as published | corresponds to | current equivalent |
|---|---|---|
| "1,735 JSONL files, 1.0 GB, single machine" ([`03_JSONL-ANALYSIS.md`](03_JSONL-ANALYSIS.md)) | a snapshot of `ubuntu-1-stock` at the time of that section's authoring | now 2,098 files / 911 MB in `ubuntu-1-stock` (still growing) |
| "35,554 requests April 1–15, 251 sessions" ([`13_PROXY-DATA.md`](13_PROXY-DATA.md)) | `ubuntu-1-stock` cc-relay proxy, April 1–15 slice | extended to **45,884 requests / 320 sessions / April 1–22** under `ubuntu-1-stock` (cc-relay source) |
| "532 JSONL files, 158.3 MB (April 1–8)" ([`06`, `13 §12`]) | `ubuntu-1-stock` bulk scan snapshot | historical; not re-run. Current `ubuntu-1-stock` corpus is larger (2,098 files), but the 532-file scan remains authoritative for the metrics computed at that time |
| "4,919 requests, zero B4/B5 after override" ([`01_BUGS.md`](01_BUGS.md)) | `ubuntu-1-stock` cc-relay proxy, April 11–14, override window | unchanged historically. The override continues; post-April 14 data, where relevant to the override measurement, is tracked against `ubuntu-1-override` |

**The historical snapshot numbers are not retracted** — they describe the state of the data at the time each analysis was written and remain reproducible from the underlying sessions. What is corrected here is the labeling: the machine/environment descriptor "single machine" was imprecise and is now replaced by the explicit `ubuntu-1-stock` label.

## 4. Storage layout

Since April 16, all three datasets above are also indexed in an internal Postgres database. Five tables keep the source files intact on disk and index metadata, parsed per-turn messages, proxy events, and flag snapshots for SQL-level cross-analysis. ETL scripts are idempotent — dataset reload is safe.

## 5. What this changes for the existing chapters

- **[`03_JSONL-ANALYSIS.md`](03_JSONL-ANALYSIS.md)** — footer environment descriptor is being updated to `ubuntu-1-stock` explicitly, with the snapshot-vs-current caveat pointed at this document.
- **[`13_PROXY-DATA.md`](13_PROXY-DATA.md)** — headline proxy totals refreshed to the current cc-relay figures (45,884 requests / 320 sessions, April 1–22). The detailed tables in §5–§11, which were built against specific time slices, retain their original figures and are annotated accordingly.
- **[`02_RATELIMIT-HEADERS.md`](02_RATELIMIT-HEADERS.md)** — environment footer updated, total request count refreshed.
- **[`README.md`](README.md)** — overview totals updated.
- **[`15_ENV-BREAKDOWN.md`](15_ENV-BREAKDOWN.md)** — new chapter: per-environment cache_read ratios (pre/post April 10, daily trend around the cut), Max 20x vs Max 5x model dispatch comparison, tier-dependent Haiku share finding.

Any new analysis published from this point forward is expected to name the dataset(s) it was computed on, using the labels in §1.

---

*Environment summary: ubuntu-1 (Linux) with two CC modes (`stock`, `override`) both on Max 20x; secondary win-1 (Windows 11) on Max 5x for validation. See §2 for the full matrix.*
