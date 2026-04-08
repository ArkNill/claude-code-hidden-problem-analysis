> 🇰🇷 이 문서는 [영어 원본](../07_TIMELINE.md)의 한국어 번역입니다.

# Claude Code Rate Limit 위기 — 전체 타임라인

> Claude Code의 rate limit(사용량 제한) / token(AI가 처리하는 텍스트 단위) 소비 문제에 대한 14개월간의 사건 기록입니다.
> 2026-04-02에 GitHub Issues API를 통해 데이터를 수집했습니다. 각 이벤트 수는 발견 시점 기준입니다.
>
> **최신 데이터 (4월 8일):** 1주일간의 프록시 데이터셋 (17,610건 요청, 129개 세션)과 업데이트된 이벤트 수는 [13_PROXY-DATA.md](../13_PROXY-DATA.md)를 참조하십시오.

---

## 요약

| 지표 | 값 |
|--------|-------|
| **기간** | 14개월 (2025-02 ~ 2026-04-03, 진행 중) |
| **주요 에스컬레이션 사이클** | 4회 |
| **최대 이슈** | #16157 — 1,422 댓글, 647 thumbs_up |
| **확인된 근본 원인** | 클라이언트 측 11건 + 서버 측 6건 (복합적) |
| **Anthropic 공식 대응** | X/Twitter만 해당 (Lydia Hallie, 4월 2-3일). 91개 이상의 이슈에서 GitHub 응답 없음 |

---

## Phase 1 — 초기 신호 (2025-02 ~ 2025-05)

여기저기서 드문드문 문제가 보고되던 시기입니다. Anthropic 측에서는 "그 사용자 개인의 문제"로 취급했습니다.

| 날짜 | 이슈 | 제목 | 댓글 수 |
|------|-------|-------|----------|
| 2025-02-24 | [#16](https://github.com/anthropics/claude-code/issues/16) | Add ability to add per-key spend limits | 3 |
| 2025-02-28 | [#236](https://github.com/anthropics/claude-code/issues/236) | API Error on AWS bedrock (429) | 2 |
| 2025-03-03 | [#274](https://github.com/anthropics/claude-code/issues/274) | IPYNB Cell Output Consumes Significant Portion of Context Window | 2 |
| 2025-03-26 | [#624](https://github.com/anthropics/claude-code/issues/624) | Potential Token Consumption Anomaly in Anthropic API Interaction | 2 |
| 2025-05-17 | [#1138](https://github.com/anthropics/claude-code/issues/1138) | [BUG] Cost increased a lot | 6 |

---

## Phase 2 — Claude 4 출시 이후 (2025-05-22 ~ 2025-06)

Claude 4가 출시되면서 첫 번째 불만의 파도가 밀려왔습니다. 사용자들이 "사용량이 10배나 빨리 소진된다"고 보고하기 시작했습니다.

| 날짜 | 이슈 | 제목 | 댓글 수 |
|------|-------|-------|----------|
| 2025-05-23 | [#1266](https://github.com/anthropics/claude-code/issues/1266) | **[BUG] Claude 4 is BURNING USAGE LIMITS x10 TIMES FASTER** | 9 |
| 2025-05-24 | [#1287](https://github.com/anthropics/claude-code/issues/1287) | Misleading Cost Command Output in Claude Code | 10 |
| 2025-05-26 | [#1328](https://github.com/anthropics/claude-code/issues/1328) | Claude AI usage limit reached | 9 |
| 2025-05-30 | [#1439](https://github.com/anthropics/claude-code/issues/1439) | Sonnet 4 is completely unusable because of 429s | 7 |
| 2025-06-02 | [#1492](https://github.com/anthropics/claude-code/issues/1492) | Hitting usage limits after 2 hours | 3 |
| 2025-06-09 | [#1836](https://github.com/anthropics/claude-code/issues/1836) | Claude AI usage limit reached | 4 |

---

## Phase 3 — 구조적 결함 발견 (2025-07 ~ 2025-08)

"내 환경 탓인가?" 수준의 개별 불만에서 벗어나, "Claude Code 자체의 설계에 문제가 있다"는 인식으로 전환된 시기입니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 핵심 발견 |
|------|-------|-------|----------|-------------|
| 2025-07-10 | [#3300](https://github.com/anthropics/claude-code/issues/3300) | ADD LIMITS TO CLAUDE TOKEN CONSUMPTION | 4 | 대문자 제목 = 사용자 좌절감 |
| 2025-07-12 | [#3406](https://github.com/anthropics/claude-code/issues/3406) | Built-in tools + MCP descriptions load on first message causing 10-20k token overhead | 7 | **근본 원인: MCP(외부 도구 연결 기능) 설명 텍스트가 매 메시지마다 10-20K token을 낭비** |
| 2025-07-15 | [#3572](https://github.com/anthropics/claude-code/issues/3572) | Anthropic API Overloaded Error with Repeated 529 Status Codes | 274 | Anthropic 서버 자체가 과부하 |
| 2025-07-17 | [#3804](https://github.com/anthropics/claude-code/issues/3804) | Unexpected Token Consumption with No Code Generation | 3 | 코드를 전혀 작성하지 않았는데도 token이 소모됨 |
| 2025-07-17 | [#3843](https://github.com/anthropics/claude-code/issues/3843) | Excessive Token Consumption and Prolonged Processing in New Sessions | 4 | 새로 시작한 세션(대화)에서도 같은 문제 발생 |
| 2025-07-21 | [#4095](https://github.com/anthropics/claude-code/issues/4095) | Massive Token Consumption — **1.67B tokens in 5 hours** | 5 | **단 5시간 만에 16억 7천만 token 소모 — 정상 사용의 수백 배** |

---

## Phase 4 — 무통보 쿼터 축소 + 대규모 반발 (2025-09 ~ 2025-10)

Anthropic이 사전 공지 없이 서버 측에서 주간 사용량 한도를 줄였습니다. 사용자 신뢰가 처음으로 크게 무너진 시점입니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 반응 |
|------|-------|-------|----------|-----------|
| 2025-10-07 | [#9094](https://github.com/anthropics/claude-code/issues/9094) | **[Meta] Unexpected change in Claude usage limits as of 2025-09-29** (30건 이상 보고) | **121** | 60 |
| 2025-10-12 | [#9424](https://github.com/anthropics/claude-code/issues/9424) | **Weekly Usage Limits Making Claude Subscriptions Unusable** | **109** | **155** |

**핵심 사실:**
- Pro 사용자: 주간 사용 가능 시간이 40-50시간 → 6-8시간으로 감소
- 모든 티어에 영향 (Pro, Max 5x, Max 20x)
- "미끼 상술(bait and switch)" 비난 — "처음에 넉넉하게 쓰게 해놓고 나중에 줄이는 것 아니냐"
- 구독 취소 및 경쟁 서비스로 이전하는 사용자 다수 발생

---

## Phase 5 — OAuth 버그 + Compaction 이슈 (2025-11 ~ 2025-12)

서버 측의 쿼터(사용량 한도) 문제에 더해, Claude Code 프로그램 자체의 버그까지 겹친 시기입니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 핵심 발견 |
|------|-------|-------|----------|-------------|
| 2025-11-01 | [#10784](https://github.com/anthropics/claude-code/issues/10784) | OAuth expiration causes retry storm consuming 100% session usage (3.75M wasted tokens) | 7 | **OAuth(로그인 인증) 만료 시 재시도가 폭주하면서 375만 token이 낭비됨** |
| 2025-11-22 | [#12149](https://github.com/anthropics/claude-code/issues/12149) | Excessive weekly usage limit consumption (71% in 2 prompts) | 3 | 질문 단 2개에 주간 사용량의 71% 소진 |
| 2025-12-01 | [#12786](https://github.com/anthropics/claude-code/issues/12786) | Rate limit incorrectly applied when switching between Max accounts | 3 | 계정을 전환해도 rate limit이 잘못 적용됨 |

---

## Phase 6 — 메가 스레드: Issue #16157 (2026-01)

Claude Code 역사상 가장 많은 사람이 참여한 단일 이슈(문제 보고 게시글)입니다. 댓글이 1,400건을 넘었습니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 반응 |
|------|-------|-------|----------|-----------|
| **2026-01-03** | [**#16157**](https://github.com/anthropics/claude-code/issues/16157) | **[BUG] Instantly hitting usage limits with Max subscription** | **1,422** | **666** (647 thumbs_up) |
| 2026-01-04 | [#16270](https://github.com/anthropics/claude-code/issues/16270) | Usage limits bugged after double limits expired on new years | 40 | 71 |
| 2026-01-08 | [#16856](https://github.com/anthropics/claude-code/issues/16856) | Excessive token usage in Claude Code 2.1.1 — 4x+ faster rate consumption | **63** | 73 |
| 2026-01-09 | [#17016](https://github.com/anthropics/claude-code/issues/17016) | Claude hitting usage limits in no time, something broken for sure | 22 | 9 |
| 2026-01-09 | [#17084](https://github.com/anthropics/claude-code/issues/17084) | Opus 4.5 usage limits significantly reduced since January 2026 | **39** | 54 |
| 2026-01-21 | [#19673](https://github.com/anthropics/claude-code/issues/19673) | You've hit your limit — While usage is still at 84% | **99** | 74 |

**핵심 사실:**
- Max 구독자(최상위 요금제 사용자)가 리셋 직후 곧바로 한도에 도달하는 현상 발생
- 사용량 표시기(미터)에 84%로 표시되는데 "한도에 도달했습니다" 메시지가 뜸 — 내부 정산이 맞지 않는다는 증거
- #16157이 모든 rate limit 관련 불만이 모이는 초대형 토론 게시글이 됨

---

## Phase 7 — Opus 4.6 + 두 번째 폭발 (2026-02)

Opus 4.6 모델이 사전 안내 없이 자동으로 업그레이드되었습니다. 새 모델은 token을 더 많이 소비하는데, compaction(대화 요약 압축) 버그까지 겹쳤습니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 반응 |
|------|-------|-------|----------|-----------|
| 2026-02-01 | [#22435](https://github.com/anthropics/claude-code/issues/22435) | Inconsistent and Undisclosed Quota Accounting Changes. **Legal liability claim ready.** | 11 | 17 |
| 2026-02-06 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | **Opus 4.6 token consumption significantly higher than 4.5** | **41** | 72 |
| 2026-02-08 | [#24179](https://github.com/anthropics/claude-code/issues/24179) | **Compaction death spiral — 211 compactions** consuming all tokens with zero progress | 8 | 2 |
| 2026-02-08 | [#24283](https://github.com/anthropics/claude-code/issues/24283) | Memory files loaded twice in git worktrees causing premature compaction | 3 | — |
| 2026-02-09 | [#24315](https://github.com/anthropics/claude-code/issues/24315) | Repeated context compaction drains weekly token budget — 9 consecutive compactions | 4 | — |
| 2026-02-14 | [#25769](https://github.com/anthropics/claude-code/issues/25769) | Misleading 'Rate limit reached' error during service outage | 2 | — |
| 2026-02-17 | [#26404](https://github.com/anthropics/claude-code/issues/26404) | Weekly token usage limit draining abnormally fast after Sonnet 4.6 update | 6 | 1 |
| 2026-02-23 | [#27869](https://github.com/anthropics/claude-code/issues/27869) | Chrome MCP screenshots accumulate — 17% of Max plan for 5 trivial turns | 6 | — |
| 2026-02-26 | [#28848](https://github.com/anthropics/claude-code/issues/28848) | **Max plan usage limits silently reduced since Claude 4.6 release** | **26** | 31 |
| 2026-02-28 | [#29579](https://github.com/anthropics/claude-code/issues/29579) | **Rate limit reached despite Max subscription and only 16% usage** | **139** | 79 |

**핵심 사실:**
- Opus 4.6이 사용자 동의 없이 자동 업그레이드됨
- 211회 연속 compaction(대화 요약 압축) = 실제 작업 없이 사용량만 소모하는 무한 루프
- 사용량이 16%인데 "사용 한도에 도달했습니다" 메시지가 뜸 = 명백한 정산 오류
- 법적 책임 문제가 처음으로 거론됨

---

## Phase 8 — Cache 리그레션 확인 + 3월 위기 (2026-03)

실제 측정 데이터를 통해 문제의 원인이 확인된 시기입니다. cache(이전 대화를 재활용하는 기능)가 제대로 작동하지 않으면 같은 내용을 매번 처음부터 다시 보내야 해서 비용이 수십 배로 뜁니다.

| 날짜 | 이슈 | 제목 | 댓글 수 | 반응 | 중요도 |
|------|-------|-------|----------|-----------|-------------|
| 2026-03-15 | [**#34629**](https://github.com/anthropics/claude-code/issues/34629) | **Prompt cache regression since v2.1.69: cache_read never grows, ~20x cost increase** | 18 | 36 | **근본 원인 #1 — 데이터로 확인됨** |
| 2026-03-22 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Claude Code Usage for Max Plan hitting limits extremely fast | **59** | 35 | 3월 22일 문제 심화 시점 |
| 2026-03-24 | [**#38335**](https://github.com/anthropics/claude-code/issues/38335) | **Session limits exhausted abnormally fast since March 23** | **313** | **258** | **세 번째 메가 스레드(초대형 토론 게시글)** |
| 2026-03-24 | [#38357](https://github.com/anthropics/claude-code/issues/38357) | Max 20x: Usage meter climbing abnormally fast — 1-2% per simple message | 8 | 3 | 정밀 분석 결과: cache 92.7%로 정상인데도 사용량 표시기는 비정상 = 서버 측 문제 |
| 2026-03-29 | [**#40524**](https://github.com/anthropics/claude-code/issues/40524) | **Conversation history invalidated on subsequent turns** | **59** | **197** | **근본 원인 #2: 대화를 주고받을 때마다 cache를 처음부터 다시 만드는 현상** |
| 2026-03-31 | [#41249](https://github.com/anthropics/claude-code/issues/41249) | Excessive token consumption — usage depleting faster than expected | 9 | 16 | 1시간도 안 되어 사용량 전부 소진 |
| 2026-03-31 | [#41506](https://github.com/anthropics/claude-code/issues/41506) | Token usage increased ~3-5x without configuration change (since ~March 28-29) | 9 | 9 | 플러그인 57개 + MCP 서버 9개 = 요청 한 번에 약 40,000 token이 부가 비용으로 소모 |
| 2026-03-31 | [#41590](https://github.com/anthropics/claude-code/issues/41590) | **New account immediately hits rate limit with zero usage** | 14 | 0 | 신규 계정이 아무것도 안 했는데 즉시 사용 한도에 도달 — 시스템 자체의 결함 |

---

## Phase 9 — 2026년 4월: 진행 중 (현재)

하루 만에 15건 이상의 새로운 rate limit 이슈가 접수되었습니다. 전 세계에서 동시에 보고가 들어왔습니다.

| 날짜 | 이슈 | 제목 | 댓글 수 |
|------|-------|-------|----------|
| 04-01 | [#41788](https://github.com/anthropics/claude-code/issues/41788) | Max 20: 100% in ~70 minutes (my report) | 11 |
| 04-01 | [#42052](https://github.com/anthropics/claude-code/issues/42052) | Max 20x: 100% usage after 2 hours of light work | 13 |
| 04-01 | [#42095](https://github.com/anthropics/claude-code/issues/42095) | 5 hour limit reached with just 4 prompts | — |
| 04-01 | [#42183](https://github.com/anthropics/claude-code/issues/42183) | Excessive token consumption without reading any files | — |
| 04-01 | [#42249](https://github.com/anthropics/claude-code/issues/42249) | Extreme token consumption — quota depleted in minutes | — |
| 04-01 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | Resume loads disproportionate tokens from opaque thinking signatures | — |
| 04-01 | [#42272](https://github.com/anthropics/claude-code/issues/42272) | Excessive consumption since 2.1.88 — 66% usage in 2 questions | — |
| 04-01 | [#42261](https://github.com/anthropics/claude-code/issues/42261), [#42259](https://github.com/anthropics/claude-code/issues/42259), [#42145](https://github.com/anthropics/claude-code/issues/42145), [#42132](https://github.com/anthropics/claude-code/issues/42132), [#42129](https://github.com/anthropics/claude-code/issues/42129), [#42104](https://github.com/anthropics/claude-code/issues/42104) | "You've hit your limit" — 동시 보고 (런던, 베를린, 도쿄, 캘커타, 파리, 이스탄불) | — |
| 04-02 | [#42338](https://github.com/anthropics/claude-code/issues/42338) | Session resume invalidates entire prompt cache | — |
| 04-02 | [#42390](https://github.com/anthropics/claude-code/issues/42390) | Rate limit triggered despite 0% usage in /usage | 3 |
| 04-02 | [#42409](https://github.com/anthropics/claude-code/issues/42409) | Excessive API usage consumption during active session | 4 |
| 04-02 | [**#42542**](https://github.com/anthropics/claude-code/issues/42542) | **Silent context degradation — 3 microcompact mechanisms strip tool results** | 2 | **근본 원인 #3: GrowthBook(A/B 테스트 플랫폼)이 제어하는 compaction이 대화 품질을 저하시킴 (cache는 99%+ 유지되지만 내용이 손실됨)** |
| 04-02 | [**#42569**](https://github.com/anthropics/claude-code/issues/42569) | **1M context incorrectly shown as extra billable usage on Max plan** | 1 | **서버 측 과금 오류 — 무료여야 할 1M context가 추가 요금으로 분류됨** |
| 04-02 | [#42583](https://github.com/anthropics/claude-code/issues/42583) | You've hit your limit — 1M actual vs 120-160K expected (v2.1.90) | 1 |
| 04-02 | [#42590](https://github.com/anthropics/claude-code/issues/42590) | Context compaction too aggressive on 1M context window (Opus 4.6) | 1 |
| 04-02 | [#42592](https://github.com/anthropics/claude-code/issues/42592) | Token consumption 100x faster after v2.1.88 — 21 min to 5-hour limit | 4 |
| 04-02 | [#42609](https://github.com/anthropics/claude-code/issues/42609) | Reached limit session in under 5 minutes (resume-triggered) | 1 |
| 04-02 | [**#42616**](https://github.com/anthropics/claude-code/issues/42616) | **Spurious 429 "Extra usage required" at 23K tokens on Max plan with 1M** | 1 | **서버 측: 디버그 로그가 "서버가 정상 요청을 거부했음"을 증명** |

### Rate Limit 헤더 분석 (4월 6일)

투명 proxy(중간에서 요청을 기록하는 도구, cc-relay)가 3,702건의 요청(4월 4~6일)에서 Anthropic 서버의 `anthropic-ratelimit-unified-*` 응답 헤더(서버가 돌려보내는 사용량 정보)를 캡처했습니다. 이를 통해 서버 측 쿼터 구조가 밝혀졌습니다:

- **듀얼 슬라이딩 윈도우**: 5시간 + 7일의 독립적인 사용량 카운터가 존재합니다. `representative-claim` = `five_hour`가 요청의 100% — 5시간 윈도우가 항상 병목입니다. 즉, 주간 한도보다 5시간 한도에 먼저 걸립니다.
- **1%당 비용**: 5시간 사용률 1% 포인트당 약 9K–16K의 눈에 보이는 출력과, 1.5M–2.1M의 cache_read(이전 대화 재활용)가 소비됩니다. cache_read가 전체 비용의 96–99%를 차지합니다.
- **thinking token 사각지대**: API 응답에 포함된 `output_tokens`는 thinking token(AI가 내부적으로 생각하는 데 쓰는 token)을 제외합니다. 눈에 보이는 출력만으로는 관찰된 사용량의 50% 미만만 설명할 수 있습니다 — 나머지는 thinking token, cache-read 가중치, 또는 둘 다일 가능성이 있습니다.
- **커뮤니티 교차 검증**: @fgrosswig이 [64배 예산 축소 포렌식](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)을 공개했습니다 (2대의 컴퓨터에서 18일간 기록한 JSONL 로그: 3/26에는 32억 token 제한 없이 사용 가능 → 4/5에는 8,800만 token에서 90% 도달). @Commandershadow9이 [34–143배 용량 축소 분석](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)을 공개하여 cache 수정을 확인하면서도, cache 버그와 무관한 용량 감소를 문서화했습니다. 양측 모두 독립적으로 thinking token을 유력한 원인으로 지목했습니다.
- **v2.1.89 분리**: 깨끗한 비교를 위한 분석 프레임워크가 수립되었습니다 — 골든 기간 (3/23–27, cache 98–99%) vs v2.1.89 버그 기간 (3/28–4/1, 데이터가 오염되어 분석에서 제외) vs 수정 후 기간 (4/2+, cache 84–97%, 그런데도 용량 감소가 관찰됨).

전체 분석: [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)

### 새로 발견된 근본 원인 (4월 2-3일)

커뮤니티 조사를 통해 두 가지 중요한 문제가 추가로 발견되었습니다:

**버그 3 — 클라이언트 측 거짓 rate limiter(가짜 사용 한도 차단)** ([#40584](https://github.com/anthropics/claude-code/issues/40584), [@rwp65](https://github.com/rwp65) 발견):
Claude Code 내부의 rate limiter가 Anthropic 서버에 실제로 요청을 보내지도 않고, 스스로 "사용 한도에 도달했습니다"라는 가짜 오류를 만들어 냈습니다 (`model: "<synthetic>"`, `input_tokens: 0`). 대화가 길어지거나 여러 sub-agent(하위 작업 AI)가 동시에 작동할 때 발생했습니다. [@marlvinvu](https://github.com/marlvinvu)가 [#40438](https://github.com/anthropics/claude-code/issues/40438), [#39938](https://github.com/anthropics/claude-code/issues/39938), [#38239](https://github.com/anthropics/claude-code/issues/38239)에서 같은 현상을 확인했습니다. **v2.1.90에서도 수정되지 않았습니다.**

**버그 4 — 무통보 microcompact(초소형 압축)으로 인한 대화 품질 저하** ([#42542](https://github.com/anthropics/claude-code/issues/42542), [@Sn3th](https://github.com/Sn3th) 발견):
세 가지 compaction 메커니즘 (`microCompact.ts:422`, `microCompact.ts:305`, `sessionMemoryCompact.ts:57`)이 모든 AI 호출 시 서버 측 GrowthBook(A/B 테스트 플랫폼) 플래그의 제어 하에 tool 결과(도구 실행 결과)를 사용자 모르게 삭제했습니다. proxy 테스트(4월 3일)에서는 **삭제가 진행되는 중에도 메인 세션의 cache 비율이 99%+ 유지**된다는 것이 확인되었습니다 — 데이터가 안정적으로 교체되어 cache 자체는 깨지지 않았습니다. 실제 피해는 **대화 품질 저하**였습니다 (AI가 이전에 도구로 확인한 결과에 접근할 수 없게 됨). 또한 오래된 Docker 버전(클라이언트 업데이트 없이 사용 중인 환경)이 최근 갑자기 사용량이 급증한 이유도 설명해 줍니다 ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — 클라이언트를 업데이트하지 않아도 서버 측 플래그 변경만으로 동작이 바뀐 것입니다. **v2.1.91까지 수정되지 않았고, 서버에서 제어됩니다.**

**버그 5 — Tool result budget 강제(도구 결과 용량 제한)** ([@Sn3th](https://github.com/Sn3th) 발견, 4월 3일 proxy로 확인):
`applyToolResultBudget()`라는 기능이 microcompact보다 먼저 실행되어, `tengu_hawthorn_window`라는 GrowthBook 플래그를 통해 모든 도구 결과에 합산 200K 상한을 적용했습니다. 개별 도구별 상한 (Grep 20K, Bash 30K)은 `tengu_pewter_kestrel` 플래그로 설정되었습니다. 한 세션에서 261건의 budget 이벤트가 측정되었는데, 도구 결과가 1-41자까지 잘려 나갔습니다 (원래 수백~수천 자일 수 있는 결과가 거의 보이지 않게 된 것입니다). v2.1.91의 `maxResultSizeChars` 오버라이드는 MCP(외부 도구) 전용이었습니다. **기본 내장 도구에는 수정되지 않았습니다.**

**서버 측 1M 과금 리그레션(오류 복귀)** ([#42616](https://github.com/anthropics/claude-code/issues/42616), [#42569](https://github.com/anthropics/claude-code/issues/42569)):
Max 플랜의 1M context(3월 13일 이후 무료로 전환됨)가 "추가 사용료(extra usage)"로 잘못 분류되었습니다. 디버그 로그에서 request ID `req_011CZf8TJf84hAUziB6LuRoc`에 대해 고작 23K token 요청에서 429 응답(사용 한도 초과 거부)이 확인되었습니다. **서버 측 버그로, 수정되지 않았습니다.**

### 공식 수정 배포 (v2.1.89-90, 4월 1일)

Anthropic이 GitHub 이슈에 대한 어떠한 공식 답변도 없이 v2.1.89-90 업데이트에서 cache 관련 수정을 조용히 배포했습니다. 수정 사실은 [changelog(변경 이력)](https://code.claude.com/docs/en/changelog)와 Anthropic 직원의 개인 X(구 Twitter) 게시물을 통해서만 확인할 수 있었습니다.

| 버전 | 수정 내용 | 대상 |
|---------|-----|-----------|
| v2.1.89 | 세션 중 tool schema(도구 정의 데이터) 바이트 변경으로 인한 prompt cache miss | 버그 1 (부분) |
| v2.1.89 | StructuredOutput schema cache 버그 (~50% 실패율) | Cache 안정성 |
| v2.1.89 | Autocompact thrash 루프(무한 압축 반복) 감지 + 중지 | Token 소모 루프 |
| v2.1.89 | LRU cache 키의 메모리 누수 | 장시간 세션 안정성 |
| v2.1.89 | 중첩 CLAUDE.md 재주입 (수십 회 반복 로드) | 컨텍스트 비대화 |
| v2.1.90 | **`--resume`(이전 대화 이어가기) 시 전체 prompt-cache miss (v2.1.69 이후 발생한 리그레션)** | **버그 2 (공식 수정)** |
| v2.1.90 | **매 턴마다 MCP tool schema의 JSON.stringify 제거** | Cache 키 안정성 |
| v2.1.90 | Rate-limit 옵션 다이얼로그 무한 루프 | 세션 크래시 |
| v2.1.90 | SSE 대형 프레임의 이차(n^2) → 선형(n) 처리 | 성능 |

**v2.1.91 (4월 2-3일):**

| 버전 | 수정 내용 | 대상 |
|---------|-----|-----------|
| v2.1.91 | `_meta["anthropic/maxResultSizeChars"]` 500K까지 확대 | 버그 10 — **MCP(외부 도구) 전용 임시 해결책** (기본 내장 도구에는 적용 안 됨) |
| v2.1.91 | `--resume` transcript 체인 단절 수정 | 버그 2 (추가 수정) |
| v2.1.91 | Edit tool에서 더 짧은 `old_string` 앵커 사용 | output token 감소 |

**직원 응답 (4월 2-3일):**
- [Lydia Hallie](https://x.com/lydiahallie/status/2039800715607187906) (스레드): *"피크 시간 한도가 더 타이트해졌고 1M context 세션이 커졌습니다. 대부분 그 영향입니다. 몇 가지 버그를 수정했지만, 과금이 잘못된 것은 아니었습니다."* 커뮤니티 반응은 매우 부정적이었습니다 — 여러 독립 조사자들의 측정 데이터가 아직 수정되지 않은 추가 버그의 존재를 보여줬기 때문입니다 ([분석 레포](https://github.com/ArkNill/claude-code-cache-analysis) 참조).
- [Lydia Hallie](https://x.com/lydiahallie/status/2039107775314428189) (이전): *"Claude Code 측에서 도움이 될 몇 가지 수정을 배포했습니다"*
- [Thariq Shihipar](https://x.com/trq212/status/2027232172810416493): prompt caching 버그를 조사 중이라고 확인 (이전 사건)

**독립 검증:** 제어된 벤치마크(동일 조건 반복 테스트)에서 v2.1.90-91이 전체 82-86% cache read와 안정 세션에서 95-99%를 달성함을 확인했습니다. v2.1.91은 npm/standalone 간 격차를 해소했습니다 (84.7% 동일 cold start). [04_BENCHMARK.md](04_BENCHMARK.md) 참조.

---

## 확인된 근본 원인

이 문제들은 복합적입니다 — 여러 버그가 동시에 작동하면서 서로 영향을 주고, 사용자가 경험하는 증상을 만들어 냈습니다.

### 클라이언트 측 버그

사용자의 컴퓨터에서 실행되는 Claude Code 프로그램 자체의 문제입니다.

| # | 근본 원인 | 이슈 | 버전 | 영향 |
|---|-----------|-------|---------|--------|
| 1 | **Cache sentinel 치환** — standalone 바이너리(독립 실행 파일)의 Bun 포크(내부 엔진)가 cache prefix(캐시 식별 표지)를 깨뜨림 | [#40524](https://github.com/anthropics/claude-code/issues/40524) | 모든 standalone | 대화할 때마다 cache를 처음부터 다시 만들어야 해서 사용량이 급증. **v2.1.89-90에서 부분 수정** ([changelog](https://code.claude.com/docs/en/changelog)) |
| 2 | **Resume cache 리그레션** — `deferred_tools_delta` 불일치가 대화 이어가기(resume) 시 cache를 무효화 | [#34629](https://github.com/anthropics/claude-code/issues/34629) | v2.1.69+ | **20배 비용 증가** (측정됨). **v2.1.90에서 수정** ([changelog](https://code.claude.com/docs/en/changelog)) |
| 3 | **Compaction 무한 루프** — 실제 작업 없이 211회 연속 대화 압축만 반복 | [#24179](https://github.com/anthropics/claude-code/issues/24179) | v2.1.x | 사용량 예산 전부 소비. **v2.1.89에서 무한 반복 감지+중지 기능 추가** |
| 4 | **OAuth retry storm** — 로그인 인증 만료 시 재시도가 폭주 | [#10784](https://github.com/anthropics/claude-code/issues/10784) | — | 375만 token 낭비 |
| 5 | **MCP tool description 오버헤드** — 외부 도구 설명이 모든 메시지에 매번 포함됨 | [#3406](https://github.com/anthropics/claude-code/issues/3406) | 모두 | 메시지 하나 보낼 때마다 10-20K token 추가 소모 |
| 6 | **Memory file 이중 로드** — git worktree(별도 작업 디렉토리) 환경에서 설정 파일이 두 번 로드됨 | [#24283](https://github.com/anthropics/claude-code/issues/24283) | — | 대화 압축이 불필요하게 일찍 시작됨 |
| 7 | **Thinking signature 재전송** — AI의 내부 사고 데이터(불투명 base64 블록)가 대화 이어가기 시 다시 전송됨 | [#42260](https://github.com/anthropics/claude-code/issues/42260) | — | 대화를 이어갈 때마다 500K+ token 추가 소모 |
| 8 | **클라이언트 측 거짓 rate limiter** — 서버에 요청을 보내지도 않고 프로그램이 스스로 "사용 한도 초과" 오류를 생성 | [#40584](https://github.com/anthropics/claude-code/issues/40584) | 모두 | `model: "<synthetic>"`, `input_tokens: 0`으로 즉시 "Rate limit reached" 표시. **수정되지 않음** |
| 9 | **무통보 microcompact로 인한 대화 품질 저하** — 서버의 GrowthBook(A/B 테스트) 플래그가 제어하는 압축이 도구 결과를 삭제 | [#42542](https://github.com/anthropics/claude-code/issues/42542) | v2.1.89+ | 대화 품질이 저하됨 (cache 99%+ 유지되지만, AI가 이전 작업 결과를 볼 수 없게 됨). **수정되지 않음, 서버에서 제어** |
| 10 | **Tool result budget 강제** — `applyToolResultBudget()` 기능이 도구 결과를 합산 200K로 제한 | GrowthBook 플래그 | 모두 | 한도 초과 시 도구 결과가 1-41자로 잘려 나감 (원본의 극히 일부만 남음). **수정되지 않음** (v2.1.91에서 MCP 외부 도구만 우회 가능) |
| 11 | **JSONL 로그 중복** — extended thinking(확장 사고 기능)이 API 호출 한 번에 로그를 2-5배로 중복 기록 | [#41346](https://github.com/anthropics/claude-code/issues/41346) | 모두 | 로컬 로그에서 token이 2.87배로 부풀려 보임. 서버 측 실제 영향은 불명. **수정되지 않음** |

### 서버 측 이슈

Anthropic의 서버에서 발생하는 문제로, 사용자가 직접 해결할 수 없습니다.

| # | 근본 원인 | 근거 | 영향 |
|---|-----------|----------|--------|
| 1 | **무통보 쿼터 축소** — 주간/5시간 사용량 한도를 사전 공지 없이 줄임 | [#9094](https://github.com/anthropics/claude-code/issues/9094), [#28848](https://github.com/anthropics/claude-code/issues/28848) | 사용 가능 시간이 5-8배 감소 |
| 2 | **정산 불일치** — 사용량 표시기에 16-84%로 나오는데 "한도에 도달했습니다" 메시지 표시 | [#19673](https://github.com/anthropics/claude-code/issues/19673), [#29579](https://github.com/anthropics/claude-code/issues/29579) | 언제 차단될지 예측 불가능 |
| 3 | **조직 수준 쿼터 공유** — 같은 조직에 속한 계정들이 사용량 풀을 함께 공유 (`organizationUuid` 키 기준) | 소스 코드 분석 | 다른 팀원이 사용량을 소비하면 내 한도도 줄어드는 현상 |
| 4 | **Opus 4.6 자동 업그레이드** — 더 많은 token을 소비하는 새 모델로 강제 전환, 거부 불가 | [#23706](https://github.com/anthropics/claude-code/issues/23706) | 같은 작업을 해도 기본 비용이 증가 |
| 5 | **서버 측 정산 방식 변경** — 클라이언트를 업데이트하지 않은 오래된 Docker 환경에서도 사용량이 갑자기 빠르게 소모 | [#37394](https://github.com/anthropics/claude-code/issues/37394) | Claude Code 버그와 무관한 서버 측 원인이 존재한다는 증거 |
| 6 | **1M context 과금 리그레션** — Max 플랜에서 무료여야 하는 1M 요청이 "추가 사용료"로 잘못 분류됨 | [#42616](https://github.com/anthropics/claude-code/issues/42616), [#42569](https://github.com/anthropics/claude-code/issues/42569) | 고작 23K token에서 허위 429(사용 거부) 응답, 잘못된 과금 |

---

## 규모 지표

커뮤니티 참여도가 가장 높았던 이슈들입니다. 숫자가 클수록 많은 사용자가 같은 문제를 겪고 있다는 뜻입니다.

| 이슈 | 댓글 수 | Thumbs Up | 날짜 | 제목 |
|-------|----------|-----------|------|-------|
| [#16157](https://github.com/anthropics/claude-code/issues/16157) | **1,422** | **647** | 2026-01-03 | Instantly hitting usage limits |
| [#38335](https://github.com/anthropics/claude-code/issues/38335) | **313** | **258** | 2026-03-24 | Session limits exhausted abnormally fast |
| [#3572](https://github.com/anthropics/claude-code/issues/3572) | **274** | — | 2025-07-15 | 529 Status Codes (server overload) |
| [#29579](https://github.com/anthropics/claude-code/issues/29579) | **139** | 79 | 2026-02-28 | Rate limit at 16% usage |
| [#9094](https://github.com/anthropics/claude-code/issues/9094) | **121** | 60 | 2025-10-07 | Silent quota change |
| [#9424](https://github.com/anthropics/claude-code/issues/9424) | **109** | **155** | 2025-10-12 | Subscriptions unusable |
| [#19673](https://github.com/anthropics/claude-code/issues/19673) | **99** | 74 | 2026-01-21 | Limit at 84% usage |
| [#16856](https://github.com/anthropics/claude-code/issues/16856) | **63** | 73 | 2026-01-08 | 4x+ faster consumption |

---

## 에스컬레이션 패턴

아래 차트는 시간 순서에 따라 문제의 심각도가 어떻게 변해왔는지 보여줍니다. 막대가 길수록 더 심각한 시기입니다.

```
2025-02  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  산발적 (Phase 1)
2025-05  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Claude 4 출시 (Phase 2)
2025-07  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  구조적 결함 (Phase 3)
2025-09  █████████████░░░░░░░░░░░░░░░░░░░░░░░░  무통보 쿼터 축소 (Phase 4)
2025-11  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  OAuth/compaction (Phase 5)
2026-01  ██████████████████████████████░░░░░░░░  메가 스레드 #16157 (Phase 6)
2026-02  ████████████████████░░░░░░░░░░░░░░░░░  Opus 4.6 + 법적 책임 (Phase 7)
2026-03  ████████████████████████████░░░░░░░░░  Cache 리그레션 확인 (Phase 8)
2026-04  ████████████████████████████████████░░  전 세계 동시 발생 (Phase 9, 진행 중)
```

새 모델 출시 또는 버전 업데이트가 있을 때마다 다음 에스컬레이션 사이클(문제 격화 주기)의 트리거가 되었습니다.

---

*2026-04-02 수집, 2026-04-06 업데이트 (rate limit 헤더 분석, 커뮤니티 교차 참조, v2.1.89 분리 프레임워크). 근본 원인 분석은 [README.md](README.md), 쿼터 아키텍처 분석은 [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) 참조.*
