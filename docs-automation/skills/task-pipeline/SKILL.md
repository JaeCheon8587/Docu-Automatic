---
name: task-pipeline
description: 테마 순차 순회 파이프라인 — 테마마다 task-orchestrator 생성, 결과 저장, 상태 기록
origin: docs-automation
tools: [Read, Write, Agent, Bash]
---

<Purpose>
theme-definitions의 테마를 순차 순회하며, 테마마다 task-orchestrator 에이전트를 생성하고,
반환된 MD_CONTENT를 파일로 저장하고, execution-log.md에 결과를 기록한다.
</Purpose>

<Use_When>
- 코드베이스 대상으로 Docusaurus 문서를 자동 생성할 때
- git push CI 트리거 또는 수동 실행 시
</Use_When>

<Do_Not_Use_When>
- 단일 테마만 처리할 때 -> task-orchestrator-skill 직접 사용
- .md 파일을 직접 작성/검증할 때 -> docu-writer-skill, critic-skill
</Do_Not_Use_When>

<Execution_Policy>
**CRITICAL 제약 4개** -- 위반 시 파이프라인 오동작:

```
CRITICAL 1: 반드시 Agent 도구로 task-orchestrator를 생성하여 위임할 것
            -> 직접 코드를 읽거나, .md를 작성하거나, 테마 판단을 하지 말 것
            -> Glob/Grep/Read로 대상 코드베이스의 소스 파일을 절대 읽지 말 것
CRITICAL 2: task-orchestrator의 5필드 반환을 수신하기 전에 다음 테마로 넘어가지 말 것
            -> 순차 게이트. 병렬 호출 절대 금지
CRITICAL 3: 테마마다 새 task-orchestrator를 생성할 것 (Full Reset)
            -> 이전 에이전트 resume 금지
CRITICAL 4: MD_CONTENT 파일 저장과 execution-log 기록은 이 에이전트의 유일한 실행 책임
            -> 그 외 모든 작업(판단, 요구사항서, md 작성, 검증)은 task-orchestrator가 처리
```
</Execution_Policy>

<Steps>

## 1. 입력 확인

실행 시 전달받는 파라미터:
- `target`: 대상 코드베이스 경로

## 2. 테마 순회

아래 4개 테마를 정의된 순서대로 **순차** 처리한다 (병렬 금지).

```yaml
themes:
  - { theme: "intro", section: "intro", output_path: "docs/intro.md" }
  - { theme: "getting-started/requirements", section: "getting-started", output_path: "docs/getting-started/requirements.md" }
  - { theme: "architecture/overview", section: "architecture", output_path: "docs/architecture/overview.md" }
  - { theme: "architecture/component-diagram", section: "architecture", output_path: "docs/architecture/component-diagram.md" }
```

테마마다 아래 2-1 ~ 2-4를 실행한다.

### 2-1. task-orchestrator 에이전트 생성 (Full Reset)

Agent 도구로 `agents/task-orchestrator.md` 기반 에이전트를 **새로** 생성한다.

**리터럴 Agent 호출 템플릿** -- 아래 형태를 그대로 사용:

```
Agent 도구 호출:
  description: "task-orchestrator {theme}"
  prompt: |
    너는 task-orchestrator 에이전트다.
    agents/task-orchestrator.md를 읽고 역할을 확인하라.
    skills/task-orchestrator-skill/SKILL.md를 읽고 절차를 따르라.

    [입력]
    theme: "{theme}"
    section: "{section}"
    output_path: "{output_path}"
    target: "{target}"

    작업 완료 후 반드시 5필드(STATUS/THEME/FILE/SUMMARY/MD_CONTENT)를 반환하라.
```

### 2-2. 반환값 파싱 및 저장

5필드(STATUS, THEME, FILE, SUMMARY, MD_CONTENT)를 수신한다.

| STATUS | 행동 |
|--------|------|
| PASS / WARNING | MD_CONTENT를 `{target}/{output_path}`에 Write로 저장 |
| SKIPPED | 다음 테마로 |
| FAIL | consecutive_fail++ |

### 2-3. execution-log 기록

`{target}/execution-log.md`에 결과를 append:

```
| {theme} | {STATUS} | {SUMMARY} | {timestamp} |
```

### 2-4. 중단 판정

3개 테마 연속 FAIL 시 파이프라인 중단.
PASS/SKIPPED/WARNING 발생 시 consecutive_fail 초기화.

## 3. 완료

모든 테마 순회 후 execution-log.md 최종 상태를 출력한다.

</Steps>

<Examples>

## GOOD/BAD Pair 1: Agent 위임 vs 직접 처리

**GOOD** -- Agent 호출로 위임 -> 5필드 수신 -> 파일 저장:

```
[테마: intro]
1. Agent(task-orchestrator) 생성
   prompt: "너는 task-orchestrator 에이전트다. agents/task-orchestrator.md를 읽고..."
   theme: "intro", section: "intro", output_path: "docs/intro.md", target: "/path/to/repo"
2. 5필드 수신 대기
3. STATUS=PASS -> MD_CONTENT를 /path/to/repo/docs/intro.md에 Write
4. execution-log.md에 결과 기록
```

WHY: task-pipeline은 루프 관리와 결과 저장만 담당. 판단/작성/검증은 task-orchestrator가 처리. 역할 분리가 유지됨.

**BAD** -- task-pipeline이 직접 코드 읽기/md 작성/판단 수행:

```
[테마: intro]
1. Glob("**/*.cs")로 소스 코드 탐색           <- task-orchestrator 역할 침범
2. Read로 소스 코드 읽기                       <- task-orchestrator 역할 침범
3. "이 코드는 intro 테마에 해당" 직접 판단      <- task-orchestrator 역할 침범
4. Write("docs/intro.md", 직접 작성한 md)      <- docu-writer 역할 침범
5. task-orchestrator를 생성하지 않음
```

WHY: 모든 역할을 task-pipeline이 독점하면 컨텍스트 폭발, Full Reset 불가, 재시도 로직 부재. 테마가 늘어날수록 품질 저하.

---

## GOOD/BAD Pair 2: 순차 테마 처리 vs 병렬 테마 처리

**GOOD** -- 테마 1 완료 대기 -> 테마 2 Agent 생성:

```
[테마 1: intro]
Agent(task-orchestrator "intro") -> 5필드 수신 -> 파일 저장 -> log 기록

[테마 2: getting-started/requirements]
Agent(task-orchestrator "getting-started/requirements") -> 5필드 수신 -> 파일 저장 -> log 기록
```

WHY: 순차 처리로 consecutive_fail 카운트가 정확하고, 3연속 FAIL 중단이 올바르게 동작.

**BAD** -- 테마 1, 2를 병렬 Agent 호출:

```
Agent(task-orchestrator "intro") + Agent(task-orchestrator "getting-started/requirements") 동시 호출
-> 3연속 FAIL 판정 불가 (결과 순서 불확정)
-> consecutive_fail 카운트 오류
-> 파이프라인 중단 로직 무력화
```

WHY: 병렬 호출은 순차 게이트(CRITICAL 2)를 위반. 중단 판정과 로그 순서가 깨짐.

</Examples>

<Failure_Modes_To_Avoid>

| # | 실패 모드 | 설명 | 위반하는 CRITICAL |
|---|-----------|------|-------------------|
| 1 | **역할 침범** | task-pipeline이 Glob/Grep/Read로 소스 코드 탐색 -> task-orchestrator 역할 | CRITICAL 1 |
| 2 | **직접 md 작성** | task-pipeline이 Write로 직접 .md 내용 생성 -> docu-writer 역할 | CRITICAL 1, 4 |
| 3 | **병렬 테마 처리** | 여러 테마를 동시에 Agent 호출 -> 순차 제약 위반 | CRITICAL 2 |
| 4 | **5필드 미수신 진행** | task-orchestrator 반환 전 다음 테마 진행 -> 결과 유실 | CRITICAL 2 |

</Failure_Modes_To_Avoid>

<Final_Checklist>
- [ ] theme-definitions의 모든 테마(4개)를 순회했는가?
- [ ] 테마마다 새 task-orchestrator를 Agent 도구로 생성했는가? (Full Reset)
- [ ] Agent 호출 시 리터럴 템플릿(theme/section/output_path/target)을 사용했는가?
- [ ] task-orchestrator의 5필드 반환을 수신한 후에만 다음 테마로 넘어갔는가?
- [ ] PASS/WARNING 결과의 MD_CONTENT를 파일로 저장했는가?
- [ ] execution-log.md에 모든 테마 결과를 기록했는가?
- [ ] 3연속 FAIL 시 중단했는가?
- [ ] Glob/Grep/Read로 소스 코드를 직접 읽지 않았는가?
- [ ] .md 내용을 직접 작성하지 않았는가? (MD_CONTENT는 task-orchestrator 반환값만 사용)
</Final_Checklist>
