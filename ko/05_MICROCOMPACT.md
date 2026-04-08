> 🇰🇷 이 문서는 [영어 원본](../05_MICROCOMPACT.md)의 한국어 번역입니다.

# 대화 내용이 몰래 지워지는 문제 — Bug 4 & 5

Claude Code에는 사용자 모르게 대화 내용을 지우는 두 가지 숨겨진 기능이 있습니다. 둘 다 사용자가 끌 수 있는 설정이 없고, v2.1.91에서도 여전히 작동하고 있습니다.

- **Bug 4 (Microcompact, 이전 도구 결과를 자동으로 지우는 내부 기능):** 이전에 Claude가 파일을 읽거나 명령을 실행한 결과가 `[Old tool result content cleared]`라는 문구로 몰래 바뀝니다. 총 327건 발생이 측정됐습니다. 캐시(재사용 가능한 임시 저장소) 비용에는 영향이 없지만, Claude가 참고할 수 있는 정보의 품질이 크게 떨어집니다.
- **Bug 5 (Budget enforcement, 용량 제한):** tool result(Claude가 파일을 읽거나 명령을 실행한 결과)의 총 글자 수가 20만 자를 넘으면 결과가 잘립니다. 총 261건 발생이 측정됐습니다. 원래 수천 자짜리 결과가 1~41자로 줄어듭니다.

> **최신 데이터 (4월 8일):** 1주일간의 프록시 데이터에서 B4: 3,782건 (15,998개 항목 삭제), B5: 72,839건 (100% 잘림 비율)이 확인되었습니다. 전체 데이터셋은 [13_PROXY-DATA.md](../13_PROXY-DATA.md)를 참조하십시오.

두 기능 모두 GrowthBook(서버에서 원격으로 기능을 켜고 끄는 A/B 테스트 시스템)이라는 시스템으로 Anthropic 서버에서 제어됩니다. Anthropic은 사용자가 Claude Code를 업데이트하지 않아도 동작을 바꿀 수 있습니다. `/export` 명령으로 대화를 내보내면 원본 그대로가 보이지만, 실제로 AI에게 전달되는 내용은 잘린 버전입니다. 즉, 내보내기 결과만 보면 문제를 알 수 없습니다.

> **이슈:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)
> **발견:** [@Sn3th](https://github.com/Sn3th), 2026년 4월 2일
> **상태:** v2.1.91에서 미수정 | **테스트:** npm + standalone, 2026년 4월 3일

---

## Bug 4: Microcompact — 실제로 무슨 일이 벌어지는가?

대화 초반에 Claude가 읽었던 파일이나 실행했던 명령의 결과가 `[Old tool result content cleared]`라는 문구로 몰래 바뀝니다. 압축이 일어났다는 알림도 없고, 어떤 경고도 뜨지 않습니다. Claude는 원래 내용에 접근할 수 없는 채로 계속 작업합니다. 그래서 이전에 읽었던 파일 내용을 인용하거나, 검색 결과를 참조하거나, 이전 명령 결과를 확인하는 것이 불가능해집니다.

**파일 15~20개를 읽은 후부터 이전에 읽었던 파일 내용이 몰래 지워집니다.** 도구를 많이 쓰는 세션(파일 읽기, 검색, bash 명령 50회 이상)에서 [@Sn3th](https://github.com/Sn3th)는 1M context window(AI가 한 번에 참고할 수 있는 대화 범위)를 사용하는데도 실제로 참고 가능한 범위가 약 40K~80K로 떨어진다고 보고했습니다. 프록시(중간 기록 서버) 데이터도 이를 뒷받침합니다. 메시지 251개짜리 세션에서 11개의 서로 다른 위치가 지워졌고, 메시지 135개짜리 세션에서는 12개 위치가 지워졌습니다. 지우기는 처음 60개 메시지 안에서 시작되며, 대화가 길어질수록 점점 더 많이 지워집니다.

## 세 가지 압축 메커니즘

이 기능들은 [GrowthBook](https://www.growthbook.io/)이라는 기능 제어 시스템으로 관리됩니다. Claude Code는 Anthropic 서버에서 설정값을 받아와 `~/.claude.json` 파일에 저장합니다. 사용자는 이 설정을 끌 수 없습니다. Anthropic이 서버에서 직접 제어하며, 사용자가 Claude Code를 업데이트하지 않아도 동작을 바꿀 수 있습니다.

소스: Claude Code 바이너리의 `src/services/compact/`.

| 메커니즘 | 소스 | 트리거 | 제어 |
|----------|------|--------|------|
| **시간 기반 microcompact** | `microCompact.ts:422` | 마지막 assistant 메시지 이후 간격이 임계값 초과 | GrowthBook: `getTimeBasedMCConfig()` |
| **캐시된 microcompact** | `microCompact.ts:305` | 카운트 기반 트리거, `cache_edits` API를 사용해 서버 cache에서 오래된 tool 결과 삭제 | GrowthBook: `getCachedMCConfig()` |
| **Session memory compact** | `sessionMemoryCompact.ts:57` | autocompact 전에 실행, 최근 메시지의 ~40K token만 유지 | GrowthBook: `tengu_session_memory` / `tengu_sm_compact` |

세 가지 모두 사용자가 설정할 수 있는 `DISABLE_AUTO_COMPACT`와 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 환경 변수를 무시합니다. SM compact에 영향을 주는 유일한 환경 변수는 `DISABLE_CLAUDE_CODE_SM_COMPACT=true`인데, 이것마저도 microcompact는 막지 못합니다.

## GrowthBook 설정값 조사

Claude Code는 GrowthBook 설정을 `~/.claude.json` 파일의 `cachedGrowthBookFeatures` 항목에 저장합니다. 아래 스크립트로 확인할 수 있습니다:

```bash
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
for k in ['tengu_slate_heron', 'tengu_session_memory', 'tengu_sm_compact',
          'tengu_sm_compact_config', 'tengu_cache_plum_violet']:
    print(f'{k}: {json.dumps(gb.get(k, \"NOT PRESENT\"), indent=2)}')
"
```

### 수집 데이터 (2026년 4월 2-3일)

4대의 컴퓨터, 4개의 계정에서 설정값을 비교했습니다. 결과는 모두 동일했습니다:

| Flag | 머신 1 (Sn3th, dev) | 머신 2 (Sn3th, prod) | 머신 3 (ArkNill, ZBook) | 머신 4 (arizonawayfarer, Win) |
|------|------------------------|------------------------|---------------------------|-------------------------------|
| `tengu_slate_heron` | `{"enabled": false, ...}` | 동일 | 동일 | 동일 |
| `tengu_session_memory` | `false` | 동일 | 동일 | 동일 |
| `tengu_sm_compact` | `false` | 동일 | 동일 | 동일 |
| `tengu_sm_compact_config` | `{"minTokens": 2000, "maxTokens": 20000, ...}` | 동일 | 동일 | 동일 |
| `tengu_cache_plum_violet` | `true` | 동일 | 동일 | 동일 |

### Flag 용어집

각 설정값이 무엇을 하는지 정리한 표입니다:

| Flag | 용도 | 현재 값 |
|------|------|---------|
| `tengu_slate_heron` | 시간 기반 microcompact 제어 (간격 임계값, 최근 유지 수) | `enabled: false` |
| `tengu_session_memory` | session memory compact 활성화 여부 | `false` |
| `tengu_sm_compact` | SM compact 활성화 여부 (session memory와 별개) | `false` |
| `tengu_sm_compact_config` | SM compact 임계값 (최소/최대 token, 메시지 수) | `{minTokens: 2000, maxTokens: 20000}` |
| `tengu_cache_plum_violet` | 용도 불명 — 조사한 모든 머신에서 유일하게 켜져 있는 flag | `true` |
| `tengu_hawthorn_window` | **총 도구 결과 용량 제한** — 모든 tool result에 허용되는 총 글자 수 (Bug 5) | `200000` |
| `tengu_pewter_kestrel` | **개별 도구별 크기 제한** — 도구 종류별 최대 글자 수 (Bug 5) | `{global: 50000, Bash: 30000, Grep: 20000}` |
| `tengu_summarize_tool_results` | 지워진 tool result가 있을 수 있다고 AI에게 미리 알려주는 시스템 프롬프트 설정 | `true` |

주목할 점은, 모든 microcompact 관련 설정이 "꺼짐"으로 표시되어 있는데도 대화 내용 지우기가 계속된다는 것입니다. 이는 budget enforcement flag(`hawthorn_window`, `pewter_kestrel`)가 실제로 작동 중인 메커니즘이거나, 이 설정값을 확인하지 않는 숨겨진 코드 경로가 있다는 뜻입니다.

**4대의 머신, 4개의 계정 (3 Linux, 1 Windows), 모두 Max 플랜, v2.1.90 — 모든 설정이 꺼짐으로 표시됩니다. 그런데도 대화 내용은 여전히 지워지고 있습니다.**

## SM Compact 임계값 문제

설정이 꺼져 있어도, 서버에서 받아온 설정값에는 코드에 내장된 기본값보다 훨씬 공격적인(더 빨리 지우는) 임계값이 포함되어 있습니다:

| 파라미터 | 원격 설정 | 코드 기본값 | 차이 |
|----------|-----------|------------|------|
| `minTokens` | **2,000** | 10,000 | **5배 낮은 하한** |
| `maxTokens` | **20,000** | 40,000 | **2배 낮은 상한** |
| `minTextBlockMessages` | 5 | — | — |

이 설정이 A/B 테스트를 통해 갑자기 활성화되면, 세션에서 유지되는 내용이 최대 20K 토큰뿐이 됩니다. 1M context에 비용을 지불하는 사용자가 기대하는 것에 비하면 극히 일부분입니다. 즉, **100만 토큰짜리 대화 범위를 구매했는데 실제로는 2만 토큰만 쓸 수 있게 되는 것**과 같습니다.

## 캐시 비용에 대한 영향 — 수정된 이해 (4월 3일)

**처음 가설 (4월 2일):** microcompact가 캐시(이전 대화를 재활용하는 기능)를 망가뜨려서 매번 처음부터 다시 계산하게 되고, 그로 인해 비용이 전액 청구될 것으로 예상했습니다.

**측정 결과:** 이 가설은 **부분적으로 틀렸습니다.** 327건의 이벤트(모든 세션 합산)에 대한 프록시 데이터를 보면:

| 컨텍스트 | 삭제 중 cache 비율 | 설명 |
|----------|-------------------|------|
| 메인 세션 | **99%+** — 영향 없음 | 삭제 시 동일한 마커를 일관되게 대체하므로, 호출 간 prefix가 변경되지 않음 |
| Sub-agent 콜드 스타트 | **0-39%** — 유의미한 하락 | Sub-agent가 이미 삭제된 context에서 새 cache를 구축하며, 콜드 스타트 페널티 발생 |
| Sub-agent 워밍 완료 (5회+ 요청) | **94-99%** — 정상 | 워밍 후 삭제된 콘텐츠와 무관하게 안정화 |

**진짜 문제는 캐시 비용이 아니라, Claude가 참고할 수 있는 정보의 품질입니다.** tool result가 지워지면:
- Claude가 이전에 읽었던 파일 내용, 검색 결과, 명령 출력을 정확하게 참조할 수 없습니다
- 이미 시도했던 것을 볼 수 없어서 같은 시도를 반복하는 문제가 생깁니다
- 장시간 세션이 1M 대화 범위를 쓰고 있는데도 실제로는 약 40K~80K만 활용 가능해집니다 ([@Sn3th](https://github.com/Sn3th) 보고)

**관련 관찰:**
- 오래된 Docker 고정 버전 (v2.1.74/86, 업데이트 안 함)에서도 최근 지우기가 시작됐습니다 ([#37394](https://github.com/anthropics/claude-code/issues/37394)). 이는 GrowthBook 설정이 사용자의 업데이트 없이 서버에서 변경된다는 증거입니다
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` 설정은 지우기를 막지 못합니다 (microcompact가 이 설정을 완전히 무시합니다)
- 지우기는 사용자마다 간헐적으로 나타납니다. 이는 GrowthBook A/B 테스트가 누가 영향받을지와 얼마나 강하게 적용할지를 제어하는 것과 일치합니다

## 로컬 재현 (진행 중)

### 설정

프록시(중간 기록 서버) 기반 요청 본문 스캐너 (cc-relay + `ANTHROPIC_BASE_URL`)를 사용해서, API 요청이 컴퓨터를 떠나기 전에 `[Old tool result content cleared]`가 포함되어 있는지 탐지했습니다. 또한 GrowthBook 파일 감시기가 10초마다 `~/.claude.json`의 변경을 기록했습니다.

### 확인된 결과 (2026년 4월 3일)

향상된 cc-relay 프록시를 사용해 여러 세션에 걸쳐 체계적으로 테스트했습니다 (3,500건 이상의 요청 기록, 총 327건의 microcompact 이벤트):

**지우기 패턴:**
- 총 **327건의 이벤트** 탐지 — 두 주요 v2.1.90 세션에서 67건과 71건, 나머지는 v2.1.91 테스트 세션과 sub-agent에서 발생
- 지워지는 위치가 **시간에 따라 확대됩니다**: [20,22]에서 시작해서 [20,22,38,40,44,46,162,166,172,174,206]까지 늘어났습니다
- **지워진 모든 위치 번호가 짝수입니다** → 도구 요청/결과 쌍을 정확히 겨냥한다는 뜻입니다
- 같은 위치가 이후 모든 API 호출에서 일관되게 지워집니다 (안정적 대체)

**캐시 영향 — 핵심 발견:**
- **메인 세션: 캐시 영향 없음.** 활발하게 지우는 중에도 비율이 99%+ 유지됩니다. 같은 위치를 같은 마커 문구로 일관되게 바꾸기 때문에, 캐시 기준점이 변하지 않습니다.
- **Sub-agent 콜드 스타트: 0~39% 캐시 하락** 관찰. sub-agent(하위 작업을 처리하는 별도 AI)가 이미 지워진 대화 내용을 바탕으로 새 캐시를 만들기 때문입니다.
- **실질적 피해는 캐시 비용이 아니라, Claude가 참고할 수 있는 정보의 품질입니다.** 이전 도구 결과에 접근할 수 없어서 정확하게 인용하거나 참조할 수 없게 됩니다.

**Budget Enforcement (Bug 5) — 새롭게 발견된 문제:**

microcompact 외에, **별도의 사전 처리 파이프라인**이 서버에서 제어하는 GrowthBook 임계값에 따라 tool result를 잘라냅니다:

```
tengu_hawthorn_window:       200,000 (aggregate tool result cap)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: true
```

이것은 `applyToolResultBudget()`을 통해 microcompact **이전에** 실행됩니다. 실제 작동을 확인한 내용:
- 단일 세션에서 **261건의 budget 이벤트** 탐지
- tool result가 **1~41자**로 줄어들었습니다 (원래는 수천 자 이상)
- **242,094자** 시점에서 budget cap(도구 결과의 총 용량 제한) 초과 (> 200K 상한)
- v2.1.91의 `maxResultSizeChars` 오버라이드는 **MCP 전용**입니다 — 기본 내장 도구인 Read/Bash/Grep에는 적용되지 않습니다

**v2.1.91 검증:**
- v2.1.91 테스트 세션에서 14건의 microcompact 이벤트, 71건의 budget 이벤트
- **v2.1.90과 변경 없음** — 두 버그 모두 동일하게 지속됩니다

### 남은 검증 사항

1. **바이너리 코드 경로 식별** — 디컴파일된 바이너리에서 `Old tool result content cleared` 검색
2. **GrowthBook 네트워크 가로채기** — 파일에 저장된 설정은 변화가 없는데 실제 동작은 다릅니다. 메모리상의 불일치를 의심하고 있습니다
3. **Budget 잘림 임계값** — 잘림이 시작되는 정확한 토큰 수 확인 (글자 수가 아닌 토큰 수 기준)

## 미해결 질문

### 1. GrowthBook 저장 설정과 실제 동작이 다른 문제

`~/.claude.json` 파일에 저장된 설정은 GrowthBook이 마지막으로 평가한 결과의 스냅샷입니다. 하지만 GrowthBook은 실행 중에 세션별 속성(사용자 ID, 요금제, 세션 ID, 시각 등)을 사용해서 기능을 평가합니다. Claude Code가 세션 도중에 다른 조건으로 다시 평가하면 — 예를 들어 토큰 사용량이 특정 임계값을 넘었을 때, 또는 세션 시간이 길어졌을 때 — 파일에 저장된 설정에는 절대 반영되지 않을 수 있습니다.

로컬 테스트에서 활발한 지우기가 일어나는 동안 파일 설정의 변화는 관찰되지 않았습니다. 하지만 이것이 메모리상의 불일치를 완전히 배제하지는 못합니다.

**테스트 방법:** 지우기가 활발히 발생하는 세션에서 GrowthBook SDK 호출(또는 GrowthBook API 엔드포인트)을 가로채서, 실시간 설정값과 파일에 저장된 설정을 비교해야 합니다.

### 2. 서버에서 지우는 건지, 클라이언트에서 지우는 건지

`[Old tool result content cleared]`가 나타날 때, 이것이 Anthropic 서버의 API 응답에 원래 포함된 건지, 아니면 Claude Code가 로컬에서 넣는 건지 확인이 필요합니다.

현재까지의 증거는 **클라이언트 측**(Claude Code가 보내는 요청 본문에서 마커를 발견)을 가리키지만, 더 긴 세션과 확실한 최초 지우기 캡처로 검증이 필요합니다.

### 3. 네 번째 경로

문서화된 세 가지 GrowthBook 설정이 모두 꺼져 있는데도 지우기가 발생합니다. 로컬에서 확인했습니다. 가능한 원인:
- GrowthBook 설정을 아예 확인하지 않는 코드 경로가 바이너리 안에 있을 수 있습니다
- 특정 빌드에서만 활성화되는 컴파일 타임 설정이 있을 수 있습니다
- 서버 응답의 헤더나 메타데이터 필드가 클라이언트 측 지우기를 유발할 수 있습니다
- `cache_edits` API 경로 (캐시된 microcompact)가 GrowthBook 설정과 무관하게 독립적으로 작동할 수 있습니다

## 효과 없는 환경 변수들

"이 설정을 바꾸면 해결되지 않을까?" 생각할 수 있는 환경 변수들을 정리했습니다. 결론부터 말하면, 현재 microcompact를 막을 수 있는 환경 변수는 없습니다:

| 환경 변수 | 제어 대상 | Microcompact를 차단하는가? |
|----------|----------|--------------------------|
| `DISABLE_AUTO_COMPACT=true` | Autocompact만 | **아니오** |
| `DISABLE_COMPACT=true` | 수동 `/compact` 포함 모든 compaction | 아마도, 하지만 수동 compact도 같이 무력화됩니다 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Autocompact 임계값 | **아니오** |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=true` | Session memory compact만 | **아니오** (시간 기반 또는 캐시된 MC는 커버 안 됩니다) |

**필요한 것:** `DISABLE_MICROCOMPACT=true` — microcompact만 독립적으로 끌 수 있는 전용 환경 변수가 필요합니다.

## 데이터 기여

대화 내용이 몰래 지워지는 현상을 겪고 있다면, 아래 정보를 공유해 주시기 바랍니다:

1. **GrowthBook flag** — 위의 추출 스크립트를 실행하고 값을 공유해 주시기 바랍니다. 만약 `tengu_sm_compact: true` 또는 `tengu_slate_heron.enabled: true`를 보여주는 사례가 있다면, A/B 테스트 가설이 확인됩니다.
2. **프록시 캡처** — `ANTHROPIC_BASE_URL`을 기록용 프록시로 설정하고, `[Old tool result content cleared]`가 나타날 때의 원본 API 응답을 캡처해 주시기 바랍니다. 이것으로 클라이언트에서 지우는 건지, 서버에서 지우는 건지 판별할 수 있습니다.
3. **세션 상세** — 사용 버전, 요금제, 운영체제, 지우기가 관찰됐을 때의 대략적인 세션 길이와 도구 사용량을 알려주시기 바랍니다.

## 참조

- [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542) — 소스 코드 분석이 포함된 원본 이슈
- [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524) — Bug 1: Sentinel cache 손상
- [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629) — Bug 2: Resume cache 퇴행
- [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584) — Bug 3: 클라이언트 측 false rate limiter
- [anthropics/claude-code#37394](https://github.com/anthropics/claude-code/issues/37394) — Docker 고정 구버전 드레인 (서버 측 증거)
- [anthropics/claude-code#42590](https://github.com/anthropics/claude-code/issues/42590) — 1M에서 context compaction이 지나치게 공격적
