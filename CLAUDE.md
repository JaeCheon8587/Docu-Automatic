# Docusaurus 문서 자동화 파이프라인

팀 내 제품 레포의 git push 시 AI가 기술 문서를 자동 생성하여 Docusaurus 사이트로 빌드/배포하는 파이프라인.
대상 언어: C++, C#, JavaScript, Python. 코드에 주석이 거의 없어 AI가 코드를 직접 분석하여 문서를 생성한다.

```
git push (제품 레포) → CI → AI .md 생성 (테마 순회) → docs-auto 브랜치 → 중앙 배치 빌드 → Docusaurus 배포
```

## 폴더 구조

| 경로 | 역할 |
|------|------|
| `docs-automation/` | **프로젝트 본체** — 설계 문서 + 산출물 (스킬/에이전트) |
| `AI Article/everything-claude-code-main/` | **참조** — ECC 스킬/에이전트 패턴 (58개 스킬) |
| `AI Article/oh-my-claudecode-main/` | **참조** — OMC 스킬/에이전트 패턴 (35개 스킬) |
| `AI Article/SKILL_Standard_Report.md` | **참조** — 스킬 작성 표준 (ECC+OMC 분석 결과) |
| `SW-RCS-Documentation-구조분석.md` | **참조** — 기존 Docusaurus 사이트 구조 분석 |
| `SW-RCS-Documentation-개요분석.md` | **참조** — 기존 Docusaurus 사이트 개요 분석 |

> **규칙**: `docs-automation/`만 수정 대상. 나머지는 읽기 전용 참조.

## 문서 체계 (docs-automation/)

4계층 구조. 상위 문서가 하위 문서의 WHY를 제공한다.

| 계층 | 파일 | 역할 |
|------|------|------|
| L0 | `00.README.md` | 엔트리포인트 — 30초 전체 파악 |
| L1 | `01.설계-결정사항.md` | WHY+WHAT — 설계 결정과 근거 |
| L2 | `02.아키텍처.md` | HOW (상세) — 에이전트/스킬 구현 설계 v3 |
| L3 | `03.SETUP-GUIDE.md` | HOW (실행) — 인프라 설치/운영 |

## 아키텍처 핵심 (v3)

### 2단계 오케스트레이션

- **메인 (task-pipeline)**: 테마 루프 관리 + 작업 오케스트레이션 생성/폐기 + 상태 추적 (execution-log.md)
- **작업 오케스트레이션**: 테마별 4단계 사이클 처리 (판단 → md 작성 → 검증 → 재시도 판정)

### 테마 순차 순회

**1차 스코프 (4개 테마)**:
`intro → getting-started/requirements → architecture/overview → architecture/component-diagram`

1 테마 = 1 md 파일. 테마 간 독립. 병렬 처리 금지. 테마별 차이는 작업 오케스트레이션이 작성하는 요구사항서의 내용으로 구현.

### Full Reset 전략

매 테마마다 모든 에이전트(작업 오케스트레이션, docu-writer, critic) 신규 생성. 이전 에이전트 resume 금지.
근거: 컨텍스트 오염 방지, 절차 준수 안정성, 디버깅 용이성, 토큰 비용 절감 (1.0x vs Resume 1.8~3.6x).

### 재시도

검증 실패 시 작업 오케스트레이션 내부에서 docu-writer 재호출 (최대 2회). 2회 초과 시 `auto_generated_warning` 태그 부착.
3개 테마 연속 FAIL 시 전체 파이프라인 중단.

## I/O 계약

**메인 → 작업 오케스트레이션 (입력)**:

| 구분 | 항목 |
|------|------|
| 메인 전달 | `theme`, `section`, `output_path` |
| 스킬 지시로 자체 조달 | git diff, theme_definition (theme-definitions.md), 관련 코드 파일 |

**작업 오케스트레이션 → 메인 (반환 5필드)**:

```yaml
STATUS: PASS | SKIPPED | WARNING | FAIL
THEME: {theme_id}
FILE: {output_path}
SUMMARY: "변경 내용 한 줄 요약"
MD_CONTENT: |
  (YAML frontmatter 포함 .md 전체 내용)
```

## 산출물 목록

| # | 파일 | 유형 | 템플릿 | 상태 |
|---|------|------|--------|------|
| 1 | `skills/task-pipeline/SKILL.md` | 테마 순회 파이프라인 스킬 | Lite | **완료** |
| 2 | `skills/task-orchestrator-skill/SKILL.md` | 작업 오케스트레이션 스킬 | Full | **완료** |
| 3 | `skills/docu-writer-skill/SKILL.md` | md 작성 스킬 | Lite | **완료** |
| 4 | `skills/critic-skill/SKILL.md` | 검증 스킬 | Lite | **완료** |
| 5 | `skills/theme-definitions/SKILL.md` | 페이지 단위 테마 정의 (1차: 4개) | — | **완료** |
| 6 | `agents/task-pipeline.md` | 테마 순회 파이프라인 에이전트 | — | **완료** |
| 7 | `agents/task-orchestrator.md` | 작업 오케스트레이션 에이전트 | — | **완료** |
| 8 | `agents/docu-writer.md` | md 작성 에이전트 | — | **완료** |
| 9 | `agents/critic.md` | 검증 에이전트 | — | **완료** |

> 모든 산출물은 `docs-automation/` 하위에 생성.

## 스킬/에이전트 작성 표준

`AI Article/SKILL_Standard_Report.md` 기준. 3-Layer 하이브리드 형식을 사용한다.

### 3-Layer 구조

| 계층 | 형식 | 역할 |
|------|------|------|
| 메타데이터 | YAML frontmatter | 스킬 발견, 카탈로그 (`name`, `description`, `origin` 필수) |
| 행동 구간 | XML 시맨틱 태그 | LLM 역할 인식, 경계 준수 (`<Purpose>`, `<Use_When>` 등) |
| 콘텐츠 | Markdown | 코드블록, 테이블, 체크리스트 |

### 에이전트 .md 작성 규칙

YAML frontmatter로 에이전트 메타 정의 + XML 태그로 역할/출력 형식 정의:

```yaml
---
name: agent-name
description: 한 줄 설명
model: sonnet  # 또는 haiku, opus
tools: [Read, Write, Glob, Grep, Bash, Agent]
---
```

### 스킬 Full Template (오케스트레이션용 — 절차.md, task-orchestrator-skill.md)

필수 태그: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Why_This_Exists>`, `<Execution_Policy>`, `<Steps>`, `<Tool_Usage>`, `<Examples>` (Good/Bad+WHY), `<Escalation_And_Stop_Conditions>`, `<Final_Checklist>`

### 스킬 Lite Template (단순 작업용 — docu-writer-skill.md, critic-skill.md)

필수 태그: `<Purpose>`, `<Use_When>`, `<Do_Not_Use_When>`, `<Final_Checklist>`
콘텐츠: Markdown `##` 섹션 + GOOD/BAD 코드 쌍

### 핵심 규칙

- Good/Bad 예시에 반드시 WHY 포함 (원칙 학습)
- 반복 루프에 Hard Cap + Soft Warning + 정체 감지
- 출력 형식을 파일 경로/구조/필수 필드까지 명시 (I/O 계약)
- 행동 구간은 XML, 콘텐츠 구간은 Markdown (혼용 금지)
- 크기 가이드라인: 유틸리티 50-100줄, 표준 150-300줄, 복합 300-600줄, 최대 800줄

## 참조 경로

| 용도 | 경로 |
|------|------|
| ECC 에이전트 예시 | `AI Article/everything-claude-code-main/agents/*.md` |
| ECC 스킬 예시 | `AI Article/everything-claude-code-main/.agents/skills/*/SKILL.md` |
| OMC 에이전트 예시 | `AI Article/oh-my-claudecode-main/agents/*.md` |
| OMC 스킬 예시 | `AI Article/oh-my-claudecode-main/skills/*/SKILL.md` |
| 스킬 작성 표준 | `AI Article/SKILL_Standard_Report.md` |
| Docusaurus 사이트 구조 | `SW-RCS-Documentation-구조분석.md` |
| 설계 결정 근거 | `docs-automation/01.설계-결정사항.md` |
| 상세 구현 설계 | `docs-automation/02.아키텍처.md` |
