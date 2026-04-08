> 🇰🇷 이 문서는 [영어 원본](../README.md)의 한국어 번역입니다.

# Claude Code 숨겨진 문제점 분석

> **TL;DR:** Claude Code에는 **4개 계층에 걸친 6개의 확인된 클라이언트 측 버그(소프트웨어 결함)**가 있습니다. 이 버그들 때문에 사용량이 예상보다 훨씬 빠르게 소진됩니다. 캐시(이전에 처리한 내용을 재사용하는 기능) 관련 버그(1-2)는 v2.1.91에서 수정되었습니다. 나머지 4개는 미수정 (B3, B4, B5, B8)입니다. 또한, 프록시(중간에서 통신을 중계하는 서버)로 캡처한 rate limit(일정 기간 내 사용할 수 있는 양의 제한) 헤더 정보를 분석한 결과, **5시간/7일 이중 윈도우 쿼터(사용량 할당) 시스템**과 심각한 **thinking 토큰(AI의 내부 사고 과정에 소비되는 처리 단위) 사각지대**가 발견되었습니다 — 사용자에게 보이는 출력만으로는 실제 사용량의 절반도 설명하지 못합니다. 모든 발견 사항은 프록시 측정 데이터로 뒷받침됩니다.
>
> **최종 업데이트:** 2026년 4월 6일 — README 구조 개편, rate limit 헤더 분석 추가

---

> **용어 안내**
>
> 이 문서에서 자주 등장하는 용어를 정리하였습니다. 처음 접하는 독자는 여기를 참고하시기 바랍니다.
>
> | 용어 | 뜻 |
> |------|-----|
> | **토큰(token)** | AI가 텍스트를 처리하는 단위입니다. 대략 한글 1글자, 영어 단어 1개 정도에 해당합니다. |
> | **캐시(cache)** | 이전에 처리한 내용을 재사용하는 기능입니다. 캐시가 잘 작동하면 같은 내용을 다시 보낼 때 비용이 크게 줄어듭니다. |
> | **rate limit** | 일정 기간 동안 사용할 수 있는 양의 제한입니다. 한도에 도달하면 일정 시간 기다려야 다시 사용할 수 있습니다. |
> | **프록시(proxy)** | 중간에서 통신을 중계하는 서버입니다. 이 분석에서는 Claude Code가 주고받는 데이터를 기록하기 위해 사용하였습니다. |
> | **버그(bug)** | 소프트웨어의 결함이나 오류입니다. 의도하지 않은 동작을 일으킵니다. |
> | **thinking 토큰** | AI가 답변을 만들기 전에 내부적으로 "생각"하는 과정에서 소비되는 토큰입니다. 사용자에게는 보이지 않지만, 사용량에는 포함됩니다. |
> | **JSONL** | Claude Code가 세션(작업 기록)을 저장하는 로그 파일 형식입니다. |
> | **context** | AI에게 보내는 전체 대화 내용입니다. 대화가 길어지면 context도 커지고, 비용도 올라갑니다. |
> | **세션(session)** | Claude Code를 열어서 작업하는 하나의 작업 단위입니다. 새 세션 = 새로 대화를 시작하는 것입니다. |
> | **standalone 바이너리** | npm 없이 직접 다운로드해서 설치하는 Claude Code 실행 파일입니다. |

---

## 최신 업데이트 (4월 6일)

### Rate limit 헤더 분석 — [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)

투명 프록시(cc-relay)가 **3,702건의 요청**(4월 4-6일)에서 `anthropic-ratelimit-unified-*` 헤더(서버가 보내는 사용량 정보)를 캡처하였습니다. 이를 통해 서버 측 쿼터가 어떻게 구성되어 있는지 밝혀냈습니다.

**이중 슬라이딩 윈도우(시간대별 사용량 측정 구간) 시스템:**
- 두 개의 독립적인 사용량 카운터가 존재합니다: **5시간**(`5h-utilization`)과 **7일**(`7d-utilization`)
- `representative-claim` = `five_hour`가 요청의 **100%** — 사실상 5시간 윈도우가 항상 사용량의 병목(가장 먼저 한도에 걸리는 부분)입니다
- 5시간 윈도우는 대략 5시간 간격으로 초기화되고, 7일 윈도우는 매주 초기화됩니다 (이 계정의 경우 4월 10일 12:00 KST)

**사용량 1%당 비용** (Max 20x / $200/mo 기준, 5개 활성 윈도우에서 측정):

| 지표 | 범위 | 비고 |
|------|------|------|
| 1%당 출력 | 9K-16K | 사용자에게 보이는 출력만 (thinking 제외) |
| 1%당 Cache Read | 1.5M-2.1M | 전체 보이는 토큰의 96-99%를 차지 |
| 1%당 총 가시적 토큰 | 1.5M-2.1M | Output + Cache Read + Input 합산 |
| 7일 누적 비율 | 0.12-0.17 | 5시간 피크 대비 7일 변화량 |

**Thinking 토큰 사각지대:** Extended thinking(AI의 확장 사고 기능) 토큰은 API가 알려주는 `output_tokens` 수치에 **포함되지 않습니다**. 사용량 1%당 보이는 출력이 9K-16K 토큰이므로, 5시간 윈도우 전체(100%)를 쓴다 해도 보이는 출력 토큰은 0.9M-1.6M에 불과합니다. 이는 Opus 모델로 몇 시간을 작업한 것치고는 너무 적은 수치입니다. 이 차이는 thinking 토큰이 사용량 할당에 포함되지만, 정확한 방식은 사용자 측에서 확인할 수 없다는 점과 일치합니다. Thinking 비활성화 후 비교 테스트는 4월 6일 주간에 계획되어 있습니다.

**커뮤니티 교차 검증:**
- [@fgrosswig](https://github.com/fgrosswig): [64배 용량 감소](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) — 두 대의 컴퓨터에서 18일간 JSONL 로그를 비교 분석 (3월 26일: 32억 토큰까지 제한 없이 사용 → 4월 5일: 8,800만 토큰에서 90% 도달)
- [@Commandershadow9](https://github.com/Commandershadow9): [34-143배 용량 감소](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296) — 캐시 수정 확인 후에도 용량 감소가 독립적으로 존재함을 확인, thinking 토큰 가설 제시

**v2.1.89 분리:** 캐시 문제(3월 28일 - 4월 1일)는 별도의 이슈이며 이미 해결되었습니다. 용량 감소는 캐시 문제와 관계없이 독립적으로 존재합니다. 이를 확인하기 위해 캐시가 정상인 기간끼리 비교하였습니다 — 골든 피리어드(3월 23-27일, 캐시 98-99%)와 수정 후(4월 2일 이후, 캐시 84-97%) 모두 캐시가 정상인 상태에서 비교한 결과입니다. 4월 10일(전체 7일 주기)까지 데이터 수집이 진행 중입니다.

---

## 현재 상태 (2026년 4월 6일)

캐시 문제(v2.1.89)는 v2.1.90-91에서 **수정**되었습니다. 하지만 4개의 추가 클라이언트 측(사용자 컴퓨터에서 작동하는 프로그램의) 버그(B3, B4, B5, B8)와 서버 측 쿼터 변경은 여전히 남아 있습니다.

| 버그 | 현상 | 영향 | 상태 (v2.1.91) | 상세 |
|------|------|------|-----------------|------|
| **B1** Sentinel | standalone 바이너리가 캐시 작동에 필요한 식별 정보를 손상시킴 | 캐시 활용률이 4-17%로 하락 (v2.1.89) | **수정됨** | [01_BUGS.md](01_BUGS.md#bug-1--sentinel-replacement-standalone-binary-only) |
| **B2** Resume | `--resume`(이어쓰기) 명령이 이전 대화 전체를 캐시 없이 다시 전송 | 전체 캐시 미스 | **수정됨** | [01_BUGS.md](01_BUGS.md#bug-2--resume-cache-breakage-v2169) |
| **B3** False RL | 프로그램이 실제 서버 오류가 아닌데도 가짜 오류를 만들어 API 호출을 차단 | 갑자기 "Rate limit reached"(사용량 한도 도달) 메시지가 표시됨 | **미수정** | [01_BUGS.md](01_BUGS.md#bug-3--client-side-false-rate-limiter-all-versions) |
| **B4** Microcompact | 세션 중간에 도구 실행 결과가 조용히 삭제됨 | AI가 참고할 수 있는 정보의 품질이 저하됨 | **미수정** | [01_BUGS.md](01_BUGS.md#bug-4--silent-microcompact--context-quality-degradation-all-versions-server-controlled), [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| **B5** Budget cap | 도구 실행 결과에 200K(약 20만) 토큰 총량 제한이 적용됨 | 오래된 결과가 1-41자로 잘려나감 | **미수정** | [01_BUGS.md](01_BUGS.md#bug-5--tool-result-budget-enforcement-all-versions), [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| **B8** Log inflation | Extended thinking이 JSONL 로그에 중복 항목을 생성 | 로컬 토큰 수가 실제의 2.87배로 부풀려져 기록됨 | **미수정** | [01_BUGS.md](01_BUGS.md#bug-8--jsonl-log-duplication-all-versions) |
| **서버** | 쿼터 구조 변경 + thinking 토큰 사용량 산입 | 실질적으로 사용 가능한 양이 줄어듦 | **설계상 의도** | [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |

### 할 수 있는 조치

1. **v2.1.91로 업데이트하십시오** — 캐시 문제가 수정됩니다 (사용량 소진의 가장 큰 원인이었습니다)
2. **npm이든 standalone이든 v2.1.91에서는 모두 정상입니다** (Sentinel 버그가 해소되었습니다)
3. **`--resume`이나 `--continue`는 사용하지 마십시오** — 이전 대화 전체가 유료 입력으로 다시 전송됩니다
4. **주기적으로 새 세션을 시작하십시오** — 200K 도구 결과 상한(B5) 때문에 오래된 결과가 알림 없이 잘려나갑니다
5. **`/dream`과 `/insights` 명령은 자제하십시오** — 사용자 모르게 백그라운드에서 API를 호출하여 사용량이 소진됩니다

설정 가이드와 자가 진단 방법은 [09_QUICKSTART.md](09_QUICKSTART.md)를 참조하십시오.

---

## 서버 측 요인 (미해결)

캐시가 95-99%로 정상 작동해도 사용량 소진은 계속됩니다. 최소 4가지 서버 측 이슈가 여기에 기여하고 있습니다.

**1. 서버 측 과금 변경:** 오래된 Docker(컨테이너 환경) 버전(v2.1.74, v2.1.86 — 한 번도 업데이트하지 않은 것)이 최근 갑자기 빠르게 소진되기 시작하였습니다. 프로그램을 업데이트하지 않았는데도 소진이 빨라졌다는 것은, 이 문제가 순수하게 사용자 측 소프트웨어 문제만은 아니라는 증거입니다 ([#37394](https://github.com/anthropics/claude-code/issues/37394)).

**2. 1M context 과금 오류:** 3월 말에 발생한 문제로, 서버가 Max 플랜의 1M(100만 토큰) context 요청을 "추가 사용량"으로 잘못 분류합니다. 디버그 로그를 보면 겨우 약 23K(2만 3천) 토큰밖에 안 쓰는데도 429 에러(사용량 초과 오류)가 발생합니다 ([#42616](https://github.com/anthropics/claude-code/issues/42616)).

**3. 이중 윈도우 쿼터 구조 (4월 6일 발견):** 5시간 + 7일 두 개의 독립적인 사용량 측정 구간이 존재합니다. 5시간 윈도우 기준 사용량 1%당 약 1.5M-2.1M의 토큰이 소비되며, 그중 96-99%는 캐시 읽기입니다. [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) 참조.

**3a. Thinking 토큰 사각지대 (4월 6일 발견):** 사용자에게 보이는 출력은 사용량 1%당 9K-16K 토큰에 불과합니다. 이는 thinking 토큰이 사용량에 포함되지만 사용자에게는 보이지 않는다는 가설과 일치합니다. [@Commandershadow9](https://github.com/Commandershadow9)와 [@fgrosswig](https://github.com/fgrosswig)의 커뮤니티 분석도 JSONL 로그 분석을 통해 독립적으로 동일한 결론에 도달하였습니다. [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) 참조.

**4. 조직 수준 쿼터 공유:** 같은 조직에 소속된 계정들이 rate limit 한도를 공유합니다. 서버 내부에서 `passesEligibilityCache`와 `overageCreditGrantCache`가 개인 계정(`accountUuid`) 단위가 아니라 조직(`organizationUuid`) 단위로 관리됩니다. 즉, 같은 조직 소속이면 다른 사람의 사용이 내 한도에도 영향을 줄 수 있습니다.

---

## 사용 시 주의사항

| 행동 | 이유 | 권고 |
|------|------|------|
| `--resume` / `--continue` | 이전 대화 전체가 유료 입력으로 다시 전송됩니다 | **사용 자제** — 새 세션을 시작하십시오 |
| `/dream`, `/insights` | 사용자 모르게 백그라운드에서 API를 호출하여 사용량이 소진됩니다 | **사용 자제** |
| 여러 터미널에서 동시 사용 | 터미널 간에 캐시를 공유하지 않아서 각각 따로 사용량이 소진됩니다 | **하나만 사용하십시오** |
| 큰 CLAUDE.md / context 파일 | 매 대화마다 전송되어 캐시 읽기량에 비례하여 비용이 증가합니다 | **간결하게 유지하십시오** |
| v2.1.89 이하 standalone | Sentinel 버그로 캐시 활용률이 4-17%에 머뭅니다 | **v2.1.91로 업데이트하십시오** |

---

## 배경

### 시작 경위

2026년 4월 1일, 평소와 같은 코딩 작업 중이었는데 Max 20 플랜($200/mo)이 약 70분 만에 사용량 100%에 도달하였습니다. JSONL 로그를 분석한 결과, 해당 세션의 평균 캐시 활용률이 **36.1%**(최솟값 21.1%)로 나타났습니다. 정상이라면 90% 이상이어야 하는 수치입니다. 캐시가 작동하지 않으니 모든 토큰이 정가(캐시 할인 없이)로 과금되고 있었습니다.

v2.1.89에서 v2.1.68로 버전을 낮추자 캐시가 즉시 **97.6%**로 회복되었습니다. 이것으로 문제가 특정 버전에서 발생한 것임을 확인하였습니다. 이후 요청별 데이터를 기록하기 위해 투명 모니터링 프록시(cc-relay)를 구축하였습니다.

개인적인 문제 해결로 시작한 작업이 빠르게 확장되었습니다. 수십 명의 사용자가 [91건 이상의 GitHub 이슈](10_ISSUES.md)에서 동일한 증상을 보고하고 있었습니다. 커뮤니티 멤버들 — [@Sn3th](https://github.com/Sn3th), [@rwp65](https://github.com/rwp65), [@fgrosswig](https://github.com/fgrosswig), [@Commandershadow9](https://github.com/Commandershadow9), 그리고 [12명의 기여자들](10_ISSUES.md#contributors--acknowledgments) — 이 독립적으로 퍼즐의 각기 다른 조각을 발견하였습니다.

**조사 타임라인:**

| 날짜 | 발견 내용 |
|------|----------|
| 4월 1일 | 70분 만에 100% 소진 → v2.1.89 문제 확인, 프록시 구축 |
| 4월 2일 | 버그 3-4 발견 (가짜 rate limiter, 조용한 microcompact). Anthropic의 Lydia Hallie가 X에 게시 |
| 4월 3일 | 버그 5 발견 (200K budget cap). v2.1.91 벤치마크: 캐시 수정 확인, 4개 다른 버그 지속. [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md) |
| 4월 4-6일 | cc-relay가 rate limit 헤더와 함께 3,702건 요청 캡처. 커뮤니티 분석 계속 |
| 4월 6일 | 이중 윈도우 쿼터 분석 발표. 커뮤니티 교차 검증 (fgrosswig 64배, Commandershadow9 34-143배). [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |

전체 14개월 연대기 (2025년 2월 – 2026년 4월): [07_TIMELINE.md](07_TIMELINE.md)

### Anthropic의 입장 (4월 2일)

Lydia Hallie (Anthropic, 제품 담당)가 [X에 게시](https://x.com/lydiahallie/status/2039800715607187906)하였습니다:

> *"피크 시간대 제한이 더 엄격해졌고 1M context 세션이 커졌습니다. 여러분이 느끼는 대부분이 이것입니다. 과정에서 몇 가지 버그를 수정했지만, 과다 청구된 것은 없습니다."*

Lydia Hallie는 Sonnet을 기본 모델로 사용하고, effort 레벨을 낮추고, 이어쓰기(resume) 대신 새로 시작하고, `CLAUDE_CODE_AUTO_COMPACT_WINDOW=200000`으로 context 크기를 제한할 것을 [권장](https://x.com/lydiahallie/status/2039800718371307603)하였습니다.

**이 분석의 데이터가 위 입장과 다른 지점들:**

- **"과다 청구된 것은 없습니다"** — 버그 5는 도구 결과가 총 200K 토큰을 넘으면 오래된 결과를 1-41자로 조용히 잘라냅니다. 1M context에 비용을 지불하는 사용자들이 실제로는 도구 결과에 200K 토큰만 사용할 수 있는 셈입니다. 단일 세션에서 261건의 잘림 이벤트가 측정되었습니다.
- **"몇 가지 버그를 수정했습니다"** — 캐시 버그(B1-B2)는 수정되었지만, 버그 3-5와 B8은 v2.1.91에서 여전히 남아 있습니다. 클라이언트 측 가짜 rate limiter(B3)는 이 분석 환경에서 65개 세션에 걸쳐 151개의 가짜 "Rate limit reached" 오류를 만들어냈습니다. 실제로 서버에 보낸 API 호출은 단 한 건도 없었습니다.
- **"피크 시간대 제한이 더 엄격해졌습니다"** — 4월 6일 프록시 데이터에 따르면 병목은 항상 5시간 윈도우입니다 (`representative-claim` = `five_hour`가 3,702건 요청의 100%). 시간대와 상관없이 동일합니다. 주말이나 한가한 시간대 데이터도 같은 패턴을 보입니다.
- **Thinking 토큰 과금** — Extended thinking 토큰은 API가 알려주는 `output_tokens`에 표시되지 않습니다. 그런데 사용자에게 보이는 출력만으로는 실제 사용량의 절반도 설명이 안 됩니다. 만약 thinking 토큰이 출력 토큰과 같은 비율로 사용량에 산입된다면, 사용자가 확인하거나 조절할 수 없는 상당한 숨겨진 비용이 존재하는 셈입니다.

**GitHub 응답:** 91건 이상의 rate limit 관련 이슈에 대해 공식 응답 없음 (2개월 이상 침묵). 모든 공식 소통은 개인 X(구 트위터) 게시물과 변경 로그를 통해서만 이루어졌습니다. 전체 응답 이력은 [10_ISSUES.md](10_ISSUES.md#anthropic-official-response)를 참조하십시오.

### 캐시 TTL (버그 아님)

[@luongnv89](https://github.com/luongnv89)가 13시간 이상 사용하지 않으면 캐시가 완전히 재구축된다고 [분석](https://github.com/luongnv89/cc-context-stats/blob/main/context-stats-cache-misses.md)하였습니다. Anthropic은 캐시 TTL(유효 시간)을 5분으로 문서화하고 있지만, 이 분석의 데이터에서는 5-26분 간격에서도 96% 이상 캐시가 유지되는 경우가 있었습니다. 실제 유효 시간은 공식 수치보다 길 수 있습니다. 버그는 아니지만 알아두면 유용합니다.

---

## 문서 목록

| 파일 | 내용 | 날짜 |
|------|------|------|
| **[README.md](README.md)** | 이 파일 — 개요, 최신 업데이트, 현재 상태 | 4월 6일 |
| **[02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)** | 이중 5h/7d 윈도우 구조, 1%당 비용, thinking 토큰 사각지대 | 4월 6일 |
| **[03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md)** | 세션 로그 분석: 초기 토큰 부풀림, 하위 에이전트 비용, 수명주기 곡선, 프록시 교차 검증 | 4월 6일 |
| **[01_BUGS.md](01_BUGS.md)** | 버그 1-5, 8 기술적 상세 + 측정 데이터 | 4월 3일 |
| **[05_MICROCOMPACT.md](05_MICROCOMPACT.md)** | 심층 분석: 조용한 context 삭제 (버그 4) + 도구 결과 예산 (버그 5) | 4월 3일 |
| **[04_BENCHMARK.md](04_BENCHMARK.md)** | npm vs standalone 벤치마크 + 요청별 원시 데이터 | 4월 3일 |
| **[06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md)** | 4월 3일 통합 테스트 결과 — 전체 버그 검증 | 4월 3일 |
| **[07_TIMELINE.md](07_TIMELINE.md)** | rate limit 이슈 14개월 연대기 (Phase 1-9) | 4월 6일 |
| **[09_QUICKSTART.md](09_QUICKSTART.md)** | 설정 가이드 + 자가 진단 | 4월 3일 |
| **[08_UPDATE-LOG.md](08_UPDATE-LOG.md)** | 일일 조사 로그 — 무엇을, 언제, 어떻게 발견했는지 | 4월 1-6일 |
| **[10_ISSUES.md](10_ISSUES.md)** | 91건 이상 관련 이슈 + 커뮤니티 도구 + 기여자 | 4월 6일 |
| **[11_USAGE-GUIDE.md](11_USAGE-GUIDE.md)** | 필수 사용 가이드 — 세션, context, CLAUDE.md, 토큰 절약 | 4월 8일 |
| **[12_ADVANCED-GUIDE.md](12_ADVANCED-GUIDE.md)** | 고급 사용 가이드 — hooks, 서브에이전트, 모니터링, rate limit 전략 | 4월 8일 |

## 환경

- **플랜:** Max 20 ($200/mo)
- **OS:** Linux (Ubuntu), HP ZBook Ultra G1a
- **테스트 버전:** v2.1.91, v2.1.90, v2.1.89, v2.1.68
- **모니터링:** cc-relay v2 투명 프록시 (8,794건 요청, 3,702건 rate limit 헤더 포함)
- **날짜:** 2026년 4월 6일

---

## 기여자

이 분석은 많은 커뮤니티 멤버들의 작업을 기반으로 합니다. 전체 내용은 [10_ISSUES.md](10_ISSUES.md#contributors--acknowledgments)를 참조하십시오.

| 기여자 | 주요 기여 |
|--------|----------|
| [@Sn3th](https://github.com/Sn3th) | microcompact 메커니즘(버그 4), GrowthBook 플래그, budget 파이프라인(버그 5) 발견, 다수 머신에서 서버 측 context 변조 확인 |
| [@rwp65](https://github.com/rwp65) | 클라이언트 측 가짜 rate limiter(버그 3) 발견 + 상세 로그 증거 |
| [@fgrosswig](https://github.com/fgrosswig) | 64배 용량 감소 분석 — 두 대의 컴퓨터에서 18일간 JSONL 전후 비교 ([#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)) |
| [@Commandershadow9](https://github.com/Commandershadow9) | 34-143배 용량 감소 분석, thinking 토큰 가설 제시 ([#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)) |
| [@kolkov](https://github.com/kolkov) | [ccdiag](https://github.com/kolkov/ccdiag) 진단 도구 개발, v2.1.91 `--resume` 관련 문제 3건 식별 ([#43044](https://github.com/anthropics/claude-code/issues/43044)) |
| [@simpolism](https://github.com/simpolism) | v2.1.90 변경 로그 분석, resume 캐시 수정 패치 적용 후 캐시 99.7-99.9% 달성 |
| [@luongnv89](https://github.com/luongnv89) | 캐시 TTL(유효 시간) 분석, [CUStats](https://custats.info)와 [context-stats](https://github.com/luongnv89/cc-context-stats) 도구 개발 |
| [@dancinlife](https://github.com/dancinlife) | 조직 단위 쿼터 공유(organizationUuid) 및 계정 간 오염 버그 발견 |
| [@weilhalt](https://github.com/weilhalt) | rate limit 헤더 모니터링용 [BudMon](https://github.com/weilhalt/budmon) 도구 개발 |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | Windows 환경에서의 GrowthBook 플래그 데이터로 운영체제 간 일관성 확인 |
| [@dbrunet73](https://github.com/dbrunet73) | 실제 운영 환경에서의 비교 데이터 (v2.1.88 vs v2.1.90)로 캐시 개선 확인 |
| [@maiarowsky](https://github.com/maiarowsky) | v2.1.90에서 버그 3 확인, 13개 세션에 걸쳐 26개 가짜 오류 항목 |
| [@edimuj](https://github.com/edimuj) | grep/file-read 작업의 토큰 낭비 측정 (350만 토큰 / 1,800회 이상 호출), [tokenlean](https://github.com/edimuj/tokenlean) 도구 개발 |
| [@amicicixp](https://github.com/amicicixp) | v2.1.90 캐시 개선을 전후 테스트로 검증 |
| [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) | 오래된 Docker 버전에서의 사용량 소진 보고 — 서버 측 문제의 핵심 증거 ([#37394](https://github.com/anthropics/claude-code/issues/37394)) |
| [@SC7639](https://github.com/SC7639) | 3월 중순 시점의 추가 데이터로 타임라인 확인 |
| Reddit 커뮤니티 | 캐시 sentinel 메커니즘의 [리버스 엔지니어링 분석](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) |

*이 분석은 커뮤니티 리서치와 개인 측정을 기반으로 합니다. Anthropic이 보증한 것이 아닙니다. 모든 우회 방법은 공식 도구와 문서화된 기능만 사용합니다.*
