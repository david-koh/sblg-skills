# codex-plugin-cc 분석

## 1. 아키텍처 개요

codex-plugin-cc는 Claude Code에서 OpenAI Codex CLI를 호출하여 코드 리뷰, 작업 위임, 진단을 수행하는 프로덕션 플러그인이다. OpenAI가 직접 제작했다.

**디렉토리 구조:**

```
plugins/codex/
  .claude-plugin/plugin.json     # 플러그인 메타데이터
  agents/codex-rescue.md         # 서브에이전트 정의 (1개)
  commands/                      # 사용자 대면 슬래시 커맨드 (7개)
    rescue.md, review.md, adversarial-review.md,
    setup.md, status.md, result.md, cancel.md
  hooks/hooks.json               # 라이프사이클 훅 정의
  prompts/                       # 템플릿화된 프롬프트 (2개)
    adversarial-review.md, stop-review-gate.md
  schemas/                       # 출력 검증용 JSON Schema
    review-output.schema.json
  scripts/                       # Node.js 런타임 레이어
    codex-companion.mjs          # 메인 CLI 엔트리포인트
    session-lifecycle-hook.mjs   # SessionStart/End 훅 핸들러
    stop-review-gate-hook.mjs    # Stop 훅 핸들러
    lib/                         # 분리된 라이브러리 모듈 (12개)
  skills/                        # 내부 전용 스킬 (3개, 모두 user-invocable: false)
    codex-cli-runtime/SKILL.md
    codex-result-handling/SKILL.md
    gpt-5-4-prompting/SKILL.md
tests/                           # Node.js 테스트 스위트 (7개 파일)
```

**핵심 아키텍처 원칙: 3계층 분리**

1. **Command Layer** (markdown) — LLM이 읽는 지시문. "무엇을 할지"를 정의
2. **Script Layer** (Node.js) — 실제 실행 로직. `codex-companion.mjs`가 단일 진입점
3. **Skill Layer** (markdown) — LLM의 내부 행동 규범. 서브에이전트에만 노출

sblg-skills는 현재 Skill Layer만 존재하며 Command/Script Layer가 없다.

---

## 2. 잘 만들어진 핵심 패턴 (8개)

### 패턴 1: Command = 계약서 (Behavioral Contract)

각 command `.md` 파일은 LLM에 대한 법적 계약서처럼 작성되어 있다. 단순한 설명이 아니라 허용/금지 행위, 실행 조건 분기, 정확한 도구 호출 형식까지 명세한다.

**`commands/review.md:1-6` — YAML frontmatter로 도구 접근 제어:**
```yaml
disable-model-invocation: true
allowed-tools: Read, Glob, Grep, Bash(node:*), Bash(git:*), AskUserQuestion
```
`disable-model-invocation: true`는 LLM이 자체 판단으로 행동하는 것을 차단하고, `allowed-tools`로 사용 가능한 도구를 화이트리스트한다. 이 조합은 LLM이 "리뷰 결과를 보고 코드를 직접 고치는" 행동을 구조적으로 불가능하게 만든다.

**`commands/review.md:47-49` — 금지 행위 명시적 반복:**
```
- Do not fix issues, apply patches, or suggest that you are about to make changes.
- Your only job is to run the review and return Codex's output verbatim to the user.
```

**`commands/rescue.md:3-4` — context: fork로 서브에이전트 격리:**
```yaml
context: fork
allowed-tools: Bash(node:*), AskUserQuestion
```
`context: fork`는 서브에이전트가 메인 세션의 컨텍스트를 오염시키지 않도록 격리한다.

sblg-skills와의 차이: sblg-skills의 SKILL.md는 워크플로우를 설명하지만, 도구 접근 제어나 금지 행위를 구조적으로 강제하지 않는다.

### 패턴 2: Thin Forwarder 서브에이전트

`agents/codex-rescue.md:11`에서 서브에이전트의 역할을 한 문장으로 정의한다:

> "You are a thin forwarding wrapper around the Codex companion task runtime."

서브에이전트가 "생각"하거나 "분석"하는 것을 명시적으로 금지한다. `agents/codex-rescue.md:26-27`:
```
- Do not inspect the repository, read files, grep, monitor progress, poll status,
  fetch results, cancel jobs, summarize output, or do any follow-up work of your own.
```

유일하게 허용되는 지적 작업은 `gpt-5-4-prompting` 스킬을 사용한 프롬프트 리라이트뿐이다 (`agents/codex-rescue.md:24`). 이 패턴은 LLM이 "도움이 되려고" 지시 범위를 벗어나는 행동을 근본적으로 차단한다.

### 패턴 3: Skills = 내부 전용 행동 규범 (user-invocable: false)

세 개의 스킬 모두 `user-invocable: false`로 설정되어 사용자가 직접 호출할 수 없다.

- `codex-cli-runtime/SKILL.md:5` — `user-invocable: false`
- `codex-result-handling/SKILL.md:5` — `user-invocable: false`
- `gpt-5-4-prompting/SKILL.md:5` — `user-invocable: false`

이 스킬들은 서브에이전트의 "내부 지침서" 역할을 한다. `codex-cli-runtime`은 어떤 CLI 명령을 쓸지, `codex-result-handling`은 출력을 어떻게 표시할지, `gpt-5-4-prompting`은 프롬프트를 어떻게 작성할지를 규정한다.

특히 `gpt-5-4-prompting/SKILL.md:11`:
> "Prompt Codex like an operator, not a collaborator."

이 한 줄이 프롬프트 작성의 전체 철학을 압축한다. 참조 문서(`references/prompt-blocks.md`)에는 재사용 가능한 XML 블록 라이브러리가 있어 프롬프트를 조립식으로 구성한다.

### 패턴 4: JSON Schema로 출력 계약 강제

`schemas/review-output.schema.json`은 리뷰 출력의 정확한 형태를 JSON Schema 2020-12로 정의한다:

```json
"required": ["verdict", "summary", "findings", "next_steps"],
"verdict": { "enum": ["approve", "needs-attention"] },
"findings": { "items": { "required": ["severity", "title", "body", "file", "line_start", "line_end", "confidence", "recommendation"] } }
```

`additionalProperties: false`로 스키마에 정의되지 않은 필드를 차단한다 (`schemas/review-output.schema.json:3`). 각 finding에는 `confidence` (0~1 float)까지 포함하여 LLM이 자신의 확신도를 수치로 표현하도록 강제한다.

`render.mjs:24-41`의 `validateReviewResultShape()`가 런타임에서 이 스키마를 검증한다.

### 패턴 5: 3-Hook 라이프사이클 관리

`hooks/hooks.json`은 세 개의 라이프사이클 이벤트를 등록한다:

1. **SessionStart** (`session-lifecycle-hook.mjs:76-79`) — 세션 ID를 환경 변수로 주입하고, `CLAUDE_ENV_FILE`에 영속화한다. 이후 모든 job이 세션에 바인딩된다.

2. **SessionEnd** (`session-lifecycle-hook.mjs:81-112`) — 브로커 셧다운, 실행 중인 job의 프로세스 트리 종료, 세션 job 정리를 순차적으로 수행한다. `terminateProcessTree()`로 좀비 프로세스까지 처리한다.

3. **Stop** (`stop-review-gate-hook.mjs`) — 사용자가 세션을 종료하려 할 때 Codex를 호출하여 마지막 Claude 턴의 코드 변경을 자동 리뷰한다. `BLOCK`/`ALLOW` 이진 판정으로 세션 종료를 게이팅한다. 15분 타임아웃.

이 훅 시스템은 sblg-skills에 완전히 부재한다.

### 패턴 6: 상태 관리 — workspace-scoped + session-scoped 이중 격리

**Workspace 격리** (`state.mjs:29-44`): 워크스페이스 루트 경로의 SHA-256 해시 상위 16자로 고유 디렉토리를 생성한다:
```javascript
const hash = createHash("sha256").update(canonicalWorkspaceRoot).digest("hex").slice(0, 16);
return path.join(stateRoot, `${slug}-${hash}`);
```

**Session 격리** (`tracked-jobs.mjs:62`): `SESSION_ID_ENV`로 job을 세션에 바인딩하여, 같은 워크스페이스에서 여러 Claude 세션이 동시에 돌아도 서로의 job이 섞이지 않는다.

**Job 생명주기** (`tracked-jobs.mjs:142-204`): `runTrackedJob()`은 try/catch로 감싸서 성공이든 실패든 반드시 상태를 업데이트한다.

**자동 정리** (`state.mjs:80-84`): `pruneJobs()`가 50개 이상 쌓인 job을 자동 삭제하며, log 파일과 JSON 파일도 함께 제거한다.

**sblg-skills와의 근본적 차이:** sblg-skills는 `~/.sblg/{skill}/`에 markdown 파일로 상태를 저장한다. job 개념이 없어 병렬 실행 추적 불가, 세션 간 격리 없음, 자동 정리 없음, 프로그래매틱 조회 불가.

### 패턴 7: Prompt Template + Interpolation 시스템

`prompts/adversarial-review.md`는 `{{TARGET_LABEL}}`, `{{USER_FOCUS}}`, `{{REVIEW_INPUT}}` 플레이스홀더를 사용하며, `prompts.mjs:9-12`의 `interpolateTemplate()`가 치환한다:

```javascript
export function interpolateTemplate(template, variables) {
  return template.replace(/\{\{([A-Z_]+)\}\}/g, (_, key) => {
    return Object.prototype.hasOwnProperty.call(variables, key) ? variables[key] : "";
  });
}
```

프롬프트 자체도 XML 태그로 구조화 — `<role>`, `<task>`, `<operating_stance>`, `<attack_surface>`, `<structured_output_contract>`, `<grounding_rules>`, `<calibration_rules>`, `<final_check>`. 각 태그가 하나의 행동 규칙을 캡슐화한다.

### 패턴 8: Command 텍스트 자체를 테스트하는 테스트 스위트

`tests/commands.test.mjs`는 **커맨드 markdown의 텍스트 내용을 테스트한다**:

```javascript
test("review command uses AskUserQuestion and background Bash while staying review-only", () => {
  const source = read("commands/review.md");
  assert.match(source, /Do not fix issues/i);
  assert.match(source, /review-only/i);
  assert.match(source, /return Codex's output verbatim to the user/i);
});
```

이 패턴은 "누군가가 커맨드 파일을 수정할 때 핵심 안전 장치를 실수로 제거하는 것"을 CI에서 잡아낸다. LLM 행동 계약의 회귀 테스트다.

---

## 3. Skill 설계 철학

| 관점 | codex-plugin-cc | sblg-skills |
|------|----------------|-------------|
| **스킬의 역할** | 서브에이전트 내부 행동 규범 | 사용자 대면 워크플로우 전체 |
| **user-invocable** | 전부 `false` | 전부 (암묵적) `true` |
| **LLM 재량** | 최소화 — "이것만 해라" | 최대화 — 전문가 페르소나, 분석, 판단 모두 위임 |
| **출력 형식** | JSON Schema로 강제 | markdown 관행으로 암시 |
| **금지 행위** | 명시적, 반복적 | 일부 존재하나 구조적 강제 없음 |
| **참조 문서** | `references/` 하위 블록 라이브러리 | 없음 |

**효과적인 이유:**

1. **Negative instruction이 긍정 instruction보다 많다.** `codex-cli-runtime/SKILL.md`에서 "Do not" 문장이 10개 이상이다. 행동 공간을 좁히기 때문에 LLM은 "무엇을 해라"보다 "무엇을 하지 마라"에 더 잘 반응한다.

2. **스킬 간 조합이 명시적이다.** `agents/codex-rescue.md:6-8`에서 어떤 스킬을 조합해서 사용하는지 선언한다:
   ```yaml
   skills:
     - codex-cli-runtime
     - gpt-5-4-prompting
   ```

3. **"왜"보다 "어떻게"에 집중한다.** sblg-skills는 "왜 이 방법이 효과적인가"를 많이 설명하는 반면, codex-plugin-cc는 "이 상황에서 정확히 이 명령을 실행해라"라고 지시한다.

---

## 4. Hook / State / Session 관리

### 세션 라이프사이클 흐름

```
[SessionStart Hook]
    → session-lifecycle-hook.mjs: appendEnvVar(SESSION_ID_ENV, input.session_id)
    → 세션 ID가 환경변수로 모든 후속 스크립트에 전파

[사용자 작업: /codex:review, /codex:rescue, ...]
    → codex-companion.mjs → tracked-jobs.mjs
    → createJobRecord() — sessionId를 job에 바인딩
    → runTrackedJob() — 상태를 running → completed/failed로 전이
    → upsertJob() + writeJobFile() — 이중 기록

[Stop Hook] (stopReviewGate 활성화 시)
    → stop-review-gate-hook.mjs
    → ALLOW → 세션 종료 허용
    → BLOCK → 세션 종료 차단 + 이유 표시

[SessionEnd Hook]
    → sendBrokerShutdown() + cleanupSessionJobs() + teardownBrokerSession()
```

### 상태 저장 구조

```
$CLAUDE_PLUGIN_DATA/state/
└── {workspace-slug}-{sha256-16}/
    ├── state.json         # 전체 상태 인덱스
    └── jobs/
        ├── {job-id}.json  # 개별 job 상세
        └── {job-id}.log   # 실시간 진행 로그
```

---

## 5. sblg-skills에 적용 가능한 것들 (우선순위 순)

### P0: 구조적 금지 행위 강화 (노력: 낮음 / 영향: 높음)

**현재 문제:** `sblg-spot/SKILL.md:43-47`, `sblg-build/SKILL.md:44-49`에 "절대 하지 않는 것"이 있지만 모든 스킬에 일관되게 적용되지 않으며 구조적 강제가 없다.

**적용 방법:** 각 Phase 끝에 "이 Phase에서 절대 하지 않는 것" 목록 추가. 열거형 금지가 가장 효과적이다 (예: "Do not use Phase 1 diagnosis as the answer").

### P1: Command Layer 도입 (노력: 중간 / 영향: 높음)

**현재 문제:** SKILL.md가 사용자 인터페이스와 실행 로직을 모두 담는다. 하나의 파일이 너무 많은 책임을 진다.

**적용 방법:**
```
skills/sblg-solve/
  SKILL.md              # 내부 행동 규범
  commands/
    solve.md            # /sblg-solve 커맨드 정의
    continue.md         # continue 커맨드
    status.md           # status 커맨드
```
frontmatter로 도구 접근과 실행 컨텍스트를 제어할 수 있다.

### P2: _status.md를 JSON으로 전환 (노력: 낮음 / 영향: 중간)

**현재 문제:** `_status.md`는 markdown YAML-like 포맷. 프로그래매틱 파싱이 어렵고 LLM이 형식을 깨뜨릴 수 있다.

**적용 방법:** `state.mjs:58-77`의 패턴으로 `_state.json` 전환. 기본값 병합으로 부분 손상에도 안전하게.

### P3: Phase 전이에 검증 게이트 추가 (노력: 중간 / 영향: 높음)

**현재 문제:** Phase 간 전이가 LLM 재량. Phase 2를 대충 끝내고 Phase 3으로 넘어가는 것을 막을 방법이 없다.

**적용 방법:** ALLOW/BLOCK 이진 판정으로 Phase 전이를 게이팅. 예:
- sblg-solve Phase 2: `constants_count > 0 && variables_removed > 0 && root_bottleneck != ""`
- sblg-spot Phase 4: `leaks_verified >= 5 && 각 leak에 2개 이상 독립 출처`

### P4: 출력 계약을 Schema로 강제 (노력: 중간 / 영향: 중간)

산출물 형식을 JSON Schema로 정의. 특히 `verdict.md`의 누수 지점 형식, `decision-record.md`의 필수 항목.

### P5: 커맨드 텍스트 회귀 테스트 (노력: 낮음 / 영향: 중간)

```javascript
test("sblg-solve SKILL requires expert personas from adjacent domains", () => {
  const source = fs.readFileSync("skills/sblg-solve/SKILL.md", "utf8");
  assert.match(source, /인접 분야/);
  assert.match(source, /병렬 subagent/);
  assert.match(source, /상수.*변수/);
});
```

### P6: 스킬 간 의존성을 frontmatter에 선언 (노력: 낮음 / 영향: 중간)

`sblg-build/SKILL.md:56-57`에 sblg-solve 의존성이 암묵적으로만 있다. frontmatter에서 명시적으로:
```yaml
dependencies:
  - sblg-solve   # prototype.md 또는 solutions.md 참조
```

### P7: Prompt Template 시스템 도입 (노력: 높음 / 영향: 높음)

전문가 페르소나 생성, 소크라테스식 심문, 구조적 유사성 매핑 패턴이 5개 SKILL.md에 반복 inline되어 있다. `prompts/` 디렉토리로 공통 블록 모듈화.

---

## 참조 파일

- `codex-plugin-cc/plugins/codex/commands/review.md:1-6` — YAML frontmatter 도구 접근 제어
- `codex-plugin-cc/plugins/codex/agents/codex-rescue.md:11` — thin forwarder 패턴
- `codex-plugin-cc/plugins/codex/skills/gpt-5-4-prompting/SKILL.md:11` — "operator, not collaborator"
- `codex-plugin-cc/plugins/codex/schemas/review-output.schema.json:3` — additionalProperties: false
- `codex-plugin-cc/plugins/codex/hooks/hooks.json` — 3-hook 라이프사이클
- `codex-plugin-cc/plugins/codex/scripts/lib/state.mjs:29-44` — workspace-scoped 상태
- `codex-plugin-cc/plugins/codex/scripts/lib/state.mjs:80-84` — 자동 job pruning
- `codex-plugin-cc/plugins/codex/scripts/lib/prompts.mjs:4-13` — 템플릿 시스템
- `codex-plugin-cc/plugins/codex/scripts/stop-review-gate-hook.mjs:69-96` — ALLOW/BLOCK 판정
- `codex-plugin-cc/tests/commands.test.mjs` — 커맨드 텍스트 회귀 테스트
