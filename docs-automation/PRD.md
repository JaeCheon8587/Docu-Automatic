> **문서 맵**: [README](00.README.md) → [설계 결정사항](01.설계-결정사항.md) → [아키텍처](02.아키텍처.md) → [셋업 가이드](03.SETUP-GUIDE.md) | **PRD**

# PRD: 산출물 6개 구현 요구사항

6개 파일(스킬 3 + 에이전트 2 + 참조 1) 구현 → 파이프라인 실행 가능 상태 달성.
**범위 외**: CI/CD 설정, Docusaurus 빌드, 인프라, 제품별 커스터마이징.

## 1. 구현 순서 (의존성 기반)

| Phase | 파일 | 근거 |
|-------|------|------|
| 1 | `skills/theme-definitions/SKILL.md` | 의존성 없음. 다른 파일이 참조 |
| 2 (병렬) | `agents/docu-writer.md` + `skills/docu-writer-skill/SKILL.md` | 리프 노드 |
| 2 (병렬) | `agents/critic.md` + `skills/critic-skill/SKILL.md` | 리프 노드 |
| 3 | `skills/task-pipeline/SKILL.md` | Phase 2 참조 |

---

## 2. 파일별 요구사항

### 2.1 skills/theme-definitions/SKILL.md

**목적**: 4개 테마의 관점/독자/스타일/포함/제외를 구조화한 참조 데이터.
**유형**: 참조 데이터 | **크기**: 50-100줄

- **입력**: 없음 (정적 데이터)
- **출력**: Main(task-pipeline 스킬), critic이 읽기 전용 참조

**필수 구조**: 테마당 YAML 블록 (4개). 각 블록에 다음 필드 포함:
`id`, `name`, `section`, `perspective`, `audience`, `writing_style`, `must_cover`, `do_not_cover`, `docs_path`

**핵심 규칙**:
1. 01.설계-결정사항.md 섹션4의 4개 테마와 1:1 대응
2. `must_cover`/`do_not_cover`가 동일 깊이/범위에서 테마 간 중복 없음
3. YAML frontmatter 포함 (name, description, origin -- Claude Code 스킬 발견 규칙 준수)

**수용 기준**: [ ] 4개 테마 정의 완비 [ ] 필수 필드/테마 존재 [ ] 테마 간 must/do_not 교차 검증됨

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
2. frontmatter 필수 필드: title, sidebar_label, sidebar_position, section, theme, auto_generated, source_files, last_commit, generated_at
3. fail 시 critic_feedback에 문제 지점(라인 번호) 구체 기술
4. frontmatter_valid 또는 theme_fitness 중 하나라도 fail -> result=fail

**수용 기준**: [ ] 2단계 검증 정의됨 [ ] 필수 필드 목록 명시 [ ] 출력 4필드 정의됨 [ ] 피드백이 docu-writer 수정 가능한 구조

### 2.4 skills/docu-writer-skill/SKILL.md

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

### 2.5 skills/critic-skill/SKILL.md

**목적**: critic 에이전트의 검증 지침 스킬.
**유형**: 스킬 Lite Template | **크기**: 120-200줄 | **참조**: ECC `security-review` SKILL.md 구조

**필수 frontmatter**: name, description, origin
**필수 태그**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Final_Checklist>`
**필수 Markdown 섹션**: `## 1단계: Frontmatter 검증` (필수 필드 체크리스트), `## 2단계: 테마 적합성 검증`, `## 출력 형식`, `## GOOD/BAD 예시` (2쌍+WHY)

**핵심 규칙**:
1. 1단계(frontmatter)는 규칙 기반 판단, 2단계(테마 적합성)는 AI 판단
2. fail 시 critic_feedback을 docu-writer 수정 가능하도록 구체적 작성 (라인 번호 포함)
3. frontmatter_valid/theme_fitness 중 하나라도 fail -> result=fail
4. theme-definitions/SKILL.md를 참조하여 적합성 판단

**수용 기준**: [ ] Lite 필수 태그 4개 존재 [ ] 2단계 검증 분리됨 [ ] GOOD/BAD+WHY 2쌍 이상 [ ] 출력 4필드 정의됨 [ ] 필수 필드 체크리스트 포함

### 2.6 skills/task-pipeline/SKILL.md

**목적**: Main CLI의 전체 행동을 정의하는 통합 오케스트레이션 스킬 (테마 루프 + 판단 + 요구사항서 + Agent 위임 + 재시도 + 저장).
**유형**: 스킬 Full Template | **크기**: 300-500줄 | **참조**: OMC `ralph` SKILL.md, `plan` SKILL.md

**필수 frontmatter**: name, description, origin, tools
**필수 XML 태그 (Full 전체)**: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Why_This_Exists>`, `<Execution_Policy>`, `<Steps>`, `<Tool_Usage>`, `<Examples>`, `<Escalation_And_Stop_Conditions>`, `<Final_Checklist>`

**`<Steps>` 구조**:
- **Phase 1 -- 입력 확인**: target 경로 확인
- **Phase 2 -- 테마 루프**: 4개 테마 순차 순회 x (Step A: 판단+요구사항서 -> Step B: Agent(docu-writer) -> Step C: Agent(critic) -> Step D: 재시도 판정+저장+로깅)
- **Phase 3 -- 완료 처리**: execution-log.md 최종 기록, 결과 요약

**`<Execution_Policy>` CRITICAL 제약 7개**:
1. .md 내용 직접 작성 금지 -> Agent(docu-writer)에 위임
2. 검증 직접 수행 금지 -> Agent(critic)에 위임
3. docu-writer 결과 수신 전 critic 호출 금지 (순차)
4. 테마 순회 반드시 순차 (병렬 금지)
5. 테마마다 새 docu-writer/critic 생성 (Full Reset)
6. 재시도 최대 2회 Hard Cap
7. 3연속 FAIL 시 파이프라인 중단

**수용 기준**: [ ] Full 필수 태그 10개 존재 [ ] Phase 1~3 + Step A~D 정의됨 [ ] CRITICAL 제약 7개 [ ] Escalation에 Hard Cap+무응답 처리 포함 [ ] GOOD/BAD+WHY 4쌍 이상 [ ] Agent 호출 리터럴 템플릿 명시 [ ] execution-log 형식 정의됨

---

## 3. I/O 계약 요약 (교차 검증용)

| # | 송신 → 수신 | 데이터 |
|---|------------|--------|
| 1 | Main 자체 조달 | git diff (Bash), theme_definition (Read), code_files (Read/Grep/Glob) |
| 2 | Main → docu-writer | 요구사항서 (YAML) + 관련 코드 + (재시도 시) critic 피드백 |
| 3 | docu-writer → Main | YAML frontmatter 포함 .md 내용 (문자열) |
| 4 | Main → critic | md 내용 + theme_definition + 요구사항서 |
| 5 | critic → Main | result/frontmatter_valid/theme_fitness/critic_feedback (4필드) |

**교차 검증**: #2 입력 = docu-writer Input_Contract, #4 입력 = critic Input_Contract.
