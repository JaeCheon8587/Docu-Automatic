> **문서 맵**: [README](00.README.md) → [설계 결정사항](01.설계-결정사항.md) → [아키텍처](02.아키텍처.md) → [셋업 가이드](03.SETUP-GUIDE.md) | **PRD**

# PRD: 산출물 8개 구현 요구사항

8개 파일(스킬 4 + 에이전트 3 + 참조 1) 구현 → 파이프라인 실행 가능 상태 달성.
**범위 외**: CI/CD 설정, Docusaurus 빌드, 인프라, 제품별 커스터마이징.

## 1. 구현 순서 (의존성 기반)

| Phase | 파일 | 근거 |
|-------|------|------|
| 1 | `skills/theme-definitions/SKILL.md` | 의존성 없음. 다른 파일이 참조 |
| 2 (병렬) | `agents/docu-writer.md` + `skills/docu-writer-skill/SKILL.md` | 리프 노드 |
| 2 (병렬) | `agents/critic.md` + `skills/critic-skill/SKILL.md` | 리프 노드 |
| 3 | `agents/task-orchestrator.md` + `skills/task-orchestrator-skill/SKILL.md` | Phase 2 참조 |
| 4 | `skills/절차/SKILL.md` | Phase 3 참조 |

---

## 2. 파일별 요구사항

### 2.1 skills/theme-definitions/SKILL.md

**목적**: 7개 테마의 관점/독자/스타일/포함/제외를 구조화한 참조 데이터.
**유형**: 참조 데이터 | **크기**: 50-100줄

- **입력**: 없음 (정적 데이터)
- **출력**: 작업 오케스트레이션, critic이 읽기 전용 참조

**필수 구조**: 테마당 YAML 블록 (7개). 각 블록에 다음 필드 포함:
`id`, `name`, `perspective`, `audience`, `writing_style`, `must_cover`, `do_not_cover`, `docs_path`

**핵심 규칙**:
1. 01.설계-결정사항.md 섹션4의 7개 테마와 1:1 대응
2. `must_cover`/`do_not_cover`가 동일 깊이/범위에서 테마 간 중복 없음 (예: '아키텍처'를 overview에서 개요 수준으로, tech-spec에서 구현 수준으로 다루는 것은 허용)
3. YAML frontmatter 포함 (name, description, origin — Claude Code 스킬 발견 규칙 준수)

**수용 기준**: [ ] 7개 테마 정의 완비 [ ] 8개 필드/테마 존재 [ ] 테마 간 must/do_not 교차 검증됨

### 2.2 agents/docu-writer.md

**목적**: 요구사항서 기반 .md 작성 에이전트.
**유형**: 에이전트 .md (YAML + XML) | **크기**: 80-120줄 | **참조**: OMC `agents/writer.md` (87줄)

- **입력**: 요구사항서 + 관련 코드 + (재시도 시) critic 피드백
- **출력**: YAML frontmatter 포함 .md 전체 내용 (문자열)

**필수 frontmatter**: `name: docu-writer`, `model: sonnet`, `tools: [Read, Grep, Glob]`
**필수 XML 태그**: `<Role>`, `<Input_Contract>`, `<Output_Format>`, `<Rules>`
- `<Rules>`에 `skills/docu-writer-skill/SKILL.md`를 Read로 읽고 따르도록 지시 포함

**핵심 규칙**:
1. 요구사항서의 perspective/writing_style 엄격 준수
2. do_not_cover 항목 절대 포함 금지
3. frontmatter 스키마: 01.설계-결정사항.md 섹션5 준수
4. Agent 도구 사용 금지 (리프 노드)

**수용 기준**: [ ] frontmatter 4필드 존재 [ ] 입력 계약 정의됨 [ ] 출력 형식 정의됨 [ ] 코드 분석 지시 포함

### 2.3 agents/critic.md

**목적**: frontmatter 유효성 + 테마 적합성 검증 에이전트.
**유형**: 에이전트 .md (YAML + XML) | **크기**: 80-120줄 | **참조**: OMC `agents/critic.md` (102줄)

- **입력**: md 내용 + theme_definition(해당 테마) + 요구사항서
- **출력**: `result`/`frontmatter_valid`/`theme_fitness`/`critic_feedback` (YAML 4필드)

**필수 frontmatter**: `name: critic`, `model: sonnet`, `tools: [Read, Grep, Glob]`
**필수 XML 태그**: `<Role>`, `<Input_Contract>`, `<Output_Format>`, `<Verification_Steps>`
- `<Rules>`에 `skills/critic-skill/SKILL.md`를 Read로 읽고 따르도록 지시 포함

**핵심 규칙**:
1. 2단계 검증: (1) frontmatter 규칙 기반 (2) 테마 적합성 AI 판단
2. frontmatter 필수 필드: title, sidebar_label, sidebar_position, product, theme, auto_generated, source_files, last_commit, generated_at
3. fail 시 critic_feedback에 문제 지점(라인 번호) 구체 기술
4. frontmatter_valid 또는 theme_fitness 중 하나라도 fail → result=fail

**수용 기준**: [ ] 2단계 검증 정의됨 [ ] 필수 필드 목록 명시 [ ] 출력 4필드 정의됨 [ ] 피드백이 docu-writer 수정 가능한 구조

### 2.4 agents/task-orchestrator.md

**목적**: 단일 테마 4단계 사이클(판단→작성→검증→재시도) 오케스트레이터 에이전트.
**유형**: 에이전트 .md (YAML + XML) | **크기**: 100-150줄 | **참조**: OMC `agents/executor.md`

- **입력**: theme, product_id, output_path (메인 전달) + git diff, theme_definition, code_files (자체 조달)
- **출력**: STATUS/THEME/FILE/SUMMARY/MD_CONTENT (5필드)

**필수 frontmatter**: `name: task-orchestrator`, `model: sonnet`, `tools: [Read, Grep, Glob, Bash, Agent]`
**필수 XML 태그**: `<Role>`, `<Input_Contract>`, `<Output_Format>`, `<Orchestration_Flow>`, `<Rules>`
- `<Rules>`에 `skills/task-orchestrator-skill/SKILL.md`를 Read로 읽고 따르도록 지시 포함

**핵심 규칙**:
1. Agent 도구로 docu-writer, critic 생성 (하위 에이전트 관리)
2. 재시도 시에도 docu-writer, critic 새로 생성 (Full Reset)
3. 재시도 최대 2회 Hard Cap
4. docu-writer 결과 수신 전 critic 호출 금지 (순차 필수)
5. 모든 경로(PASS/SKIPPED/WARNING/FAIL)에서 5필드 반환

**수용 기준**: [ ] 입력 6항목 정의됨 [ ] 4단계(A~D) 흐름 정의됨 [ ] 반환 5필드 형식 명시 [ ] docu-writer/critic 생성 지시 포함 [ ] Full Reset 명시

### 2.5 skills/docu-writer-skill/SKILL.md

**목적**: docu-writer 에이전트의 문서 작성 지침 스킬.
**유형**: 스킬 Lite Template | **크기**: 120-200줄 | **참조**: ECC `security-review` SKILL.md 구조

**필수 frontmatter**: name, description, origin
**필수 태그**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Final_Checklist>`
**필수 Markdown 섹션**: `## 문서 작성 규칙`, `## YAML Frontmatter 작성법`, `## GOOD/BAD 예시` (2쌍+WHY), `## Quick Reference`

**핵심 규칙**:
1. 요구사항서의 perspective/writing_style 최우선 준수
2. do_not_cover 항목 포함 시 반드시 제거
3. frontmatter 스키마: 01.설계-결정사항.md 섹션5 준수
4. 코드 분석 시 Read/Grep/Glob 활용 지시 포함

**수용 기준**: [ ] Lite 필수 태그 4개 존재 [ ] GOOD/BAD+WHY 2쌍 이상 [ ] frontmatter 작성법 포함 [ ] 테마 관점 준수 지침 명시

### 2.6 skills/critic-skill/SKILL.md

**목적**: critic 에이전트의 검증 지침 스킬.
**유형**: 스킬 Lite Template | **크기**: 120-200줄 | **참조**: ECC `security-review` SKILL.md 구조

**필수 frontmatter**: name, description, origin
**필수 태그**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Final_Checklist>`
**필수 Markdown 섹션**: `## 1단계: Frontmatter 검증` (필수 필드 체크리스트), `## 2단계: 테마 적합성 검증`, `## 출력 형식`, `## GOOD/BAD 예시` (2쌍+WHY)

**핵심 규칙**:
1. 1단계(frontmatter)는 규칙 기반 판단, 2단계(테마 적합성)는 AI 판단
2. fail 시 critic_feedback을 docu-writer 수정 가능하도록 구체적 작성 (라인 번호 포함)
3. frontmatter_valid/theme_fitness 중 하나라도 fail → result=fail
4. theme-definitions/SKILL.md를 참조하여 적합성 판단

**수용 기준**: [ ] Lite 필수 태그 4개 존재 [ ] 2단계 검증 분리됨 [ ] GOOD/BAD+WHY 2쌍 이상 [ ] 출력 4필드 정의됨 [ ] 필수 필드 체크리스트 포함

### 2.7 skills/task-orchestrator-skill/SKILL.md

**목적**: task-orchestrator 에이전트의 4단계 사이클 행동을 정의하는 오케스트레이션 스킬.
**유형**: 스킬 Full Template | **크기**: 250-400줄 | **참조**: OMC `ralph` SKILL.md (208줄), `plan` SKILL.md (257줄)

**필수 frontmatter**: name, description, origin, tools
**필수 XML 태그 (Full 전체)**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Why_This_Exists>`, `<Execution_Policy>`, `<Steps>`, `<Tool_Usage>`, `<Examples>`, `<Escalation_And_Stop_Conditions>`, `<Final_Checklist>`

**`<Steps>` 4단계**:
- **A. 판단+요구사항서**: git diff 분석 → 테마 필요 여부 → Yes이면 요구사항서 작성 (YAML 형식)
  - 요구사항서 필드: `theme`, `perspective`, `audience`, `writing_style`, `must_cover[]`, `do_not_cover[]`, `key_changes` (git diff 요약), `related_files[]`, `output_path`
- **B. md 작성**: docu-writer Agent 생성 → 요구사항서+코드 전달 → md 수신
- **C. 검증**: critic Agent 생성 → md+테마정의+요구사항서 전달 → pass/fail 수신
- **D. 재시도 판정**: pass→5필드 반환, fail(1~2회)→B 복귀(피드백 포함), fail(2회 초과)→WARNING+경고태그

**`<Execution_Policy>` CRITICAL 제약**:
1. docu-writer 결과 수신 전 critic 호출 금지
2. critic 결과 수신 전 반환 금지
3. 재시도 시 docu-writer, critic 새로 생성 (Full Reset)
4. 재시도 최대 2회 (Hard Cap)
5. Step A에서 No → 즉시 STATUS=SKIPPED 반환

**수용 기준**: [ ] Full 필수 태그 10개 존재 [ ] Step A~D 정의됨 [ ] I/O 계약(입력6/출력5) 정의됨 [ ] CRITICAL 제약 3개 이상 [ ] Escalation에 Hard Cap+무응답 처리 포함 [ ] GOOD/BAD+WHY 2쌍 이상 [ ] 요구사항서 포함 항목 명시

### 2.8 skills/절차/SKILL.md

**목적**: 메인 에이전트의 전체 행동을 정의하는 최상위 오케스트레이션 스킬 (테마 루프 + 상태 추적).
**유형**: 스킬 Full Template | **크기**: 300-500줄 | **참조**: OMC `ralph` SKILL.md, `team` SKILL.md (933줄)

**필수 frontmatter**: name, description, origin, tools
**필수 XML 태그 (Full 전체)**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Why_This_Exists>`, `<Execution_Policy>`, `<Steps>`, `<Tool_Usage>`, `<Examples>`, `<Escalation_And_Stop_Conditions>`, `<Final_Checklist>`

**`<Steps>` 3 Phase**:
- **Phase 1 — 입력 수집**: product_id 확인, git diff 실행, 관련 파일 경로 수집
- **Phase 2 — 테마 루프**: 7개 테마 순차 순회 × (작업 오케스트레이션 Agent 생성 → 반환 수신 → 파싱/저장/로깅 → 에이전트 폐기)
- **Phase 3 — 완료 처리**: execution-log.md 최종 기록, docs-auto 브랜치 커밋, 결과 요약

**`<Execution_Policy>` CRITICAL 제약**:
1. 테마 순회 반드시 순차 (병렬 절대 금지)
2. 작업 오케스트레이션 반환 수신 전 다음 테마 진행 금지
3. 매 테마마다 새 작업 오케스트레이션 생성 (Full Reset)
4. 메인은 판단/요구사항서/재시도 직접 수행 안 함 (위임)
5. execution-log.md에 매 테마 결과 즉시 기록

**상태 추적**: execution-log.md (파일 기반). 필드: pipeline_id, product_id, started_at, themes[] (theme, status, file, summary).

**핵심 규칙**:
1. 테마 순서 고정: overview → features → tech-spec → config → api → lib → guide
2. 3개 테마 연속 FAIL → 전체 파이프라인 중단 + 로그
3. STATUS별 처리: PASS/WARNING → Write 도구로 .md 저장, SKIPPED → 다음 테마, FAIL → 에러 로깅
4. MD_CONTENT를 파싱하여 output_path에 저장

**수용 기준**: [ ] Full 필수 태그 10개 존재 [ ] Phase 1~3 정의됨 [ ] 7개 테마 순차 순회 명시 [ ] execution-log.md 형식 정의됨 [ ] CRITICAL 제약 3개 이상 [ ] 3연속 FAIL→중단 포함 [ ] GOOD/BAD+WHY 2쌍 이상 [ ] Agent 호출 패턴 명시 [ ] 반환값 파싱+저장 절차 정의됨

---

## 3. I/O 계약 요약 (교차 검증용)

| # | 송신 → 수신 | 데이터 |
|---|------------|--------|
| 1 | 메인 → task-orch | `theme`, `product_id`, `output_path` |
| 2 | task-orch 자체 조달 | git diff (Bash), theme_definition (Read), code_files (Read/Grep/Glob) |
| 3 | task-orch → docu-writer | 요구사항서 (YAML 형식) + 관련 코드 + (재시도 시) critic 피드백 |
| 4 | docu-writer → task-orch | YAML frontmatter 포함 .md 내용 (문자열) |
| 5 | task-orch → critic | md 내용 + theme_definition + 요구사항서 |
| 6 | critic → task-orch | result/frontmatter_valid/theme_fitness/critic_feedback |
| 7 | task-orch → 메인 | STATUS/THEME/FILE/SUMMARY/MD_CONTENT (5필드) |

**교차 검증**: #3 입력 = docu-writer Input_Contract, #5 입력 = critic Input_Contract, #7 출력 = 절차.md 파싱 대상.
