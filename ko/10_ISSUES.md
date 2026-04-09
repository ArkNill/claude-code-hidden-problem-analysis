> 이 문서는 [영어 원본](../10_ISSUES.md)의 한국어 번역입니다.
>
> ⚠️ **이 번역은 4월 6일 기준입니다.** 4월 9일 추가된 신규 이슈, 도구, 기여자, Anthropic 응답 업데이트는 아직 번역되지 않았습니다. 최신 내용은 [영어 원본](../10_ISSUES.md)을 참조하십시오.

# 관련 이슈, 커뮤니티 도구 및 기여자

> 2026년 4월 6일 기준: **91개 이상의 고유 이슈에 180건 이상의 코멘트**가 접수되었습니다. GitHub에서의 Anthropic 공식 응답: **없음**.

---

## 카테고리별 이슈 인덱스

### 근본 원인 버그

이 문제들을 일으키는 핵심 버그들입니다. 각 버그의 상세 설명은 [01_BUGS.md](01_BUGS.md)를 참조하십시오.

- [#40524](https://github.com/anthropics/claude-code/issues/40524) — 대화 히스토리 무효화 (버그 1: sentinel) — **v2.1.89-91에서 개선**
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume 캐시 리그레션(이전 대화 이어가기 시 캐시 재활용 실패) (버그 2: deferred_tools_delta) — **v2.1.90-91에서 개선**
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= 빌링 해시 치환(과금 식별자가 잘못 바뀌는 문제)
- [#40584](https://github.com/anthropics/claude-code/issues/40584) — **클라이언트 측 허위 rate limiter**(실제 서버 제한이 아닌데 "제한 도달"이라고 뜨는 현상) (버그 3: 151건의 synthetic 항목 확인 (전체 기간; 4월 1-6일 분석 기간에는 24건 — [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) 참조)) — **미수정**
- [#42542](https://github.com/anthropics/claude-code/issues/42542) — **조용한 microcompact → 컨텍스트 품질 저하**(대화 맥락이 사용자 몰래 삭제되는 현상) (버그 4: 327건 이벤트, 캐시 영향 없음) — **미수정**
- 버그 5: **Tool 결과 budget enforcement**(도구가 돌려준 결과를 강제로 잘라내는 현상) (200K 누적 캡, GrowthBook 플래그를 통해 발견) — **미수정** (v2.1.91 MCP 오버라이드만 가능)
- [#41346](https://github.com/anthropics/claude-code/issues/41346) — **JSONL 로그 중복**(세션 기록에 같은 내용이 여러 번 기록되는 현상) (버그 8: 2.87배 PRELIM 인플레이션 (단일 세션 측정치; 전체 세션 평균 1.93x — [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) 참조)) — **미수정**

### 서버 측 빌링 버그

Anthropic 서버 쪽에서 발생하는 과금 관련 문제들입니다.

- [#42616](https://github.com/anthropics/claude-code/issues/42616) — 1M context Max 플랜에서 23K 토큰밖에 안 썼는데 허위 429 "Extra usage required" 발생
- [#42569](https://github.com/anthropics/claude-code/issues/42569) — Max 플랜에서 1M context가 추가 과금 사용량으로 잘못 표시
- [#37394](https://github.com/anthropics/claude-code/issues/37394) — 오래된 Docker 버전(업데이트되지 않은)이 빠르게 소진 — 서버 측 과금 변경 때문

### 토큰 인플레이션 메커니즘

실제보다 토큰이 더 많이 소비되는 것처럼 보이거나, 실제로 더 많이 소비되게 만드는 문제들입니다.

- [#41663](https://github.com/anthropics/claude-code/issues/41663) — Prompt 캐시(이전 대화 재활용 기능)가 오히려 과도한 토큰 소비 유발
- [#41607](https://github.com/anthropics/claude-code/issues/41607) — 중복 compaction 서브에이전트(대화 압축 보조 AI가 같은 작업을 5번 반복)
- [#41767](https://github.com/anthropics/claude-code/issues/41767) — v2.1.89에서 Auto-compact 루프(자동 압축이 끝없이 반복)
- [#42260](https://github.com/anthropics/claude-code/issues/42260) — Resume이 thinking signature(AI 사고 과정 서명)를 input 토큰으로 재생
- [#42256](https://github.com/anthropics/claude-code/issues/42256) — Read 도구가 매 메시지마다 과대 이미지를 재전송
- [#42590](https://github.com/anthropics/claude-code/issues/42590) — 1M context window에서 context compaction(대화 압축)이 너무 공격적

### 세션 로딩 / JSONL 파이프라인

세션(대화)을 불러오거나 기록하는 과정에서 발생하는 문제들입니다.

- [#43044](https://github.com/anthropics/claude-code/issues/43044) — **v2.1.91에서 `--resume`이 0% 컨텍스트 로드** — 세션 로딩 파이프라인의 세 가지 리그레션(이전 버전보다 못해진 점) ([@kolkov](https://github.com/kolkov) 작성)

### Rate Limit 보고 (주요 스레드)

사용량 한도에 너무 빨리 도달한다는 사용자 보고들 중 가장 많은 논의가 이루어진 스레드입니다.

- [#16157](https://github.com/anthropics/claude-code/issues/16157) — 사용량 제한에 즉시 도달 (1400건 이상의 코멘트)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — 세션 제한이 비정상적으로 빠르게 소진 (300건 이상의 코멘트)
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — 본 저자의 최초 보고 (Max 20, 약 70분 만에 100%)

---

## Anthropic 공식 대응

**GitHub:** 91개 이상의 rate-limit 이슈에 대한 응답 없음 (2개월 이상의 침묵).

**2026년 4월 2일 — Lydia Hallie (Anthropic, Product)가 X에 게시:**

> *"피크 시간대 제한이 더 엄격해졌고 1M-context 세션이 커졌습니다, 여러분이 느끼는 것의 대부분은 그것입니다. 몇 가지 버그를 수정했지만, 과다 청구된 것은 없습니다."*

**우리의 측정 데이터는 이 평가에 의문을 제기합니다:**
- **버그 5 (200K 캡):** 도구 결과가 누적 200K를 넘으면 조용히 1-41자로 잘립니다. 1M context에 대해 비용을 지불하는 사용자들은 실제로는 내장 도구에서 200K만큼의 결과만 받고 있고 — 나머지는 사용자 모르게 버려집니다.
- **버그 3 (synthetic RL, 가짜 제한):** 자체 환경만으로도 65개 세션에서 151건의 `<synthetic>` 항목이 나왔습니다. 클라이언트가 서버에 물어보지도 않고 스스로 API 호출을 차단해서 — 사용자는 실제로 사용량을 쓰지 않았는데도 "Rate limit reached"를 보게 됩니다.
- **버그 8 (PRELIM 중복):** Extended thinking(확장 사고) 세션이 실제 API 호출보다 2-3배 많은 토큰 항목을 기록합니다. 서버 측 rate limiter가 이 부풀려진 수치를 기준으로 사용량을 계산하는지는 아직 답이 없는 질문입니다.

| 누구 | 플랫폼 | 내용 | 링크 |
|------|--------|------|------|
| **Lydia Hallie** | X | rate limit에 대한 전체 성명 (스레드 시작) | [게시물 1](https://x.com/lydiahallie/status/2039800715607187906) |
| **Lydia Hallie** | X | 사용 팁 후속 게시 (스레드 끝) | [게시물 2](https://x.com/lydiahallie/status/2039800718371307603) |
| **Lydia Hallie** | X | *"도움이 될 수정 사항을 배포했습니다"* (이전) | [게시물](https://x.com/lydiahallie/status/2039107775314428189) |
| **Thariq Shihipar** | X | *"prompt caching 관련 버그... 핫픽스 완료"* (이전 사건) | [게시물](https://x.com/trq212/status/2027232172810416493) |
| **공식 Changelog** | GitHub | v2.1.89-91 수정 항목 | [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

---

## 커뮤니티 분석 및 도구

커뮤니티 구성원들이 이 문제를 분석하고 대응하기 위해 만든 도구들입니다.

### 분석 도구

문제를 진단하고 사용량을 모니터링하는 데 도움이 되는 도구들입니다.

- [Reddit: Claude Code 캐시 버그 리버스 엔지니어링 분석](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — 커뮤니티 캐시 패치 + 테스트 툴킷
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy(네트워크 트래픽 분석 도구) 기반 Claude Code 트래픽 분석
- [ccdiag](https://github.com/kolkov/ccdiag) — Go 기반 JSONL 세션 로그 복구 및 DAG(세션 흐름 그래프) 분석 도구 ([@kolkov](https://github.com/kolkov) 제작)
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Claude Code용 투명 프록시(통신을 중간에서 기록하는 도구)
- [CUStats](https://custats.info) — 실시간 사용량 추적 및 시각화
- [context-stats](https://github.com/luongnv89/cc-context-stats) — 대화별 캐시 메트릭(측정값) 내보내기 및 분석 ([@luongnv89](https://github.com/luongnv89) 제작)
- [BudMon](https://github.com/weilhalt/budmon) — rate-limit 헤더(사용량 제한 정보) 모니터링용 데스크톱 대시보드
- [claude-usage-dashboard](https://github.com/fgrosswig/claude-usage-dashboard) — 상세 분석, 여러 컴퓨터 데이터 통합, 숨겨진 budget 추정 기능이 있는 독립형 Node.js JSONL 대시보드 ([@fgrosswig](https://github.com/fgrosswig) 제작)
- [Resume cache fix patch](https://gist.github.com/simpolism/302621e661f462f3e78684d96bf307ba) — v2.1.91에서 남아있는 두 가지 `--resume` 캐시 미스(캐시 재활용 실패) 수정 ([@simpolism](https://github.com/simpolism) 제작)

### 토큰 최적화 도구

토큰을 절약하는 데 도움이 되는 도구들입니다.

- [rtk](https://github.com/rtk-ai/rtk) — Tool output 압축(도구 결과를 작게 줄여서 토큰 절약)
- [tokenlean](https://github.com/edimuj/tokenlean) — 에이전트용 54개 CLI 도구 (전체 파일을 읽는 대신 필요한 부분만 추출해서 토큰 절약)

### 관련 세션 로딩 분석

[@kolkov](https://github.com/kolkov)가 v2.1.91 `--resume` 리그레션(이전 대화 이어가기 기능의 퇴보)을 **JSONL DAG / 세션 로딩 레이어**(세션 기록이 로드되는 내부 단계)에서 독립적으로 분석했습니다 — 이 레포에서 다루는 API 레벨 캐시 버그와는 다른 계층의 문제입니다.

**이슈:** [anthropics/claude-code#43044](https://github.com/anthropics/claude-code/issues/43044)

| # | 리그레션 | 영향 |
|---|---------|------|
| 1 | `walkChainBeforeParse` 제거 | 5MB 이상 JSONL 파일에서 fork pruning 불가 |
| 2 | `ExY` 타임스탬프 fallback이 fork 경계를 넘어 연결 | 타임스탬프 경계를 넘어 fork가 병합됨 |
| 3 | `leafUuids` 검사 누락 | 마지막 세션 감지가 잘못된 로그 파일을 선택 |

**도구:** [ccdiag](https://github.com/kolkov/ccdiag). 이것은 **API 이전 파이프라인**(JSONL 로그가 재구성되는 방식, 즉 서버에 보내기 전 단계)을 다루며, 이 레포의 API 레벨 분석과는 다른 영역입니다.

---

<details>
<summary><strong>근본 원인 분석 + v2.1.91 업데이트가 게시된 91개 전체 이슈</strong> (클릭하여 펼치기)</summary>

| # | 이슈 | 제목 |
|---|------|------|
| 1 | [#6457](https://github.com/anthropics/claude-code/issues/6457) | 5시간 제한이 1시간 30분 이내에 도달 |
| 2 | [#16157](https://github.com/anthropics/claude-code/issues/16157) | Max 구독으로 사용량 제한에 즉시 도달 |
| 3 | [#16856](https://github.com/anthropics/claude-code/issues/16856) | Claude Code 2.1.1에서 과도한 토큰 사용 — 4배 이상 빠른 소비 |
| 4 | [#17016](https://github.com/anthropics/claude-code/issues/17016) | Claude가 순식간에 사용량 제한에 도달 |
| 5 | [#22435](https://github.com/anthropics/claude-code/issues/22435) | 일관성 없고 공개되지 않은 쿼터 과금 변경 |
| 6 | [#22876](https://github.com/anthropics/claude-code/issues/22876) | 대시보드에 사용 가능한 쿼터가 표시됨에도 rate limit 429 오류 |
| 7 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | Opus 4.6 토큰 소비가 4.5보다 현저히 높음 |
| 8 | [#34410](https://github.com/anthropics/claude-code/issues/34410) | Max x20 플랜 — 5시간 쿼터가 약 10회 프롬프트에 소진 |
| 9 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Max 플랜 Claude Code 사용량이 극도로 빠르게 제한에 도달 |
| 10 | [#38239](https://github.com/anthropics/claude-code/issues/38239) | 극도로 빠른 토큰 소비 |
| 11 | [#38335](https://github.com/anthropics/claude-code/issues/38335) | 3월 23일 이후 세션 제한이 비정상적으로 빠르게 소진 |
| 12 | [#38345](https://github.com/anthropics/claude-code/issues/38345) | 비정상적 rate limit 소비 — 1-2회 프롬프트에 5시간 윈도우 소진 |
| 13 | [#38350](https://github.com/anthropics/claude-code/issues/38350) | 비정상 / 부풀린 rate limit / 세션 사용량 |
| 14 | [#38896](https://github.com/anthropics/claude-code/issues/38896) | 사용량 0%임에도 rate limit 도달 |
| 15 | [#39938](https://github.com/anthropics/claude-code/issues/39938) | 새 세션에서 약 3분 만에 93% rate limit 소비 |
| 16 | [#39966](https://github.com/anthropics/claude-code/issues/39966) | Opus 4.5에서 비정상적 토큰 소비율 — 약 1시간에 100% 사용 |
| 17 | [#40438](https://github.com/anthropics/claude-code/issues/40438) | 아무 입력 없이 rate limit 도달, /insights가 비정상적 토큰 사용 유발 |
| 18 | [#40524](https://github.com/anthropics/claude-code/issues/40524) | 이후 턴에서 대화 히스토리 무효화 **(근본 원인: Sentinel)** |
| 19 | [#40535](https://github.com/anthropics/claude-code/issues/40535) | ClaudeMax — 일일 제한 내 세션 스로틀링 |
| 20 | [#40652](https://github.com/anthropics/claude-code/issues/40652) | CLI가 cch= 빌링 해시 치환으로 과거 tool 결과를 변조 **(근본 원인)** |
| 21 | [#40790](https://github.com/anthropics/claude-code/issues/40790) | 3월 23일 이후 과도한 토큰 소비 급증 |
| 22 | [#40851](https://github.com/anthropics/claude-code/issues/40851) | Opus 4.6 (Max $100) — 최소한의 프롬프트 후 쿼터 93% 도달 |
| 23 | [#40871](https://github.com/anthropics/claude-code/issues/40871) | Max x5에서 프롬프트 하나에 20%로 급증 |
| 24 | [#40895](https://github.com/anthropics/claude-code/issues/40895) | Max 플랜에서 Opus 4.6: 프롬프트당 사용량이 불균형적으로 높음 |
| 25 | [#40903](https://github.com/anthropics/claude-code/issues/40903) | Max 플랜 사용량 제한이 2주 전 대비 크게 감소 |
| 26 | [#40956](https://github.com/anthropics/claude-code/issues/40956) | MAX+ 플랜에서 슬래시 명령어 호출 시 rate limit 오류 |
| 27 | [#41001](https://github.com/anthropics/claude-code/issues/41001) | Team Premium 시트 주간 제한이 102M 토큰만으로 100%에 고정 |
| 28 | [#41055](https://github.com/anthropics/claude-code/issues/41055) | 주간 사용량 리셋 실패 + $200 Max 플랜에서 부정확한 토큰 소비율 |
| 29 | [#41076](https://github.com/anthropics/claude-code/issues/41076) | Max5 플랜에서 매일 세션 제한 도달 (오프피크 시간 포함) |
| 30 | [#41084](https://github.com/anthropics/claude-code/issues/41084) | 일일 사용량은 0%이나 주간은 41% — 팬텀 사용량 |
| 31 | [#41158](https://github.com/anthropics/claude-code/issues/41158) | 과도한 토큰 소비 |
| 32 | [#41173](https://github.com/anthropics/claude-code/issues/41173) | Max $200 플랜에서 16% 세션 사용량에 rate limit 도달 |
| 33 | [#41174](https://github.com/anthropics/claude-code/issues/41174) | 10분 만에 사용량 제한 도달 |
| 34 | [#41212](https://github.com/anthropics/claude-code/issues/41212) | 모든 서비스에서 rate limit 도달 — Max 20x, 18% 사용량, 비피크 |
| 35 | [#41249](https://github.com/anthropics/claude-code/issues/41249) | 과도한 토큰 소비율 — 사용량이 예상보다 빠르게 소진 |
| 36 | [#41346](https://github.com/anthropics/claude-code/issues/41346) | Extended thinking이 중복 .jsonl 로그 항목 생성 (약 2-3배 토큰 인플레이션) |
| 37 | [#41377](https://github.com/anthropics/claude-code/issues/41377) | 상태 표시줄 사용량 %가 웹사이트 세션 사용량과 불일치 |
| 38 | [#41424](https://github.com/anthropics/claude-code/issues/41424) | Claude Max 플랜에서 빈번한 529 Overloaded 오류 |
| 39 | [#41504](https://github.com/anthropics/claude-code/issues/41504) | Max 플랜: 하루에 두 번 사용량이 비정상적으로 빠르게 소진 |
| 40 | [#41506](https://github.com/anthropics/claude-code/issues/41506) | 설정 변경 없이 토큰 사용량이 약 3-5배 증가 |
| 41 | [#41508](https://github.com/anthropics/claude-code/issues/41508) | 20x Max 플랜 사용자가 단일 세션 후 "Extra Usage" 제한 도달 |
| 42 | [#41521](https://github.com/anthropics/claude-code/issues/41521) | Max 플랜 5x 팬텀 소비 및 대량 토큰 사용 증가 |
| 43 | [#41550](https://github.com/anthropics/claude-code/issues/41550) | 기능 기획 시작 5-10분 만에 극도로 높은 사용량 |
| 44 | [#41567](https://github.com/anthropics/claude-code/issues/41567) | 실제 작업도 안 했는데 40% (Max x5) |
| 45 | [#41583](https://github.com/anthropics/claude-code/issues/41583) | Pro 플랜에서 26% 사용량에 rate limit 오류 |
| 46 | [#41590](https://github.com/anthropics/claude-code/issues/41590) | 새 계정이 사용량 0에서 즉시 rate limit 도달 |
| 47 | [#41605](https://github.com/anthropics/claude-code/issues/41605) | Pro Max에서 /fast 토글이 "extra usage credits exhausted" 보고 |
| 48 | [#41607](https://github.com/anthropics/claude-code/issues/41607) | 중복 compaction 서브에이전트 생성 (5배 동일 작업) |
| 49 | [#41617](https://github.com/anthropics/claude-code/issues/41617) | 최근 업데이트 이후 과도한 토큰 소비 (Max 플랜) |
| 50 | [#41640](https://github.com/anthropics/claude-code/issues/41640) | 과도한 API 사용량 소비 및 응답 성능 저하 |
| 51 | [#41651](https://github.com/anthropics/claude-code/issues/41651) | Claude Max 플랜에서 빈번한 529 Overloaded 오류 |
| 52 | [#41663](https://github.com/anthropics/claude-code/issues/41663) | Prompt 캐시가 과도한 토큰 소비 유발 (10-20배 인플레이션) |
| 53 | [#41666](https://github.com/anthropics/claude-code/issues/41666) | v2.1.88 철회된 릴리스가 세션 낭비 + 상당한 토큰 손실 유발 |
| 54 | [#41674](https://github.com/anthropics/claude-code/issues/41674) | 예상치 못한 높은 API 사용량 및 rate limit 도달 |
| 55 | [#41728](https://github.com/anthropics/claude-code/issues/41728) | 제한 도달 · 오전 5시 리셋 (Europe/Istanbul) |
| 56 | [#41749](https://github.com/anthropics/claude-code/issues/41749) | 예상치 못한 과도한 API 사용량 소비 |
| 57 | [#41750](https://github.com/anthropics/claude-code/issues/41750) | 빈 applied_edits로 매 턴마다 context management 실행 |
| 58 | [#41767](https://github.com/anthropics/claude-code/issues/41767) | v2.1.89에서 Auto-Compact 연속 루프 |
| 59 | [#41776](https://github.com/anthropics/claude-code/issues/41776) | API 오류 / rate limit 도달 |
| 60 | [#41779](https://github.com/anthropics/claude-code/issues/41779) | 이전 버전 대비 과도한 토큰 소비 |
| 61 | [#41781](https://github.com/anthropics/claude-code/issues/41781) | 제한 도달 · 자정 리셋 (America/Los_Angeles) |
| 62 | [#41788](https://github.com/anthropics/claude-code/issues/41788) | **Max 20 플랜: 약 70분 만에 100% 소진 (본 저자의 최초 보고)** |
| 63 | [#41802](https://github.com/anthropics/claude-code/issues/41802) | 과도한 토큰 소비와 출력 없이 세션 정지 |
| 64 | [#41812](https://github.com/anthropics/claude-code/issues/41812) | 시작 오버헤드로 인한 과도한 토큰 사용 |
| 65 | [#41853](https://github.com/anthropics/claude-code/issues/41853) | 상응하는 출력 없이 과도한 토큰 소비 |
| 66 | [#41854](https://github.com/anthropics/claude-code/issues/41854) | 토큰 사용률 계산이 주간 제한을 예상치 못하게 초과할 수 있음 |
| 67 | [#41859](https://github.com/anthropics/claude-code/issues/41859) | 과도한 토큰 소비로 API 비용 사용 불가 수준 |
| 68 | [#41866](https://github.com/anthropics/claude-code/issues/41866) | Claude Code CLI에서 극단적 토큰 소모 |
| 69 | [#41891](https://github.com/anthropics/claude-code/issues/41891) | API 호출 없이 rate limit 소비 |
| 70 | [#41928](https://github.com/anthropics/claude-code/issues/41928) | 가속화된 제한 및 토큰 사용 |
| 71 | [#41930](https://github.com/anthropics/claude-code/issues/41930) | 치명적: 광범위한 비정상 사용량 소진 — 다수의 근본 원인 |
| 72 | [#41944](https://github.com/anthropics/claude-code/issues/41944) | CLI 모드에서 응답 후 사용량 제한 미표시 (2.1.89) |
| 73 | [#42003](https://github.com/anthropics/claude-code/issues/42003) | 제한 도달 · 오후 12시 리셋 (America/Sao_Paulo) |
| 74 | [#42244](https://github.com/anthropics/claude-code/issues/42244) | 2.1.89 — 터미널 콘텐츠 사라짐 |
| 75 | [#42247](https://github.com/anthropics/claude-code/issues/42247) | 전송되는 턴 히스토리 양을 제한하는 플래그 |
| 76 | [#42249](https://github.com/anthropics/claude-code/issues/42249) | 극단적 토큰 소비 — 쿼터가 수 분 만에 소진 |
| 77 | [#42256](https://github.com/anthropics/claude-code/issues/42256) | Read 도구가 매 후속 메시지마다 과대 이미지 전송 |
| 78 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | Resume이 불투명한 thinking signature에서 불균형적 토큰 로드 |
| 79 | [#42261](https://github.com/anthropics/claude-code/issues/42261) | 제한 도달 · 오전 1시 리셋 (Europe/London) |
| 80 | [#42272](https://github.com/anthropics/claude-code/issues/42272) | 2.1.88 이후 과도한 토큰 소비 — 2개 질문에 66% 사용 |
| 81 | [#42277](https://github.com/anthropics/claude-code/issues/42277) | 새 세션이 사용량 제한을 올바르게 리셋하지 않음 |
| 82 | [#42290](https://github.com/anthropics/claude-code/issues/42290) | /export 잘림 + /resume이 불완전한 컨텍스트 전달 |
| — | [#42338](https://github.com/anthropics/claude-code/issues/42338) | 세션 resume이 전체 prompt 캐시를 무효화 |
| 83 | [#42390](https://github.com/anthropics/claude-code/issues/42390) | /usage에서 0% 사용량인데도 rate limit 발동 |
| 84 | [#42409](https://github.com/anthropics/claude-code/issues/42409) | 활성 세션 중 과도한 API 사용량 소비 |
| 85 | [#42542](https://github.com/anthropics/claude-code/issues/42542) | **조용한 컨텍스트 품질 저하 — 3가지 microcompact 메커니즘 (버그 4: 근본 원인)** |
| 86 | [#42569](https://github.com/anthropics/claude-code/issues/42569) | Max 플랜에서 1M context window가 추가 과금 사용량으로 잘못 표시 |
| 87 | [#42583](https://github.com/anthropics/claude-code/issues/42583) | 제한 도달 — 실제 1M vs 예상 120-160K |
| 88 | [#42592](https://github.com/anthropics/claude-code/issues/42592) | v2.1.88 이후 토큰 소비 100배 빨라짐 — 21분 만에 5시간 제한 도달 |
| 89 | [#42609](https://github.com/anthropics/claude-code/issues/42609) | 5분 이내에 세션 제한 도달 (resume으로 발동) |
| 90 | [#42616](https://github.com/anthropics/claude-code/issues/42616) | 1M context Max 플랜에서 23K 토큰 시점에 허위 429 "Extra usage required" |
| 91 | [#42590](https://github.com/anthropics/claude-code/issues/42590) | 1M context window에서 context compaction이 너무 공격적 (Opus 4.6) |

</details>

---

## 기여자 및 감사의 말

이 분석은 이 문제들을 독립적으로 조사하고 측정한 많은 커뮤니티 구성원들의 작업을 기반으로 합니다.

| 누구 | 기여 내용 |
|------|----------|
| [@Sn3th](https://github.com/Sn3th) | 세 가지 microcompact 메커니즘(버그 4) 발견, GrowthBook 플래그 추출, `applyToolResultBudget()` 파이프라인(버그 5), 여러 머신에서 서버 측 컨텍스트 변조 확인 |
| [@rwp65](https://github.com/rwp65) | 상세한 로그 증거와 함께 클라이언트 측 허위 rate limiter(버그 3) 발견 |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | 크로스 플랫폼 일관성을 확인하는 Windows GrowthBook 플래그 덤프 |
| [@dbrunet73](https://github.com/dbrunet73) | 캐시 개선을 확인하는 실제 OTel 비교 데이터 (v2.1.88 vs v2.1.90) |
| [@maiarowsky](https://github.com/maiarowsky) | 13개 세션에서 26건의 synthetic 항목으로 v2.1.90에서 버그 3 확인 |
| [@luongnv89](https://github.com/luongnv89) | 캐시 TTL 분석, [CUStats](https://custats.info) 및 [context-stats](https://github.com/luongnv89/cc-context-stats) 제작 |
| [@edimuj](https://github.com/edimuj) | grep/file-read 토큰 낭비 측정 (3.5M 토큰 / 1,800건 이상 호출), [tokenlean](https://github.com/edimuj/tokenlean) 제작 |
| [@amicicixp](https://github.com/amicicixp) | 전후 비교 테스트로 v2.1.90 캐시 개선 검증 |
| [@simpolism](https://github.com/simpolism) | v2.1.90 changelog 상관관계 확인, resume 캐시 수정 패치 제작 (99.7-99.9% 적중률) |
| [@kolkov](https://github.com/kolkov) | [ccdiag](https://github.com/kolkov/ccdiag) 제작, 세 가지 v2.1.91 `--resume` 리그레션 확인 ([#43044](https://github.com/anthropics/claude-code/issues/43044)) |
| [@weilhalt](https://github.com/weilhalt) | 실시간 rate-limit 헤더 모니터링용 [BudMon](https://github.com/weilhalt/budmon) 제작 |
| [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) | 오래된 Docker 버전 소진 보고 — 핵심 서버 측 증거 ([#37394](https://github.com/anthropics/claude-code/issues/37394)) |
| [@dancinlife](https://github.com/dancinlife) | `organizationUuid` 기반 쿼터 풀링 및 `oauthAccount` 교차 오염 버그 발견 |
| [@SC7639](https://github.com/SC7639) | 3월 중순 타임라인을 확인하는 추가 리그레션 데이터 |
| [@fgrosswig](https://github.com/fgrosswig) | 64배 budget 감소 포렌식 — 듀얼 머신 18일간 JSONL 전후 비교 ([#38335](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)) |
| [@Commandershadow9](https://github.com/Commandershadow9) | 캐시 수정 확인, 34-143배 용량 감소 문서화, thinking 토큰 가설 제기 ([#41506](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)) |
| Reddit 커뮤니티 | 캐시 sentinel 메커니즘의 [리버스 엔지니어링 분석](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6) |

*이 분석은 커뮤니티 리서치와 개인 측정을 기반으로 합니다. Anthropic이 보증하지 않습니다. 모든 해결 방법은 공식 도구와 문서화된 기능만을 사용합니다.*
