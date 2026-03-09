---
name: docu-writer-skill
description: 요구사항서 기반 기술 문서(.md) 작성 지침
origin: docs-automation
---

<Purpose>
작업 오케스트레이션이 작성한 요구사항서를 기반으로 Docusaurus용 .md 파일을 작성하는 지침을 제공한다.
docu-writer 에이전트가 이 스킬을 Read로 읽고 따른다.
</Purpose>

<Use_When>
- 작업 오케스트레이션이 docu-writer 에이전트를 호출할 때
- 요구사항서가 전달되어 .md 작성이 필요할 때
- critic 검증 실패로 재작성이 필요할 때
- 새로운 테마 문서를 처음 생성할 때
- 기존 테마 문서를 업데이트할 때
</Use_When>

<Do_Not_Use_When>
- 테마 필요 여부 판단 시 → 작업 오케스트레이션이 직접 수행
- frontmatter/테마 적합성 검증 시 → critic-skill 사용
- 테마 정의 참조 시 → theme-definitions/SKILL.md 직접 읽기
- 파이프라인 전체 흐름 제어 시 → 절차/SKILL.md 사용
</Do_Not_Use_When>

## 문서 작성 규칙

1. **perspective 최우선**: 요구사항서의 perspective가 문서의 관점을 결정한다. overview는 "이것이 무엇이고 왜 필요한지", tech-spec은 "어떻게 구현되었는지" 등.
2. **writing_style 엄격 준수**: 서술형/목록형/기술적 서술/참조형/명세형/절차형 — 요구사항서에 명시된 스타일로 작성한다.
3. **do_not_cover 절대 금지**: 작성 완료 후 do_not_cover 항목 각각을 문서에서 검색한다. 해당 내용 발견 시 즉시 제거한다.
4. **must_cover 전수 포함**: must_cover의 모든 항목이 문서에 포함되었는지 확인한다. 누락 시 추가한다.
5. **audience 적합 용어 수준**: "신규 팀원, 관리자"이면 전문 용어 최소화, "개발자"이면 정확한 기술 용어 사용.

## YAML Frontmatter 작성법

| 필드 | 타입 | 규칙 | 예시 |
|------|------|------|------|
| title | string | 페이지 제목 | "시스템 요구사항" |
| sidebar_label | string | title의 축약형 | "시스템 요구사항" |
| sidebar_position | number | 사이드바 내 순서 | 1 |
| section | string | 사이트 섹션 | "getting-started" |
| theme | string | theme-definitions의 테마 ID | "getting-started/requirements" |
| auto_generated | boolean | 항상 true | true |
| source_files | string[] | 분석한 소스 파일 경로 목록 | ["src/main.cpp"] |
| last_commit | string | 요구사항서의 last_commit 값을 그대로 사용 | "a1b2c3d" |
| generated_at | string | 요구사항서의 generated_at 값을 그대로 사용 | "2026-03-05T14:30:00" |
| auto_generated_warning | string (선택) | 검증 미통과 시 경고 메시지 | "" |
| tags | string[] (선택) | 검색/분류용 태그 | ["vnc", "rfb"] |

**GOOD**:

```yaml
---
title: "시스템 요구사항"
sidebar_label: "시스템 요구사항"
sidebar_position: 1
section: "getting-started"
theme: "getting-started/requirements"
auto_generated: true
source_files:
  - "CMakeLists.txt"
  - "src/server/main.cpp"
last_commit: "a1b2c3d"
generated_at: "2026-03-05T14:30:00"
---
```

**BAD**:

```yaml
---
title: "요구사항"          # 너무 축약
sidebar_position: "1"      # 문자열 (숫자여야 함)
theme: "requirements"      # 유효한 테마 ID가 아님
auto_generated: false       # 자동 생성인데 false
---
```

WHY: 필수 필드 누락(sidebar_label, section, source_files, last_commit, generated_at), 타입 오류, 잘못된 테마 ID.

## 코드 분석 방법

단계별 절차:

1. **기준 경로 확인**: 요구사항서의 `target` 필드에서 대상 코드베이스 절대 경로 확인
2. **related_files 읽기**: `{target}/{related_file}` 형태로 절대 경로 조합 → Read로 소스 코드 읽기
3. **키워드 탐색**: key_changes/must_cover 키워드로 `{target}` 하위 Grep 검색
4. **확장 탐색**: Glob으로 관련 헤더/인터페이스 파일 탐색 → 필요 시 Read로 추가 읽기
5. **추측 금지**: Read로 확인하지 않은 함수 동작, 클래스 구조를 서술하지 말 것

**GOOD** — target + related_files → Read → Grep 확장:

```
target: "/home/user/project"
related_files: ["src/main.cpp", "include/server.h"]

1. Read("/home/user/project/src/main.cpp") -> 코드 확인
2. Read("/home/user/project/include/server.h") -> 헤더 확인
3. Grep(pattern="VNCServer", path="/home/user/project/src/") -> 추가 참조 파일 발견
4. Read로 추가 파일 확인 -> 문서 작성
```

WHY: 실제 코드를 읽고 분석한 결과로 문서를 작성. 정확성 보장.

**BAD** — 요구사항서 텍스트만 보고 추측:

```
target: "/home/user/project"
related_files: ["src/main.cpp", "include/server.h"]

1. Read/Grep/Glob 사용 안 함
2. 요구사항서의 key_changes "VNC 서버 초기화 로직 변경"만 보고 추측으로 서술
3. "main.cpp에서 VNCServer 클래스를 초기화합니다" <- 실제 코드 미확인
```

WHY: 코드를 읽지 않으면 실제 구현과 다른 내용을 서술할 위험. critic이 fail 판정하여 불필요한 재시도 발생.

## GOOD/BAD 예시

### Pair 1: perspective 준수 vs 위반

**GOOD** — overview 테마에서 평이한 제품 설명:

```markdown
# MiRcsServer 개요

MiRcsServer는 원격 제어 세션을 관리하는 서버 애플리케이션입니다.
VNC 프로토콜을 기반으로 클라이언트의 원격 접속 요청을 처리하며,
다중 세션을 동시에 관리할 수 있습니다.

## 주요 역할
- 원격 제어 세션 생성 및 관리
- 클라이언트 인증 처리
- 화면 전송 및 입력 이벤트 중계
```

WHY: perspective("이것이 무엇이고 왜 필요한지")에 부합하는 개요 수준 서술. 비전문가도 이해 가능.

**BAD** — overview 테마에서 구현 세부사항 설명:

```markdown
# MiRcsServer 개요

MiRcsServer는 RFBSConn 클래스를 상속받아 구현됩니다.
SConnectionST::processMsg() 메서드에서 RFB 3.8 프로토콜의
SecurityType 핸드셰이크를 처리하며, VeNCrypt 확장을 통해
TLS 1.2 이상의 암호화 채널을 수립합니다.
```

WHY: overview 관점이 아닌 tech-spec 관점으로 작성됨. 클래스명, 메서드명, 프로토콜 버전 등은 tech-spec 영역.

### Pair 2: critic 피드백 핀포인트 수정 vs 전면 재작성

**GOOD** — 지적된 라인만 수정:

```
critic_feedback: "2단계: 라인 45-52에 RFBSConn 클래스 설명 -> do_not_cover 해당"
-> 라인 45-52를 제거하고 해당 위치에 개요 수준 설명으로 대체
-> 나머지 문서 구조/내용 유지
```

WHY: 최소 변경으로 문제 해결. 기존 검증 통과 부분을 보존.

**BAD** — 문서 전체 재생성:

```
critic_feedback: "2단계: 라인 45-52에 RFBSConn 클래스 설명 -> do_not_cover 해당"
-> 문서 전체를 처음부터 새로 작성
-> 이전에 검증 통과했던 부분도 변경됨
```

WHY: 불필요한 전면 재작성은 새로운 오류 유입 위험. critic이 통과한 부분까지 변경하면 재검증 비용 증가.

## Quick Reference

| 항목 | 핵심 |
|------|------|
| perspective | 요구사항서의 perspective가 문서 관점을 결정 |
| writing_style | 서술형/목록형/기술적/참조형/명세형/절차형 중 하나 |
| do_not_cover | 작성 후 반드시 자체 검토하여 해당 내용 제거 |
| must_cover | 모든 항목이 문서에 포함되었는지 확인 |
| frontmatter | 9필수 필드 + 타입/규칙 준수 |
| 코드 분석 | target + related_files로 코드 직접 Read, Grep/Glob 확장 탐색, 추측 금지 |
| 출력 방식 | draft_path에 Write 저장 후 경량 확인서(status + draft_path) 반환. md 문자열 반환 금지 |
| critic 수정 | 지적 부분만 핀포인트 수정, 전면 재작성 금지 |

<Final_Checklist>
- [ ] perspective/writing_style이 요구사항서와 일치하는가?
- [ ] do_not_cover 항목이 문서 어디에도 포함되지 않았는가?
- [ ] must_cover 항목이 모두 문서에 포함되었는가?
- [ ] YAML frontmatter 9필드가 모두 존재하고 타입이 올바른가?
- [ ] (재시도 시) critic 피드백의 지적사항만 수정하고 나머지는 유지했는가?
- [ ] target 경로의 related_files를 Read로 직접 읽었는가?
- [ ] md를 draft_path에 Write 저장했는가? (md 문자열 반환이 아닌 파일 저장)
- [ ] 경량 확인서(status + draft_path)를 반환했는가?
</Final_Checklist>
