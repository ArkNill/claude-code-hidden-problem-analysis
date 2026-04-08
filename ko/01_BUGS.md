> 🇰🇷 이 문서는 [영어 원본](../01_BUGS.md)의 한국어 번역입니다.

# 버그 상세 — 기술적 근본 원인 분석

> 버그 1-2 (캐시 레이어)는 v2.1.91에서 **수정됨**. 버그 3-5 및 8은 v2.1.91 기준으로 **미수정** 상태.
>
> 버그 1-2는 커뮤니티 리버스 엔지니어링(프로그램 내부를 역추적해서 동작 원리를 파악하는 작업)을 통해 확인되었습니다 ([Reddit](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)). 버그 3-5 및 8은 2026년 4월 2-3일 프록시(API 요청을 중간에서 가로채 기록하는 도구) 기반 테스트를 통해 발견되었습니다.

---

## 버그 1 — Sentinel 치환 (standalone 바이너리 전용)

**GitHub Issue:** [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524)

standalone 바이너리(npm 없이 단독으로 설치한 실행 파일)에 내장된 Bun fork(Claude Code가 사용하는 자바스크립트 런타임의 변형판)에는 `cch=00000`이라는 sentinel(프로그램이 캐시를 식별하기 위해 사용하는 표식) 치환 메커니즘이 포함되어 있습니다. 특정 조건에서 메시지 안의 sentinel이 잘못 치환되면서 캐시 prefix(캐시로 재사용할 수 있도록 고정해둔 대화 앞부분)가 깨지고, 전체를 처음부터 다시 만들어야 하는 상황이 발생합니다.

- **v2.1.89:** 치명적 — 정상적인 경우 90% 이상이어야 하는 캐시 재사용률이 4-17%까지 떨어졌고, 복구가 불가능했습니다
- **v2.1.90:** 부분 완화 — 처음 시작(cold start) 시에는 여전히 영향이 있었지만(47-67%), 대화가 진행되면서(warming) 94-99%로 복구되었습니다
- **npm:** 영향 없음 — npm으로 설치한 JavaScript 번들에는 이 로직이 포함되지 않습니다

**v2.1.89-90 공식 수정 ([changelog](https://code.claude.com/docs/en/changelog)):**
- v2.1.89: *"Fixed prompt cache misses in long sessions caused by tool schema bytes changing mid-session"*
- v2.1.90: *"Improved performance: eliminated per-turn JSON.stringify of MCP tool schemas on cache-key lookup"*

---

## 버그 2 — Resume 캐시 파손 (v2.1.69+)

**GitHub Issue:** [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629)

`deferred_tools_delta`(v2.1.69에서 도입된 도구 지연 로딩 기능) 때문에 `--resume`(이전 대화를 이어서 하는 옵션)을 사용하면 첫 번째 메시지의 구조가 서버에 저장된 캐시 버전과 달라집니다. 그래서 캐시가 완전히 무효화(캐시 미스)됩니다. 전체 대화를 처음부터 다시 전송하는 것과 동일한 비용이 발생합니다.

**v2.1.90 공식 수정 ([changelog](https://code.claude.com/docs/en/changelog)):**
> *"Fixed --resume causing a full prompt-cache miss on the first request for users with deferred tools, MCP servers, or custom agents (regression since v2.1.69)"*

**참고:** `--continue`(마지막 대화를 이어서 하는 옵션)도 동일한 캐시 무효화 동작을 보입니다 ([#42338](https://github.com/anthropics/claude-code/issues/42338) 확인됨). 완전히 검증될 때까지는 `--resume`과 `--continue` 모두 사용을 피하고, 새 세션을 시작하는 것을 권장합니다.

---

## 버그 3 — 클라이언트 측 거짓 Rate Limiter (전 버전)

**GitHub Issue:** [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584)

로컬 rate limiter(사용량을 제한하는 내장 기능)가 Anthropic API를 전혀 호출하지 않고도 **가짜 "Rate limit reached" 에러**를 만들어냅니다. 실제로는 서버에 아무 요청도 보내지 않았는데, 마치 한도에 도달한 것처럼 에러가 뜨는 것입니다. 이 에러는 세션 로그에서 다음과 같이 식별할 수 있습니다:

```json
{
  "model": "<synthetic>",
  "usage": { "input_tokens": 0, "output_tokens": 0 }
}
```

대화 기록이 매우 클 때, 그리고 여러 sub-agent(Claude Code가 내부적으로 병렬 실행하는 보조 에이전트)가 동시에 생성될 때 이 현상이 발생합니다. [@rwp65](https://github.com/rwp65)가 [#40584](https://github.com/anthropics/claude-code/issues/40584)에서 약 74MB 크기의 대화 기록으로 이 현상을 관찰했습니다. rate limiter가 `컨텍스트 크기 × 동시 요청 수`를 곱해서 한도를 계산하는 것으로 보이며, 개별 요청이 작더라도 멀티 에이전트 워크플로우 전체가 차단됩니다.

- **발견:** [@rwp65](https://github.com/rwp65), [#40584](https://github.com/anthropics/claude-code/issues/40584) (2026년 3월 29일)
- **교차 참조:** [@marlvinvu](https://github.com/marlvinvu), [#40438](https://github.com/anthropics/claude-code/issues/40438), [#39938](https://github.com/anthropics/claude-code/issues/39938), [#38239](https://github.com/anthropics/claude-code/issues/38239) 간 교차 분석
- **상태:** **미수정** — v2.1.91까지 전 버전에 존재
- **영향:** 몇 시간 동안 아무것도 안 하고 쉬다가 다시 사용해도 즉시 "Rate limit reached"가 뜰 수 있습니다. 이 시점에서 사용량 한도는 완전히 리셋되어 있어야 정상입니다. API 호출이 실제로 이루어지지 않았으므로, 이 에러는 전적으로 내 컴퓨터의 Claude Code가 만들어낸 거짓 에러입니다.

---

## 버그 4 — Silent Microcompact → 컨텍스트 품질 저하 (전 버전, 서버 제어)

**GitHub Issue:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)

`src/services/compact/` 경로에 있는 세 가지 compaction(대화 내용을 자동 압축하는) 메커니즘이 **매 API 호출마다 사용자에게 알리지 않고 자동으로** 실행됩니다. 이 과정에서 이전에 도구가 반환한 결과(파일 내용, 명령 실행 결과 등)가 제거됩니다.

| 메커니즘 | 소스 | 트리거 | 제어 |
|----------|------|--------|------|
| **시간 기반 microcompact** | `microCompact.ts:422` | 마지막 assistant 메시지 이후 간격이 임계값 초과 | GrowthBook: `getTimeBasedMCConfig()` |
| **캐시된 microcompact** | `microCompact.ts:305` | 횟수 기반 트리거, `cache_edits` API를 사용하여 이전 tool 결과 삭제 | GrowthBook: `getCachedMCConfig()` |
| **세션 메모리 compact** | `sessionMemoryCompact.ts:57` | autocompact 전에 실행 | GrowthBook flag |

**주요 발견:**
- 세 가지 모두 `DISABLE_AUTO_COMPACT` 및 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`(사용자가 자동 압축을 끄기 위해 설정하는 환경 변수) 설정을 무시하고 작동합니다
- **서버 측 GrowthBook A/B 테스트 flag**(Anthropic 서버에서 원격으로 기능을 켜고 끌 수 있는 스위치)로 제어됩니다 — Anthropic이 사용자의 프로그램을 업데이트하지 않고도 동작을 바꿀 수 있습니다
- 도구 결과가 `[Old tool result content cleared]`라는 빈 문구로 조용히 교체됩니다 — 압축이 일어났다는 알림이 전혀 표시되지 않습니다
- 프록시를 통해 **3,782건의 내용 삭제 이벤트** (15,998 항목 삭제)가 감지되었습니다 (4월 1-8일 스냅샷). 버그 발견은 4월 2일; 프록시 시작은 4월 1일. 이전 수치 327건은 4월 3일 집중 테스트만 포함
- 삭제된 인덱스가 모두 **짝수** → 도구 호출과 그 결과 쌍(tool_use/tool_result)을 정확하게 타겟팅하고 있습니다
- 대화가 길어짐에 따라 삭제 범위가 **점진적으로 확장**됩니다

**캐시 영향 (4월 3일 업데이트 — 측정치):**

프록시 기반 테스트 결과, 초기 가설에 대한 **수정사항**이 밝혀졌습니다: microcompact가 메인 세션에서 지속적인 캐시 문제를 일으키지는 않습니다.

| 컨텍스트 | clearing 중 캐시 비율 |
|----------|----------------------|
| 메인 세션 | **99%+** — 영향 없음 (동일한 표식으로 일관되게 치환되어 prefix가 유지됨) |
| Sub-agent cold start | **0-39%** — 삭제 시점에서 하락 관찰 |
| Sub-agent warmed | **94-99%** — 정상 복구 |

캐시 비율이 높게 유지되는 이유는 동일한 `[Old tool result content cleared]` 마커가 매번 같은 내용으로 치환되어 대화 앞부분(prompt prefix)이 보존되기 때문입니다. 하지만 진짜 문제는 따로 있습니다. 모델이 더 이상 원본 파일 내용이나 명령 실행 결과를 볼 수 없고, 빈 placeholder만 보게 됩니다. 실질적으로 이는 Claude가 이전에 읽었던 파일 내용을 정확히 인용할 수 없고, 이미 시도했던 접근 방식을 다시 시도할 수 있다는 뜻입니다. [@Sn3th](https://github.com/Sn3th)는 1M(100만) 토큰 윈도우가 있는데도 도구를 50회 이상 사용한 세션에서 실제로 활용 가능한 컨텍스트가 약 4만~8만 토큰으로 줄어든다고 보고했습니다. 이용 가능한 컨텍스트의 92-96%가 사라지는 셈입니다.

- **업데이트 (4월 3일):** 4대 머신 / 4개 계정에 걸쳐 GrowthBook flag를 조사한 결과 **모든 스위치가 비활성화** 상태였습니다 — 그런데도 컨텍스트가 여전히 삭제되었습니다. 이는 위에서 설명한 세 가지 GrowthBook 제어 메커니즘과는 별개인 다른 코드 경로가 있다는 뜻입니다. 전체 분석은 [05_MICROCOMPACT.md](05_MICROCOMPACT.md)를 참고하십시오.
- **발견:** [@Sn3th](https://github.com/Sn3th), [#42542](https://github.com/anthropics/claude-code/issues/42542) (2026년 4월 2일)
- **상태:** v2.1.91에서 **미수정**. v2.1.91 테스트 세션에서 동일한 패턴의 14건 이벤트가 감지되었습니다.

---

## 버그 5 — Tool Result Budget 적용 (전 버전)

**발견:** 2026년 4월 3일 (cc-relay 프록시 개선을 통해)
**소스:** [@Sn3th](https://github.com/Sn3th)가 GrowthBook flag를 식별; 우리가 동작 활성화를 확인.

**별도의 요청 전 파이프라인**인 `applyToolResultBudget()`이 서버에서 제어하는 임계값에 따라 도구 결과를 잘라냅니다. 이것은 microcompact(버그 4)보다 먼저 실행되며 독립적으로 동작합니다. 쉽게 말해, 도구가 반환한 결과의 총 크기에 상한선을 두고, 초과하면 이전 결과부터 잘라내는 것입니다.

**활성 GrowthBook flag (`~/.claude.json`에서 확인):**
```
tengu_hawthorn_window:       200,000  (aggregate tool result cap across all messages)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: true    (system prompt tells model to expect clearing)
```

**측정된 영향:**
- 20개 세션에 걸쳐 **72,839건의 budget 이벤트**가 감지되었습니다 (4월 1-8일 스냅샷). 이전 수치 261건은 단일 세션 (4월 3일)만 포함
- **100% 잘림률** — 모든 이벤트가 내용 축소를 초래
- **90.6%의 이벤트**가 11-100자로 잘림; 9.4%는 0-10자로 잘림 (평균: 24자)
- **242,094자**(20만 자 상한 초과)에서 budget 임계값을 넘었습니다
- 대략 파일 읽기 15-20회 이후부터 이전에 읽은 결과들이 조용히 잘려 나가기 시작합니다

**v2.1.91:** `_meta["anthropic/maxResultSizeChars"]` (최대 500K) 추가 — 그러나 이는 **MCP tool(외부 도구 서버) 결과**에만 적용됩니다. Claude Code의 내장 도구(Read, Bash, Grep, Glob, Edit)는 이 override의 **영향을 받지 않습니다**. 일반 사용에서 20만 자 합산 상한은 그대로 유지됩니다.

**환경 변수로 끌 수 있는 방법이 없습니다.** `DISABLE_AUTO_COMPACT`, `DISABLE_COMPACT` 및 기타 알려진 모든 환경 변수가 이 코드 경로에 영향을 주지 않습니다.

> **넘버링 참고:** 버그 6과 7은 조사 과정에서 확인되었으나(compaction 무한 루프와 OAuth 재시도 폭주), 재현이 어려운 과거 이슈로 [07_TIMELINE.md](07_TIMELINE.md)에서 별도로 추적합니다. 이 문서는 현재 측정 가능한 버그에 집중합니다.

---

## 버그 8 — JSONL 로그 중복 (전 버전)

**GitHub Issue:** [anthropics/claude-code#41346](https://github.com/anthropics/claude-code/issues/41346)

extended thinking(Claude가 답변 전에 내부적으로 길게 생각하는 모드)이 활성화되면, 세션 JSONL 파일(Claude Code가 남기는 로그 파일)에 API 호출 한 번당 **2-5개의 PRELIM(중간) 항목**이 기록됩니다. 이 중간 항목들도 FINAL(최종) 항목과 동일한 `cache_read_input_tokens` 및 `cache_creation_input_tokens` 값을 포함합니다. 결과적으로 로컬에서 집계하는 토큰 사용량이 실제보다 부풀려집니다.

**측정치 (4월 3일, 단일 세션):**

| 세션 유형 | PRELIM | FINAL | 비율 | 토큰 인플레이션 |
|----------|--------|-------|------|----------------|
| 메인 세션 | 79 | 82 | 0.96x | **2.87x** |
| Sub-agent | 39 | 20 | 1.95x | — |
| Sub-agent | 12 | 7 | 1.71x | — |
| 이전 세션 | 16 | 6 | **2.67x** | — |

**대량 스캔 (4월 8일, 532 파일):** B8은 **전 세션 공통**입니다 — 가장 큰 세션 상위 10개 전부에서 PRELIM/FINAL 중복이 나타납니다. 평균 토큰 인플레이션: **2.37x** (범위 1.45x-4.42x). 최악의 경우: 734f00e7 세션에서 4.42x (기록된 입력 토큰의 77%가 중복된 PRELIM 항목).

**미결 질문:** 서버 측 rate limiter가 PRELIM 항목도 함께 카운트하는가? 만약 그렇다면, extended thinking 세션은 실제 API 사용량의 2-3배에 해당하는 rate limit이 부과되는 셈입니다.

---

## 버그 규모 개요 (4월 1-8일)

```mermaid
xychart-beta
    title "버그 유형별 감지 이벤트 (4월 1-8일, 로그 스케일 근사)"
    x-axis ["B3 합성 RL","B4 Microcompact","B5 Budget Cap","B8 PRELIM 인플레이션"]
    y-axis "이벤트" 0 --> 75000
    bar [3129,3782,72839,532]
```

> B3: 257/532 파일에서 3,129건의 rate limit 텍스트 발생. B4: 3,782건의 삭제 이벤트 (15,998 항목). B5: 72,839건의 잘림 이벤트 (100% 비율). B8: 분석한 532 파일 전체에서 공통 (2.37x 평균 인플레이션). 전체 데이터: [13_PROXY-DATA.md](13_PROXY-DATA.md).

---

## 측정 데이터

### 방법론

[`ANTHROPIC_BASE_URL`](https://docs.anthropic.com/en/docs/claude-code/settings#environment-variables) (공식 환경 변수)을 사용한 투명한 로컬 모니터링 프록시를 이용했습니다. 이 프록시는 요청이나 응답을 수정하지 않고, 각 API 응답에서 `cache_creation_input_tokens`(캐시가 새로 만들어진 토큰 수)와 `cache_read_input_tokens`(캐시에서 재사용된 토큰 수)만 기록합니다 (소스 감사 완료).

### v2.1.89 Standalone — 수정 전 (파손 상태, JSONL 데이터)

| 세션 | 항목 수 | 평균 캐시 Read | 최소 | 상태 |
|------|---------|---------------|------|------|
| `64d42269` (4/1 16:33) | 233 | **47.0%** | 8.3% | drain 시작 |
| `9100c2d2` (4/1 18:04) | 89 | **36.1%** | 21.1% | 최악의 drain — 조사 촉발 |
| Session A (이전 JSONL) | 168 | **4.3%** | — | 불량 — 20배 비용 인플레이션 |

정상적이라면 캐시 재사용률이 90% 이상이어야 하는데, 4.3%까지 떨어진 세션은 같은 대화를 하는 데 약 20배의 비용이 더 든 셈입니다. v2.1.68 (npm)으로 다운그레이드하자 `892388f6` 세션이 **평균 97.6%**로 복구되었습니다 (119개 항목, 최소 60.9%).

### v2.1.90 — 수정 후 (양쪽 설치)

| 지표 | npm (Node.js) | Standalone (ELF) |
|------|--------------|-----------------|
| 완료된 시나리오 | 7 (79-report 병렬 에이전트 읽기 포함) | 4 (forge, browsegrab, feedkit 5-turn, 3-project 병렬) |
| 소비된 사용량 | 28% → 35% (**7%**) | 35% → 40% (**5%**) |
| 전체 캐시 read | **86.4%** | **86.2%** |
| 안정 세션 read | **95-99.8%** | **95-99.7%** |

요청별 상세 데이터 및 warming 곡선: **[04_BENCHMARK.md](04_BENCHMARK.md)**

### 버전 비교 요약

| 지표 | v2.1.89 Standalone | v2.1.90 npm | v2.1.90 standalone | v2.1.91 npm | v2.1.91 standalone |
|------|-------------------|------------|-------------------|------------|-------------------|
| Cold start | **4-17%** | 63-80% | **14-47%** | **84.5%** | **27.8%** |
| 95%+ 복구 | 불가 | 3-5 reqs | 3-5 reqs | **2 reqs** | **1 req** |
| Sub-agent cold | — | 79-87% | 14-47% | **54%** | **0%** |
| Sub-agent stable | — | 87-94% | 94-99% | **93-99%** | **91-99%** |
| 안정 세션 | 90-99% | **95-99.8%** | **95-99.7%** | **98-99.6%** | **94-99%** |
| 전체 | ~20% | 86.4% | 86.2% | **88.4%** | **84.1%** |
| 평가 | **사용 금지** | 양호 | 양호 | **최상** | **양호** |

v2.1.91 standalone의 cold start 수치는 워크스페이스(작업 폴더)에 따라 달라집니다 (전체 벤치마크에서 27.8% vs 단일 프롬프트 테스트에서 84.7%). 하지만 복구 속도는 v2.1.90보다 훨씬 빠릅니다 (1번의 요청 만에 vs 3-5번). 어느 설치 방식이든 대화가 진행되면 94-99%로 수렴합니다. 요청별 데이터는 **[04_BENCHMARK.md](04_BENCHMARK.md)**를 참고하십시오.

---

*참고: 버그 4-5 심층 분석은 [05_MICROCOMPACT.md](05_MICROCOMPACT.md), 4월 3일 통합 테스트 결과는 [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md), 서버 측 할당량 분석은 [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)를 참고하십시오.*
