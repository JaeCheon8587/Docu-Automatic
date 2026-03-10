---
name: critic-skill
description: frontmatter 유효성 + 테마 적합성 2단계 검증 지침
origin: docs-automation
---

<Purpose>
docu-writer가 작성한 .md 파일의 품질을 2단계(frontmatter 검증 + 테마 적합성 검증)로 판정하는 지침을 제공한다.
critic 에이전트가 이 스킬을 Read로 읽고 따른다.
</Purpose>

<Use_When>
- 작업 오케스트레이션이 critic 에이전트를 호출할 때
- docu-writer가 작성한 .md 내용의 검증이 필요할 때
- 재시도 후 수정된 .md의 재검증이 필요할 때
- frontmatter 스키마 준수 여부 확인이 필요할 때
</Use_When>

<Do_Not_Use_When>
- .md 작성 시 -> docu-writer-skill 사용
- 테마 필요 여부 판단 시 -> 작업 오케스트레이션이 직접 수행
- 파이프라인 전체 흐름 제어 시 -> 절차/SKILL.md 사용
- 테마 정의 자체를 수정할 때 -> theme-definitions/SKILL.md 직접 수정
</Do_Not_Use_When>

## 0단계: md 로드

검증 대상 md는 draft_path에서 Read로 직접 로드한다. prompt에 인라인으로 전달되지 않는다.

```
Read(draft_path) -> md_content 확보
```

파일이 없거나 비어있으면 즉시 `result: fail`, `critic_feedback: ["0단계: draft_path 파일 없음 또는 빈 파일"]` 반환.

## 1단계: Frontmatter 검증

규칙 기반 기계적 검증. 주관적 판단 없이 존재/타입/값만 확인.

| # | 필드 | 타입 | 검증 규칙 |
|---|------|------|----------|
| 1 | title | string | 존재 + 비어있지 않음 |
| 2 | sidebar_label | string | 존재 + 비어있지 않음 |
| 3 | sidebar_position | number | 존재 + 숫자 타입 |
| 4 | section | string | 존재 + 비어있지 않음 |
| 5 | theme | string | skills/theme-definitions/themes/ 디렉토리에 .yaml 파일이 존재하는 테마 ID |
| 6 | auto_generated | boolean | 존재 + true |
| 7 | source_files | string[] | 존재 + 최소 1개 항목 |
| 8 | last_commit | string | 존재 + 비어있지 않음 |
| 9 | generated_at | string | 존재 + ISO-8601 형식 (YYYY-MM-DDTHH:MM:SS) |

**판정**: 9개 모두 통과 -> `frontmatter_valid: true`
하나라도 실패 -> `frontmatter_valid: false` + critic_feedback에 실패 항목 기록

## 2단계: 테마 적합성 검증

AI 판단 기반. `theme_def_path`의 파일을 Read로 읽어 대조.

### 2-1. perspective 일치

문서 전체의 서술 관점이 theme_definition의 perspective와 부합하는지 확인.

- "설치/실행에 필요한 환경과 조건" 관점에서 내부 구현 세부사항 서술 -> fail
- "시스템 전체 구조와 제품 간 관계" 관점에서 클래스/함수 수준 세부사항 서술 -> fail

### 2-2. do_not_cover 위반 스캔

do_not_cover에 명시된 각 항목을 문서 전체에서 스캔.
위반 발견 시 해당 라인 번호를 기록.

- architecture/overview의 do_not_cover "설정 파라미터 (→ config)" -> 문서에 설정 파라미터 테이블 존재 시 fail

### 2-3. must_cover 완전성

must_cover의 모든 항목이 문서에서 다뤄졌는지 확인.
누락 항목이 있으면 feedback에 기록.

### 2-4. audience 적합성

용어 수준과 설명 깊이가 대상 독자에 적합한지 확인.

- "신규 팀원, 관리자" 대상에 코드 스니펫 과다 -> fail
- "개발자" 대상에 과도한 비전문 설명 -> 경고 (fail은 아님)

### 2-5. writing_style 부합

요구사항서에 명시된 writing_style을 문서가 따르는지 확인.

- "서술형"인데 전체가 테이블/목록 -> fail
- "참조형"인데 긴 서술 문단 위주 -> fail

**판정**: 5개 모두 통과 -> `theme_fitness: pass`
하나라도 실패 -> `theme_fitness: fail` + critic_feedback에 라인 번호 포함 위반 기록

## 출력 형식

YAML 4필드:

```yaml
# pass 예시
result: pass
frontmatter_valid: true
theme_fitness: pass
critic_feedback: []
```

```yaml
# fail 예시
result: fail
frontmatter_valid: true
theme_fitness: fail
critic_feedback:
  - "2단계: 라인 45-52에 RFBSConn 클래스 상속 구조 설명 -> overview의 do_not_cover '코드 수준 구현 세부사항' 해당"
  - "2단계: must_cover '시스템 요구사항' 항목 누락"
```

## GOOD/BAD 예시

### Pair 1: 구체적 피드백 vs 모호한 피드백

**GOOD**:
```yaml
result: fail
frontmatter_valid: true
theme_fitness: fail
critic_feedback:
  - "2단계: 라인 45-52에 RFBSConn 클래스 설명 -> do_not_cover '코드 수준 구현 세부사항 (-> tech-spec)' 해당"
  - "2단계: must_cover '시스템 요구사항' 항목이 문서에 없음"
```
WHY: 단계 번호, 라인 번호, 구체적 위반 내용을 포함하여 docu-writer가 정확히 어디를 수정해야 하는지 알 수 있음.

**BAD**:
```yaml
result: fail
frontmatter_valid: true
theme_fitness: fail
critic_feedback:
  - "문서가 테마에 맞지 않습니다"
  - "일부 내용이 누락되었습니다"
```
WHY: 라인 번호 없음, 구체적 위반 내용 없음. docu-writer가 어디를 수정해야 할지 알 수 없어 전면 재작성을 유발.

### Pair 2: AND 논리 판정 vs OR 논리 오류

**GOOD**:
```yaml
# frontmatter는 통과했지만 테마 적합성 실패 -> result=fail
result: fail
frontmatter_valid: true
theme_fitness: fail
critic_feedback:
  - "2단계: 라인 30-38에 API 엔드포인트 명세 -> overview의 do_not_cover 해당"
```
WHY: result는 frontmatter_valid AND theme_fitness의 AND 논리. 하나라도 fail이면 result=fail.

**BAD**:
```yaml
# frontmatter는 통과했지만 테마 적합성 실패 -> result=pass (오류!)
result: pass
frontmatter_valid: true
theme_fitness: fail
critic_feedback:
  - "2단계: 라인 30-38에 API 엔드포인트 명세 -> overview의 do_not_cover 해당"
```
WHY: theme_fitness=fail인데 result=pass로 판정. AND 논리 위반. docu-writer에 잘못된 통과 신호를 보냄.

## Quick Reference

| 항목 | 핵심 |
|------|------|
| md 로드 | draft_path에서 Read로 직접 로드 (인라인 전달 아님) |
| 검증 단계 | 0단계(md 로드) -> 1단계(frontmatter, 규칙 기반) -> 2단계(테마 적합성, AI 판단) |
| 판정 방법 | 1단계 9필드 + 2단계 5관점 -> AND 논리로 최종 result |
| feedback 형식 | "{단계}: 라인 {NN}-{MM}에 {구체적 위반 내용}" |
| 참조 파일 | theme_def_path (scout가 저장한 테마 정의) |
| 최종 result | frontmatter_valid=true AND theme_fitness=pass -> result=pass |

<Final_Checklist>
- [ ] 1단계에서 9개 필수 필드를 모두 기계적으로 검사했는가?
- [ ] 2단계에서 theme_definition의 5개 관점(perspective, do_not_cover, must_cover, audience, writing_style)을 모두 확인했는가?
- [ ] 모든 critic_feedback에 "단계: 라인 NN-MM에 {내용}" 형식이 포함되어 있는가?
- [ ] result가 frontmatter_valid AND theme_fitness의 AND 논리로 판정되었는가?
- [ ] do_not_cover 항목 각각을 문서 전체에서 스캔했는가?
</Final_Checklist>
