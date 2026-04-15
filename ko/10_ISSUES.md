> 이 문서는 [영어 원본](../10_ISSUES.md)의 한국어 번역입니다.

# 관련 이슈, 커뮤니티 도구 및 기여자

> 2026년 4월 9일 기준: **91개 이상의 추적 이슈 + 500건 이상의 새 이슈 (4월 6-9일)**. Anthropic GitHub 응답: **#42796에서 bcherny만** (6개 코멘트, 4월 6일). 나머지 모든 이슈에서 응답 없음.

---

## 카테고리별 이슈 인덱스

### 근본 원인 버그
- [#40524](https://github.com/anthropics/claude-code/issues/40524) — 대화 히스토리 무효화 (버그 1: sentinel) — **v2.1.89-91에서 개선**
- [#34629](https://github.com/anthropics/claude-code/issues/34629) — Resume 캐시 리그레션 (버그 2: deferred_tools_delta) — **v2.1.90-91에서 개선**
- [#40652](https://github.com/anthropics/claude-code/issues/40652) — cch= 빌링 해시 치환
- [#40584](https://github.com/anthropics/claude-code/issues/40584) — **클라이언트 측 허위 rate limiter** (버그 3: 151건의 `<synthetic>` 항목, 65개 세션 파일 (전체 기간; 4월 1-6일 분석 기간에는 24건 — [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) 참조)) — **미수정**
- [#42542](https://github.com/anthropics/claude-code/issues/42542) — **조용한 microcompact → 컨텍스트 품질 저하** (버그 4: 5,500건 이벤트, 18,858개 항목 삭제 — [13_PROXY-DATA.md](13_PROXY-DATA.md) 참조) — **미수정**
- 버그 5: **Tool 결과 budget enforcement** (200K 누적 캡, 167,818건 이벤트, 100% 잘림 — [13_PROXY-DATA.md](13_PROXY-DATA.md) 참조) — **미수정** (v2.1.91 MCP 오버라이드만 가능)
- [#41346](https://github.com/anthropics/claude-code/issues/41346) — **JSONL 로그 중복** (버그 8: 532개 파일에서 평균 2.37배 인플레이션, 최대 4.42배 — [13_PROXY-DATA.md](13_PROXY-DATA.md) 참조) — **미수정**

### 서버 측 빌링 버그
- [#42616](https://github.com/anthropics/claude-code/issues/42616) — 1M context Max 플랜에서 23K 토큰밖에 안 썼는데 허위 429 "Extra usage required" 발생
- [#42569](https://github.com/anthropics/claude-code/issues/42569) — Max 플랜에서 1M context가 추가 과금 사용량으로 잘못 표시
- [#37394](https://github.com/anthropics/claude-code/issues/37394) — 오래된 Docker 버전(업데이트되지 않은)이 빠르게 소진 — 서버 측 과금 변경 때문

### 토큰 인플레이션 메커니즘
- [#41663](https://github.com/anthropics/claude-code/issues/41663) — Prompt 캐시가 오히려 과도한 토큰 소비 유발
- [#41607](https://github.com/anthropics/claude-code/issues/41607) — 중복 compaction 서브에이전트 (5배 동일 작업)
- [#41767](https://github.com/anthropics/claude-code/issues/41767) — v2.1.89에서 Auto-compact 루프
- [#42260](https://github.com/anthropics/claude-code/issues/42260) — Resume이 thinking signature를 input 토큰으로 재생
- [#42256](https://github.com/anthropics/claude-code/issues/42256) — Read 도구가 매 메시지마다 과대 이미지를 재전송
- [#42590](https://github.com/anthropics/claude-code/issues/42590) — 1M context window에서 context compaction이 너무 공격적

### 신규 버그 (4월 9일)
- [#45419](https://github.com/anthropics/claude-code/issues/45419) — **`/branch` 컨텍스트 인플레이션** (버그 9: 한 메시지로 6%→73%, 3건의 중복 이슈) — **미수정**
- [#44703](https://github.com/anthropics/claude-code/issues/44703) — **TaskOutput deprecation → autocompact thrashing** (버그 10: 21배 컨텍스트 주입, 치명적 오류) — **수정 확인 없이 닫힘**
- [#44724](https://github.com/anthropics/claude-code/issues/44724) — **SendMessage resume 캐시 완전 미스** (버그 2a: 시스템 프롬프트 포함 `cache_read=0`, CLI resume과 구분됨) — **미수정**
- [#45286](https://github.com/anthropics/claude-code/issues/45286) — **JSONL 비원자적 쓰기 → 세션 손상** (버그 8a: 동시 도구 실행이 tool_result를 누락, [#21321](https://github.com/anthropics/claude-code/issues/21321)에 10건 이상 중복) — **미수정**
- Adaptive thinking zero-reasoning (버그 11) — **Anthropic이 HN에서 인정**, 모델팀 조사 중. 해결 방법: `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`

### 예비 발견 사항 (4월 9일, 검증 대기 중)
- [#45381](https://github.com/anthropics/claude-code/issues/45381) — 텔레메트리 비활성화 시 캐시 TTL이 1시간→5분으로 감소 (Anthropic `has repro`)
- [#45380](https://github.com/anthropics/claude-code/issues/45380) — Raw SDK 호출이 third-party 감지를 우회하여 추가 사용료 대신 플랜에 과금 (42건 이상 관련 오분류 이슈)
- v2.1.64 (3월 3일): "Output efficiency" 시스템 프롬프트 변경 — [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)에서 diff 추적

### 세션 로딩 / JSONL 파이프라인
- [#43044](https://github.com/anthropics/claude-code/issues/43044) — **v2.1.91에서 `--resume`이 0% 컨텍스트 로드** — 세션 로딩 파이프라인의 세 가지 리그레션 ([@kolkov](https://github.com/kolkov) 작성)

### Rate Limit 보고 (주요 스레드)
- [#16157](https://github.com/anthropics/claude-code/issues/16157) — 사용량 제한에 즉시 도달 (1400건 이상의 코멘트)
- [#42796](https://github.com/anthropics/claude-code/issues/42796) — Rate limit이 심각하게 저하됨 (HN 바이럴, **bcherny 응답** — Anthropic GitHub 응답이 있는 유일한 이슈)
- [#38335](https://github.com/anthropics/claude-code/issues/38335) — 세션 제한이 비정상적으로 빠르게 소진 (478건 이상 코멘트, 15일, Anthropic 응답 제로)
- [#41788](https://github.com/anthropics/claude-code/issues/41788) — 본 저자의 최초 보고 (Max 20, 약 70분 만에 100%)

---

## Anthropic 공식 대응

**GitHub (4월 9일 업데이트):** bcherny가 [#42796](https://github.com/anthropics/claude-code/issues/42796)에 4월 6일에만 6번 응답했습니다 (HN에서 58개 코멘트/일 급증으로 바이럴된 날). 나머지 모든 이슈에서 응답 없음 — #38335 (478개 코멘트, 15일), #41930 (49개 코멘트, 8일 이상), #42542를 포함합니다. 4월 6일 이후 bcherny는 GitHub에서도 침묵했습니다 (4월 7-8일: 코멘트 제로).

**HN (4월 6일):** bcherny가 adaptive thinking에 zero-reasoning 버그가 있어 조작(fabrication)이 발생한다고 인정했습니다. "모델팀과 조사 중입니다." 이것이 4월 6-9일 기간 동안 Anthropic이 인정한 유일한 버그입니다.

**2026년 4월 2일 — Lydia Hallie (Anthropic, Product)가 X에 게시:**

> *"피크 시간대 제한이 더 엄격해졌고 1M-context 세션이 커졌습니다, 여러분이 느끼는 것의 대부분은 그것입니다. 몇 가지 버그를 수정했지만, 과다 청구된 것은 없습니다."*

**우리의 측정 데이터는 이 평가에 의문을 제기합니다:**
- **버그 5 (200K 캡):** 도구 결과가 누적 200K를 넘으면 조용히 1-49자로 잘립니다. 1M context에 대해 비용을 지불하는 사용자들은 실제로는 내장 도구에서 200K만큼의 결과만 받고 있고 — 나머지는 사용자 모르게 버려집니다.
- **버그 3 (synthetic RL, 가짜 제한):** 65개 세션에서 151건의 `<synthetic>` 항목이 나왔습니다 (전체 기간; 4월 1-6일 분석 기간에는 24건 — [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) 참조). 클라이언트가 서버에 물어보지도 않고 스스로 API 호출을 차단해서 — 사용자는 실제로 사용량을 쓰지 않았는데도 "Rate limit reached"를 보게 됩니다.
- **버그 8 (PRELIM 중복):** Extended thinking 세션이 실제 API 호출보다 2-3배 많은 토큰 항목을 기록합니다. 서버 측 rate limiter가 이 부풀려진 수치를 기준으로 사용량을 계산하는지는 아직 답이 없는 질문입니다.

| 누구 | 플랫폼 | 내용 | 링크 |
|------|--------|------|------|
| **Lydia Hallie** | X | rate limit에 대한 전체 성명 (스레드 시작) | [게시물 1](https://x.com/lydiahallie/status/2039800715607187906) |
| **Lydia Hallie** | X | 사용 팁 후속 게시 (스레드 끝) | [게시물 2](https://x.com/lydiahallie/status/2039800718371307603) |
| **Lydia Hallie** | X | *"도움이 될 수정 사항을 배포했습니다"* (이전) | [게시물](https://x.com/lydiahallie/status/2039107775314428189) |
| **Thariq Shihipar** | X | *"prompt caching 관련 버그... 핫픽스 완료"* (이전 사건) | [게시물](https://x.com/trq212/status/2027232172810416493) |
| **공식 Changelog** | GitHub | v2.1.89-91 수정 항목 | [CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) |

---

## 커뮤니티 분석 및 도구

### 분석 도구
- [Reddit: Claude Code 캐시 버그 리버스 엔지니어링 분석](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)
- [cc-cache-fix](https://github.com/Rangizingo/cc-cache-fix) — 커뮤니티 캐시 패치 + 테스트 툴킷
- [cc-diag](https://github.com/nicobailey/cc-diag) — mitmproxy 기반 Claude Code 트래픽 분석
- [ccdiag](https://github.com/kolkov/ccdiag) — Go 기반 JSONL 세션 로그 복구 및 DAG 분석 도구 ([@kolkov](https://github.com/kolkov) 제작)
- [claude-code-router](https://github.com/pathintegral-institute/claude-code-router) — Claude Code용 투명 프록시
- [CUStats](https://custats.info) — 실시간 사용량 추적 및 시각화
- [context-stats](https://github.com/luongnv89/cc-context-stats) — 대화별 캐시 메트릭 내보내기 및 분석 ([@luongnv89](https://github.com/luongnv89) 제작)
- [BudMon](https://github.com/weilhalt/budmon) — rate-limit 헤더 모니터링용 데스크톱 대시보드
- [claude-usage-dashboard](https://github.com/fgrosswig/claude-usage-dashboard) — 상세 분석, 여러 컴퓨터 데이터 통합, 숨겨진 budget 추정 기능이 있는 독립형 Node.js JSONL 대시보드 ([@fgrosswig](https://github.com/fgrosswig) 제작)
- [Resume cache fix patch](https://gist.github.com/simpolism/302621e661f462f3e78684d96bf307ba) — v2.1.91에서 남아있는 두 가지 `--resume` 캐시 미스 수정 ([@simpolism](https://github.com/simpolism) 제작)

### 인터셉션 및 캐시 수정 도구 (4월 6-9일)
- [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) — `NODE_OPTIONS` fetch 인터셉터: 블록 위치 정규화, 도구 정의 순서 정렬, 이미지 carry-forward 제거. 98.3% 캐시 히트 보고 ([@cnighswonger](https://github.com/cnighswonger) 제작)
- [cc-trace](https://github.com/alexfazio/cc-trace) — mitmproxy 기반 CC API 인터셉션, 디버깅, 분석 (★152) ([@alexfazio](https://github.com/alexfazio) 제작)
- [X-Ray-Claude-Code-Interceptor](https://github.com/Renvect/X-Ray-Claude-Code-Interceptor) — 실시간 대시보드 + 오래된 도구 결과 "스마트 스트리핑"이 있는 Node.js 프록시 ([@Renvect](https://github.com/Renvect) 제작)
- [openwolf](https://github.com/cytostack/openwolf) — 토큰 사용량 안정화 (cytostack 제작)

### 시스템 프롬프트 및 컨텍스트 분석
- [claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) — 릴리스 노트가 포함된 버전별 CC 시스템 프롬프트 diff 추적 (Piebald-AI 제작)
- [context-engineering-papers](https://github.com/borcho23/context-engineering-papers) — Root Theorem: 컨텍스트 충실도 저하에 대한 형식적 프레임워크 ([@borcho23](https://github.com/borcho23) 제작)

### 토큰 최적화 도구
- [rtk](https://github.com/rtk-ai/rtk) — Tool output 압축
- [tokenlean](https://github.com/edimuj/tokenlean) — 에이전트용 54개 CLI 도구 (전체 파일 대신 심볼/스니펫만 추출하여 토큰 절약)

### 관련 세션 로딩 분석

[@kolkov](https://github.com/kolkov)가 v2.1.91 `--resume` 리그레션을 **JSONL DAG / 세션 로딩 레이어**에서 독립적으로 분석했습니다 — 이 레포에서 다루는 API 레벨 캐시 버그와는 다른 계층의 문제입니다.

**이슈:** [anthropics/claude-code#43044](https://github.com/anthropics/claude-code/issues/43044)

| # | 리그레션 | 영향 |
|---|---------|------|
| 1 | `walkChainBeforeParse` 제거 | 5MB 이상 JSONL 파일에서 fork pruning 불가 |
| 2 | `ExY` 타임스탬프 fallback이 fork 경계를 넘어 연결 | 타임스탬프 경계를 넘어 fork가 병합됨 |
| 3 | `leafUuids` 검사 누락 | 마지막 세션 감지가 잘못된 로그 파일을 선택 |

**도구:** [ccdiag](https://github.com/kolkov/ccdiag). 이것은 **API 이전 파이프라인**(JSONL 로그가 재구성되는 방식)을 다루며, 이 레포의 API 레벨 분석과는 다른 영역입니다.

---

<details>
<summary><strong>근본 원인 분석 + v2.1.91 업데이트가 게시된 91개 전체 이슈</strong> (클릭하여 펼치기 — 4월 9일 추가 이슈는 위의 카테고리별 섹션에 있습니다)</summary>

| # | 이슈 | 제목 |
|---|------|------|
| 1 | [#6457](https://github.com/anthropics/claude-code/issues/6457) | 5-hour limit reached in less than 1h30 |
| 2 | [#16157](https://github.com/anthropics/claude-code/issues/16157) | Instantly hitting usage limits with Max subscription |
| 3 | [#16856](https://github.com/anthropics/claude-code/issues/16856) | Excessive token usage in Claude Code 2.1.1 — 4x+ faster rate consumption |
| 4 | [#17016](https://github.com/anthropics/claude-code/issues/17016) | Claude hitting usage limits in no time |
| 5 | [#22435](https://github.com/anthropics/claude-code/issues/22435) | Inconsistent and Undisclosed Quota Accounting Changes |
| 6 | [#22876](https://github.com/anthropics/claude-code/issues/22876) | Rate limit 429 errors despite dashboard showing available quota |
| 7 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | Opus 4.6 token consumption significantly higher than 4.5 |
| 8 | [#34410](https://github.com/anthropics/claude-code/issues/34410) | Max x20 plan — 5-hour quota consumed in ~10 prompts |
| 9 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Claude Code Usage for Max Plan hitting limits extremely fast |
| 10 | [#38239](https://github.com/anthropics/claude-code/issues/38239) | Extremely rapid token consumption |
| 11 | [#38335](https://github.com/anthropics/claude-code/issues/38335) | Session limits exhausted abnormally fast since March 23 |
| 12 | [#38345](https://github.com/anthropics/claude-code/issues/38345) | Abnormal rate limit consumption — 1-2 prompts exhausting 5-hour window |
| 13 | [#38350](https://github.com/anthropics/claude-code/issues/38350) | Abnormal / inflated rate limit / session usage |
| 14 | [#38896](https://github.com/anthropics/claude-code/issues/38896) | Rate limit reached despite 0% usage |
| 15 | [#39938](https://github.com/anthropics/claude-code/issues/39938) | 93% rate limit consumed in ~3 minutes on fresh session |
| 16 | [#39966](https://github.com/anthropics/claude-code/issues/39966) | Abnormal token consumption rate on Opus 4.5 — 100% usage in ~1 hour |
| 17 | [#40438](https://github.com/anthropics/claude-code/issues/40438) | Rate Limit Reached without typing anything, /insights causes insane token usage |
| 18 | [#40524](https://github.com/anthropics/claude-code/issues/40524) | Conversation history invalidated on subsequent turns **(Root Cause: Sentinel)** |
| 19 | [#40535](https://github.com/anthropics/claude-code/issues/40535) | ClaudeMax — Session Throttling in Daily Limit |
| 20 | [#40652](https://github.com/anthropics/claude-code/issues/40652) | CLI mutates historical tool results via cch= billing hash substitution **(Root Cause)** |
| 21 | [#40790](https://github.com/anthropics/claude-code/issues/40790) | Excessive token consumption spike since March 23rd |
| 22 | [#40851](https://github.com/anthropics/claude-code/issues/40851) | Opus 4.6 (Max $100) — Quota reaches 93% after minimal prompting |
| 23 | [#40871](https://github.com/anthropics/claude-code/issues/40871) | Make one prompt and jump to 20% on Max x5 |
| 24 | [#40895](https://github.com/anthropics/claude-code/issues/40895) | Opus 4.6 on Max plan: usage seems disproportionately high per prompt |
| 25 | [#40903](https://github.com/anthropics/claude-code/issues/40903) | Max Plan usage limits significantly reduced compared to 2 weeks ago |
| 26 | [#40956](https://github.com/anthropics/claude-code/issues/40956) | Rate limit error on MAX+ plan when invoking slash commands |
| 27 | [#41001](https://github.com/anthropics/claude-code/issues/41001) | Team Premium seat weekly limit stuck at 100% after only 102M tokens |
| 28 | [#41055](https://github.com/anthropics/claude-code/issues/41055) | Weekly usage reset failed + incorrect token burn rate on $200 Max plan |
| 29 | [#41076](https://github.com/anthropics/claude-code/issues/41076) | Max5 plan hitting session limits daily (including off-peak hours) |
| 30 | [#41084](https://github.com/anthropics/claude-code/issues/41084) | Usage Limits Show 0% Daily But 41% Weekly — Phantom Usage |
| 31 | [#41158](https://github.com/anthropics/claude-code/issues/41158) | Excessive token consumption |
| 32 | [#41173](https://github.com/anthropics/claude-code/issues/41173) | Rate limit reached at 16% session usage on Max $200 plan |
| 33 | [#41174](https://github.com/anthropics/claude-code/issues/41174) | Usage limit reached in 10 minutes |
| 34 | [#41212](https://github.com/anthropics/claude-code/issues/41212) | Rate limit reached on ALL surfaces — Max 20x, 18% usage, non-peak |
| 35 | [#41249](https://github.com/anthropics/claude-code/issues/41249) | Excessive token consumption rate — usage depleting faster than expected |
| 36 | [#41346](https://github.com/anthropics/claude-code/issues/41346) | Extended thinking generates duplicate .jsonl log entries (~2-3x token inflation) |
| 37 | [#41377](https://github.com/anthropics/claude-code/issues/41377) | Status line usage % doesn't match website session usage |
| 38 | [#41424](https://github.com/anthropics/claude-code/issues/41424) | Claude Max + Claude Code rate limits exhausting abnormally fast |
| 39 | [#41504](https://github.com/anthropics/claude-code/issues/41504) | Max plan: usage exhausted abnormally fast, twice in one day |
| 40 | [#41506](https://github.com/anthropics/claude-code/issues/41506) | Token usage increased ~3-5x without any configuration change |
| 41 | [#41508](https://github.com/anthropics/claude-code/issues/41508) | 20x Max Plan User Hitting "Extra Usage" Limits After Single Session |
| 42 | [#41521](https://github.com/anthropics/claude-code/issues/41521) | Max Plan 5x Phantom Consumption and Large Token Usage Increased |
| 43 | [#41550](https://github.com/anthropics/claude-code/issues/41550) | Usage extremely high after 5-10 minutes into planning a feature |
| 44 | [#41567](https://github.com/anthropics/claude-code/issues/41567) | Not even doing real work and we're at 40% (Max x5) |
| 45 | [#41583](https://github.com/anthropics/claude-code/issues/41583) | Rate limit errors on Pro Plan at 26% usage |
| 46 | [#41590](https://github.com/anthropics/claude-code/issues/41590) | New account immediately hits rate limit with zero usage |
| 47 | [#41605](https://github.com/anthropics/claude-code/issues/41605) | /fast toggle reports "extra usage credits exhausted" on Pro Max |
| 48 | [#41607](https://github.com/anthropics/claude-code/issues/41607) | Duplicate compaction subagents spawned (5x identical work) |
| 49 | [#41617](https://github.com/anthropics/claude-code/issues/41617) | Excessive token consumption after recent updates (Max plan) |
| 50 | [#41640](https://github.com/anthropics/claude-code/issues/41640) | Excessive API usage consumption and degraded response performance |
| 51 | [#41651](https://github.com/anthropics/claude-code/issues/41651) | Frequent 529 Overloaded errors on Claude Max plan |
| 52 | [#41663](https://github.com/anthropics/claude-code/issues/41663) | Prompt Cache causes excessive token consumption (10-20x inflation) |
| 53 | [#41666](https://github.com/anthropics/claude-code/issues/41666) | v2.1.88 yanked release caused wasted session + significant token loss |
| 54 | [#41674](https://github.com/anthropics/claude-code/issues/41674) | Unexpectedly High API Usage and Rate Limit Hits |
| 55 | [#41728](https://github.com/anthropics/claude-code/issues/41728) | You've hit your limit · resets 5am (Europe/Istanbul) |
| 56 | [#41749](https://github.com/anthropics/claude-code/issues/41749) | Unexpected excessive API usage consumption |
| 57 | [#41750](https://github.com/anthropics/claude-code/issues/41750) | Context management fires on every turn with empty applied_edits |
| 58 | [#41767](https://github.com/anthropics/claude-code/issues/41767) | Auto-Compact loops continuously in v2.1.89 |
| 59 | [#41776](https://github.com/anthropics/claude-code/issues/41776) | API Error / Rate limit reached |
| 60 | [#41779](https://github.com/anthropics/claude-code/issues/41779) | Excessive token consumption compared to previous versions |
| 61 | [#41781](https://github.com/anthropics/claude-code/issues/41781) | You've hit your limit · resets 12am (America/Los_Angeles) |
| 62 | [#41788](https://github.com/anthropics/claude-code/issues/41788) | **Max 20 plan: 100% exhausted in ~70 min (my original report)** |
| 63 | [#41802](https://github.com/anthropics/claude-code/issues/41802) | Session hangs with excessive token consumption and no output |
| 64 | [#41812](https://github.com/anthropics/claude-code/issues/41812) | Excessive token usage due to startup overhead |
| 65 | [#41853](https://github.com/anthropics/claude-code/issues/41853) | Excessive token consumption without corresponding output |
| 66 | [#41854](https://github.com/anthropics/claude-code/issues/41854) | Token usage rate calculation may exceed weekly limits unexpectedly |
| 67 | [#41859](https://github.com/anthropics/claude-code/issues/41859) | Excessive token consumption causing unusable API costs |
| 68 | [#41866](https://github.com/anthropics/claude-code/issues/41866) | Extreme Token Burn with Claude Code CLI |
| 69 | [#41891](https://github.com/anthropics/claude-code/issues/41891) | Rate limit consumed without API calls |
| 70 | [#41928](https://github.com/anthropics/claude-code/issues/41928) | Accelerated restrictions and token usage |
| 71 | [#41930](https://github.com/anthropics/claude-code/issues/41930) | Critical: Widespread abnormal usage drain — multiple root causes |
| 72 | [#41944](https://github.com/anthropics/claude-code/issues/41944) | Usage limits not displaying after responses in CLI mode (2.1.89) |
| 73 | [#42003](https://github.com/anthropics/claude-code/issues/42003) | You've hit your limit · resets 12pm (America/Sao_Paulo) |
| 74 | [#42244](https://github.com/anthropics/claude-code/issues/42244) | 2.1.89 — terminal content disappearing |
| 75 | [#42247](https://github.com/anthropics/claude-code/issues/42247) | Flag to limit amount of turn history sent |
| 76 | [#42249](https://github.com/anthropics/claude-code/issues/42249) | Extreme token consumption — quota depleted in minutes |
| 77 | [#42256](https://github.com/anthropics/claude-code/issues/42256) | Read tool sends oversized images on every subsequent message |
| 78 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | Resume loads disproportionate tokens from opaque thinking signatures |
| 79 | [#42261](https://github.com/anthropics/claude-code/issues/42261) | You've hit your limit · resets 1am (Europe/London) |
| 80 | [#42272](https://github.com/anthropics/claude-code/issues/42272) | Excessive token consumption since 2.1.88 — 66% usage in 2 questions |
| 81 | [#42277](https://github.com/anthropics/claude-code/issues/42277) | New session doesn't reset usage limits correctly |
| 82 | [#42290](https://github.com/anthropics/claude-code/issues/42290) | /export truncates + /resume delivers incomplete context |
| — | [#42338](https://github.com/anthropics/claude-code/issues/42338) | Session resume invalidates entire prompt cache |
| 83 | [#42390](https://github.com/anthropics/claude-code/issues/42390) | Rate limit triggered despite 0% usage in /usage |
| 84 | [#42409](https://github.com/anthropics/claude-code/issues/42409) | Excessive API usage consumption during active session |
| 85 | [#42542](https://github.com/anthropics/claude-code/issues/42542) | **Silent context degradation — 3 microcompact mechanisms (Bug 4: Root Cause)** |
| 86 | [#42569](https://github.com/anthropics/claude-code/issues/42569) | 1M context window incorrectly shown as extra billable usage on Max plan |
| 87 | [#42583](https://github.com/anthropics/claude-code/issues/42583) | You've hit your limit — 1M actual vs 120-160K expected |
| 88 | [#42592](https://github.com/anthropics/claude-code/issues/42592) | Token consumption 100x faster after v2.1.88 — 21 min to 5-hour limit |
| 89 | [#42609](https://github.com/anthropics/claude-code/issues/42609) | Reached limit session in under 5 minutes (resume-triggered) |
| 90 | [#42616](https://github.com/anthropics/claude-code/issues/42616) | Spurious 429 "Extra usage required" at 23K tokens on Max plan with 1M context |
| 91 | [#42590](https://github.com/anthropics/claude-code/issues/42590) | Context compaction too aggressive on 1M context window (Opus 4.6) |

</details>

---

## 기여자 및 감사의 말

이 분석은 이 문제들을 독립적으로 조사하고 측정한 많은 커뮤니티 구성원들의 작업을 기반으로 합니다.

| 누구 | 기여 내용 |
|------|----------|
| [@Sn3th](https://github.com/Sn3th) | 세 가지 microcompact 메커니즘(버그 4) 발견, GrowthBook 플래그 추출, `applyToolResultBudget()` 파이프라인(버그 5), 여러 머신에서 서버 측 컨텍스트 변조 확인 |
| [@rwp65](https://github.com/rwp65) | 상세한 로그 증거와 함께 클라이언트 측 허위 rate limiter(버그 3) 발견 |
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
| [@cnighswonger](https://github.com/cnighswonger) | [claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix) 인터셉터 제작 — 4,700건 호출 측정, 98.3% 캐시 히트, 블록 분산(52-73%) 및 도구 순서(98%) 캐시 무효화 정량화, 이미지 carry-forward 버그(37개 이미지/266K 토큰) 식별, `ephemeral_1h`/`ephemeral_5m` 필드를 통한 TTL 티어 감지 |
| [@bilby91](https://github.com/bilby91) | [#44045](https://github.com/anthropics/claude-code/issues/44045) 최초 보고자 — resume 시 `skill_listing` + `companion_intro` 캐시 미스 식별, Agent SDK `query()` 경로로 확장 |
| [@labzink](https://github.com/labzink) | [#44724](https://github.com/anthropics/claude-code/issues/44724) — SendMessage 캐시 완전 미스(버그 2a) 식별, CLI resume 버그와 구분되는 호출별 캐시 메트릭 제공 |
| [@wpank](https://github.com/wpank) | 게이트웨이를 통해 47,810건 요청 추적, v2.1.63 vs v2.1.96 정량적 비교 ($10,700 총 지출), 18% 캐시 깨짐($927 낭비) 식별 |
| [@wjordan](https://github.com/wjordan) | [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)를 통해 v2.1.64의 "Output efficiency" 시스템 프롬프트 변경 식별 |
| [@EmpireJones](https://github.com/EmpireJones) | [#45381](https://github.com/anthropics/claude-code/issues/45381) — 텔레메트리-캐시 TTL 연결 발견 (Anthropic `has repro`) |
| [@arizonawayfarer](https://github.com/arizonawayfarer) | 크로스 플랫폼 일관성을 확인하는 Windows GrowthBook 플래그 덤프, JSONL `acompact-*` 도구 중복 분석 (35% 비율) |
| [@progerzua](https://github.com/progerzua) | [#45419](https://github.com/anthropics/claude-code/issues/45419) — `/branch` 컨텍스트 인플레이션 측정 (6%→73%, 카테고리별 분류) |

*이 분석은 커뮤니티 리서치와 개인 측정을 기반으로 합니다. Anthropic이 보증하지 않습니다. 모든 해결 방법은 공식 도구와 문서화된 기능만을 사용합니다.*
