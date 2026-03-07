# Docusaurus 문서 자동화 파이프라인

git push 시 AI가 코드를 분석하여 기술 문서(.md)를 자동 생성하고, Docusaurus 사이트로 빌드/배포하는 파이프라인.

```
git push (제품 레포) → CI → AI .md 생성 (테마 순회) → docs-auto 브랜치 → 중앙 배치 빌드 → Docusaurus 배포
```

## 배경

- 팀 내 여러 제품군이 개별 Git 레포로 관리됨 (C++, C#, JavaScript, Python)
- 기존 코드베이스에 Doxygen 주석이 거의 없어 AI가 코드를 직접 분석하여 문서 생성

## 아키텍처 (v4)

1단계 오케스트레이션 구조. Main CLI가 테마를 순차 순회하며 4단계 사이클을 반복한다.

```
Main CLI (Level 0): 테마 루프 + 판단 + 요구사항서 + 오케스트레이션 + 재시도 + 저장
  ├── docu-writer (Level 1 Agent): md 작성 (리프)
  └── critic (Level 1 Agent): 독립 검증 (리프)
```

**테마별 4단계 사이클:**

| 단계 | 수행자 | 내용 |
|------|--------|------|
| 판단 | Main | git diff + 테마 정의 분석 → 문서화 필요 여부 판단 → 요구사항서 작성 |
| 작성 | Agent(docu-writer) | 요구사항서 + 관련 코드 → YAML frontmatter 포함 .md 생성 |
| 검증 | Agent(critic) | frontmatter 유효성 + 테마 적합성 2단계 검증 |
| 저장 | Main | .md 파일 저장 + execution-log 기록. Fail 시 최대 2회 재시도 |

**핵심 전략:**
- **Full Reset**: 매 테마마다 docu-writer/critic 신규 생성 (컨텍스트 오염 방지, 비용 1.0x)
- **순차 순회**: 테마 간 병렬 처리 금지. 품질과 안전성 우선
- **3연속 FAIL 시 파이프라인 중단**

## 1차 스코프 (4개 테마)

| 테마 ID | 문서 관점 | 대상 독자 |
|---------|----------|----------|
| `intro` | 프로젝트 전체 소개 | 모든 방문자 |
| `getting-started/requirements` | 설치/실행에 필요한 환경과 조건 | 설치자, 운영자 |
| `architecture/overview` | 시스템 전체 구조와 모듈 간 관계 | 신규 팀원, 개발자 |
| `architecture/component-diagram` | S/W별 컴포넌트 구성 | 개발자 |

## 프로젝트 구조

```
docs-automation/
├── docs/                              # 설계 문서 (4계층)
│   ├── 00.README.md                   #   L0: 엔트리포인트
│   ├── 01.설계-결정사항.md              #   L1: WHY+WHAT
│   ├── 02.아키텍처.md                  #   L2: HOW (상세)
│   └── 03.SETUP-GUIDE.md              #   L3: HOW (실행)
├── agents/                            # AI 에이전트 정의
│   ├── docu-writer.md                 #   md 작성 에이전트 (Level 1 리프)
│   └── critic.md                      #   검증 에이전트 (Level 1 리프)
├── skills/                            # AI 스킬 정의
│   ├── task-pipeline/SKILL.md         #   통합 오케스트레이션 스킬 (Full)
│   ├── docu-writer-skill/SKILL.md     #   md 작성 스킬 (Lite)
│   ├── critic-skill/SKILL.md          #   검증 스킬 (Lite)
│   └── theme-definitions/SKILL.md     #   4개 테마 정의 (참조 데이터)
└── PRD.md                             # 산출물 구현 요구사항
```

## 산출물 현황

| # | 파일 | 설명 | 상태 |
|---|------|------|------|
| 1 | `skills/task-pipeline/SKILL.md` | 통합 오케스트레이션 (루프 + 판단 + Agent 위임 + 재시도 + 저장) | 완료 |
| 2 | `skills/docu-writer-skill/SKILL.md` | md 작성 규칙 | 완료 |
| 3 | `skills/critic-skill/SKILL.md` | 검증 규칙 | 완료 |
| 4 | `skills/theme-definitions/SKILL.md` | 페이지 단위 테마 정의 (1차: 4개) | 완료 |
| 5 | `agents/docu-writer.md` | md 작성기 (Level 1 리프) | 완료 |
| 6 | `agents/critic.md` | 검증기 (Level 1 리프) | 완료 |

## 실행 환경

- **런타임**: Claude Code CLI
- **실행 방식**: Main CLI가 `skills/task-pipeline/SKILL.md`를 읽고 직접 실행

## 문서 맵

상위 문서가 하위 문서의 WHY를 제공하는 4계층 구조:

| 계층 | 문서 | 역할 |
|------|------|------|
| L0 | [README](docs/00.README.md) | 엔트리포인트 -- 30초 파악 |
| L1 | [설계 결정사항](docs/01.설계-결정사항.md) | WHY + WHAT -- 설계 결정과 근거 |
| L2 | [아키텍처](docs/02.아키텍처.md) | HOW (상세) -- 에이전트/스킬 구현 설계 v4 |
| L3 | [셋업 가이드](docs/03.SETUP-GUIDE.md) | HOW (실행) -- 인프라 설치/운영 |
