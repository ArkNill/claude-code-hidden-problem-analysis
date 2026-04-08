> 이 문서는 [영어 원본](../08_UPDATE-LOG.md)의 한국어 번역입니다.

# 조사 기록 — 일별 진행 상황

> 날짜별로 조사한 내용, 발견 사항, 공개 내용을 기록합니다. 각 항목은 그날 무엇에 집중했는지, 어떤 방법을 썼는지, 무엇을 찾아냈는지를 담고 있습니다. 각 항목의 이벤트 수는 해당 날짜 기준입니다 — 누적 합계는 [13_PROXY-DATA.md](../13_PROXY-DATA.md)를 참조하십시오.

---

## 2026년 4월 1일 — 최초 발견

**계기:** Max 20 플랜($200/월)을 쓰는데, 평범한 코딩 작업 중 약 70분 만에 사용량이 100%에 도달했습니다.

**수행한 작업:**
- 세션 JSONL(세션 로그 파일, `9100c2d2`) 분석: **평균 cache read(캐시 재활용 비율) 36.1%** (최소 21.1%) — 정상이라면 90% 이상이어야 합니다
- 이전 세션(`64d42269`) 분석: **평균 47.0%** (233개 항목, 최소 8.3%) — 이미 성능 저하가 시작된 상태였습니다
- v2.1.89에서 v2.1.68로 다운그레이드(npm으로 설치): 캐시가 즉시 **97.6%**(119개 항목)로 회복되었습니다
- 확인 결과: 문제는 v2.1.89 버전에만 있었고, 계정이나 서버 문제가 아니었습니다
- 향후 요청별 모니터링(요청 하나하나의 토큰 사용을 추적)을 위해 `ANTHROPIC_BASE_URL`을 사용하는 cc-relay 투명 프록시(중간에서 통신 내용을 기록하는 도구)를 설정했습니다

**발견 사항:**
- **버그 1 (Sentinel)** 확인 — 커뮤니티 리버스 엔지니어링(프로그램 내부를 역분석하는 작업, [Reddit](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6))이 원인을 추적했습니다. standalone 바이너리(독립 실행 파일)에 내장된 Bun 포크의 `cch=00000` sentinel(캐시를 식별하는 특수 값) 교체가 문제였습니다
- v2.1.89 standalone: 4-17% cache read만 나오고 회복 불가. npm 설치 버전은 영향 없었습니다.

**공개:** 아직 없음 — 데이터를 모으는 단계였습니다.

---

## 2026년 4월 2일 — 버그 3-4 발견, Anthropic 대응

**초점:** 당일 출시된 v2.1.90을 프록시(통신 기록 도구)를 통해 테스트하고, 커뮤니티와 협업했습니다.

**수행한 작업:**
- cc-relay v2 프록시 가동 — 요청마다 `cache_creation`(캐시 생성), `cache_read`(캐시 읽기), `status_code`(응답 상태), `latency_ms`(응답 시간) 기록
- v2.1.90 벤치마크 시작: 같은 컴퓨터, 같은 프록시에서 npm vs standalone 직접 비교
- GrowthBook feature flag(서버에서 원격으로 기능을 켜고 끄는 스위치) 추출 확인([@Sn3th](https://github.com/Sn3th)) — `~/.claude.json`에 `cachedGrowthBookFeatures` 포함

**발견 사항:**
- **버그 3 (False Rate Limiter, 가짜 사용량 제한)** [@rwp65](https://github.com/rwp65)가 [#40584](https://github.com/anthropics/claude-code/issues/40584)에서 발견: 실제로 API를 호출하지 않았는데도 클라이언트가 가짜 "Rate limit reached" 오류(`model: "<synthetic>"`)를 만들어냈습니다. 대화가 길어지거나 여러 서브 에이전트(보조 AI)가 동시에 돌 때 발생했습니다.
- **버그 4 (Microcompact, 조용한 내용 삭제)** [@Sn3th](https://github.com/Sn3th)가 [#42542](https://github.com/anthropics/claude-code/issues/42542)에서 발견: 세 가지 compaction 메커니즘(대화 압축 기능)이 도구 결과를 사용자 몰래 삭제하고 있었고, 서버 측 GrowthBook A/B 테스트 플래그로 제어되고 있었습니다. 처음에는 캐시를 무효화하는 게 아닌가 의심했습니다.
- **Anthropic 대응:** Lydia Hallie가 [X에 게시](https://x.com/lydiahallie/status/2039800715607187906): *"피크 시간대 제한이 더 엄격해졌고 1M-context 세션이 커졌습니다... 몇 가지 버그를 수정했지만, 과다 청구된 것은 없습니다."*

**공개:** 아직 없음 — 테스트가 진행 중이었습니다.

---

## 2026년 4월 3일 — 버그 5 발견, v2.1.91 벤치마크, 레포 공개

**초점:** v2.1.91 전체 벤치마크(성능 비교 테스트) + 새 버그 발견 + 종합 테스트를 진행했습니다.

**수행한 작업:**
- v2.1.91 릴리스 — 전체 벤치마크 수행: npm vs standalone 직접 비교
- microcompact 탐지와 budget enforcement(예산 제한) 스캐닝 기능으로 cc-relay를 강화했습니다
- 4대 컴퓨터 / 4개 계정에 걸쳐 GrowthBook 플래그를 조사했습니다
- 버그 4 캐시 영향에 대한 **수정**: 프록시 데이터에 의하면 대화 압축 중에도 메인 세션 캐시는 99% 이상 유지되었습니다 — 진짜 문제는 캐시가 아니라 대화 맥락의 품질이 떨어지는 것이었습니다
- 91개 이상의 GitHub 이슈에 분석 결과를 교차 참조하는 코멘트를 게시했습니다

**발견 사항:**
- **버그 5 (Budget Cap, 도구 결과 예산 상한)** 발견: `applyToolResultBudget()`가 GrowthBook 플래그 `tengu_hawthorn_window`를 통해 200K 누적 캡(도구 결과의 총량 제한)을 적용하고 있었습니다. **261건의 budget 이벤트** 측정 — 도구가 돌려준 결과가 1-41자로 잘렸습니다. v2.1.91의 `maxResultSizeChars` 오버라이드는 MCP(외부 도구 연결) 전용이라 기본 내장 도구에는 적용되지 않았습니다.
- **버그 8 (JSONL Duplication, 로그 중복)** 측정: extended thinking(확장 사고 기능)이 API 호출 한 번당 2-5개의 PRELIM(예비) 항목을 생성했습니다. 메인 세션에서: **2.87배** 토큰 수 부풀림.
- **v2.1.91 벤치마크 결과:** Sentinel 격차 해소 — npm과 standalone 모두 cold start(처음 시작)에서 84.7% 달성. 캐시는 2회 요청 내 98% 이상으로 회복. 버그 1-2를 제외한 나머지 버그는 계속 남아 있었습니다.
- **버그 3 확인:** 자체 환경에서 65개 세션에 걸쳐 151개의 `<synthetic>` 항목(가짜 제한 기록)을 확인했습니다.
- **버그 4 이벤트 수:** 테스트된 모든 세션에서 327건의 microcompact clearing 이벤트(조용한 내용 삭제)가 발생했습니다.

**공개:**
- 레포지토리 생성: [claude-code-cache-analysis](https://github.com/ArkNill/claude-code-cache-analysis) (현재 이름 변경됨)
- [README.md](README.md), [04_BENCHMARK.md](04_BENCHMARK.md), [05_MICROCOMPACT.md](05_MICROCOMPACT.md), [07_TIMELINE.md](07_TIMELINE.md), [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md)
- 91개 이상의 GitHub 이슈에 코멘트

---

## 2026년 4월 4일 — Rate Limit 헤더 캡처 시작

**초점:** rate limit 헤더(사용량 제한 정보가 담긴 응답 데이터) 캡처를 위해 cc-relay를 개선하고 운영을 안정화했습니다.

**수행한 작업:**
- cc-relay를 개선해서 `anthropic-ratelimit-unified-*` 응답 헤더를 캡처하고 → `ratelimit_headers` 컬럼(JSON)에 저장하도록 했습니다
- cc-relay 스트리밍 방식을 수정했습니다: 실시간 스트리밍 프록시를 위해 `client.request()` → `client.send(req, stream=True)`로 변경
- systemd watchdog 추가 (60초 ping 루프, `WatchdogSec=120`, `RestartSec=3`) — 프록시가 죽으면 자동 재시작하는 안전장치
- 사전 로그인 서비스 시작을 위한 `loginctl enable-linger` 설정

**수집된 데이터:**
- **rate limit 헤더가 포함된 2,097건의 요청** (캡처 첫 날)
- 헤더에 포함된 정보: `5h-utilization`(5시간 사용률), `7d-utilization`(7일 사용률), `representative-claim`(대표 청구 기준), `fallback-percentage`(대체 비율), `overage-status`(초과 상태)
- 첫 관찰: 100%의 요청에서 `representative-claim` = `five_hour` — 즉, 모든 사용량이 5시간 윈도우 기준으로 산정되고 있었습니다

**커뮤니티:**
- [@dancinlife](https://github.com/dancinlife): `organizationUuid` 기반 쿼터 풀링(같은 조직의 계정이 사용량 한도를 나눠 쓰는 현상) 발견 — 같은 조직에 속한 계정끼리 rate limit을 공유하고 있었습니다. 기여자로 추가.

**공개:** dancinlife 기여자 레포 추가.

---

## 2026년 4월 5일 — 데이터 축적, 커뮤니티 분석

**초점:** 헤더 데이터를 계속 수집하고, 커뮤니티에서 나온 분석 결과를 검토했습니다.

**수집된 데이터:**
- **추가 1,333건의 헤더 포함 요청** (총 3,430건 이상)
- 프록시 DB 전체: 8,794건의 요청, 1,245건의 microcompact 이벤트(조용한 내용 삭제), 23,021건의 budget 이벤트(도구 결과 잘림)

**커뮤니티 활동:**
- **[@fgrosswig](https://github.com/fgrosswig)**: #38335에서 [64배 budget 감소 포렌식(상세 추적 분석)](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353) 공개: 두 대의 컴퓨터에서 18일간 JSONL 분석. 3월 26일에는 3.2B 토큰을 제한 없이 쓸 수 있었는데 → 4월 5일에는 90%에서 88M만 가능. 가설: cache-read 가중치(캐시로 읽은 토큰의 과금 비중)가 ~0배에서 ~1배로 변경된 것 같다는 분석이었습니다.
- **[@Commandershadow9](https://github.com/Commandershadow9)**: #41506에서 [34-143배 용량 감소](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296) 공개: 캐시 문제는 고쳐졌지만, 캐시 버그와 무관하게 쓸 수 있는 양이 34-143배나 줄었다고 보고했습니다. thinking 토큰(AI가 생각하는 과정에 쓰이는 토큰)이 사용량에 포함되는 것 아니냐는 가설을 제기했습니다.
- **[@Sn3th](https://github.com/Sn3th)**: #42542 발견 사항에 대한 지속적 협업.
- **[@marlvinvu](https://github.com/marlvinvu)**: #41346에서 PRELIM(예비 로그 항목) 2-3배 부풀림과 여러 이슈에 걸쳐 보고된 "유령 사용량"의 상관관계를 분석했습니다. PRELIM + synthetic(가짜 제한)이 합쳐지면 더 심해진다고 제시했습니다.

**공개:** kolkov resume 리그레션 참조 레포 추가.

---

## 2026년 4월 6일 — Rate Limit 헤더 분석, 레포 구조 개편

**초점:** 48시간 동안 캡처한 헤더를 분석하고, 결과를 공개하고, 레포 구조를 전면 개편했습니다.

**수행된 분석:**
- 3,702건의 요청을 `5h-reset` 헤더 타임스탬프(사용량 초기화 시간)를 기준으로 **8개의 개별 5시간 윈도우**로 분할
- **1% 사용률당 비용** 계산: 사용률 1% 포인트를 쓰는 데 출력 토큰 9K-16K, cache_read 토큰 1.5M-2.1M이 필요했습니다
- **thinking 토큰 사각지대** 확인: `output_tokens`(출력 토큰 수)에 thinking 토큰이 빠져 있어서, 눈에 보이는 출력만으로는 관찰된 사용률의 50%도 설명이 안 되었습니다
- **7일 누적 패턴** 매핑: 5시간 피크 대비 비율 ~0.12-0.17 (6개 데이터 포인트, 아직 정밀도가 낮습니다)
- **v2.1.89 분리 프레임워크** 수립: 깨끗한 비교를 위해 golden(3월 23-27일, 정상) / bug(3월 28일-4월 1일, 버그 기간) / post-fix(4월 2일 이후, 수정 후)로 시기를 구분했습니다
- fgrosswig와 Commandershadow9 데이터와 교차 검증 — cache-read 가중치 변경이나 thinking 토큰 계산 변경과 일치했습니다
- **JSONL 세션 로그 분석** (110개 메인 + 279개 서브에이전트 세션, 4월 1-6일):
  - 자체 데이터에서 24개의 synthetic rate limit 항목(버그 3, 가짜 제한)을 확인
  - PRELIM/FINAL(예비/최종 로그) 비율: 전체 0.82배, 작업량 많은 날 최대 1.14배
  - 서브에이전트 분석: cache_read의 17.3%, output의 40.4%, cache_create의 62%를 차지
  - 세션 수명 주기: cache_read가 990턴에 걸쳐 24배 증가 (턴당 25K→595K, 턴당 +575 선형 증가)
  - **프록시 대 JSONL 교차 비교: JSONL cache_read는 프록시의 1.93배** — PRELIM 이중 계산(버그 8)을 직접 확인. JSONL 로그로 사용량을 추적하는 사용자는 실제보다 약 2배 부풀려진 수치를 보게 됩니다.

**방법론 참고:**
- 윈도우 B(피크 0.04)와 H(피크 0.02)는 활동량이 너무 적어서 1% 분석에서 제외했습니다
- 5시간 경계는 대략 5시간 간격이지만 완벽하게 균일하지는 않았습니다 (19:00→01:00 = 6시간, 01:00→12:00 = 비활동 시간 중 11시간 갭)
- 윈도우 간 1% cache_read 변동 ~1.35배 — 작업 유형에 따라 달라집니다
- 이 데이터는 Max 20x/$200 플랜 기준입니다 — 다른 요금제에서는 1%당 비용이 다를 수 있습니다
- 48시간 / 8개 윈도우는 작은 샘플입니다. 전체 7일 주기 분석은 4월 10일에 완료 예정입니다.

**게시된 코멘트:**
- [#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189807108) — @fgrosswig에게 프록시 데이터 공유, 서로 보완적인 분석 (그쪽의 JSONL + 우리의 헤더)
- [#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189847482) — @Commandershadow9에게 프록시 데이터 공유, 34배 계산 및 thinking 토큰 가설에 대한 방법론 참고사항

**레포 변경:**
- 레포지토리 이름 변경: `claude-code-cache-analysis` → `claude-code-hidden-problem-analysis`
- **RATELIMIT-HEADERS.md** 생성: 윈도우별 데이터를 포함한 전체 헤더 분석
- **README.md** 구조 개편 (653→175줄): 최신 업데이트를 상단에, 조사 타임라인, Anthropic 반박 사항, 문서 인덱스
- **BUGS.md** 생성: 버그 1-5, 8 세부사항을 README에서 분리
- **QUICKSTART.md** 생성: 설정 가이드 + 자가 진단을 README에서 분리
- **ISSUES.md** 생성: 91개 이상의 이슈 + 커뮤니티 도구 + 기여자를 README에서 분리
- **UPDATE-LOG.md** 생성: 이 문서
- 기존 문서 업데이트: BENCHMARK (이슈 수), TIMELINE (4월 6일 섹션, 버그 4 문구 수정)
- 기여자 추가: @fgrosswig, @Commandershadow9, @kolkov, @simpolism (확대)

**공개:** 위의 모든 내용.

---

## 향후 계획 — 4월 7-10일

**계획:**
- 4월 10일까지 rate limit 헤더 데이터 수집을 계속할 것입니다 (7일 윈도우 리셋을 관찰하기 위해)
- **Thinking 토큰 분리 테스트**: `alwaysThinkingEnabled: false`로 세션을 실행하고 1%당 사용률 비용을 비교할 것입니다. 크게 줄면 → thinking 토큰이 주범. 안 줄면 → cache-read 가중치 변경이 주범.
- 전체 7일 주기를 윈도우별로 추적한 분석을 공개할 예정입니다
- #38335 및 #41506 코멘트에 대한 커뮤니티 반응을 지켜볼 것입니다

---

*이 로그는 중요한 조사 활동이 있을 때마다 업데이트됩니다. rate limit 위기의 14개월 전체 역사는 [07_TIMELINE.md](07_TIMELINE.md)를 참조하십시오.*
