# gstack 분석

## 1. 아키텍처 개요

gstack은 Claude Code 위에 동작하는 **SKILL.md 기반 워크플로우 엔진 + 영속 브라우저 데몬**이다. 30개 이상의 스킬이 하나의 모노레포에서 관리되며, 각 스킬은 독립된 디렉토리에 `SKILL.md.tmpl` (소스) + `SKILL.md` (생성물)로 존재한다.

### 핵심 구성 요소

```
gstack/
├── browse/           # Playwright 기반 영속 브라우저 데몬 (Bun 컴파일 바이너리)
├── design/           # GPT Image API 기반 디자인 바이너리
├── scripts/
│   ├── gen-skill-docs.ts      # .tmpl → SKILL.md 생성기
│   └── resolvers/             # 플레이스홀더별 리졸버 모듈
├── test/
│   ├── helpers/
│   │   ├── skill-parser.ts    # $B 명령어 추출 + 검증
│   │   ├── session-runner.ts  # claude -p로 E2E 세션 실행
│   │   ├── llm-judge.ts       # Sonnet으로 SKILL.md 품질 평가
│   │   └── eval-store.ts      # 증분 결과 저장
│   ├── skill-parser.test.ts        # Tier 1: 정적 검증
│   ├── skill-llm-eval.test.ts      # Tier 3: LLM-as-judge
│   └── skill-e2e-*.test.ts         # Tier 2: 실제 Claude 세션
├── bin/              # CLI 유틸리티 (gstack-config, gstack-slug, gstack-learnings-*)
└── [30+ skill dirs]  # 각각 SKILL.md.tmpl + SKILL.md
```

**가장 중요한 아키텍처 결정**: 브라우저를 MCP가 아닌 CLI 바이너리로 구현했다. 컴파일된 58MB 바이너리가 localhost HTTP로 Chromium 데몬과 통신한다. MCP 대비 명령당 0 토큰 오버헤드 (MCP 30,000-40,000 토큰 vs gstack 0 토큰 per 20-command session, `BROWSER.md:308-310`).

---

## 2. 잘 만들어진 핵심 패턴 (7개)

### 패턴 1: Template → Generated SKILL.md 파이프라인

**파일**: `scripts/gen-skill-docs.ts`, 모든 `SKILL.md.tmpl` 파일

`.tmpl` 파일에 인간이 워크플로우 로직을 작성하면, `gen-skill-docs.ts`가 소스 코드에서 메타데이터를 추출하여 `{{PLACEHOLDER}}`를 치환한다.

`ARCHITECTURE.md:186-211`에 문서화된 11개 플레이스홀더:
- `{{COMMAND_REFERENCE}}` — `commands.ts`에서 명령어 테이블 생성
- `{{SNAPSHOT_FLAGS}}` — `snapshot.ts`에서 플래그 레퍼런스 생성
- `{{PREAMBLE}}` — 업데이트 체크, 세션 추적, 학습 로드 등 공통 부트스트랩
- `{{BROWSE_SETUP}}` — 바이너리 탐색 + 셋업 안내
- `{{BASE_BRANCH_DETECT}}` — PR 대상 브랜치 동적 탐지

**왜 효과적인가**: 코드에 명령어가 추가되면 문서에 자동 반영된다. CI 검증: `gen:skill-docs --dry-run` + `git diff --exit-code`로 stale 문서를 머지 전에 잡는다 (`ARCHITECTURE.md:228-229`).

**sblg-skills와의 차이**: sblg-skills는 SKILL.md를 수동으로 직접 작성한다. 공통 섹션(Session Continuity, Web Search, Commands, Output Structure)이 5개 스킬에 거의 동일하게 반복되지만 중앙화되어 있지 않다.

### 패턴 2: Preamble — 모든 스킬의 공통 부트스트랩

**파일**: 모든 생성된 `SKILL.md`의 `## Preamble (run first)` 섹션 (`checkpoint/SKILL.md:27-87`)

단일 bash 블록에서 5가지를 처리한다:
1. **업데이트 체크** — `gstack-update-check` 호출
2. **세션 추적** — `~/.gstack/sessions/$PPID` 터치, 3+ 세션 시 ELI16 모드
3. **학습 로드** — `learnings.jsonl`에서 프로젝트별 학습 검색
4. **브랜치/모드 탐지** — 현재 브랜치, 레포 모드, 프로액티브 설정
5. **텔레메트리** — 옵트인 사용량 기록

한 번 작성하면 `{{PREAMBLE}}`으로 모든 스킬에 주입된다. 스킬 작성자는 워크플로우 로직에만 집중할 수 있다.

### 패턴 3: Hooks를 활용한 런타임 안전장치

**파일**: `careful/SKILL.md.tmpl:13-19`, `careful/bin/check-careful.sh`

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-careful.sh"
```

`check-careful.sh:1-112`는 stdin에서 JSON을 읽고, 파괴적 패턴(`rm -rf`, `DROP TABLE`, `git push --force` 등)을 감지하면 `{"permissionDecision":"ask","message":"..."}` 를 반환한다. 안전한 예외(node_modules, dist 등)는 자동 허용된다 (`check-careful.sh:29-48`).

**왜 효과적인가**: 프롬프트 레벨이 아닌 런타임 레벨에서 안전을 강제한다. LLM이 "나는 이런 걸 하지 않겠다"고 약속하는 게 아니라, 실행 자체가 차단된다.

### 패턴 4: 3-Tier 테스트 피라미드

**파일**: `package.json:16-23`, `test/helpers/skill-parser.ts`, `test/skill-e2e-*.test.ts`

| Tier | 무엇을 테스트 | 비용 | 속도 |
|------|-------------|------|------|
| 1 | `$B` 명령어 파싱, 레지스트리 대조, 스냅샷 플래그 검증 | 무료 | <5s |
| 2 | `claude -p`로 실제 세션 스폰, 스킬 실행 | ~$3.85 | ~20min |
| 3 | Sonnet이 SKILL.md의 명확성/완성도/실행가능성 채점 | ~$0.15 | ~30s |

**Diff 기반 테스트 선택** (`CLAUDE.md:33-43`): 각 E2E 테스트가 `touchfiles.ts`에 파일 의존성을 선언한다. `git diff`로 변경된 파일만 관련 테스트를 실행하여 비용을 최소화한다.

### 패턴 5: Operational Self-Improvement (세션 학습)

**파일**: 모든 preamble의 `## Operational Self-Improvement` 섹션 (`checkpoint/SKILL.md:358-370`), `learn/SKILL.md.tmpl`

매 스킬 세션 종료 시:
- "예상치 못하게 실패한 명령이 있었는가?"
- "잘못된 접근을 택하고 되돌린 적이 있는가?"
- "프로젝트 특유의 quirk을 발견했는가?"

발견하면 `gstack-learnings-log`로 JSONL에 기록한다. 다음 세션에서 preamble이 `gstack-learnings-search --limit 3`으로 관련 학습을 자동 로드한다 (`checkpoint/SKILL.md:66-73`).

`/learn` 스킬(`learn/SKILL.md.tmpl:24-192`)은 학습 관리 전용 인터페이스: 검색, 가지치기(stale 학습 제거, 모순 감지), 마크다운 내보내기, 통계.

### 패턴 6: 다중 모델 독립 검증 + Consensus Table

**파일**: `autoplan/SKILL.md.tmpl:236-307`

CEO/Design/Eng 3단계 리뷰에서 각각 Claude subagent + Codex를 **독립적으로** 실행한다:
- Claude subagent: 이전 리뷰 컨텍스트 **없이** 독립 평가
- Codex: 이전 단계의 findings를 **포함**하여 누적 검증
- 두 모델의 결과를 Consensus Table로 합산: CONFIRMED / DISAGREE

`DISAGREE`는 taste decision으로 분류되고 최종 Approval Gate에서 사용자에게 표시된다.

**User Challenge 패턴** (`autoplan/SKILL.md.tmpl:76-95`): 두 모델이 모두 사용자의 명시적 방향과 다른 의견일 때, 절대 자동 결정하지 않고 사용자에게 돌려보낸다.

### 패턴 7: AskUserQuestion 표준 포맷

**파일**: 모든 preamble의 `## AskUserQuestion Format` (`checkpoint/SKILL.md:306-314`)

4단계 구조를 강제한다:
1. **Re-ground**: 프로젝트, 현재 브랜치, 현재 작업 명시
2. **Simplify**: 16세가 이해할 수 있는 평문으로 설명
3. **Recommend**: `RECOMMENDATION: Choose [X] because [...]` + `Completeness: X/10`
4. **Options**: A) B) C) 레터 옵션, 노력 추정 포함

"사용자가 이 창을 20분 동안 안 봤다고 가정하라" (`checkpoint/SKILL.md:312`) — 모든 AskUserQuestion이 자기 완결적이어야 한다.

---

## 3. Skill 설계 철학 (SKILL.md 작성 원칙)

### 3.1 Template vs Generated 분리

`.tmpl`을 편집하고 `gen:skill-docs`를 실행한다. `.tmpl`과 `.md`를 모두 커밋한다. 머지 충돌 시 **절대** generated 파일의 어느 쪽도 수용하지 않는다 — `.tmpl`의 충돌을 해결하고 재생성한다 (`CLAUDE.md:119-132`).

### 3.2 Bash 블록은 자기 완결적

`CLAUDE.md:148-159`:
> SKILL.md.tmpl 파일은 bash 스크립트가 아니라 **Claude가 읽는 프롬프트 템플릿**이다. 각 bash 코드 블록은 별도 셸에서 실행된다 — **변수가 블록 간에 유지되지 않는다.**

규칙:
- 상태 전달은 자연어로 한다 ("Phase 0에서 감지한 base branch를 사용")
- 조건문은 영어로 표현한다 (bash if/elif 대신 번호 매긴 결정 단계)
- 브랜치 이름을 하드코딩하지 않는다 — `{{BASE_BRANCH_DETECT}}`로 동적 탐지

### 3.3 역할 부여 + Hard Gate

각 스킬은 명시적 역할과 "절대 하지 않는 것"을 최상단에 선언한다:
- `/checkpoint`: "Staff Engineer who keeps meticulous session notes" + "HARD GATE: Do NOT implement code changes"
- `/investigate`: "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
- `/cso`: "Think like an attacker, report like a defender. You do NOT make code changes."
- `/learn`: "HARD GATE: Do NOT implement code changes. This skill manages learnings only."

**"절대 하지 않는 것"은 스킬 중간이 아니라 최상단에 위치한다.** 프롬프트에서 금지 사항은 가능한 한 일찍, 가능한 한 강하게.

### 3.4 구체적 산출물 명세

스킬이 "리뷰하라"가 아니라 "이 테이블을 출력하라"를 지시한다:
- `/cso`는 `ATTACK SURFACE MAP` 테이블의 정확한 형식을 정의 (`cso/SKILL.md.tmpl:124-143`)
- `/autoplan`은 `CEO DUAL VOICES — CONSENSUS TABLE`의 정확한 ASCII 형식 정의
- `/investigate`는 `DEBUG REPORT`의 형식 정의

### 3.5 Completion Status Protocol

모든 스킬이 4가지 종료 상태 중 하나를 보고한다 (`checkpoint/SKILL.md:336-339`):
- **DONE** — 모든 단계 완료, 증거 제공
- **DONE_WITH_CONCERNS** — 완료, 사용자 주의 사항 있음
- **BLOCKED** — 진행 불가, 시도한 것과 막힌 것 명시
- **NEEDS_CONTEXT** — 필요한 정보 명시

3번 실패하면 STOP하고 에스컬레이트한다.

### 3.6 에러 메시지는 AI Agent용

`ARCHITECTURE.md:264-268`:
> 에러는 AI 에이전트용이다. 모든 에러 메시지가 actionable해야 한다:
> "Element not found" → "Element not found or not interactable. Run `snapshot -i` to see available elements."

---

## 4. Session Intelligence / State 관리

### 4.1 프로젝트별 JSONL 파일 시스템

```
~/.gstack/
├── config.yaml                    # 글로벌 설정
├── sessions/                      # 활성 세션 추적 ($PPID 기반)
├── analytics/
│   └── skill-usage.jsonl          # 스킬별 사용 로그
└── projects/
    └── {slug}/
        ├── learnings.jsonl        # 프로젝트별 학습 (append-only)
        ├── timeline.jsonl         # 세션 타임라인
        ├── checkpoints/           # /checkpoint 저장 파일
        ├── {branch}-reviews.jsonl # 브랜치별 리뷰 로그
        └── security-reports/      # /cso 리포트
```

### 4.2 Context Recovery (컴팩션 생존)

`checkpoint/SKILL.md:262-301`의 Context Recovery 섹션.

세션 시작 또는 컴팩션 후 자동으로:
1. 최근 3개 아티팩트 검색
2. 현재 브랜치의 리뷰 수 확인
3. 타임라인에서 마지막 5개 이벤트 읽기
4. 마지막 완료된 세션의 스킬/결과 표시
5. **최근 스킬 패턴 분석**: "review,ship,review" 같은 반복 패턴이면 다음 스킬 제안

Welcome back 메시지: "Welcome back to {branch}. Last session: /{skill} ({outcome}). [Checkpoint summary]."

### 4.3 Cross-session Learnings

Preamble에서 자동 로드 (`checkpoint/SKILL.md:62-73`):
```bash
_LEARN_FILE="${GSTACK_HOME:-$HOME/.gstack}/projects/${SLUG:-unknown}/learnings.jsonl"
if [ -f "$_LEARN_FILE" ]; then
  _LEARN_COUNT=$(wc -l < "$_LEARN_FILE" | tr -d ' ')
  if [ "$_LEARN_COUNT" -gt 5 ]; then
    ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 3
  fi
fi
```

5개 이상의 학습이 누적되면 상위 3개를 자동으로 표시한다. 학습은 `{skill, type, key, insight, confidence, source, files}` 구조의 JSONL이다.

### 4.4 Multi-session Awareness

3+ 세션이 실행 중이면, 모든 스킬이 "ELI16 mode"에 진입 — 모든 질문에서 컨텍스트를 다시 설명한다 (`ARCHITECTURE.md:218-219`).

세션 추적은 `~/.gstack/sessions/` 디렉토리의 파일 mtime으로 구현된다 (2시간 이상 된 파일은 자동 정리).

---

## 5. sblg-skills에 적용 가능한 것들 (우선순위 순)

### P0 (즉시 도입) — 구조적 결함 해결

**1. 공통 섹션을 Preamble bash 스크립트로 추출**

현재 sblg-skills의 5개 SKILL.md에서 다음 섹션이 거의 동일하게 반복된다:
- `## Commands` 테이블
- `## Session Continuity` (가장 최근 세션 탐색 + `_status.md` 읽기)
- `## Web Search` (검색 도구 가용성 체크)
- `## Output Structure` (디렉토리 구조)
- `## _status.md Format`

**구체적 행동**: 공통 bash 블록을 `~/.sblg/bin/sblg-preamble.sh`로 추출. 각 SKILL.md에서 한 줄로 호출. 템플릿 시스템까지 필요 없이, **공통 bash 스크립트 + source**만으로도 유지보수 부담이 1/5로 줄어든다.

**2. Completion Status Protocol 도입**

sblg-skills의 각 Phase가 "완료 후 보고" 형식을 이미 사용하고 있지만 (`sblg-solve/SKILL.md:56`, `sblg-spot/SKILL.md:77`), 최종 종료 시 전체 워크플로우 상태를 보고하는 표준이 없다.

DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT 4종 상태 도입. 특히 `continue`로 재개할 때 이전 세션이 왜 중단되었는지 알 수 있게 된다.

**3. Hard Gate를 SKILL.md 최상단으로 이동**

현재 sblg-skills에 `## 절대 하지 않는 것` 섹션이 있지만 (`sblg-build/SKILL.md:46-50`, `sblg-spot/SKILL.md:43-47`), 스킬 중간에 위치한다. gstack처럼 스킬 최상단에 **HARD GATE**로 선언해야 효과적이다.

### P1 (높은 가치) — 품질 개선

**4. AskUserQuestion 표준 포맷 도입**

현재 sblg-skills의 AskUserQuestion이 각 스킬마다 자유 형식이다. gstack의 4단계 포맷(Re-ground, Simplify, Recommend, Options)을 채택하면 UX가 일관된다.

"사용자가 20분 동안 안 봤다고 가정하라"는 전제는 sblg-skills처럼 긴 워크플로우(5 Phase, 30분+)에서 특히 중요하다.

**5. Cross-session Learnings 시스템**

sblg-skills의 `_status.md`는 세션 상태만 기록하고, **프로젝트에서 발견한 패턴**은 기록하지 않는다. `learnings.jsonl` 패턴을 도입하면:
- `/sblg-solve`가 발견한 "이 가정은 관행이었다"를 다음 세션에서 자동 참조
- `/sblg-spot`이 발견한 도메인별 지식을 `/sblg-build`가 활용
- 스킬 간 지식 전이가 파일 시스템을 통해 자연스럽게 발생

**6. 스킬 간 라우팅 규칙**

sblg-skills 5개 스킬 간 자연스러운 흐름이 있다 (spot → solve → build, decide는 독립, learn은 횡단). 이를 루트 또는 각 스킬 끝에서 명시적으로 라우팅하면 사용자가 수동으로 다음 스킬을 기억할 필요가 없다.

### P2 (중기) — 차별화

**7. 테스트 인프라 도입**

바로 적용 가능한 것은 **Tier 1 (정적 검증)**:
- 각 SKILL.md의 bash 블록 파싱
- `_status.md` 포맷의 필드 완결성 검증
- Phase 번호 연속성 검증
- 스킬 간 참조 정합성 검증 (sblg-solve가 sblg-spot 산출물을 올바른 경로에서 읽는지)

Tier 2 (E2E)와 Tier 3 (LLM judge)는 스킬이 10개 이상 늘어난 후 도입해도 된다.

**8. Operational Self-Improvement 루틴**

각 스킬 세션 종료 시 3가지 자기 점검 질문 추가:
- "이번 Phase에서 예상치 못한 패턴을 발견했는가?"
- "제거한 가정 중 다른 프로젝트에도 적용될 것이 있는가?"
- "Phase 전이 시 막힌 지점이 있었는가?"

발견하면 `~/.sblg/learnings.jsonl`에 기록. 다음 세션에서 preamble이 자동 로드.

---

## Trade-offs

| 적용 항목 | 장점 | 비용/리스크 |
|-----------|------|------------|
| Preamble 추출 | 유지보수 1/5, 일관성 보장 | bin 스크립트 관리 오버헤드 추가 |
| Template 시스템 | 코드-문서 드리프트 제거 | gen-skill-docs 같은 빌드 스텝 필요 |
| Learnings JSONL | 세션 간 지식 전이 | JSONL 파싱/검색 유틸리티 개발 필요 |
| AskUserQuestion 표준 | UX 일관성, 컨텍스트 복구 개선 | 기존 질문 모두 리팩토링 |
| Completion Protocol | continue 재개 신뢰성 향상 | _status.md 포맷 변경 필요 |
| 테스트 인프라 | SKILL.md 품질 보증 | 테스트 코드 작성 + CI 설정 |

---

## 참조 파일

- `gstack/ARCHITECTURE.md:186-242` — Template system 설계와 placeholder 목록
- `gstack/ARCHITECTURE.md:215-241` — 3-tier 테스트 구조
- `gstack/ARCHITECTURE.md:264-268` — Error philosophy (AI agent용 에러)
- `gstack/CLAUDE.md:119-159` — SKILL.md 워크플로우 및 템플릿 작성 규칙
- `gstack/ETHOS.md` — Boil the Lake, Search Before Building, User Sovereignty
- `gstack/careful/bin/check-careful.sh:1-112` — Hook 기반 안전장치 구현
- `gstack/careful/SKILL.md.tmpl:13-19` — Hook frontmatter 선언
- `gstack/autoplan/SKILL.md.tmpl:44-95` — 6 Decision Principles + User Challenge 패턴
- `gstack/checkpoint/SKILL.md:262-301` — Context Recovery 메커니즘
- `gstack/checkpoint/SKILL.md:306-314` — AskUserQuestion 표준 포맷
- `gstack/checkpoint/SKILL.md:336-370` — Completion Status + Escalation + Self-Improvement
- `gstack/investigate/SKILL.md.tmpl:42-203` — Iron Law + 3-strike + scope lock
- `gstack/cso/SKILL.md.tmpl:26-627` — 14-phase 보안 감사 (가장 정교한 단일 스킬)
- `gstack/learn/SKILL.md.tmpl:24-192` — 학습 관리 전용 스킬
- `gstack/test/helpers/skill-parser.ts:35-134` — $B 명령어 추출 및 검증
- `gstack/scripts/gen-skill-docs.ts` — 템플릿 생성기
