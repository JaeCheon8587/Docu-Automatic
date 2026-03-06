---
name: theme-definitions
description: 페이지 단위 테마 정의 참조 데이터 (작업 오케스트레이션/critic 읽기 전용)
origin: docs-automation
---

# Theme Definitions

> 페이지 단위 테마 정의 참조 데이터. 작업 오케스트레이션과 critic이 읽기 전용으로 참조.
> 1 테마 = 1 md 파일. 메인 루프가 테마를 순차 순회하며 4단계 사이클 실행.
>
> **1차 스코프**: 4개 테마 (개요 1 + 시작하기 1 + 아키텍처 2)
> **2차 확장 예정**: 제품별 페이지, 라이브러리, 개발 가이드 등

```yaml
# ============================================================
# 1차 스코프: 4개 테마
# ============================================================

- id: intro
  name: 개요
  section: intro
  perspective: "프로젝트 전체 소개 — 무엇이고 왜 필요한지"
  audience: "모든 방문자 (신규 팀원, 관리자)"
  writing_style: "서술형 — 평이한 소개, 핵심 정보 테이블 병행"
  must_cover:
    - 시스템/제품 소개 (목적과 필요성)
    - 주요 컴포넌트/모듈 역할 요약
    - 기술 스택 개요
    - 지원 플랫폼
  do_not_cover:
    - "개별 제품의 상세 기능 명세 (→ 제품별 페이지)"
    - "설치/실행 방법 (→ getting-started)"
    - "내부 구현 세부사항 (→ architecture, tech-spec)"
  docs_path: "docs/intro.md"
  site_url: "/docs/intro"

- id: getting-started/requirements
  name: 시스템 요구사항
  section: getting-started
  perspective: "설치/실행에 필요한 환경과 조건"
  audience: "설치자, 운영자"
  writing_style: "참조형 — 제품별 테이블, 간결한 설명"
  must_cover:
    - 실행 환경 (OS, 런타임, 메모리, 권한)
    - 개발 환경 (Windows/Linux 빌드 도구)
    - 네트워크 포트 목록 (해당 시)
    - 서드파티 의존성
  do_not_cover:
    - "설치 절차 (→ getting-started/installation)"
    - "사용 방법 (→ getting-started/quick-start)"
    - "내부 구현 세부사항"
  docs_path: "docs/getting-started/requirements.md"
  site_url: "/docs/getting-started/requirements"

- id: architecture/overview
  name: 제품 아키텍처
  section: architecture
  perspective: "시스템 전체 구조와 모듈 간 관계"
  audience: "신규 팀원, 개발자"
  writing_style: "서술형 — 전체 그림 중심, 다이어그램 포함"
  must_cover:
    - Mermaid 기반 시스템 전체 구조
    - Mermaid 기반 모듈 간 관계 및 통신/호출 흐름
    - 핵심 프로토콜/기술 (해당 시)
  do_not_cover:
    - "개별 제품 내부 아키텍처 (→ 제품별 tech-spec)"
    - "설정 파라미터 (→ config)"
    - "API 상세 명세 (→ api)"
  docs_path: "docs/architecture/overview.md"
  site_url: "/docs/architecture/overview"

- id: architecture/component-diagram
  name: 컴포넌트 다이어그램
  section: architecture
  perspective: "S/W별 대략적인 컴포넌트 구성"
  audience: "개발자"
  writing_style: "참조형 — 제품별 컴포넌트 테이블/다이어그램"
  must_cover:
    - Mermaid 기반 주요 컴포넌트 (라이브러리/모듈 수준)
    - Mermaid 기반 컴포넌트 간 의존 관계
  do_not_cover:
    - "클래스/함수 수준 세부사항 (→ 제품별 tech-spec)"
    - "설정 파라미터 (→ config)"
  docs_path: "docs/architecture/component-diagram.md"
  site_url: "/docs/architecture/component-diagram"
```
