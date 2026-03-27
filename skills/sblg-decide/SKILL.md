---
name: sblg-decide
description: >-
  First-principles decision making — expert panel (pro/con/neutral) parallel opinions →
  atomic decomposition (strip emotion/convention, extract real tradeoffs) →
  cross-domain precedent mapping → transfer learning into Decision Record.
  Use when: "A vs B", "which is better", "how to decide", "tradeoff analysis",
  "pros and cons", "sblg-decide".
license: MIT
metadata:
  author: sblg
  version: "1.0.0"
---

# sblg-decide — 의사결정

선택지 앞에서 감이나 관행 대신, 근본 트레이드오프를 기반으로 판단하는 스킬. 감정과 마케팅을 걷어내고 진짜 변수만 남긴다. 산출물은 Decision Record.

## Commands

| 커맨드 | 설명 |
|--------|------|
| `/sblg-decide` | 새 의사결정 시작 (Phase 1부터) |
| `status` | 현재 진행 상태 |
| `continue` | 이전 대화에서 이어서 (\_status.md 기반) |

## Session Continuity

새 대화에서 `continue` 또는 `status` 입력 시:
1. `~/.sblg/decide/` 디렉토리에서 가장 최근 세션 탐색
2. `_status.md` 읽어서 현재 Phase 파악
3. 남은 작업부터 이어서 진행

## Web Search

검색이 필요한 단계에서 사용 가능한 웹 검색 도구를 사용한다.
검색 도구가 없으면 "검색 불가 — 내부 지식으로 진행" 안내 후 계속.

## Workflow

### Phase 1: 전문가 패널 의견 수집 (입력)

사용자가 선택지(A vs B vs ...)를 설명하면 Top 0.1% 실무 전문가 페르소나 3~5명을 자동 생성한다. 균형 잡힌 판단을 위해 의도적으로 찬/반/중립을 배치한다 — 한쪽으로 쏠린 전문가 패널은 의미 없다.

**페르소나 생성 기준:**
- A 지지 전문가 1~2명 (A의 장점을 가장 잘 아는 사람)
- B 지지 전문가 1~2명 (B의 장점을 가장 잘 아는 사람)
- 중립 분석가 1명 (두 선택지의 구조적 차이를 객관적으로 분석)
- 각 페르소나에 이름, 전문 영역, 경력 배경 부여

**병렬 subagent로 각 전문가가 독립적으로 의견:**
1. "이 선택지의 근본적 장점은 무엇인가?"
2. "이 선택지를 고르면 포기하게 되는 것은?"
3. "3년 후 이 결정을 후회할 시나리오는?"
4. "이 선택지가 가장 빛나는 상황과 가장 위험한 상황은?"

완료 후 보고: "Phase 1 완료: 전문가 N명 의견 수집. 찬 M명, 반 K명, 중립 1명."

산출물: `opinions.md`

### Phase 2: 퍼스트 프린시플 분해 (원자 단위 해체)

전문가 의견은 입력 재료일 뿐이다. 이 단계의 본질은 각 선택지에서 감정, 마케팅, 관행을 걷어내고 근본 트레이드오프만 남기는 것이다. "이 결정의 진짜 변수는 딱 N개다"를 도출한다.

**Step 1: 변수/상수 분리**

각 선택지의 장단점에서:
- **상수** (물리적으로 바꿀 수 없는 트레이드오프): 구조적 한계, 비용 구조, 시간 제약
- **변수** (감정/마케팅/관행): "이게 트렌드다", "남들이 다 쓴다", "뭔가 불안하다", 마케팅 카피

**Step 2: 변수 제거 (소크라테스식 심문)**

영향력 큰 상위 3~5개 변수에 대해:
1. "이 장점이 중요하다는 근거가 뭐지? 데이터가 있나?"
2. "이 단점이 진짜 단점인가, 두려움인가?"
3. "이 트레이드오프를 실제로 경험한 사례가 있나?" — WebSearch로 검증
4. "반대 진영의 가장 강한 논거는 무엇인가?"

**Step 3: 원자 단위 도출**

변수를 제거하고 남은 상수들이 진짜 원자인지 검증. "이 결정의 진짜 변수는 딱 N개다" 형태로 정리.

**Step 4: 인과관계 재구성**

"A를 고르면 B를 포기, B를 포기하면 C에 영향, C는 우리에게 얼마나 중요한가" — 트레이드오프의 연쇄 효과를 논리적으로 전개.

완료 후 보고: "Phase 2 완료: 진짜 트레이드오프 N개. 제거된 감정/관행 M개."

산출물: `tradeoffs.md` (상수/변수 분류표 + 근본 트레이드오프 + 인과관계)

### Phase 3: 교차 학습 (연결)

사용자가 이미 깊이 아는 분야(A)에서 같은 트레이드오프를 어떻게 결정했는지 매핑한다. "다른 맥락에서 같은 구조의 결정을 했을 때 어떤 결과가 나왔는가."

**Step 1: 사용자의 기존 전문 분야 파악**

"당신이 가장 깊이 아는 분야는?" — 이미 알고 있으면 스킵.

**Step 2: 기존 분야에서 구조적으로 같은 트레이드오프 탐색**

Phase 2의 근본 트레이드오프와 구조적으로 같은 결정이 사용자의 경험에 있었는지 확인.

**Step 3: 구조적 유사성 매핑 + 외부 사례 검증**

| 현재 트레이드오프 | 기존 경험의 유사 결정 | 그때 결과 | WebSearch 사례 |
|-------------------|---------------------|----------|---------------|
| ... | ... | ... | ... |

WebSearch는 매핑을 "검증"하는 데 사용. 성공/실패 사례 각 1개 이상.

완료 후 보고: "Phase 3 완료: N개 판례 매핑. 성공 사례 X건, 실패 사례 Y건."

산출물: `precedents.md`

### Phase 4: 전이 학습 (적용) → Decision Record

Phase 3의 판례를 현재 상황에 적용하여 최종 추천을 도출한다.

**Decision Record (ADR 스타일) 포함 항목:**

1. **결정 요약** — 무엇을 결정하는가 (한 줄)
2. **맥락** — 왜 이 결정이 필요한가
3. **선택지** — 각 선택지의 근본 트레이드오프 (Phase 2)
4. **판례** — 다른 맥락의 유사 결정 결과 (Phase 3)
5. **추천** — 최종 선택 + 근거
6. **되돌림 지표** — 이 결정이 틀렸다면 어떤 신호로 알 수 있는가
7. **되돌림 비용** — 틀렸을 때 바꾸는 데 드는 비용 (가역적/비가역적)

완료 후 보고: "Phase 4 완료: 추천 — {선택지}. 이유: {1줄}. 되돌림 지표: {1줄}."

산출물: `decision-record.md`

## Output Structure

```
~/.sblg/decide/
└── {결정슬러그}-{YYYYMMDD}/
    ├── _status.md
    ├── opinions.md
    ├── tradeoffs.md
    ├── precedents.md
    └── decision-record.md
```

## _status.md Format

```markdown
# Session Status
- decision: {결정 한 줄 요약}
- options: {A, B, ...}
- current_phase: {1~4}
- started_at: {날짜}
- last_updated: {날짜}
- expert_count: {N}
- real_tradeoffs: {N}
- variables_removed: {N}
- precedents_found: {N}
- recommendation: {미정 또는 선택지}
- next_action: {다음에 해야 할 구체적 작업}
```
