---
name: task-pipeline
description: 통합 오케스트레이션 스킬 — 테마 루프 + 판단 + 요구사항서 + Agent 위임 + 재시도 + 저장
origin: docs-automation
tools: [Read, Write, Grep, Glob, Bash, Agent]
---

<Purpose>
theme-definitions의 테마를 순차 순회하며, 테마마다:
(1) git diff/코드 분석으로 문서화 필요 여부를 판단하고 요구사항서를 작성
(2) Agent(docu-writer)에 md 작성을 위임
(3) Agent(critic)에 검증을 위임
(4) 재시도/저장/execution-log 기록을 수행한다.

Main CLI가 직접 읽고 따르는 유일한 오케스트레이션 스킬.
</Purpose>

<Use_When>
- 코드베이스 대상으로 Docusaurus 문서를 자동 생성할 때
- git push CI 트리거 또는 수동 실행 시
</Use_When>

<Do_Not_Use_When>
- .md 파일을 직접 작성/검증할 때 -> docu-writer-skill, critic-skill
- 테마 정의를 참조할 때 -> theme-definitions/SKILL.md 직접 Read
</Do_Not_Use_When>

<Why_This_Exists>
1. **서브 에이전트 중첩 제약 해소**: Claude Code에서 서브 에이전트가 서브 에이전트를 생성할 수 없으므로, Main이 오케스트레이션을 직접 수행하고 scout/docu-writer/critic을 Level 1 Agent로 호출
2. **구조 단순화**: v3의 3단계(Main -> task-orchestrator -> docu-writer/critic)에서 v4의 2단계(Main -> scout/docu-writer/critic)로 평탄화
3. **Full Reset 유지**: 매 테마마다 새 scout/docu-writer/critic 생성으로 컨텍스트 오염 방지, 비용 효율(1.0x)
5. **Main 컨텍스트 보호**: 파일 기반 I/O로 에이전트 간 데이터를 `.pipeline-temp/` 파일로 주고받고, Main은 경로만 전달. 1테마당 ~340토큰 (텍스트 릴레이 대비 ~10배 절감).
4. **역할 분리 유지**: Main은 판단+오케스트레이션, docu-writer는 md 작성 전문, critic은 검증 전문
</Why_This_Exists>

<Execution_Policy>
**CRITICAL 제약 8개** -- 위반 시 파이프라인 오동작:

```
CRITICAL 1: .md 내용을 직접 작성하지 말 것
            -> 반드시 Agent(docu-writer)에 위임
            -> Write 도구로 직접 .md 본문을 생성하지 말 것
            -> docu-writer가 draft_path에 Write. Main은 PASS 시 cp, WARNING 시 frontmatter 수정 후 Write
CRITICAL 2: 검증을 직접 수행하지 말 것
            -> 반드시 Agent(critic)에 위임
            -> "내용이 적절해 보인다" 같은 직접 판단 금지
CRITICAL 3: docu-writer 결과 수신 전 critic 호출 금지 (순차)
            -> critic은 draft_path의 파일이 있어야 검증 가능
CRITICAL 4: 테마 순회는 반드시 순차 (병렬 금지)
            -> 현재 테마의 전체 사이클(판단->작성->검증->저장)이 완료된 후 다음 테마 진행
CRITICAL 5: 테마마다 새 scout/docu-writer/critic 생성 (Full Reset)
            -> 이전 에이전트 resume 금지
            -> 재시도 시에도 새 에이전트 생성
CRITICAL 6: 재시도 최대 2회 Hard Cap
            -> 2회 초과 시 auto_generated_warning 태그 부착 후 WARNING으로 저장
CRITICAL 7: 3연속 FAIL 시 파이프라인 중단
            -> PASS/SKIPPED/WARNING 발생 시 consecutive_fail 초기화
CRITICAL 8: Step A의 Glob/Grep을 Main이 직접 수행 금지
            -> 반드시 Agent(scout)에 위임
            -> Main 컨텍스트에 Glob/Grep 결과가 적재되면 테마 확장 시 오토 컴팩트 발생
```
</Execution_Policy>

<Steps>

## Phase 1: 입력 확인

실행 시 전달받는 파라미터:
- `target`: 대상 코드베이스 경로

#### 1-1. target 경로 존재 확인

target 디렉토리가 존재하는지 확인한다. 없으면 즉시 중단.

#### 1-2. last_commit 확보

Bash로 target이 독립 git repo인지 확인 후 커밋 해시를 조회:

```bash
cd {target} && [ -d .git ] && git log -1 --format=%h 2>/dev/null || echo "non-git"
```

- 독립 git repo이면 (`.git` 디렉토리 존재): 해당 repo의 최신 커밋 해시 (7자)
- 독립 git repo가 아니면 (`.git` 없음 — 하위 디렉토리, git 미초기화 등): `"non-git"`

> **주의**: `git rev-parse --is-inside-work-tree`는 부모 repo의 working tree 내부에서도 `true`를 반환하므로, 반드시 `[ -d .git ]`로 target 자체가 git root인지 확인해야 한다.

결과를 `last_commit` 변수에 저장한다. 이후 모든 테마의 요구사항서에 동일하게 전달.

#### 1-3. generated_at 확보

Bash로 현재 시각을 ISO-8601 형식으로 조회:

```bash
date -u +%Y-%m-%dT%H:%M:%S 2>/dev/null || echo "unknown"
```

결과를 `generated_at` 변수에 저장한다. 이후 모든 테마의 요구사항서에 동일하게 전달.

#### 1-4. 임시 디렉토리 생성

```bash
mkdir -p {target}/.pipeline-temp
```

docu-writer가 md 드래프트를 저장할 임시 디렉토리. Phase 3에서 정리.

## Phase 2: 테마 루프

#### 상태 복구 (오토 컴팩트 대비)

테마 처리 시작 전 `{target}/execution-log.md`가 존재하면 Read하여 YAML 상태 헤더를 파싱:
- 이미 `completed_themes`에 포함된 테마는 건너뜀
- `consecutive_fail` 값 복구
- execution-log.md가 없거나 YAML 헤더가 없으면 처음부터 시작 (하위 호환)

아래 4개 테마를 정의된 순서대로 **순차** 처리한다 (병렬 금지).

> **테마 추가 시**: 아래 리스트에 항목 추가 + `skills/theme-definitions/themes/{normalized_theme}.yaml` 파일 생성

```yaml
themes:
  - { theme: "intro", section: "intro", output_path: "docs/intro.md" }
  - { theme: "getting-started/requirements", section: "getting-started", output_path: "docs/getting-started/requirements.md" }
  - { theme: "architecture/overview", section: "architecture", output_path: "docs/architecture/overview.md" }
  - { theme: "architecture/component-diagram", section: "architecture", output_path: "docs/architecture/component-diagram.md" }
```

테마마다 아래 Step A ~ Step D를 실행한다. `retry_count`를 0으로 초기화.

---

### Step A: 판단 + 요구사항서 작성 (Scout Agent 호출)

Agent 도구로 `agents/scout.md` 기반 에이전트를 **새로** 생성한다.

> **WHY**: Glob/Grep 결과가 Main 컨텍스트에 직접 적재되면 테마당 ~5,000 토큰을 소비한다. scout가 내부에서 소화하고 Main에는 완성된 결과만 반환하여 ~200 토큰으로 축소. (CRITICAL 8)

**리터럴 Agent 호출 템플릿** -- 아래 형태를 그대로 사용:

```
Agent 도구 호출:
  description: "scout {theme}"
  subagent_type: "general-purpose"
  prompt: |
    너는 scout 에이전트다.
    agents/scout.md를 읽고 역할을 확인하라.

    [테마 정보]
    theme: "{theme}"
    section: "{section}"
    target: "{target}"
    last_commit: "{last_commit}"
    generated_at: "{generated_at}"
    output_path: "{output_path}"
    draft_path: "{draft_path}"
    req_path: "{target}/.pipeline-temp/req-{normalized_theme}.yaml"
    theme_def_path: "{target}/.pipeline-temp/theme-def-{normalized_theme}.yaml"

    절차:
    1. skills/theme-definitions/themes/{normalized_theme}.yaml을 Read하여 테마 정의를 로드하라.
    2. target 코드베이스를 Glob/Grep으로 탐색하여 관련 파일 경로를 수집하라 (코드 본문 Read 금지).
    3. 문서화 필요 여부를 판단하라.
    4. proceed면 요구사항서를 req_path에 Write, 테마 정의를 theme_def_path에 Write 저장하고, status만 반환하라.
       skipped면 사유를 반환하라.
```

> `normalized_theme`: theme ID의 `/`를 `--`로 치환. 예: `architecture/overview` → `architecture--overview`

**status를 수신하여 분기**:

- **status == "proceed"**: Step B 진행. (요구사항서는 req_path에, 테마 정의는 theme_def_path에 이미 저장됨)
- **status == "skipped"**: execution-log에 SKIPPED 기록. 다음 테마.
- **status == "fail"**: execution-log에 FAIL 기록. consecutive_fail++.

---

### Step B: md 작성 (docu-writer Agent 호출)

Agent 도구로 `agents/docu-writer.md` 기반 에이전트를 **새로** 생성한다.

**리터럴 Agent 호출 템플릿** -- 아래 형태를 그대로 사용:

```
Agent 도구 호출:
  description: "docu-writer {theme}"
  subagent_type: "general-purpose"
  prompt: |
    너는 docu-writer 에이전트다.
    agents/docu-writer.md를 읽고 역할을 확인하라.
    skills/docu-writer-skill/SKILL.md를 읽고 절차를 따르라.

    [요구사항서 경로]
    req_path: "{target}/.pipeline-temp/req-{normalized_theme}.yaml"
    이 파일을 Read하여 요구사항서(14필드)를 로드하라.

    [대상 코드베이스]
    target: "{target}"
    요구사항서의 related_files 경로를 기준으로 대상 코드베이스에서 직접 Read/Grep/Glob하여 코드를 분석하라.

    [드래프트 저장 경로]
    draft_path: "{target}/.pipeline-temp/draft-{normalized_theme}.md"
    작성한 md를 이 경로에 Write로 저장하라. md 전문을 문자열로 반환하지 말 것.

    {재시도 시에만 추가:
    [critic 피드백 경로]
    feedback_path: "{target}/.pipeline-temp/feedback-{normalized_theme}.yaml"
    이 파일을 Read하여 critic 피드백을 로드하고 지적사항을 반영하라.}

    작업 완료 후 경량 확인서를 반환하라:
    status: "done"
    draft_path: "{draft_path}"
```

경량 확인서를 수신한다. Main은 다음을 검증:
1. status가 "done"인지 확인 (아니면 STATUS=FAIL)
2. Bash로 `[ -f {draft_path} ]` 파일 존재 확인 (없으면 STATUS=FAIL)

---

### Step C: 검증 (critic Agent 호출)

Agent 도구로 `agents/critic.md` 기반 에이전트를 **새로** 생성한다.

**리터럴 Agent 호출 템플릿** -- 아래 형태를 그대로 사용:

```
Agent 도구 호출:
  description: "critic {theme}"
  subagent_type: "general-purpose"
  prompt: |
    너는 critic 에이전트다.
    agents/critic.md를 읽고 역할을 확인하라.
    skills/critic-skill/SKILL.md를 읽고 절차를 따르라.

    [드래프트 파일 경로]
    draft_path: "{target}/.pipeline-temp/draft-{normalized_theme}.md"
    이 경로의 파일을 Read로 직접 로드하여 검증하라.

    [테마 정의 경로]
    theme_def_path: "{target}/.pipeline-temp/theme-def-{normalized_theme}.yaml"
    이 파일을 Read하여 테마 정의를 로드하라.

    [요구사항서 경로]
    req_path: "{target}/.pipeline-temp/req-{normalized_theme}.yaml"
    이 파일을 Read하여 요구사항서를 로드하라.

    [피드백 저장 경로]
    feedback_path: "{target}/.pipeline-temp/feedback-{normalized_theme}.yaml"
    fail 시 critic_feedback을 이 경로에 Write 저장하라.

    작업 완료 후 3필드(result/frontmatter_valid/theme_fitness)를 반환하라.
    critic_feedback은 fail 시 feedback_path에 Write. Main에 반환하지 말 것.
```

3필드를 수신:

```yaml
result: pass|fail
frontmatter_valid: true|false
theme_fitness: pass|fail
```

> critic_feedback은 feedback_path에 저장됨. Main 컨텍스트에 적재되지 않음.

---

### Step D: 재시도 판정 + 저장

| critic result | retry_count | 행동 |
|---------------|:-----------:|------|
| pass | -- | Bash: `cp {draft_path} {target}/{output_path}` 저장. execution-log 기록. 다음 테마 |
| fail | 0 또는 1 | retry_count++. Step B로 복귀 (feedback_path 전달 — docu-writer가 직접 Read, **새 에이전트 생성**) |
| fail | 2 | Read(draft_path) → frontmatter에 `auto_generated_warning: "검증 미통과 - 수동 검토 필요"` 삽입 → `{target}/{output_path}`에 Write 저장. execution-log에 WARNING 기록. 다음 테마 |

**SKIPPED 경로** (Step A에서 판단 No):
- execution-log에 SKIPPED 기록. 다음 테마.

**FAIL 경로** (docu-writer 무응답, theme_definition 미발견 등):
- execution-log에 FAIL 기록. consecutive_fail++.

**저장 후 처리**:
- PASS/SKIPPED/WARNING 발생 시 `consecutive_fail` 초기화
- FAIL 발생 시 `consecutive_fail++`
- `consecutive_fail >= 3` -> 파이프라인 중단

### execution-log 기록

`{target}/execution-log.md`에 YAML 상태 헤더 + 결과 테이블 형식으로 기록:

```markdown
---
current_theme_index: 2
consecutive_fail: 0
completed_themes:
  - { theme: "intro", status: "PASS" }
  - { theme: "getting-started/requirements", status: "PASS" }
---

| theme | status | summary | timestamp |
|-------|--------|---------|-----------|
| intro | PASS | ... | ... |
| getting-started/requirements | PASS | ... | ... |
```

**기록 규칙**:
- 테마 시작 전: `current_theme_index` 업데이트
- 테마 완료 후: `completed_themes`에 추가, 테이블에 행 append
- `consecutive_fail`: PASS/SKIPPED/WARNING 시 0으로 초기화, FAIL 시 +1
- `timestamp`: Phase 1-3의 `generated_at`을 공통 사용 (파이프라인 실행 시점 기준). 테마별 실시간 시각이 아님.

---

## Phase 3: 완료 처리

#### 3-1. 임시 디렉토리 정리

```bash
rm -rf {target}/.pipeline-temp
```

#### 3-2. 최종 상태 출력

모든 테마 순회 후 execution-log.md 최종 상태를 출력한다.

</Steps>

> 참조: Examples / Failure_Modes_To_Avoid / Tool_Usage → `skills/task-pipeline/SKILL-REFERENCE.md`

<Escalation_And_Stop_Conditions>

| 조건 | 행동 |
|------|------|
| **3연속 FAIL** | 파이프라인 즉시 중단 + execution-log에 중단 사유 기록 |
| **Hard Cap 도달** (retry_count >= 2) | `auto_generated_warning` 태그 추가, WARNING으로 저장 |
| **docu-writer 무응답/빈 반환** | STATUS=FAIL 처리, execution-log에 "docu-writer 무응답" 기록 |
| **critic 무응답/빈 반환** | STATUS=FAIL 처리, execution-log에 "critic 무응답" 기록 |
| **theme_definition 미발견** | STATUS=FAIL 처리, execution-log에 "theme_definition 미발견: {theme}" 기록 |
| **git diff 실행 실패** | full-scan 모드로 전환 (FAIL이 아님) |
| **정체 감지** (재시도에서 동일 피드백 반복) | 현재 retry_count를 2로 강제 설정하여 WARNING 경로로 탈출 |

</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Phase 1-4에서 `.pipeline-temp/` 디렉토리를 생성했는가?
- [ ] 4개 테마를 순서대로 순차 처리했는가? (병렬 금지)
- [ ] Step A에서 Agent 도구로 scout를 새로 생성했는가? (직접 Glob/Grep 아님, CRITICAL 8)
- [ ] scout에 req_path, theme_def_path를 전달했는가?
- [ ] docu-writer에 req_path를 전달했는가? (인라인 YAML 아님)
- [ ] critic에 draft_path, theme_def_path, req_path, feedback_path를 전달했는가? (인라인 데이터 아님)
- [ ] Step B에서 Agent 도구로 docu-writer를 새로 생성했는가? (직접 md 작성 아님)
- [ ] Step B에서 docu-writer가 draft_path에 Write 저장했는가? (md 문자열 반환 아님)
- [ ] Step B 후 status 확인 + 파일 존재 확인을 수행했는가?
- [ ] Step C에서 Agent 도구로 critic을 새로 생성했는가? (직접 검증 아님)
- [ ] Step C에서 critic에 draft_path만 전달했는가? (md 인라인 삽입 아님)
- [ ] docu-writer -> critic 순서로 순차 호출했는가? (병렬 호출 없음)
- [ ] 재시도 시 새 scout/docu-writer/critic을 생성했는가? (Full Reset)
- [ ] retry_count가 Hard Cap(2회)을 초과하지 않았는가?
- [ ] PASS 시 `cp draft_path output_path`로 저장했는가?
- [ ] WARNING 시 draft를 Read → frontmatter 수정 → output_path에 Write했는가?
- [ ] execution-log.md에 모든 테마 결과를 기록했는가?
- [ ] 3연속 FAIL 시 중단했는가?
- [ ] .md 내용을 직접 작성하지 않았는가? (CRITICAL 1)
- [ ] 검증을 직접 수행하지 않았는가? (CRITICAL 2)
- [ ] Step B Agent prompt에 code_context를 삽입하지 않았는가?
- [ ] execution-log.md 상단 YAML 상태 블록을 매 테마마다 업데이트했는가?
- [ ] Phase 3에서 `.pipeline-temp/` 디렉토리를 정리했는가?
</Final_Checklist>
