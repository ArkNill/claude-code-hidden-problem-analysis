> 이 문서는 [영어 원본](../12_ADVANCED-GUIDE.md)의 한국어 번역입니다.

# Claude Code 고급 가이드 — 파워 유저 실전 가이드

> **요약:** CLAUDE.md 아키텍처를 이해하면 비용과 컨텍스트 품질을 제어할 수 있습니다. 프록시 모니터링을 사용하면 Claude Code가 숨기는 쿼터 수치를 확인할 수 있습니다. 훅(hook)과 멀티 프로젝트 격리를 통해 가드레일을 자동화하고 여러 코드베이스에 걸쳐 확장할 수 있습니다.
>
> **대상 독자:** 에이전트 개발자, 멀티 프로젝트 운영자, 팀.
> **사전 요구사항:** 설정 방법은 [09_QUICKSTART.md](09_QUICKSTART.md)를 참조하십시오. 기술적 맥락을 위해 [01_BUGS.md](01_BUGS.md)의 버그 분석과 [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md)의 쿼터 아키텍처를 참조하십시오.

---

## 1. CLAUDE.md 아키텍처

Claude Code는 매 턴마다 지시 파일을 로드하여 시스템 프롬프트(모델에게 전달되는 기본 지시문)에 연결합니다. 이 아키텍처를 이해하는 것이 동작과 비용 모두를 제어하는 핵심입니다.

### 3단계 계층 구조

| 계층 | 위치 | 범위 | 로드 시점 |
|------|------|------|----------|
| 글로벌 | `~/.claude/CLAUDE.md` | 모든 프로젝트, 모든 세션 | 항상 |
| 프로젝트 (루트) | `<project>/CLAUDE.md` | 해당 프로젝트만 | `<project>/` 디렉토리에서 작업할 때 |
| 프로젝트 (dotdir) | `<project>/.claude/CLAUDE.md` | 해당 프로젝트만 (대안) | `<project>/` 디렉토리에서 작업할 때 |

모든 계층이 연결되어 **매 API 호출마다** 시스템 프롬프트의 일부로 전송됩니다. 지연 로딩이나 조건부 포함은 없습니다.

### 설계 원칙

- **글로벌 계층:** 진정으로 보편적인 규칙만 포함합니다. 코딩 스타일, git 규칙, 보안 정책. 목표: **50줄 이하**.
- **프로젝트 계층:** 해당 코드베이스의 고유한 특성만 포함합니다. 기술 스택, 주요 파일 경로, 네이밍 규칙. 목표: **150줄 이하**.
- **합산 목표:** 모든 계층을 합쳐 200줄 이하.
- **사람이 아닌 모델을 위해 작성하십시오.** 간결한 명령문이 설명적 산문보다 효과적입니다. "시크릿에는 `os.getenv()`를 사용하고, 절대 하드코딩하지 마라"가 환경 변수의 중요성을 설명하는 문단보다 낫습니다.

### 비용 분석

200줄짜리 CLAUDE.md는 대략 2,000 토큰입니다. 일반적인 캐시 비율(96-99% `cache_read`)에서는 턴당 캐시 읽기 가격이 부과됩니다. 개별적으로는 저렴하지만 누적됩니다:

| CLAUDE.md 크기 | 토큰 수 | 캐시 비율 기준 턴당 비용 | 위험도 |
|---------------|--------|----------------------|-------|
| 50줄 | ~500 | 무시할 수준 | 없음 |
| 200줄 | ~2,000 | 낮음 | 없음 |
| 500줄 | ~5,000 | 보통 | 캐시 미스 위험 증가 |
| 1,000줄 이상 | ~10,000+ | 높음 | 수확 체감, 캐시 불안정 |

시스템 프롬프트(CLAUDE.md 포함)가 캐시 프리픽스(캐시 일치 여부를 판단하는 앞부분)를 구성합니다. 프리픽스가 클수록 해싱하고 매칭해야 할 바이트가 늘어납니다. 약 500줄을 넘으면 사소한 변경으로 인한 캐시 미스 가능성이 높아지고, 지시 수행 효과도 정체됩니다.

### MEMORY.md 패턴

Claude Code는 `~/.claude/projects/<project-path>/memory/` 아래에 `MEMORY.md` 파일을 자동으로 관리합니다. 이 파일은 세션 간에 유지됩니다.

**적합한 용도:**
- 사용자 선호 설정 (언어, git 이메일, 스타일)
- 프로젝트 상태 추적 및 상세 파일에 대한 포인터
- 세션 간에 적용되는 피드백 규칙

**부적합한 용도:**
- 코드 패턴이나 아키텍처 문서 (코드를 직접 읽는 것이 항상 더 최신입니다)
- 임시 상태 (`/compact` 요약이나 작업 파일을 사용하십시오)
- 대용량 콘텐츠 블록 (MEMORY.md는 CLAUDE.md와 마찬가지로 매 턴마다 로드됩니다)

**인덱스 패턴:** MEMORY.md를 개별 메모리 파일에 대한 포인터 인덱스로 유지하십시오. 각 파일은 하나의 주제를 다룹니다. 이렇게 하면 턴당 로드량을 작게 유지하면서도 심층 저장이 가능합니다.

### 안티 패턴

| 안티 패턴 | 문제점 | 해결 방법 |
|----------|-------|----------|
| 라이브러리 문서를 CLAUDE.md에 복사 | 매 턴마다 부풀어나며, 모델은 이미 인기 라이브러리를 알고 있음 | 문서 링크를 제공하거나 모델이 필요할 때 읽도록 함 |
| 자주 변경되는 파일 트리 포함 | 트리 변경 시 캐시 프리픽스가 무효화됨 | glob 패턴이나 주요 파일 목록 사용 |
| CLAUDE.md에 세션별 상태 저장 | 세션 간 오래된 상태가 되며 글로벌 지시를 오염시킴 | PLAN 파일이나 세션 로컬 노트 사용 |
| 계층 간 규칙 중복 | 토큰 낭비 및 모순 위험 | 각 규칙을 정확히 하나의 계층에만 배치 |

---

## 2. 서브에이전트 전략

Claude Code는 독립적인 AI 인스턴스("서브에이전트")를 생성하여 하위 작업을 병렬로 처리할 수 있습니다. 각 서브에이전트는 자체 컨텍스트 윈도우(한 번에 처리할 수 있는 텍스트 범위)를 가진 완전한 Claude Code 세션입니다.

### 비용 프로필

| 단계 | 캐시 비율 | 참고 |
|------|----------|------|
| 콜드 스타트 (첫 번째 요청) | 54-80% | 전체 캐시 구축 — 비용 높음 |
| 워밍 (2-5번째 요청) | 80-94% | 점진적 개선 |
| 안정 상태 (5번째 이후 요청) | 94-99% | 정상 운영 비용 |

각 서브에이전트는 독립적으로 자체 캐시를 구축합니다. 부모와 자식 세션 간 캐시 공유는 없습니다.

### 서브에이전트 활용이 적합한 경우

**적합한 후보:**
- 여러 디렉토리를 동시에 검색 (다른 하위 트리에 3-5개 에이전트 배치)
- 메인 세션의 컨텍스트를 오염시키지 않아야 하는 격리된 조사
- 장시간 실행되는 백그라운드 작업 (코드 생성, 테스트 작성)
- 결과만 필요하고 중간 추론 과정은 필요 없는 작업

**부적합한 후보:**
- 단순한 단일 파일 작업 (콜드 스타트 오버헤드가 이점을 초과함)
- 메인 세션의 축적된 컨텍스트가 필요한 작업 (서브에이전트는 처음부터 시작함)
- 사용량 한도에 가까운 경우 (각 서브에이전트가 동일한 쿼터 풀에서 소모함)
- 한 턴에 끝나는 짧은 작업 (단일 응답에 콜드 스타트 비용을 지불하게 됨)

### 컨텍스트 전달

서브에이전트는 CLAUDE.md를 읽지만(에이전트당 토큰 비용 발생), 부모 세션의 대화 히스토리는 **상속하지 않습니다**. 효과적인 패턴은 다음과 같습니다:

- **명시적 프롬프트:** 서브에이전트에게 정확한 파일 경로, 검색어, 예상 출력 형식을 제공하십시오. 이전에 논의한 내용을 알고 있다고 가정하지 마십시오.
- **PLAN 파일:** 복잡한 다단계 작업의 경우, 계획을 파일에 작성하고 서브에이전트가 읽도록 하십시오. 프롬프트에 컨텍스트를 채워 넣는 것보다 `Read` 호출 한 번의 비용으로 해결됩니다.
- **범위 지정 지시:** 서브에이전트에 프로젝트별 규칙이 필요한 경우, 이미 CLAUDE.md에 포함되어 있습니다. 프롬프트에서 반복하지 마십시오.

### 병렬 디스패치

탐색 작업에는 3-5개 에이전트를 동시에 실행하는 것이 효율적입니다. 각 에이전트가 독립적으로 자체 캐시를 워밍합니다. 총 비용은 콜드 스타트 오버헤드의 3-5배이지만, 실제 소요 시간 절약이 보통 이를 정당화합니다.

세션 내 모든 에이전트가 동일한 사용량 한도 쿼터에서 소모한다는 점에 유의하십시오. 5개 병렬 에이전트는 순차 작업보다 5배 빠르게 쿼터를 소진합니다.

---

## 3. 훅 & 자동화

훅(hook)은 Claude Code가 도구를 사용할 때 자동으로 실행되는 셸 명령입니다. 수동 개입 없이 가드레일, 린팅(코드 품질 검사), 정책 적용을 가능하게 합니다.

### 설정

훅은 `~/.claude/settings.json`의 `"hooks"` 키 아래에 위치합니다. 프로젝트 수준 오버라이드는 `<project>/.claude/settings.json`에 배치합니다.

### 사용 가능한 훅 지점

| 훅 | 실행 시점 | 차단 가능 여부 | 용도 |
|----|----------|-------------|------|
| `PreToolUse` | 도구 실행 전 | 가능 (exit 1) | 위험한 명령 차단 |
| `PostToolUse` | 도구 실행 후 | 불가 | 린팅, 검증, 로깅 |
| `Notification` | Claude가 알림을 보낼 때 | 불가 | 커스텀 알림 |

### 훅에서 사용 가능한 환경 변수

| 변수 | 내용 |
|------|------|
| `CC_TOOL_INPUT` | 도구 입력 파라미터의 JSON 문자열 |
| `CC_TOOL_NAME` | 호출되는 도구의 이름 |

### 실전 예시: Python 파일 편집 후 자동 린팅

이 훅은 Claude가 편집하거나 작성하는 모든 Python 파일에 대해 프로젝트의 로컬 가상 환경을 사용하여 `ruff`를 실행합니다:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "filepath=$(echo \"$CC_TOOL_INPUT\" | grep -oP '\"file_path\"\\s*:\\s*\"\\K[^\"]+'); if [ -z \"$filepath\" ] || [ ! -f \"$filepath\" ]; then exit 0; fi; case \"$filepath\" in *.py) d=$(dirname \"$filepath\"); while [ \"$d\" != \"/\" ]; do if [ -f \"$d/.venv/bin/ruff\" ] && [ -f \"$d/pyproject.toml\" ]; then \"$d/.venv/bin/ruff\" check --no-fix \"$filepath\" 2>&1 | head -20; exit 0; fi; d=$(dirname \"$d\"); done ;; esac",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

주요 설계 결정 사항:
- 디렉토리 트리를 위로 탐색하여 가장 가까운 `.venv/bin/ruff`를 찾습니다 — 멀티 프로젝트 환경에서도 동작합니다
- `.py` 파일에서만 트리거됩니다 (`case` 문)
- Python이 아닌 파일에서는 깔끔하게 종료합니다 (exit 0) — 거짓 실패 없음
- `--no-fix`는 파일을 수정하지 않고 문제만 보고합니다
- `head -20`은 출력을 제한하여 컨텍스트 범람을 방지합니다

### 예시: 위험한 명령 차단

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$CC_TOOL_INPUT\" | grep -qiE 'rm\\s+-rf\\s+/|drop\\s+table|git\\s+push\\s+--force\\s+(origin\\s+)?main' && echo 'BLOCKED: Dangerous command detected' && exit 1 || exit 0",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

이 훅은 `rm -rf /`, `DROP TABLE`, main 브랜치로의 강제 푸시를 차단합니다. 위험 허용 수준에 맞게 정규식을 조정하십시오.

### 글로벌 환경 변수

settings.json의 `"env"` 키는 모든 Claude Code 세션에 환경 변수를 설정합니다:

```json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1",
    "ANTHROPIC_BASE_URL": "http://localhost:8080"
  }
}
```

`DISABLE_AUTOUPDATER`는 작업 중 예기치 않은 버전 변경을 방지합니다. `ANTHROPIC_BASE_URL`은 모든 API 트래픽을 로컬 프록시를 통해 라우팅합니다 (섹션 4 참조).

### 훅 설계 가이드라인

- **훅은 빠르게 유지하십시오.** `timeout` 필드의 단위는 밀리초입니다. 10초 타임아웃은 린터에 충분히 여유 있는 값이며, 차단 훅은 5초 이내로 유지하십시오.
- **성공 시 exit 0, 차단 시 exit 1.** `PreToolUse` 훅만 차단할 수 있으며, `PostToolUse`의 종료 코드는 무시됩니다.
- **훅에서 파일을 수정하지 마십시오.** 파일 시스템에 쓰는 훅은 Claude의 자체 쓰기 작업과 경쟁 조건(race condition)을 유발할 수 있습니다.
- **훅을 먼저 독립적으로 테스트하십시오.** 샘플 `CC_TOOL_INPUT`으로 명령을 수동 실행한 후 settings.json에 추가하십시오.

---

## 4. 모니터링 & 프록시 설정

Claude Code의 내장 사용량 바는 대략적인 퍼센트만 표시하며, 실제 쿼터 메커니즘은 숨겨져 있습니다. 투명 프록시를 사용하면 전체 그림을 볼 수 있습니다.

### 모니터링이 필요한 이유

- 요청별 정확한 `cache_creation_input_tokens`와 `cache_read_input_tokens` 확인
- Claude Code가 버리는 사용량 한도 헤더 캡처 (`representative-claim`만 읽음)
- API에 도달하기 전에 컨텍스트 변형(마이크로컴팩트, 예산 제한) 감지
- 가시적 토큰 수와 사용률 퍼센트 간의 상관관계 파악

### ANTHROPIC_BASE_URL 프록시

`ANTHROPIC_BASE_URL`은 공식 환경 변수입니다. 이를 설정하면 모든 API 요청이 로컬 프록시를 경유합니다:

```bash
# settings.json에서
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:8080"
  }
}
```

프록시는 모든 요청과 응답을 수정 없이 로깅합니다. 헤더, 토큰 사용량, 요청 본문을 캡처합니다.

### 사용량 한도 헤더

모든 API 응답에는 `anthropic-ratelimit-unified-*` 헤더가 포함됩니다 ([02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) 참조):

| 헤더 | 의미 | 예시 |
|------|------|------|
| `5h-utilization` | 현재 5시간 윈도우 사용량 (0.0-1.0) | `0.26` |
| `7d-utilization` | 현재 7일 윈도우 사용량 | `0.19` |
| `representative-claim` | 어느 윈도우가 병목인지 | `five_hour` |
| `5h-reset` | 5시간 윈도우 리셋 Unix 타임스탬프 | `1775455200` |
| `7d-reset` | 7일 윈도우 리셋 Unix 타임스탬프 | `1775973600` |
| `fallback-percentage` | 용량 할당 비율 | `0.5` |
| `overage-status` | 초과 사용 과금 상태 | `allowed` |

**3,702건의 측정 요청에서 얻은 핵심 인사이트:** 5시간 윈도우가 항상 바인딩 제약 조건입니다 (100%의 요청에서 `representative-claim` = `five_hour`). 5시간 윈도우의 1%당 대략 1.5M-2.1M 가시적 토큰이 소비됩니다 — 하지만 그 중 96-99%가 `cache_read`입니다. 1%당 가시적 출력은 9K-16K 토큰에 불과하며, 이는 사고 토큰(thinking token, API 응답에서 보이지 않는 추론 과정)이 쿼터 소비의 상당 부분을 차지함을 시사합니다.

### JSONL 세션 로그

Claude Code는 `~/.claude/projects/<project-path>/`에 JSONL 파일로 세션 로그를 기록합니다. 이것은 모든 API 상호작용의 클라이언트 측 기록입니다.

**중요한 주의사항:** 확장 사고(Extended thinking)는 토큰 수를 중복 계산하는 PRELIM 항목을 생성합니다. 측정 데이터에서 JSONL은 프록시가 보는 것 대비 `cache_read` 토큰을 **1.93배** 기록합니다. 실제 비용을 계산할 때는 항상 `stop_reason: "end_turn"`으로 필터링하여 FINAL 항목만 사용하십시오.

서브에이전트 세션은 같은 디렉토리에 별도의 JSONL 파일을 가집니다. 파일 이름에 서브에이전트의 세션 ID가 포함됩니다.

---

## 5. 멀티 프로젝트 워크플로우

### 컨텍스트 격리

각 프로젝트 디렉토리는 다음의 고유한 항목을 가집니다:
- CLAUDE.md (프로젝트 수준 지시)
- `.claude/settings.json` (프로젝트 수준 설정 오버라이드)
- 세션 로그 (`~/.claude/projects/<project-path>/` 아래 JSONL 파일)
- MEMORY.md (영속 메모리)

### 설정 계층

| 수준 | 파일 | 오버라이드 |
|------|------|----------|
| 글로벌 | `~/.claude/settings.json` | 기본 설정 |
| 프로젝트 | `<project>/.claude/settings.json` | 글로벌 설정을 확장/오버라이드 |

프로젝트 설정은 글로벌 설정을 대체하지 않고 병합합니다. 양 수준의 훅이 모두 실행됩니다. 프로젝트 수준 환경 변수가 글로벌 수준을 오버라이드합니다.

### 프로젝트 전환

- **프로젝트를 전환할 때 새 세션을 시작하십시오.** 프로젝트 A의 컨텍스트를 프로젝트 B로 가져가지 마십시오 — 토큰 낭비이며 모델을 혼란시킵니다.
- **각 프로젝트는 자체 완결적이어야 합니다.** CLAUDE.md에 Claude가 글로벌 파일의 프로젝트별 규칙 설명 없이도 독립적으로 작업할 수 있을 만큼의 컨텍스트가 있어야 합니다.
- **표준은 공유하되, 세부 사항은 공유하지 마십시오.** 글로벌 CLAUDE.md: "모든 Python 함수에 타입 힌트를 사용하라." 프로젝트 CLAUDE.md: "이 프로젝트는 SQLAlchemy async를 사용하는 FastAPI, Python 3.12를 사용한다."

### 프로젝트 간 Git 워크플로우

- **지시 파일을 자동 커밋하지 마십시오.** CLAUDE.md와 PLAN 파일은 로컬 작업 문서입니다. 레포에 포함하려면 의도적으로 커밋하십시오.
- **커밋 전에 diff를 검토하십시오.** Claude는 `git log`에서 보이는 패턴을 따르므로, 깔끔한 커밋 히스토리를 유지하면 향후 커밋 품질이 향상됩니다.
- **관심사를 분리하십시오.** Claude가 여러 서브시스템에 걸쳐 변경을 가한 경우, 하나의 큰 커밋 대신 집중된 커밋으로 분리하는 것을 고려하십시오.

---

## 6. 사용량 한도 아키텍처 & 전술

쿼터 시스템을 이해하면 작업 세션을 계획하고 예기치 않은 상황을 방지할 수 있습니다. 여기의 모든 데이터는 Max 20x ($200/월) 플랜에서 프록시로 캡처한 헤더 기준입니다 — 다른 티어는 절대값이 다르지만 아키텍처는 동일합니다.

### 이중 윈도우 시스템

두 개의 독립적인 슬라이딩 윈도우가 쿼터를 제어합니다:

| 윈도우 | 기간 | 역할 | 리셋 |
|--------|------|------|------|
| **5시간** | ~5시간 | 항상 병목 | ~5시간마다 (`5h-reset` 확인) |
| **7일** | 7일 | 누적기 | 고정 주간 타임스탬프 |

7일 카운터는 각 5시간 윈도우 최대 사용량의 약 12-17%를 누적합니다. 실제로는 5시간 윈도우가 항상 사용량 한도를 트리거하는 제약 조건입니다.

### 1%당 비용 (Max 20x, 측정값)

| 지표 | 범위 | 참고 |
|------|------|------|
| 1%당 출력 | 9K-16K 토큰 | 가시적 출력만 |
| 1%당 캐시 읽기 | 1.5M-2.1M 토큰 | 가시적 볼륨의 96-99% |
| 1%당 총 가시적 토큰 | 1.6M-2.1M 토큰 | 모든 측정 필드의 합계 |

전체 5시간 윈도우(100%)는 **0.9M-1.6M 가시적 출력 토큰**에 불과합니다. 수 시간에 걸친 Opus급 작업에 대해 이 수치는 낮습니다 — 이 차이는 확장 사고 토큰이 출력 토큰 요금으로 쿼터에 계산되되, 클라이언트에게는 보이지 않는다는 것과 일관됩니다.

### 사고 토큰 사각지대

확장 사고 토큰은 API 응답의 `output_tokens`에 **포함되지 않습니다**. 클라이언트 측에서 이 비용을 보거나 제어할 수 없습니다. 실질적인 영향은 다음과 같습니다:

- 복잡한 추론 턴에 15K 출력 토큰이 사용된 것으로 보입니다
- 실제 쿼터 비용은 50K-200K+ (사고 + 출력 합산)일 수 있습니다
- 사용량 바가 가시적 토큰 수와 일치하지 않는 양만큼 점프합니다
- 클라이언트 측 완화 방법이 없습니다

### 실용적 전술

| 전술 | 효과 | 트레이드오프 |
|------|------|------------|
| 무거운 작업을 5시간 윈도우에 분산 | 더 오래 한도 이내로 유지 | 리셋 시간에 맞춘 계획 필요 |
| 다중 터미널 회피 | 병렬 소진 방지 | 병렬성 감소 |
| 세션을 간결하게 유지 | 턴당 `cache_read` 감소 | 더 잦은 새 시작 필요 |
| 프록시로 리셋 시간 확인 | 쿼터 회복 시점 파악 | 프록시 설정 필요 |
| 긴 세션보다 새 세션 | Bug 4/5 컨텍스트 열화 방지 | 축적된 컨텍스트 손실 |

### 한도에 도달했을 때

1. **실제인지 Bug B3(거짓 사용량 한도)인지 확인하십시오.** 세션 JSONL에서 `"model": "<synthetic>"`을 찾으십시오. 존재하면 클라이언트가 생성한 가짜 오류입니다.
2. **실제인 경우:** 프록시 로그에서 `5h-reset` 헤더 타임스탬프를 확인하십시오. 리셋을 기다리십시오.
3. **Claude Code를 반복적으로 재시작하지 마십시오.** 도움이 되지 않습니다 — 한도는 서버 측이며 세션이 아닌 계정에 연결되어 있습니다.
4. **새 터미널을 열지 마십시오.** 각 새 세션은 콜드 스타트 비용을 지불하여 쿼터를 더 빠르게 소진합니다.

---

## 7. MCP 서버 고려사항

MCP(Model Context Protocol, 외부 도구 연동 프로토콜) 서버는 Claude Code를 외부 도구로 확장합니다. 비용 프로필을 이해하면 컨텍스트 예산을 관리하는 데 도움이 됩니다.

### 컨텍스트 비용

활성화된 모든 MCP 도구의 스키마(도구 정의 구조)가 매 턴의 시스템 프롬프트에 포함됩니다. 도구가 많을수록 = 턴당 토큰이 늘어남 = 캐시 증가 속도 가속.

| MCP 설정 | 턴당 스키마 오버헤드 |
|----------|-------------------|
| MCP 도구 0개 | 기준선 |
| MCP 도구 5개 | +500-2,000 토큰 |
| MCP 도구 20개 | +2,000-8,000 토큰 |
| MCP 도구 57개 (관측값) | +10,000+ 토큰 |

### 도구 결과 크기 제한

도구 유형에 따라 두 가지 다른 상한이 적용됩니다:

| 도구 유형 | 결과당 제한 | 집계 제한 | 오버라이드 가능 여부 |
|----------|-----------|----------|-------------------|
| 내장 (Read, Bash, Grep, Glob, Edit) | 30K-50K 문자 (GrowthBook 플래그 기준) | **200K 문자** (Bug 5) | 불가 |
| MCP 도구 | 최대 500K 문자 | 동일한 200K 집계 | 가능 (`_meta["anthropic/maxResultSizeChars"]`) |

200K 집계 예산 (`tengu_hawthorn_window`)은 대화 내 모든 도구 결과에 걸쳐 적용됩니다. 파일 읽기를 약 15-20회 수행하면, 이전 결과가 소리 없이 1-41자로 잘립니다. 이것이 Bug 5이며, 이를 비활성화할 환경 변수는 없습니다.

### 모범 사례

- **실제로 사용하는 MCP 서버만 활성화하십시오.** 사용하지 않는 서버를 비활성화하여 턴당 스키마 오버헤드를 줄이십시오.
- **200K 집계 상한을 인지하십시오.** 대용량 MCP 응답은 내장 도구 결과와 동일한 예산을 소비합니다.
- **대용량 MCP 응답은 컴팩션(대화 압축)을 더 빠르게 트리거합니다.** 100K 문자의 단일 응답이 집계 예산의 절반을 소비합니다.
- **스키마 크기를 확인하십시오.** MCP 서버가 많은 도구를 노출하면, 각 도구가 매 턴의 토큰 수에 추가됩니다. 집중된 서버로 분리하는 것을 고려하십시오.

---

## 8. GrowthBook 기능 플래그

Claude Code는 서버 측 기능 제어를 위해 [GrowthBook](https://www.growthbook.io/)을 사용합니다. 이 시스템을 이해하면 클라이언트 업데이트 없이도 동작이 변경될 수 있는 이유를 설명할 수 있습니다.

### GrowthBook이 제어하는 항목

| 플래그 | 제어 대상 | 현재 기본값 |
|--------|---------|------------|
| `tengu_hawthorn_window` | 집계 도구 결과 예산 (문자) | `200000` |
| `tengu_pewter_kestrel` | 도구별 결과 크기 상한 | `{global: 50000, Bash: 30000, Grep: 20000}` |
| `tengu_slate_heron` | 시간 기반 마이크로컴팩트 | `enabled: false` |
| `tengu_session_memory` | 세션 메모리 컴팩트 | `false` |
| `tengu_sm_compact` | SM 컴팩트 게이트 | `false` |
| `tengu_cache_plum_violet` | 불명 (유일하게 활성화된 플래그) | `true` |
| `tengu_summarize_tool_results` | 모델용 시스템 프롬프트 플래그 | `true` |

### 중요한 이유

- Anthropic은 클라이언트 업데이트를 배포하지 않고도 이 값을 **서버 측에서** 변경할 수 있습니다
- Docker로 고정한 이전 버전(v2.1.74/86)이 플래그 값이 원격으로 변경되었을 때 소진 현상을 경험했습니다 ([#37394](https://github.com/anthropics/claude-code/issues/37394))
- `~/.claude.json`의 디스크 캐시 (`cachedGrowthBookFeatures`)는 스냅샷이며, 런타임 값은 세션 속성에 따라 다를 수 있습니다
- 기능 플래그를 통해 A/B 테스트가 가능하므로, 같은 플랜과 버전의 다른 사용자와 경험이 다를 수 있습니다

### 플래그 확인 방법

```bash
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
for k in ['tengu_slate_heron', 'tengu_session_memory', 'tengu_sm_compact',
          'tengu_sm_compact_config', 'tengu_cache_plum_violet',
          'tengu_hawthorn_window', 'tengu_pewter_kestrel']:
    print(f'{k}: {json.dumps(gb.get(k, \"NOT PRESENT\"), indent=2)}')
"
```

---

## 9. 세션 관리 패턴

### 세션 생명주기

| 단계 | 일반적인 길이 | 캐시 비율 | 컨텍스트 품질 |
|------|-------------|----------|-------------|
| 콜드 스타트 | 1-3턴 | 54-80% | 완전 (제거된 것 없음) |
| 워밍 | 4-50턴 | 94-99% | 완전 |
| 성숙 | 50-150턴 | 96-99% | 열화 중 (Bug 4/5 활성) |
| 후기 | 150턴 이상 | 96-99% | 상당히 열화됨 |

마이크로컴팩트가 동일한 마커를 일관되게 대체하기 때문에 캐시 비율은 전체적으로 높게 유지됩니다. 그러나 컨텍스트 **품질**은 열화됩니다: 모델이 세션 초반의 원본 도구 결과를 더 이상 볼 수 없게 됩니다.

### 새 세션을 시작해야 할 시점

- 자체 완결적인 작업을 완료한 후 (오래된 컨텍스트를 다음으로 가져가지 마십시오)
- Claude가 접근 방식을 반복하거나 이전 작업을 참조하지 못할 때 (Bug 4의 징후)
- 15-20회 이상 파일 읽기를 수행한 후 (Bug 5의 200K 집계 상한이 활성화되었을 가능성)
- 다른 프로젝트나 서브시스템으로 전환하기 전
- 사용량 한도에 도달한 후 (5시간 리셋을 기다리는 동안)

### 컴팩션 제어

| 방법 | 기능 | 사용 시점 |
|------|------|----------|
| `/compact` | 대화 히스토리를 요약 | 세션 내 주제 전환 전 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | 자동 컴팩트 임계값 설정 | 컨텍스트가 필요할 때 컴팩션 지연 |
| `DISABLE_AUTO_COMPACT=true` | 자동 컴팩트 비활성화 | 수동 제어를 원할 때 |

참고: 이들 중 어느 것도 마이크로컴팩트(Bug 4)나 예산 제한(Bug 5)은 제어하지 않습니다. 이들은 별도의 코드 경로에서 작동합니다.

---

## 10. 커뮤니티 도구 & 리소스

### 모니터링

| 도구 | 용도 | 링크 |
|------|------|------|
| CUStats | 웹 기반 세션 통계 뷰어 | [custats.info](https://custats.info) |
| BudMon | 사용량 한도 헤더 모니터 | [github.com/weilhalt/budmon](https://github.com/weilhalt/budmon) |
| context-stats | JSONL 기반 캐시 미스 분석 | [github.com/luongnv89/cc-context-stats](https://github.com/luongnv89/cc-context-stats) |

### 최적화

| 도구 | 용도 | 링크 |
|------|------|------|
| tokenlean | 에이전트 토큰 최적화를 위한 54가지 CLI 도구 | [github.com/edimuj/tokenlean](https://github.com/edimuj/tokenlean) |
| tokpress | 지능형 도구 출력 압축 (23 필터) | [pypi.org/project/tokpress](https://pypi.org/project/tokpress/) |

### 진단

| 도구 | 용도 | 링크 |
|------|------|------|
| ccdiag | CLI 진단 도구 | [github.com/kolkov/ccdiag](https://github.com/kolkov/ccdiag) |
| llm-relay | 멀티 CLI 프록시 + 세션 진단 (8 탐지기) | [pypi.org/project/llm-relay](https://pypi.org/project/llm-relay/) |

### 이 레포지토리의 문서

| 문서 | 주제 |
|------|------|
| [01_BUGS.md](01_BUGS.md) | 7가지 확인된 버그 — 기술적 근본 원인 분석 |
| [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) | 3,702건 프록시 요청 기반 이중 5시간/7일 쿼터 아키텍처 |
| [03_JSONL-ANALYSIS.md](03_JSONL-ANALYSIS.md) | 세션 로그 분석 방법론 (메인 110 + 서브에이전트 279 세션) |
| [04_BENCHMARK.md](04_BENCHMARK.md) | 성능 벤치마크 |
| [05_MICROCOMPACT.md](05_MICROCOMPACT.md) | 무음 컨텍스트 변형 심층 분석 (Bug 4 & 5) |
| [06_TEST-RESULTS-0403.md](06_TEST-RESULTS-0403.md) | 4월 3일 체계적 테스트 결과 |
| [07_TIMELINE.md](07_TIMELINE.md) | 14개월 이슈 연대기 (91건 이상 이슈, 4차례 에스컬레이션 사이클) |
| [08_UPDATE-LOG.md](08_UPDATE-LOG.md) | 문서 업데이트 로그 |
| [09_QUICKSTART.md](09_QUICKSTART.md) | 설정 및 자가 진단 |
| [10_ISSUES.md](10_ISSUES.md) | 관련 이슈, 커뮤니티 도구, 기여자 |

---

## 부록 A: settings.json 레퍼런스

이 가이드의 패턴을 결합한 완전한 `settings.json`입니다:

```json
{
  "env": {
    "DISABLE_AUTOUPDATER": "1"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$CC_TOOL_INPUT\" | grep -qiE 'rm\\s+-rf\\s+/|drop\\s+table|git\\s+push\\s+--force\\s+(origin\\s+)?main' && echo 'BLOCKED: Dangerous command detected' && exit 1 || exit 0",
            "timeout": 5000
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "filepath=$(echo \"$CC_TOOL_INPUT\" | grep -oP '\"file_path\"\\s*:\\s*\"\\K[^\"]+'); if [ -z \"$filepath\" ] || [ ! -f \"$filepath\" ]; then exit 0; fi; case \"$filepath\" in *.py) d=$(dirname \"$filepath\"); while [ \"$d\" != \"/\" ]; do if [ -f \"$d/.venv/bin/ruff\" ] && [ -f \"$d/pyproject.toml\" ]; then \"$d/.venv/bin/ruff\" check --no-fix \"$filepath\" 2>&1 | head -20; exit 0; fi; d=$(dirname \"$d\"); done ;; esac",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

## 부록 B: 빠른 참조 카드

### 환경 변수

| 변수 | 용도 | 권장 여부 |
|------|------|----------|
| `DISABLE_AUTOUPDATER=1` | 자동 업데이트 방지 | 예 |
| `ANTHROPIC_BASE_URL=http://...` | 프록시를 통한 라우팅 | 모니터링 시 |
| `DISABLE_AUTO_COMPACT=true` | 자동 컴팩트 비활성화 | 상황에 따라 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70` | 컴팩트 임계값 설정 | 상황에 따라 |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=true` | SM 컴팩트만 비활성화 | 제한적 효과 |
| `DISABLE_COMPACT=true` | 모든 컴팩션 비활성화 | 위험 (수동 기능도 비활성화) |

### 제어할 수 없는 메커니즘

| 메커니즘 | 이유 | 상세 |
|---------|------|------|
| 마이크로컴팩트 (Bug 4) | GrowthBook으로 서버 제어, 환경 변수 없음 | [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| 도구 결과 예산 (Bug 5) | 서버 제어, 200K 집계 상한 | [05_MICROCOMPACT.md](05_MICROCOMPACT.md) |
| 사고 토큰 비용 | API 응답에 노출되지 않음 | [02_RATELIMIT-HEADERS.md](02_RATELIMIT-HEADERS.md) |
| GrowthBook 플래그 변경 | 서버 측, 클라이언트 업데이트 없이 변경 가능 | 섹션 8 |
| PRELIM 로그 항목 (Bug 8) | 확장 사고 동작 | [01_BUGS.md](01_BUGS.md) |

### 세션 상태 점검

```bash
# GrowthBook 플래그 확인
python3 -c "
import json
gb = json.load(open('$HOME/.claude.json')).get('cachedGrowthBookFeatures', {})
print('Budget cap:', gb.get('tengu_hawthorn_window', 'NOT SET'))
print('Microcompact:', gb.get('tengu_slate_heron', {}).get('enabled', 'N/A'))
"

# 세션 로그에서 PRELIM vs FINAL 항목 수 확인
grep -c '"stop_reason":"end_turn"' ~/.claude/projects/*/YOUR_SESSION.jsonl
grep -c 'PRELIM' ~/.claude/projects/*/YOUR_SESSION.jsonl

# 합성 사용량 한도 확인
grep -c '"model":"<synthetic>"' ~/.claude/projects/*/YOUR_SESSION.jsonl
```

---

*이 가이드는 2026년 4월 기준 (Claude Code v2.1.91) 조사 결과를 반영합니다. Claude Code는 활발히 개발 중이므로, 특정 버전 세부 사항에 의존하기 전에 현재 동작을 확인하십시오. 기반 데이터와 방법론은 이 레포지토리의 링크된 분석 문서를 참조하십시오.*
