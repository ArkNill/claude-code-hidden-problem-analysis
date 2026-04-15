> 이 문서는 [영어 원본](../05_MICROCOMPACT.md)의 한국어 번역입니다.

# 대화 내용이 몰래 변조되는 문제 — Bug 4 & 5

두 가지 별도의 메커니즘이 대화 내용을 API에 보내기 전에 몰래 수정합니다. 둘 다 환경 변수로 제어할 수 없으며, v2.1.91에서도 여전히 활성 상태입니다.

- **Bug 4 (Microcompact):** 이전 tool result가 `[Old tool result content cleared]`로 대체됩니다. 327건의 이벤트가 측정되었습니다 (4월 3일 집중 테스트). 캐시에는 영향이 없지만 (99%+ 유지) context 품질이 파괴됩니다.
- **Bug 5 (Budget enforcement):** 총 20만 자를 초과하면 tool result가 잘립니다. 261건의 이벤트가 측정되었습니다 (4월 3일 집중 테스트). 결과가 1~41자로 축소됩니다 (4월 3일 세션; 전체 주간 최대: 49자).

> **최신 데이터 (4월 14일):** 프록시 데이터 (30,477건, 4월 1~14일)에서 B4: **5,500건** (18,858개 항목 삭제), B5: **167,818건** (100% 잘림 비율, 100% 50자 이하) — 모두 **미수정 기준선 기간** (4월 1~10일)의 데이터입니다. 4월 10일에 배포된 프록시 기반 GrowthBook flag 오버라이드가 두 문제를 완전히 제거했습니다 (167,818 → 0, 5,500 → 0, 이후 4,919건). 통제된 테스트 방법론은 [01_BUGS.md](01_BUGS.md#growthbook-flag-override--controlled-elimination-test-april-1014)를, 전체 데이터셋은 [13_PROXY-DATA.md](13_PROXY-DATA.md)를 참조하십시오.

두 기능 모두 GrowthBook feature flag를 통해 서버에서 제어됩니다 — Anthropic은 클라이언트 업데이트 없이 동작을 변경할 수 있습니다. `/export`는 변조된 버전을 보여주지 않습니다 — 전체 context를 보여주지만, API에 전달되는 것은 잘린 버전입니다.

> **이슈:** [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542)
> **발견:** [@Sn3th](https://github.com/Sn3th), 2026년 4월 2일
> **상태:** v2.1.91에서 미수정 | **테스트:** npm + standalone, 2026년 4월 3일

---

## Bug 4: Microcompact — 무슨 일이 벌어지는가

세션 초반의 tool result가 `[Old tool result content cleared]`로 몰래 대체됩니다. 압축 알림도 표시되지 않고, 어떤 hook도 실행되지 않습니다. 모델은 원래 tool output에 접근할 수 없는 채로 계속 작동합니다 — 파일 내용을 인용하거나, 특정 grep 결과를 참조하거나, 이전 명령 출력을 확인하는 것이 불가능해집니다.

도구를 많이 사용하는 세션(50회 이상의 파일 읽기, grep, bash 명령)에서 [@Sn3th](https://github.com/Sn3th)는 1M 윈도우에도 불구하고 실질 context가 약 40~80K로 떨어진다고 보고했습니다. 프록시 데이터도 이를 뒷받침합니다: 251개 메시지 세션에서 11개의 서로 다른 메시지 인덱스가 삭제되었고, 135개 메시지 세션에서는 12개 인덱스가 삭제되었습니다. 삭제는 처음 60개 메시지 안에서 시작되며 대화가 커질수록 확대됩니다.

## 세 가지 압축 메커니즘

이 메커니즘들은 feature-flagging 시스템인 [GrowthBook](https://www.growthbook.io/)으로 제어됩니다. Claude Code CLI는 Anthropic 서버에서 flag를 가져와 `~/.claude.json`에 캐시합니다. 사용자는 이 flag를 비활성화할 수 없습니다 — Anthropic이 서버에서 제어하며 클라이언트 업데이트 없이 동작을 변경할 수 있습니다.

소스: Claude Code 바이너리의 `src/services/compact/` ([#42542](https://github.com/anthropics/claude-code/issues/42542)에 문서화된 바와 같이, 공개된 npm 번들에서 확인).

| 메커니즘 | 소스 | 트리거 | 제어 |
|-----------|--------|---------|---------|
| **시간 기반 microcompact** | `microCompact.ts:422` | 마지막 assistant 메시지 이후 시간 간격이 임계값 초과 | GrowthBook: `getTimeBasedMCConfig()` |
| **캐시된 microcompact** | `microCompact.ts:305` | 카운트 기반 트리거, `cache_edits` API를 사용해 서버 캐시에서 오래된 tool result 삭제 | GrowthBook: `getCachedMCConfig()` |
| **Session memory compact** | `sessionMemoryCompact.ts:57` | autocompact 전에 실행, 최근 메시지의 ~40K 토큰만 유지 | GrowthBook: `tengu_session_memory` / `tengu_sm_compact` |

세 가지 모두 `DISABLE_AUTO_COMPACT` 및 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`를 무시합니다. SM compact에 영향을 주는 유일한 환경 변수는 `DISABLE_CLAUDE_CODE_SM_COMPACT=true`이며, 이것도 microcompact는 다루지 않습니다.

## GrowthBook Feature Flag 조사

CLI는 GrowthBook 페이로드를 `~/.claude.json`의 `cachedGrowthBookFeatures` 아래에 캐시합니다. 다음 스크립트로 추출할 수 있습니다:

```bash
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
for k in ['tengu_slate_heron', 'tengu_session_memory', 'tengu_sm_compact',
          'tengu_sm_compact_config', 'tengu_cache_plum_violet']:
    print(f'{k}: {json.dumps(gb.get(k, \"NOT PRESENT\"), indent=2)}')
"
```

### 수집 데이터 (2026년 4월 2~3일)

| Flag | 머신 1 (Sn3th, dev) | 머신 2 (Sn3th, prod) | 머신 3 (ArkNill, ubuntu-1) | 머신 4 (arizonawayfarer, Win) |
|------|------------------------|------------------------|---------------------------|-------------------------------|
| `tengu_slate_heron` | `{"enabled": false, ...}` | 동일 | 동일 | 동일 |
| `tengu_session_memory` | `false` | 동일 | 동일 | 동일 |
| `tengu_sm_compact` | `false` | 동일 | 동일 | 동일 |
| `tengu_sm_compact_config` | `{"minTokens": 2000, "maxTokens": 20000, ...}` | 동일 | 동일 | 동일 |
| `tengu_cache_plum_violet` | `true` | 동일 | 동일 | 동일 |

### Flag 용어집

| Flag | 용도 | 현재 값 |
|------|---------|---------------|
| `tengu_slate_heron` | 시간 기반 microcompact 제어 (간격 임계값, 최근 유지 수) | `enabled: false` |
| `tengu_session_memory` | Session memory compact 게이트 | `false` |
| `tengu_sm_compact` | SM compact 게이트 (session memory와 별개) | `false` |
| `tengu_sm_compact_config` | SM compact 임계값 (최소/최대 토큰, 메시지 수) | `{minTokens: 2000, maxTokens: 20000}` |
| `tengu_cache_plum_violet` | 용도 불명 — 조사한 모든 머신에서 유일하게 활성화된 flag | `true` |
| `tengu_hawthorn_window` | **총 tool result 용량 제한** — 모든 tool result에 허용되는 총 글자 수 (Bug 5) | `200000` |
| `tengu_pewter_kestrel` | **개별 tool result 크기 제한** — 도구별 출력 제한 (Bug 5) | `{global: 50000, Bash: 30000, Grep: 20000}` |
| `tengu_summarize_tool_results` | 삭제된 tool result가 있을 수 있다고 모델에 알리는 시스템 프롬프트 flag | `true` |

모든 microcompact 게이트가 비활성화로 표시됨에도 불구하고 context 제거가 지속됩니다 — 이는 budget enforcement flag(`hawthorn_window`, `pewter_kestrel`)가 활성 메커니즘이거나, 이 게이트를 확인하지 않는 문서화되지 않은 코드 경로가 있다는 것을 가리킵니다.

**4대의 머신, 4개의 계정 (3 Linux, 1 Windows), 모두 Max 플랜, v2.1.90 — 모든 게이트가 비활성화로 표시됩니다. Context는 여전히 제거되고 있습니다.**

## SM Compact 임계값 문제

게이트가 꺼져 있어도, 원격 설정에는 코드 기본값보다 훨씬 더 공격적인 임계값이 포함되어 있습니다:

| 파라미터 | 원격 설정 | 코드 기본값 | 차이 |
|-----------|--------------|-------------|------------|
| `minTokens` | **2,000** | 10,000 | **5배 낮은 하한** |
| `maxTokens` | **20,000** | 40,000 | **2배 낮은 상한** |
| `minTextBlockMessages` | 5 | — | — |

이 게이트가 A/B 테스트를 통해 활성화되면, 세션은 최대 20K 토큰만 유지하게 됩니다 — 1M context에 비용을 지불하는 사용자가 기대하는 것의 극히 일부입니다.

## 캐시 영향 — 수정된 이해 (4월 3일)

**초기 가설 (4월 2일):** microcompact가 프롬프트 캐시 prefix를 무효화하여 캐시 비율이 0%가 되고 전액 과금될 것으로 예상했습니다.

**측정 결과:** 이 가설은 **부분적으로 틀렸습니다.** 327건의 이벤트(모든 세션 합산)에 대한 프록시 데이터는 다음을 보여줍니다. 상세한 sub-agent 캐시 측정은 [04_BENCHMARK.md](04_BENCHMARK.md)를 참조하십시오.

| Context | 삭제 중 캐시 비율 | 설명 |
|---------|---------------------------|-------------|
| 메인 세션 | **99%+** — 영향 없음 | 동일한 마커로 일관되게 대체하므로 호출 간 prefix가 변경되지 않음 |
| Sub-agent 콜드 스타트 | **0~39%** — 유의미한 하락 | Sub-agent가 이미 삭제된 context에서 새 캐시를 구축하며 콜드 스타트 페널티 발생 |
| Sub-agent 워밍 완료 (5회+ 요청) | **94~99%** — 정상 | 워밍 후 삭제된 콘텐츠와 무관하게 안정화 |

**진짜 비용은 캐시 과금이 아니라 context 품질입니다.** Tool result가 삭제되면:
- 모델이 이전 파일 내용, grep 결과, 명령 출력을 정확하게 참조할 수 없습니다
- 이미 시도한 것을 볼 수 없어 반복 실패가 발생합니다 (모델이 이전에 무엇을 시도했는지 확인 불가)
- 장시간 세션이 1M 윈도우임에도 실질 context가 약 40~80K로 저하됩니다 ([@Sn3th](https://github.com/Sn3th) 보고)

**관련 관측:**
- 오래된 Docker 고정 버전 (v2.1.74/86, 업데이트 안 함)에서도 최근 소진이 시작되었습니다 ([#37394](https://github.com/anthropics/claude-code/issues/37394)) — 이는 GrowthBook flag가 클라이언트 업데이트 없이 서버에서 변경된다는 것을 확인합니다
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70`은 삭제를 방지하지 못합니다 (microcompact가 이를 완전히 무시)
- 소진은 사용자마다 간헐적으로 나타납니다 — GrowthBook A/B 테스트가 누가 영향받는지와 얼마나 공격적으로 적용되는지를 제어하는 것과 일치합니다

## 로컬 재현 (진행 중)

### 설정

프록시 기반 요청 본문 스캐너 (cc-relay + `ANTHROPIC_BASE_URL`)가 API 요청이 머신을 떠나기 전에 `[Old tool result content cleared]`를 탐지합니다. GrowthBook 파일 감시기가 10초마다 `~/.claude.json`을 비교합니다.

### 확인된 결과 (2026년 4월 3일)

향상된 cc-relay 프록시를 사용하여 여러 세션에 걸친 체계적 테스트 (3,500건 이상의 로깅 요청, 총 327건의 microcompact 이벤트). 전체 세션 로그 분석은 [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md)를 참조하십시오.

**삭제 패턴:**
- 총 **327건의 이벤트** 탐지 — 두 주요 v2.1.90 세션에서 67건과 71건, v2.1.91 테스트 세션에서 14건, 나머지 175건은 v2.1.90 테스트 중 생성된 sub-agent 세션에서 발생
- 삭제 인덱스가 **시간에 따라 확대**: [20,22]에서 시작하여 [20,22,38,40,44,46,162,166,172,174,206]으로 증가
- **삭제된 모든 인덱스가 짝수** → tool_use/tool_result 쌍을 정확히 타겟팅
- 동일 인덱스가 이후 모든 API 호출에서 일관되게 삭제 (안정적 대체)

**캐시 영향 — 핵심 발견:**
- **메인 세션: 캐시 영향 없음.** 활발한 삭제 중에도 비율이 99%+ 유지. 동일 인덱스를 동일 마커 텍스트로 일관되게 대체하므로 프롬프트 prefix가 변경되지 않아 캐시가 유효하게 유지됩니다.
- **Sub-agent 콜드 스타트: 0~39% 캐시 하락** 관측. 삭제가 활성화된 시점에서 발생. Sub-agent가 이미 삭제된 context에서 새 캐시를 구축하기 때문입니다.
- **실질적 피해는 context 품질이지 캐시 비용이 아닙니다.** 에이전트가 이전 tool result에 접근할 수 없어 정확한 인용이나 참조가 불가능해집니다.

**Budget Enforcement (Bug 5) — 새로운 발견:**

microcompact 외에, **별도의 사전 요청 파이프라인**이 서버 제어 GrowthBook 임계값에 따라 tool result를 잘라냅니다:

```
tengu_hawthorn_window:       200,000 (aggregate tool result cap)
tengu_pewter_kestrel:        {global: 50000, Bash: 30000, Grep: 20000, Snip: 1000}
tengu_summarize_tool_results: true
```

이것은 `applyToolResultBudget()`을 통해 microcompact **이전에** 실행됩니다. 확인된 활성 상태:
- 단일 세션에서 **261건의 budget 이벤트** 탐지
- Tool result가 **1~41자**로 축소 (원래 수천 자 이상; 전체 주간 최대: 49자 — [13_PROXY-DATA.md](13_PROXY-DATA.md) 참조)
- **242,094자** 시점에서 budget 초과 (> 200K 상한)
- v2.1.91의 `maxResultSizeChars` 오버라이드는 **MCP 전용** — 내장 Read/Bash/Grep에는 영향 없음

**v2.1.91 검증:**
- v2.1.91 테스트 세션에서 14건의 microcompact 이벤트, 71건의 budget 이벤트
- **v2.1.90에서 변경 없음** — 두 버그 모두 동일하게 지속

### 남은 검증 사항

1. **바이너리 코드 경로 식별** — 디컴파일된 바이너리에서 `Old tool result content cleared` 검색
2. **GrowthBook 네트워크 가로채기** — 디스크 캐시에서는 삭제 중 변화가 없음; 메모리 내 불일치가 의심됨
3. **Budget 잘림 임계값** — 잘림이 시작되는 정확한 토큰 수 (글자 수가 아닌)

## 미해결 질문

### 1. GrowthBook 런타임과 디스크 캐시의 불일치

`~/.claude.json`의 디스크 캐시는 마지막 GrowthBook 평가의 스냅샷입니다. 하지만 GrowthBook은 세션별 속성(사용자 ID, 플랜 등급, 세션 ID, 타임스탬프 등)을 사용하여 런타임에 기능을 평가합니다. CLI가 세션 중간에 다른 속성으로 재평가하면 — 토큰 임계값을 넘은 후, 또는 세션 지속 시간에 따라 — 디스크 캐시에는 절대 반영되지 않습니다.

예비 로컬 테스트에서 활발한 삭제 중 디스크 캐시 변화는 관측되지 않았지만, 메모리 내 불일치를 배제하지는 못합니다.

**테스트:** 제거가 활발히 발생하는 세션에서 GrowthBook SDK 호출(또는 GrowthBook API 엔드포인트)을 가로채서, 런타임 feature 값과 디스크 캐시를 비교합니다.

### 2. 서버 측 콘텐츠 삭제

`[Old tool result content cleared]`가 나타날 때, 이것이 Anthropic의 원시 API 응답에 포함된 것인지, 아니면 CLI가 로컬에서 삽입하는 것인지 확인이 필요합니다.

예비 증거는 **클라이언트 측**(나가는 요청 본문에서 마커 발견)을 가리키지만, 더 긴 세션과 확인된 최초 삭제 캡처로 검증이 필요합니다.

### 3. 네 번째 경로

문서화된 세 가지 GrowthBook 게이트가 모두 꺼져 있는데도 삭제가 발생합니다. 로컬에서 확인했습니다. 후보:
- GrowthBook을 전혀 확인하지 않는 바이너리 내 코드 경로
- 특정 빌드에서만 활성화되는 컴파일 타임 flag
- 클라이언트 측 삭제를 유발하는 응답 헤더 또는 메타데이터 필드
- GrowthBook 게이트와 독립적으로 작동하는 `cache_edits` API 경로 (캐시된 microcompact)

## 효과 없는 환경 변수

| 환경 변수 | 제어 대상 | Microcompact를 차단하는가? |
|---------|-----------------|---------------------------|
| `DISABLE_AUTO_COMPACT=true` | Autocompact만 | **아니오** |
| `DISABLE_COMPACT=true` | 수동 `/compact` 포함 모든 compaction | 아마도, 하지만 수동 compact도 같이 무력화됩니다 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | Autocompact 임계값 | **아니오** |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=true` | Session memory compact만 | **아니오** (시간 기반 또는 캐시된 MC는 다루지 않음) |

**필요한 것:** `DISABLE_MICROCOMPACT=true` — microcompact를 독립적으로 비활성화하는 전용 환경 변수.

## 데이터 기여

대화 내용이 몰래 제거되는 현상을 겪고 있다면, 다음을 공유해 주십시오:

1. **GrowthBook flag** — 위의 추출 스크립트를 실행하고 값을 공유해 주십시오. 만약 `tengu_sm_compact: true` 또는 `tengu_slate_heron.enabled: true`를 보여주는 사례가 있다면, A/B 테스트 가설이 확인됩니다.
2. **프록시 캡처** — `ANTHROPIC_BASE_URL`을 로깅 프록시로 설정하고, `[Old tool result content cleared]`가 나타날 때의 원시 API 응답을 캡처해 주십시오. 이것으로 클라이언트 측인지 서버 측인지 판별할 수 있습니다.
3. **세션 상세** — 버전, 플랜 등급, OS, 제거가 관측되었을 때의 대략적인 세션 길이 및 도구 사용량.

## 참조

- [anthropics/claude-code#42542](https://github.com/anthropics/claude-code/issues/42542) — 소스 코드 분석이 포함된 원본 이슈
- [anthropics/claude-code#40524](https://github.com/anthropics/claude-code/issues/40524) — Bug 1: Sentinel cache 손상
- [anthropics/claude-code#34629](https://github.com/anthropics/claude-code/issues/34629) — Bug 2: Resume cache 퇴행
- [anthropics/claude-code#40584](https://github.com/anthropics/claude-code/issues/40584) — Bug 3: 클라이언트 측 false rate limiter
- [anthropics/claude-code#37394](https://github.com/anthropics/claude-code/issues/37394) — Docker 고정 구버전 소진 (서버 측 증거)
- [anthropics/claude-code#42590](https://github.com/anthropics/claude-code/issues/42590) — 1M에서 context compaction이 지나치게 공격적
