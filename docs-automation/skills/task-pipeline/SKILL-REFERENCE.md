---
name: task-pipeline-reference
description: task-pipeline SKILL.md의 Examples, Failure_Modes, Tool_Usage 참조 문서
origin: docs-automation
---

> 이 문서는 `task-pipeline/SKILL.md`에서 분리된 참조 자료입니다.
> 런타임 실행에는 SKILL.md만 필요합니다. 학습/디버깅 시 참조하세요.

<Tool_Usage>

| 도구 | 용도 | Step |
|------|------|------|
| **Bash** | `cd {target} && [ -d .git ] && git log -1 --format=%h \|\| echo "non-git"` last_commit 확보 | Phase 1 |
| **Bash** | `date -u +%Y-%m-%dT%H:%M:%S` generated_at 확보 | Phase 1 |
| **Bash** | `mkdir -p {target}/.pipeline-temp` 임시 디렉토리 생성 | Phase 1 |
| **Agent** | scout 에이전트 생성 (Glob/Grep + 판단 + 요구사항서) — Main 컨텍스트 보호 | A |
| **Read** | execution-log.md 상태 복구 | Phase 2 |
| **Agent** | docu-writer 에이전트 생성 (md를 draft_path에 Write) | B |
| **Agent** | critic 에이전트 생성 (draft_path에서 Read로 md 로드) | C |
| **Bash** | `cp {draft_path} {target}/{output_path}` PASS 시 파일 복사 | D |
| **Write** | WARNING 시 frontmatter에 auto_generated_warning 삽입 후 저장 | D |
| **Write** | execution-log.md 기록 | D |
| **Bash** | `rm -rf {target}/.pipeline-temp` 임시 디렉토리 정리 | Phase 3 |

**Agent 호출 필수 규칙**:
- Step A의 탐색+판단+요구사항서는 **반드시** Agent 도구로 scout를 생성하여 위임 (CRITICAL 8)
- Step B의 md 작성은 **반드시** Agent 도구로 docu-writer를 생성하여 위임 (CRITICAL 1)
- Step C의 검증은 **반드시** Agent 도구로 critic을 생성하여 위임 (CRITICAL 2)
- docu-writer Agent 완료 후 critic Agent 호출 (CRITICAL 3)
- 테마마다 새 scout/docu-writer/critic Agent 생성 (CRITICAL 5)

</Tool_Usage>

<Examples>

## GOOD/BAD Pair 1: 파일 기반 핸드오프 vs md 문자열 인라인

**GOOD** -- docu-writer가 draft_path에 Write, Main은 경로만 수신, critic이 Read로 로드:

```
Step B:
1. Agent(docu-writer) 생성
   prompt에 draft_path 포함: "{target}/.pipeline-temp/draft-intro.md"
   docu-writer가 md를 draft_path에 Write
2. Main은 경량 확인서 수신: { status: "done", draft_path: "..." }
3. Main이 파일 존재 확인 (Bash: [ -f {draft_path} ])

Step C:
4. Agent(critic) 생성
   prompt에 draft_path 전달 (경로만, ~50 tokens)
   critic이 Read(draft_path)로 md 직접 로드
5. critic이 4필드 반환

Step D:
6. result=pass -> Bash: cp {draft_path} {target}/{output_path}
```

WHY: md_content가 Main 컨텍스트에 적재되지 않음. 테마당 ~6,000 tokens 절감. 4테마 기준 ~24,000 tokens 절감.

**BAD** -- md 전문을 문자열로 Main에 반환 후 critic에 재삽입:

```
Step B:
1. Agent(docu-writer) -> md_content 문자열 반환 (~3,000 tokens Main에 적재)

Step C:
2. Agent(critic) prompt에 md_content 인라인 삽입 (~3,000 tokens Main에서 발신)
   -> 테마당 ~6,000 tokens × 4테마 = ~24,000 tokens 소비
```

WHY: md_content가 Main 컨텍스트에 2번 적재(수신 1회 + 발신 1회). 재시도 시 추가 누적. auto-compact 위험 증가.

---

## GOOD/BAD Pair 2: Agent 위임 vs 직접 md 작성

**GOOD** -- Agent 도구로 docu-writer 생성하여 md 작성 위임:

```
Step A 완료: 요구사항서 + 관련 파일 경로 수집

Step B:
1. Agent(docu-writer) 생성
   prompt: "너는 docu-writer 에이전트다. agents/docu-writer.md를 읽고..."
   전달: 요구사항서 + 대상 코드베이스 경로 + draft_path
2. 경량 확인서 수신 (status + draft_path)

Step C:
3. Agent(critic) 생성
   prompt: "너는 critic 에이전트다. agents/critic.md를 읽고..."
   전달: draft_path + theme_definition + 요구사항서
4. critic이 반환한 4필드 수신

Step D:
5. result=pass -> cp draft_path -> output_path
6. execution-log.md에 PASS 기록
```

WHY: 역할 분리 유지. docu-writer는 md 작성 전문, critic은 검증 전문. Main은 오케스트레이션과 저장에 집중.

**BAD** -- Main이 직접 md 작성 + 직접 검증:

```
Step B (위반):
1. Agent(docu-writer)를 생성하지 않음
2. 직접 Write("docs/intro.md", 자체 작성 md 내용)  <- CRITICAL 1 위반

Step C (위반):
3. Agent(critic)를 생성하지 않음
4. "내용이 적절해 보인다" 직접 판단                   <- CRITICAL 2 위반
```

WHY: docu-writer 스킬의 Docusaurus 규칙/frontmatter 검증이 누락됨. critic 스킬의 체계적 검증이 누락됨.

---

## GOOD/BAD Pair 3: 순차 테마 처리 vs 병렬 테마 처리

**GOOD** -- 테마 1 완료 대기 -> 테마 2 진행:

```
[테마 1: intro]
Step A -> Step B (docu-writer) -> Step C (critic) -> Step D (저장)

[테마 2: getting-started/requirements]
Step A -> Step B (docu-writer) -> Step C (critic) -> Step D (저장)
```

WHY: 순차 처리로 consecutive_fail 카운트가 정확하고, 3연속 FAIL 중단이 올바르게 동작.

**BAD** -- 병렬 Agent 호출:

```
Agent(docu-writer "intro") + Agent(docu-writer "getting-started/requirements") 동시 호출
-> CRITICAL 4 위반 (테마 순차 처리 필수)
```

WHY: 병렬 호출은 CRITICAL 4를 위반. 중단 판정과 로그 순서가 깨짐.

---

## GOOD/BAD Pair 4: Scout 위임 vs 직접 Glob/Grep

**GOOD** -- Scout Agent가 탐색+판단+요구사항서 작성:

```
Step A:
1. Agent(scout) 생성
   prompt에 theme, section, target, last_commit 등 전달
   scout가 내부에서 Glob/Grep 소화 → 요구사항서 작성
2. Main은 경량 결과 수신: { status, requirement_spec, theme_definition } ~200tok
3. Main 컨텍스트 소비: ~200 tokens
```

WHY: Glob/Grep 결과(경로 수십 개 × 긴 절대경로)가 scout의 일회용 컨텍스트에서 소화됨. Main에는 완성된 요구사항서만 반환. 40테마 시 ~192,000 토큰 절감. 오토 컴팩트 방지.

**BAD** -- Main이 직접 Glob/Grep 수행:

```
Step A (위반):
1. Main: Glob(**/*) → 60개 절대경로 ~3000tok 직접 적재
2. Main: Grep(keywords) → ~2000tok 직접 적재
3. Main: 요구사항서 작성
   -> 테마당 ~5,000 tokens × 40테마 = ~200,000 tokens 소비  <- CRITICAL 8 위반
   -> 8테마 내 오토 컴팩트 발생 → 파이프라인 붕괴
```

WHY: Main 컨텍스트 보호. 40테마 시 ~192,000 토큰 절감. 오토 컴팩트 방지.

---

## GOOD/BAD Pair 5: Full Reset vs Resume

**GOOD** -- 재시도 시 새 에이전트 생성:

```
[1차] Agent(docu-writer, NEW) -> draft_path에 Write -> Agent(critic, NEW) -> fail
[재시도] Agent(docu-writer, NEW) -> draft_path에 Write -> Agent(critic, NEW) -> pass
```

WHY: 새 에이전트는 깨끗한 컨텍스트. 비용 1.0x. 이전 오류 패턴 비전파.

**BAD** -- resume:

```
[1차] Agent(docu-writer) -> fail
[재시도] Agent(docu-writer, RESUME) -> 이전 컨텍스트 + 오류 잔존
```

WHY: Resume 시 비용 1.8~3.6x. 동일 오류 반복 가능.

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
| 10 | **md 인라인 전달** | docu-writer가 md를 문자열로 반환하거나 Main이 critic에 md 문자열을 삽입 | -- (토큰 낭비) |
| 11 | **직접 Glob/Grep** | scout를 호출하지 않고 Main이 직접 파일 탐색 수행 (토큰 낭비, 오토 컴팩트 위험) | CRITICAL 8 |

</Failure_Modes_To_Avoid>
