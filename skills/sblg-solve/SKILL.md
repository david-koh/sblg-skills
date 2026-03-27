---
name: sblg-solve
description: >-
  First-principles problem solving — expert panel parallel diagnosis →
  atomic decomposition (separate constants from assumptions) →
  cross-domain pattern mapping → transfer learning to reconstruct solutions.
  Use when: "I'm stuck", "can't solve this", "need a breakthrough", "why isn't this working",
  "different approach", "sblg-solve".
license: MIT
metadata:
  author: sblg
  version: "1.0.0"
---

# sblg-solve — 막힌 문제의 해결책 도출

기존 방식으로 안 풀리는 문제를 원자 단위로 분해하고, 다른 분야에서 해결 패턴을 가져와 재구성하는 스킬. "원래 이렇게 하는 거야"를 깨는 것이 목적.

## Commands

| 커맨드 | 설명 |
|--------|------|
| `/sblg-solve` | 새 문제 해결 시작 (Phase 1부터) |
| `status` | 현재 진행 상태 |
| `continue` | 이전 대화에서 이어서 (\_status.md 기반) |

## Session Continuity

새 대화에서 `continue` 또는 `status` 입력 시:
1. `~/.sblg/solve/` 디렉토리에서 가장 최근 세션 탐색
2. `_status.md` 읽어서 현재 Phase와 진도 파악
3. 남은 작업부터 이어서 진행

## Web Search

검색이 필요한 단계에서 사용 가능한 웹 검색 도구를 사용한다.
검색 도구가 없으면 "검색 불가 — 내부 지식으로 진행" 안내 후 계속.

## Workflow

### Phase 1: 전문가 패널 진단 (입력)

사용자가 막힌 문제를 설명하면 해당 도메인의 Top 0.1% 실무 전문가 페르소나 3~5명을 자동 생성한다. 같은 분야에만 갇히면 같은 관행 안에서 돈다. 의도적으로 인접 분야 전문가를 포함해 다른 시각을 확보한다.

**페르소나 생성 기준:**
- 같은 분야 2명 (문제의 직접적 맥락 이해)
- 인접 분야 1~3명 (다른 시각 — 이 문제와 구조적으로 비슷한 문제를 다른 분야에서 풀어본 사람)
- 각 페르소나에 이름, 전문 영역, 경력 배경 부여

**병렬 subagent로 각 전문가가 독립적으로 진단:**
1. "이 문제의 진짜 원인은 무엇인가?"
2. "사람들이 당연시하는 가정은 무엇인가?"
3. "우리 분야에서 비슷한 문제를 어떻게 풀었는가?"
4. "이 문제에서 가장 흔한 실패 패턴은?"

완료 후 보고: "Phase 1 완료: 전문가 N명 진단 수집. [공통 진단/의견 차이 1줄 요약]"

산출물: `diagnosis.md`

### Phase 2: 퍼스트 프린시플 분해 (원자 단위 해체)

전문가 진단은 입력 재료일 뿐이다. 이 단계의 본질은 문제에 깔린 가정들을 모두 드러내고, 진짜 제약(상수)과 제거 가능한 관행(변수)을 분리하는 것이다.

**Step 1: 변수/상수 분리**

문제에 등장하는 모든 요소를 나열하고 분류:
- **상수** (물리적으로 바꿀 수 없는 제약): 법적 규제, 물리법칙, 수학적 한계, 하드웨어 제약
- **변수** (일반적 착각/관행): "원래 이렇게 하는 거야", 비용 가정, 업계 상식, 조직 관성

**Step 2: 변수 제거 (소크라테스식 심문)**

영향력 큰 상위 3~5개 변수에 대해:
1. "이 가정의 출처가 뭐지? 누가 처음 말했지?"
2. "이게 물리적 제약인가, 관행인가?"
3. "이 가정을 깨고 성공한 사례가 있나?" — WebSearch로 반증 탐색
4. "이 가정이 틀리면 어떤 해결책이 열리나?"

**Step 3: 원자 단위 도출**

변수를 제거하고 남은 상수들이 "더 이상 쪼갤 수 없는 원자"인지 검증. 쪼갤 수 있으면 한 번 더 분해.

**Step 4: 인과관계 재구성**

남은 원자들의 인과관계: "이 병목이 존재하는 이유는 A이고, A가 존재하는 이유는 B다." 가장 근본적인 병목 하나를 특정.

완료 후 보고: "Phase 2 완료: 진짜 제약 N개, 제거 가능한 가정 M개. 근본 병목: {한 줄}."

산출물: `constraints.md` (상수/변수 분류표 + 인과관계 + 근본 병목)

### Phase 3: 교차 학습 (연결)

사용자가 이미 깊이 아는 분야(A)에서 근본 원리를 추출하고, 현재 문제(B)의 제약(Phase 2 산출물)과 구조적으로 1:1 매핑한다. 비유가 아니라 같은 제약이 다른 맥락에서 어떻게 풀렸는지.

**Step 1: 사용자의 기존 전문 분야 파악**

"당신이 가장 깊이 아는 분야는?" — 이미 알고 있으면 스킵.

**Step 2: 기존 분야(A)에서 비슷한 제약을 어떻게 풀었는지 추출**

Phase 2의 근본 병목과 구조적으로 같은 문제가 A에 있었는지 확인.

**Step 3: 구조적 유사성 매핑**

| 현재 문제(B)의 제약 | A 분야의 유사 제약 | A에서의 해결법 | 구조적 유사성 근거 |
|--------------------|--------------------|---------------|-------------------|
| ... | ... | ... | ... |

**Step 4: 매핑 검증**

WebSearch로 "이 구조적 유사성이 실제로 성립하는가?" 검증. 검색은 연결을 발견하는 게 아니라 검증하는 데 사용.

완료 후 보고: "Phase 3 완료: {A} 분야에서 N개 해결 패턴 매핑."

산출물: `cross-patterns.md`

### Phase 4: 전이 학습 (적용) → 해결책 도출

Phase 3의 매핑을 바탕으로 다른 분야의 해결 패턴을 현재 문제에 재구성한다.

**3개 해결책 제안:**

1. **보수적** — 가장 적은 가정을 제거. 안전하지만 개선 폭 작음.
2. **급진적** — 핵심 가정을 제거. 큰 개선이지만 리스크 있음.
3. **창의적** — 다른 분야 패턴을 직접 전이. 예상치 못한 접근.

각 해결책에 반드시 포함:
- "어떤 가정을 제거했는가" (Phase 2 변수 참조)
- "어떤 분야의 패턴을 가져왔는가" (Phase 3 매핑 참조)
- "되돌림 지표" — 이 해결책이 틀렸다면 어디서 알 수 있는가

완료 후 보고: "Phase 4 완료: 3개 해결책 제안. 추천: {N번} — {이유 1줄}."

산출물: `solutions.md`

## Output Structure

```
~/.sblg/solve/
└── {문제슬러그}-{YYYYMMDD}/
    ├── _status.md
    ├── diagnosis.md
    ├── constraints.md
    ├── cross-patterns.md
    └── solutions.md
```

## _status.md Format

```markdown
# Session Status
- problem: {문제 한 줄 요약}
- current_phase: {1~4}
- started_at: {날짜}
- last_updated: {날짜}
- expert_count: {N}
- constants_count: {N}
- variables_removed: {N}
- root_bottleneck: {한 줄}
- solutions_count: {N}
- next_action: {다음에 해야 할 구체적 작업}
```
