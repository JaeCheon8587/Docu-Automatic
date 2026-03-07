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
1. **서브 에이전트 중첩 제약 해소**: Claude Code에서 서브 에이전트가 서브 에이전트를 생성할 수 없으므로, Main이 판단+오케스트레이션을 직접 수행하고 docu-writer/critic만 Level 1 Agent로 호출
2. **구조 단순화**: v3의 3단계(Main -> task-orchestrator -> docu-writer/critic)에서 v4의 2단계(Main -> docu-writer/critic)로 평탄화
3. **Full Reset 유지**: 매 테마마다 새 docu-writer/critic 생성으로 컨텍스트 오염 방지, 비용 효율(1.0x)
4. **역할 분리 유지**: Main은 판단+오케스트레이션, docu-writer는 md 작성 전문, critic은 검증 전문
</Why_This_Exists>

<Execution_Policy>
**CRITICAL 제약 7개** -- 위반 시 파이프라인 오동작:

```
CRITICAL 1: .md 내용을 직접 작성하지 말 것
            -> 반드시 Agent(docu-writer)에 위임
            -> Write 도구로 직접 .md 본문을 생성하지 말 것
            -> md 저장은 docu-writer가 반환한 md_content를 그대로 파일에 Write
CRITICAL 2: 검증을 직접 수행하지 말 것
            -> 반드시 Agent(critic)에 위임
            -> "내용이 적절해 보인다" 같은 직접 판단 금지
CRITICAL 3: docu-writer 결과 수신 전 critic 호출 금지 (순차)
            -> critic은 md_content가 있어야 검증 가능
CRITICAL 4: 테마 순회는 반드시 순차 (병렬 금지)
            -> 현재 테마의 전체 사이클(판단->작성->검증->저장)이 완료된 후 다음 테마 진행
CRITICAL 5: 테마마다 새 docu-writer/critic 생성 (Full Reset)
            -> 이전 에이전트 resume 금지
            -> 재시도 시에도 새 에이전트 생성
CRITICAL 6: 재시도 최대 2회 Hard Cap
            -> 2회 초과 시 auto_generated_warning 태그 부착 후 WARNING으로 저장
CRITICAL 7: 3연속 FAIL 시 파이프라인 중단
            -> PASS/SKIPPED/WARNING 발생 시 consecutive_fail 초기화
```
</Execution_Policy>

<Steps>

## Phase 1: 입력 확인

실행 시 전달받는 파라미터:
- `target`: 대상 코드베이스 경로

## Phase 2: 테마 루프

아래 4개 테마를 정의된 순서대로 **순차** 처리한다 (병렬 금지).

```yaml
themes:
  - { theme: "intro", section: "intro", output_path: "docs/intro.md" }
  - { theme: "getting-started/requirements", section: "getting-started", output_path: "docs/getting-started/requirements.md" }
  - { theme: "architecture/overview", section: "architecture", output_path: "docs/architecture/overview.md" }
  - { theme: "architecture/component-diagram", section: "architecture", output_path: "docs/architecture/component-diagram.md" }
```

테마마다 아래 Step A ~ Step D를 실행한다. `retry_count`를 0으로 초기화.

---

### Step A: 판단 + 요구사항서 작성

#### A-1. 변경 파일 수집

Bash로 git diff 실행:

```bash
git diff HEAD~1 --name-only
```

**2가지 모드**:

| 모드 | 조건 | 동작 |
|------|------|------|
| **diff 모드** | git diff 결과가 있음 | 변경 파일 기반으로 판단 + 선별적 문서화 |
| **full-scan 모드** | git diff 결과가 없음 (초기 생성, 수동 실행, diff 실패) | 현재 코드베이스 전체를 대상으로 문서화 |

diff 모드: 변경 파일 목록을 수집하여 A-3, A-4에서 활용.
full-scan 모드: Glob/Grep으로 section 관련 소스 파일을 탐색하여 코드베이스 파악.

#### A-2. 테마 정의 읽기

Read로 `skills/theme-definitions/SKILL.md`를 읽고, 전달받은 `theme`에 해당하는 블록을 추출한다.
추출 대상: perspective, audience, writing_style, must_cover, do_not_cover, docs_path, section.

theme_definition을 찾을 수 없으면 즉시 STATUS=FAIL 처리 (execution-log에 원인 기록).

#### A-3. 관련 코드 읽기

**diff 모드**: 변경 파일 목록에서 관련 소스 코드를 Read로 읽는다. Grep/Glob으로 변경 파일이 참조하는 헤더/인터페이스/설정 파일도 탐색한다.

**full-scan 모드**: 테마의 section과 must_cover 항목에 따라 코드베이스를 탐색한다.

| section | 탐색 전략 |
|---------|----------|
| `intro` | 프로젝트 루트 파일(README, 설정), 진입점(main), 전체 디렉토리 구조 |
| `getting-started` | 빌드 설정 파일(CMakeLists.txt, .csproj, package.json), 설정 파일, 네트워크 포트 정의, 의존성 목록 |
| `architecture` | 프로젝트 구조, 진입점(main), 모듈 간 의존성, 통신 코드, 프로토콜 관련 소스 |

Glob으로 관련 디렉토리를 탐색하고, 주요 파일을 Read로 읽는다. 테마의 must_cover 항목에 해당하는 코드를 Grep으로 검색한다.

#### A-4. 판단

**diff 모드**: 변경 내용이 현재 테마의 must_cover/perspective에 해당하는지 판단:
- **No** -> 이 테마를 SKIPPED 처리. execution-log 기록 후 다음 테마로. Step B 진행 금지.
- **Yes** -> 요구사항서 작성 후 Step B 진행.

**full-scan 모드**: 코드베이스에 현재 테마에 해당하는 내용이 존재하는지 판단:
- 해당 내용 없음 -> SKIPPED 처리.
- 해당 내용 있음 -> 요구사항서 작성 후 Step B 진행.

#### A-5. 요구사항서 작성 (판단 Yes일 때)

다음 10필드 YAML 형식으로 요구사항서를 작성:

```yaml
theme: "{theme_id}"
section: "{section}"
perspective: "{theme_definition.perspective}"
audience: "{theme_definition.audience}"
writing_style: "{theme_definition.writing_style}"
must_cover:
  - "{변경과 관련된 must_cover 항목}"
do_not_cover:
  - "{theme_definition.do_not_cover 전체}"
key_changes: "git diff 변경 내용 요약"
related_files:
  - "{변경 파일 경로}"
  - "{연관 파일 경로}"
output_path: "{output_path}"
```

**규칙**:
- must_cover (diff 모드): theme_definition의 must_cover 중 변경과 관련된 항목만 선별
- must_cover (full-scan 모드): theme_definition의 must_cover 전체를 포함
- do_not_cover: theme_definition의 do_not_cover 전체를 그대로 포함
- key_changes (diff 모드): git diff 내용을 1~3문장으로 요약
- key_changes (full-scan 모드): `"초기 문서 생성 -- 현재 코드베이스 기반"` 고정값
- related_files: 변경 파일 + A-3에서 탐색한 연관 파일

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

    [요구사항서]
    {requirement_spec YAML 전체}

    [관련 코드]
    {code_context - A-3에서 읽은 소스 코드 내용}

    {재시도 시에만 추가:
    [critic 피드백]
    {이전 critic의 critic_feedback 목록}}

    작업 완료 후 Docusaurus용 .md 전체 내용(YAML frontmatter 포함)을 반환하라.
```

md_content를 수신한다. md 내용이 비어있거나 YAML frontmatter가 없으면 STATUS=FAIL 처리.

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

    [md 내용]
    {docu-writer가 반환한 .md 전체 내용}

    [테마 정의]
    {A-2에서 추출한 theme_definition 블록}

    [요구사항서]
    {requirement_spec YAML 전체}

    작업 완료 후 4필드(result/frontmatter_valid/theme_fitness/critic_feedback)를 반환하라.
```

4필드를 수신:

```yaml
result: pass|fail
frontmatter_valid: true|false
theme_fitness: pass|fail
critic_feedback:
  - "{피드백 목록}"
```

---

### Step D: 재시도 판정 + 저장

| critic result | retry_count | 행동 |
|---------------|:-----------:|------|
| pass | -- | md_content를 `{target}/{output_path}`에 Write 저장. execution-log 기록. 다음 테마 |
| fail | 0 또는 1 | retry_count++. Step B로 복귀 (critic_feedback 포함, **새 에이전트 생성**) |
| fail | 2 | md의 frontmatter에 `auto_generated_warning: "검증 미통과 - 수동 검토 필요"` 추가. Write 저장. execution-log에 WARNING 기록. 다음 테마 |

**SKIPPED 경로** (Step A에서 판단 No):
- execution-log에 SKIPPED 기록. 다음 테마.

**FAIL 경로** (docu-writer 무응답, theme_definition 미발견 등):
- execution-log에 FAIL 기록. consecutive_fail++.

**저장 후 처리**:
- PASS/SKIPPED/WARNING 발생 시 `consecutive_fail` 초기화
- FAIL 발생 시 `consecutive_fail++`
- `consecutive_fail >= 3` -> 파이프라인 중단

### execution-log 기록

`{target}/execution-log.md`에 결과를 append:

```
| {theme} | {STATUS} | {SUMMARY} | {timestamp} |
```

---

## Phase 3: 완료 처리

모든 테마 순회 후 execution-log.md 최종 상태를 출력한다.

</Steps>

<Tool_Usage>

| 도구 | 용도 | Step |
|------|------|------|
| **Bash** | `git diff HEAD~1 --name-only` 변경 파일 목록 | A-1 |
| **Bash** | `git diff HEAD~1` 상세 변경 내역 | A-1 |
| **Bash** | `git log -1 --format=%h` 커밋 해시 조회 | A-1 |
| **Read** | `skills/theme-definitions/SKILL.md` 테마 정의 읽기 | A-2 |
| **Read** | 변경 파일 소스 코드 읽기 | A-3 |
| **Grep** | 변경 파일이 참조하는 심볼/함수 검색 | A-3 |
| **Glob** | 관련 헤더/인터페이스 파일 탐색 | A-3 |
| **Agent** | docu-writer 에이전트 생성 | B |
| **Agent** | critic 에이전트 생성 | C |
| **Write** | md 파일 저장 (`{target}/{output_path}`) | D |
| **Write** | execution-log.md 기록 | D |

**Agent 호출 필수 규칙**:
- Step B의 md 작성은 **반드시** Agent 도구로 docu-writer를 생성하여 위임 (CRITICAL 1)
- Step C의 검증은 **반드시** Agent 도구로 critic을 생성하여 위임 (CRITICAL 2)
- docu-writer Agent 완료 후 critic Agent 호출 (CRITICAL 3)
- 테마마다 새 Agent 생성 (CRITICAL 5)

</Tool_Usage>

<Examples>

## GOOD/BAD Pair 1: Agent 위임 vs 직접 md 작성

**GOOD** -- Agent 도구로 docu-writer 생성하여 md 작성 위임:

```
Step A 완료: 요구사항서 + 코드 컨텍스트 준비

Step B:
1. Agent(docu-writer) 생성
   prompt: "너는 docu-writer 에이전트다. agents/docu-writer.md를 읽고..."
   전달: 요구사항서 + 코드 컨텍스트
2. docu-writer가 반환한 md_content 수신

Step C:
3. Agent(critic) 생성
   prompt: "너는 critic 에이전트다. agents/critic.md를 읽고..."
   전달: md_content + theme_definition + 요구사항서
4. critic이 반환한 4필드 수신

Step D:
5. result=pass -> md_content를 파일로 Write 저장
6. execution-log.md에 PASS 기록
```

WHY: 역할 분리 유지. docu-writer는 md 작성 전문, critic은 검증 전문. Main은 오케스트레이션과 저장에 집중.

**BAD** -- Main이 직접 md 작성 + 직접 검증:

```
Step A 완료: 요구사항서 + 코드 컨텍스트 준비

Step B (위반):
1. Agent(docu-writer)를 생성하지 않음
2. 직접 Write("docs/intro.md", 자체 작성 md 내용)  <- CRITICAL 1 위반

Step C (위반):
3. Agent(critic)를 생성하지 않음
4. "내용이 적절해 보인다" 직접 판단                   <- CRITICAL 2 위반
5. 파일 저장
```

WHY: docu-writer 스킬의 Docusaurus 규칙/frontmatter 검증이 누락됨. critic 스킬의 체계적 검증이 누락됨. 품질 보증 없는 문서 생성.

---

## GOOD/BAD Pair 2: 순차 테마 처리 vs 병렬 테마 처리

**GOOD** -- 테마 1 완료 대기 -> 테마 2 진행:

```
[테마 1: intro]
Step A: 판단 + 요구사항서
Step B: Agent(docu-writer "intro") -> md_content 수신
Step C: Agent(critic "intro") -> result=pass
Step D: 파일 저장 + log 기록

[테마 2: getting-started/requirements]
Step A: 판단 + 요구사항서
Step B: Agent(docu-writer "getting-started/requirements") -> md_content 수신
...
```

WHY: 순차 처리로 consecutive_fail 카운트가 정확하고, 3연속 FAIL 중단이 올바르게 동작.

**BAD** -- 테마 1, 2의 docu-writer를 병렬 Agent 호출:

```
Agent(docu-writer "intro") + Agent(docu-writer "getting-started/requirements") 동시 호출
-> CRITICAL 4 위반 (테마 순차 처리 필수)
-> 3연속 FAIL 판정 불가 (결과 순서 불확정)
```

WHY: 병렬 호출은 CRITICAL 4를 위반. 중단 판정과 로그 순서가 깨짐.

---

## GOOD/BAD Pair 3: Full Reset vs Resume

**GOOD** -- 재시도 시 새 에이전트 생성:

```
[1차 시도]
Agent(docu-writer) -> md_v1
Agent(critic) -> fail + feedback

[재시도 - 새 에이전트]
Agent(docu-writer, NEW) -> 전달: requirement_spec + code_context + critic_feedback
                        -> md_v2 (피드백 반영)
Agent(critic, NEW) -> pass
```

WHY: 새 에이전트는 깨끗한 컨텍스트에서 시작. 비용 1.0x. 이전 오류 패턴이 전파되지 않음.

**BAD** -- 기존 에이전트에 "수정해줘":

```
[1차 시도]
Agent(docu-writer) -> md_v1
Agent(critic) -> fail + feedback

[재시도 - resume]
Agent(docu-writer, RESUME) -> "이전 md를 수정해줘: {feedback}"
                           -> md_v2 (이전 컨텍스트 + 오류 패턴 잔존)
```

WHY: Resume 시 이전 컨텍스트(오류 포함)가 누적. 비용 1.8~3.6x. 동일 오류 반복 가능.

---

## GOOD/BAD Pair 4: SKIPPED 즉시 처리 vs 불필요 에이전트 생성

**GOOD** -- 판단 No시 즉시 SKIPPED:

```
theme: architecture/component-diagram
git diff: settings.ini, deploy.sh 변경

판단: settings.ini/deploy.sh는 component-diagram의 must_cover에 해당하지 않음
-> SKIPPED 처리, execution-log 기록
-> docu-writer/critic 호출 없음
```

WHY: 불필요한 에이전트 생성/호출 비용 절감. 무의미 문서 생성 방지.

**BAD** -- 판단 없이 무조건 docu-writer 호출:

```
theme: architecture/component-diagram
git diff: settings.ini, deploy.sh 변경

판단 건너뜀
-> Agent(docu-writer) -> settings.ini 변경 기반 컴포넌트 다이어그램 문서 생성
-> 설정 파일 내용이 아키텍처 문서에 혼입
```

WHY: 판단 없이 호출하면 테마와 무관한 문서가 생성됨. critic이 fail 판정해도 불필요한 2회 재시도 소모.

</Examples>

<Failure_Modes_To_Avoid>

| # | 실패 모드 | 설명 | 위반하는 CRITICAL |
|---|-----------|------|-------------------|
| 1 | **직접 md 작성** | Agent(docu-writer)를 생성하지 않고 직접 .md 내용을 Write로 작성 | CRITICAL 1 |
| 2 | **직접 검증** | Agent(critic)를 생성하지 않고 직접 품질/적합성을 판단 | CRITICAL 2 |
| 3 | **병렬 호출** | docu-writer + critic을 동시에 Agent 호출 | CRITICAL 3 |
| 4 | **병렬 테마** | 여러 테마를 동시에 처리 | CRITICAL 4 |
| 5 | **Resume 시도** | 재시도 시 이전 에이전트를 resume하여 수정 요청 | CRITICAL 5 |
| 6 | **무한 재시도** | retry_count 확인 없이 계속 재시도 | CRITICAL 6 |
| 7 | **중단 무시** | 3연속 FAIL에도 계속 진행 | CRITICAL 7 |
| 8 | **판단 없는 작성** | Step A 판단을 건너뛰고 무조건 docu-writer 호출 | -- |
| 9 | **검증 전 저장** | critic 결과 수신 전에 md 파일을 저장 | CRITICAL 2, 3 |

</Failure_Modes_To_Avoid>

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
- [ ] skills/theme-definitions/SKILL.md에서 테마 정의를 읽었는가?
- [ ] git diff를 Bash로 실행하여 변경 파일을 수집했는가?
- [ ] 4개 테마를 순서대로 순차 처리했는가? (병렬 금지)
- [ ] 테마마다 Step A에서 필요 여부를 판단했는가? (No이면 SKIPPED)
- [ ] 요구사항서 10필드를 모두 작성했는가?
- [ ] Step B에서 Agent 도구로 docu-writer를 새로 생성했는가? (직접 md 작성 아님)
- [ ] Step C에서 Agent 도구로 critic을 새로 생성했는가? (직접 검증 아님)
- [ ] docu-writer -> critic 순서로 순차 호출했는가? (병렬 호출 없음)
- [ ] 재시도 시 새 docu-writer/critic을 생성했는가? (Full Reset)
- [ ] retry_count가 Hard Cap(2회)을 초과하지 않았는가?
- [ ] PASS/WARNING 결과의 md_content를 파일로 저장했는가?
- [ ] execution-log.md에 모든 테마 결과를 기록했는가?
- [ ] 3연속 FAIL 시 중단했는가?
- [ ] .md 내용을 직접 작성하지 않았는가? (CRITICAL 1)
- [ ] 검증을 직접 수행하지 않았는가? (CRITICAL 2)
</Final_Checklist>
