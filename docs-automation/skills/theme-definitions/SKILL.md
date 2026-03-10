---
name: theme-definitions
description: 페이지 단위 테마 정의 레지스트리
origin: docs-automation
---

# Theme Definitions

> 1 테마 = 1 yaml 파일 = 1 md 파일.
> 개별 테마 정의는 `themes/` 디렉토리에 YAML 파일로 관리.

## 테마 레지스트리

| theme ID | section | 파일 | docs_path |
|----------|---------|------|-----------|
| intro | intro | themes/intro.yaml | docs/intro.md |
| getting-started/requirements | getting-started | themes/getting-started--requirements.yaml | docs/getting-started/requirements.md |
| architecture/overview | architecture | themes/architecture--overview.yaml | docs/architecture/overview.md |
| architecture/component-diagram | architecture | themes/architecture--component-diagram.yaml | docs/architecture/component-diagram.md |

## 파일명 컨벤션

theme ID의 `/`를 `--`로 치환: `architecture/overview` → `architecture--overview.yaml`

## 테마 추가 절차

1. `themes/{normalized_theme_id}.yaml` 파일 생성 (기존 테마 파일을 템플릿으로 복사)
2. 위 레지스트리 테이블에 행 추가
3. `skills/task-pipeline/SKILL.md`의 themes 리스트에 항목 추가
