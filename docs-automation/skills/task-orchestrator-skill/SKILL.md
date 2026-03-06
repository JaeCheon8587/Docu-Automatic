---
name: task-orchestrator-skill
description: 단일 테마 4단계 사이클(판단-작성-검증-재시도) 오케스트레이션 지침
origin: docs-automation
tools: [Read, Grep, Glob, Bash, Agent]
---

<Purpose>
단일 테마에 대한 4단계 사이클(판단 -> md 작성 -> 검증 -> 재시도 판정)의 상세 행동 지침을 제공한다.
task-orchestrator 에이전트가 작업 시작 시 이 스킬을 Read로 읽고 따른다.
메인(절차.md)이 테마 루프에서 작업 오케스트레이션을 호출할 때마다 이 지침에 따라 동작한다.
</Purpose>

<Use_When>
- 메인이 테마 루프에서 작업 오케스트레이션 에이전트를 생성할 때
- 테마별 판단/작성/검증/재시도 사이클이 필요할 때
- git diff 변경이 특정 테마의 문서화 대상인지 판단해야 할 때
- docu-writer/critic을 조율하여 .md를 생성해야 할 때
- 검증 실패 후 재시도 판정이 필요할 때
</Use_When>

<Do_Not_Use_When>
- 테마 루프 전체를 관리할 때 -> task-pipeline/SKILL.md (메인 오케스트레이션)
- .md 파일을 직접 작성할 때 -> docu-writer-skill/SKILL.md
- .md 파일을 검증할 때 -> critic-skill/SKILL.md
- 테마 정의를 참조할 때 -> theme-definitions/SKILL.md (직접 Read)
</Do_Not_Use_When>

<Why_This_Exists>
1. **메인 컨텍스트 보호**: 테마 수 x 4단계 복합 로직을 메인에 누적하면 컨텍스트 오염 발생. 테마별 독립 에이전트로 격리
2. **Full Reset 보장**: 매 테마마다 새 에이전트로 생성되므로 이전 테마의 판단/피드백이 현재 테마에 간섭하지 않음
3. **재시도 캡슐화**: 재시도 루프(최대 2회)를 작업 오케스트레이션 내부에서 처리하여 메인의 상태 관리 단순화
4. **역할 분리**: 판단 + 요구사항서 작성은 오케스트레이터, md 작성은 docu-writer, 검증은 critic으로 명확 분리
</Why_This_Exists>

<Execution_Policy>
**CRITICAL 제약 5개** -- 위반 시 파이프라인 오동작:

```
CRITICAL 1: docu-writer 결과를 수신하기 전에 critic을 호출하지 말 것
            -> critic은 md_content가 있어야 검증 가능
CRITICAL 2: critic 결과를 수신하기 전에 5필드를 반환하지 말 것
            -> 검증 전 반환은 품질 보증 우회
CRITICAL 3: 재시도 시 docu-writer, critic을 새로 생성할 것 (Full Reset)
            -> resume 금지. 이전 컨텍스트가 수정 방향을 왜곡
CRITICAL 4: 재시도 최대 2회 Hard Cap
            -> 2회 초과 시 WARNING 반환. 무한 루프 방지
CRITICAL 5: Step A 판단 No -> 즉시 STATUS=SKIPPED 반환
            -> Step B 진행 금지. 불필요한 에이전트 생성 방지
```
</Execution_Policy>

<Steps>

## Step A: 판단 + 요구사항서 작성

### A-1. 변경 파일 수집

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

### A-2. 테마 정의 읽기

Read로 `skills/theme-definitions/SKILL.md`를 읽고, 전달받은 `theme`에 해당하는 블록을 추출한다.
추출 대상: perspective, audience, writing_style, must_cover, do_not_cover, docs_path, section.

theme_definition을 찾을 수 없으면 즉시 `STATUS=FAIL` 반환 (SUMMARY에 원인 기록).

### A-3. 관련 코드 읽기

**diff 모드**: 변경 파일 목록에서 관련 소스 코드를 Read로 읽는다. Grep/Glob으로 변경 파일이 참조하는 헤더/인터페이스/설정 파일도 탐색한다.

**full-scan 모드**: 테마의 section과 must_cover 항목에 따라 코드베이스를 탐색한다.

| section | 탐색 전략 |
|---------|----------|
| `getting-started` | 빌드 설정 파일(CMakeLists.txt, .csproj, package.json), 설정 파일, 네트워크 포트 정의, 의존성 목록 |
| `architecture` | 프로젝트 구조, 진입점(main), 모듈 간 의존성, 통신 코드, 프로토콜 관련 소스 |

Glob으로 관련 디렉토리를 탐색하고, 주요 파일을 Read로 읽는다. 테마의 must_cover 항목에 해당하는 코드를 Grep으로 검색한다.

### A-4. 판단

**diff 모드**: 변경 내용이 현재 테마의 must_cover/perspective에 해당하는지 판단:
- **No** -> `STATUS=SKIPPED` 5필드 즉시 반환. Step B 진행 금지.
- **Yes** -> 요구사항서 작성 후 Step B 진행.

**full-scan 모드**: 코드베이스에 현재 테마에 해당하는 내용이 존재하는지 판단:
- 해당 내용 없음 -> `STATUS=SKIPPED` 반환.
- 해당 내용 있음 -> 요구사항서 작성 후 Step B 진행.

### A-5. 요구사항서 작성 (판단 Yes일 때)

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
- key_changes (full-scan 모드): `"초기 문서 생성 — 현재 코드베이스 기반"` 고정값
- related_files: 변경 파일 + A-3에서 탐색한 연관 파일

---

## Step B: md 작성 (docu-writer 호출)

### B-1. docu-writer 에이전트 생성

Agent 도구로 `agents/docu-writer.md` 기반 에이전트를 생성한다.

전달 내용:
1. **requirement_spec**: Step A에서 작성한 요구사항서 (10필드 YAML)
2. **code_context**: 관련 소스 코드 파일 내용
3. **critic_feedback**: 재시도 시에만 전달. 이전 critic의 피드백 목록

프롬프트 구성:

```
다음 요구사항서에 따라 Docusaurus용 .md를 작성하라.

[요구사항서]
{requirement_spec YAML}

[관련 코드]
{code_context}

{재시도 시: [critic 피드백]\n{critic_feedback}}
```

### B-2. md 수신

docu-writer가 반환한 .md 내용을 수신한다.
md 내용이 비어있거나 YAML frontmatter가 없으면 `STATUS=FAIL` 반환.

---

## Step C: 검증 (critic 호출)

### C-1. critic 에이전트 생성

Agent 도구로 `agents/critic.md` 기반 에이전트를 생성한다.

전달 내용:
1. **md_content**: Step B에서 수신한 .md 전체 내용
2. **theme_definition**: Step A-2에서 추출한 테마 정의 블록
3. **requirement_spec**: Step A에서 작성한 요구사항서

프롬프트 구성:

```
다음 .md 파일을 검증하라.

[md 내용]
{md_content}

[테마 정의]
{theme_definition}

[요구사항서]
{requirement_spec YAML}
```

### C-2. 4필드 수신

critic이 반환한 4필드를 수신:

```yaml
result: pass|fail
frontmatter_valid: true|false
theme_fitness: pass|fail
critic_feedback:
  - "{피드백 목록}"
```

---

## Step D: 재시도 판정

`retry_count`를 0으로 초기화하고 Step B/C 사이클을 추적한다.

| critic result | retry_count | 행동 |
|---------------|:-----------:|------|
| pass | -- | `STATUS=PASS`, MD_CONTENT에 docu-writer md 포함, 5필드 반환 |
| fail | 0 또는 1 | retry_count++, Step B로 복귀 (critic_feedback 포함, **새 에이전트 생성**) |
| fail | 2 | md의 frontmatter에 `auto_generated_warning: "검증 미통과 - 수동 검토 필요"` 추가, `STATUS=WARNING` 5필드 반환 |

**재시도 시 Full Reset**:
- 이전 docu-writer/critic 에이전트를 resume하지 않는다
- Agent 도구로 새 에이전트를 생성하여 critic_feedback + 원본 요구사항서를 전달한다
- 비용: 1.0x (Resume 시 1.8~3.6x 대비 절감)

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
| **Agent** | `agents/docu-writer.md` 기반 에이전트 생성 | B-1 |
| **Agent** | `agents/critic.md` 기반 에이전트 생성 | C-1 |

**Agent 호출 시 주의사항**:
- docu-writer/critic 에이전트는 `agents/` 디렉토리의 .md 파일을 에이전트 정의로 참조
- 실제 Agent 도구 호출 시 해당 에이전트의 역할/입력을 프롬프트에 명시
- 순차 호출 필수: docu-writer Agent 완료 -> critic Agent 호출
</Tool_Usage>

<Examples>

## GOOD/BAD Pair 1: 순차 호출 vs 병렬 호출

**GOOD** -- docu-writer 결과 대기 후 critic 호출:

```
1. Agent(docu-writer) -> md_content 수신 대기
2. md_content 수신 확인
3. Agent(critic) -> 4필드 수신 대기
4. 4필드 기반 판정
```

WHY: critic은 md_content가 있어야 검증 가능. 순차 호출로 정확한 입력 보장.

**BAD** -- docu-writer + critic 동시 호출:

```
1. Agent(docu-writer) + Agent(critic) 동시 호출
2. critic에 md_content 없이 빈 입력 전달
3. critic이 빈 문서에 대해 fail 판정
```

WHY: 병렬 호출 시 critic에 검증 대상이 없어 항상 fail. 무의미한 재시도 루프 진입.

---

## GOOD/BAD Pair 2: Full Reset vs Resume

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

## GOOD/BAD Pair 3: SKIPPED 즉시 반환 vs 불필요 에이전트 생성

**GOOD** -- 판단 No시 즉시 SKIPPED:

```
theme: architecture/component-diagram
git diff: settings.ini, deploy.sh 변경

판단: settings.ini/deploy.sh는 component-diagram의 must_cover(제품별 컴포넌트, 의존 관계, 플랫폼 차이)에 해당하지 않음
-> STATUS=SKIPPED 즉시 반환
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

<Escalation_And_Stop_Conditions>

| 조건 | 행동 |
|------|------|
| **Hard Cap 도달** (retry_count >= 2) | `auto_generated_warning` 태그 추가, `STATUS=WARNING` 반환 |
| **docu-writer 무응답/빈 반환** | `STATUS=FAIL` 반환, SUMMARY에 "docu-writer 무응답" 기록 |
| **critic 무응답/빈 반환** | `STATUS=FAIL` 반환, SUMMARY에 "critic 무응답" 기록 |
| **theme_definition 미발견** | `STATUS=FAIL` 반환, SUMMARY에 "theme_definition 미발견: {theme}" 기록 |
| **git diff 실행 실패** | full-scan 모드로 전환 (FAIL이 아님) |
| **정체 감지** (재시도에서 동일 피드백 반복) | 현재 retry_count를 2로 강제 설정하여 WARNING 경로로 탈출 |

모든 비정상 경로에서 반드시 5필드를 반환한다. 빈 필드는 있어도 5필드 구조는 유지.
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] skills/task-orchestrator-skill/SKILL.md를 Read로 읽었는가?
- [ ] skills/theme-definitions/SKILL.md에서 해당 테마 블록을 추출했는가?
- [ ] git diff를 Bash로 실행하여 변경 파일을 수집했는가?
- [ ] Step A에서 테마 필요 여부를 판단했는가? (No이면 SKIPPED 즉시 반환)
- [ ] 요구사항서 10필드(theme, section, perspective, audience, writing_style, must_cover, do_not_cover, key_changes, related_files, output_path)가 모두 작성되었는가?
- [ ] docu-writer -> critic 순서로 순차 호출했는가? (병렬 호출 없음)
- [ ] 재시도 시 docu-writer/critic을 새로 생성했는가? (Full Reset, resume 없음)
- [ ] 5필드(STATUS/THEME/FILE/SUMMARY/MD_CONTENT) 모두 포함하여 반환했는가?
- [ ] retry_count가 Hard Cap(2회)을 초과하지 않았는가?
</Final_Checklist>
