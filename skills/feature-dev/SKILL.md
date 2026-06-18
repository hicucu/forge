---
name: feature-dev
description: >
  기능개발 전체 사이클 오케스트레이터. 아이디어 수신부터 PR까지.
  "기능 개발해줘", "feature dev 시작", "개발 프로세스 시작",
  "새 기능", "기능 추가 시작" 등으로 트리거.
  brainstorming → 설계문서 → 스펙 분리 → 브랜치 → 스펙별 구현 → PR 순서로 진행.
---

# Feature Dev — 전체 기능개발 사이클 오케스트레이터

아이디어 수신부터 PR까지 기능개발 전체 사이클을 조율한다.

## 핵심 원칙

- forge 스킬을 사용하더라도 **중간 산출물은 `_workspaces/{branch-slug}/`에 저장**한다.
- 브랜치명 slug = 설계 내용 기반 kebab-case (예: `feature/user-auth` → `user-auth`)
- 커밋은 사용자가 명시적으로 요청할 때만 실행한다.

## Phase 0: 프로젝트 컨텍스트 준비

`_workspaces/project-context.md` 존재 여부와 탐색 일시를 확인한다:

| 상태                    | 처리                                             |
| ----------------------- | ------------------------------------------------ |
| 미존재                  | project-context 에이전트 호출 (mode: full)       |
| 존재 + 탐색 24시간 이내 | 재사용                                           |
| 존재 + 탐색 24시간 경과 | project-context 에이전트 호출 (mode: full)       |
| 브랜치 변경 감지        | project-context 에이전트 호출 (mode: full)       |
| 신규 의존성 추가 감지   | project-context 에이전트 호출 (mode: stack-only) |

호출 프롬프트:

```
{팀_위치}/agents/project-context.agent.md를 읽고 그 지침에 따라 작업한다.
CWD: {프로젝트_루트}
mode: full
output: _workspaces/project-context.md
```

`{팀_위치}`는 이 SKILL.md가 위치한 `.claude/`의 절대 경로다.

## Phase 1: 설계

`forge:brainstorming` 스킬을 호출한다.

> **[필수] 저장 경로 강제 재정의**
> 파일 저장 시 반드시 아래 경로를 사용한다. 스킬 내부 기본값보다 이 지시가 우선한다.
> brainstorming이 설계 문서를 커밋하는 단계는 **건너뛴다** (커밋은 사용자가 명시적으로 요청할 때만).

**저장 경로 (고정):**

- 설계 문서: `_workspaces/{branch-slug}/design.md`

**저장 후 검증:** 파일이 `_workspaces/{branch-slug}/design.md`에 존재하는지 확인한다.
다른 경로에 저장됐으면 즉시 올바른 경로로 이동하고 잘못된 파일을 삭제한다.

Phase 1 실행 순서:

1. `_workspaces/project-context.md` 내용을 brainstorming 컨텍스트에 포함한다
2. brainstorming 스킬 흐름 진행:
   - 요구사항 명확화 Q&A (1회 1문항)
   - 접근방식 2~3개 제안 + 추천 (장단점 포함)
   - 설계안 제시 → 사용자 승인
3. 설계 승인 후 branch-slug 결정 (kebab-case)
4. `_workspaces/{branch-slug}/` 디렉토리 생성
5. `_workspaces/{branch-slug}/design.md` 저장 후 경로 검증

### 브랜치 생성

설계 문서 저장 직후:

```bash
git checkout -b feature/{slug}
```

실패 시 (이미 존재): `git checkout feature/{slug}` 로 전환.

## Phase 2: 계획

`forge:writing-plans` 스킬을 호출한다.

> **[필수] 저장 경로 강제 재정의**
> 파일 저장 시 반드시 아래 경로를 사용한다. 스킬 내부 기본값보다 이 지시가 우선한다.

**저장 경로 (고정):**

- 구현 계획: `_workspaces/{branch-slug}/specs/spec-{a,b,...}.md`

**저장 후 검증:** 파일이 `_workspaces/{branch-slug}/specs/`에 존재하는지 확인한다.
다른 경로에 저장됐으면 즉시 올바른 경로로 이동하고 잘못된 파일을 삭제한다.

### 스펙 분리 원칙

- 독립 구현 가능 단위 → 별도 스펙 (순서 무관)
- 순서 의존 (A 완료 후 B 가능) → 반드시 별도 스펙
- 스펙 1개 = 커밋 1개 단위

### 스펙 파일 형식

```markdown
---
complexity: simple | complex
depends-on: none | spec-a | spec-b
---

# Spec {A}: {제목}

## 목표

## 구현 범위

## 완료 기준

## 제외 범위
```

## Phase 3: 스펙별 구현

**스펙 참조 경로:** `_workspaces/{branch-slug}/specs/spec-{a,b,...}.md`
다른 경로에서 스펙을 찾지 않는다.

`depends-on` 순서대로 실행. 각 스펙 완료 후 문서 갱신 → 커밋 요청.

### 복잡도별 실행 방식

| complexity | 방식                           | 흐름                                                               |
| ---------- | ------------------------------ | ------------------------------------------------------------------ |
| `simple`   | 직접 구현                      | 구현 완료 → 커밋 직전 문서 갱신 → 사용자에게 커밋 요청             |
| `complex`  | feature-pipeline 에이전트 투입 | feature-pipeline 실행 → 커밋 직전 문서 갱신 → 사용자에게 커밋 요청 |

### 커밋 직전 문서 갱신

각 스펙 구현 완료 후, 커밋 전에 `sync-docs-from-diff` 기반으로 실행한다:

1. 변경 파일 분석
2. 영향받는 문서 감지 (README, docs/, inline \*.md)
3. 업데이트 제안 → 사용자 승인 후 반영
4. 변경 없으면 이 단계 생략

## Phase 4: 완료

1. `mirabell:code-review` 실행 (전체 브랜치 대상)
2. 리뷰 이슈 수정
3. PR 생성 (모든 스펙 커밋 포함)
4. `_workspaces/{branch-slug}/` 디렉토리 삭제 (기능 완료 정리):

```bash
rm -rf _workspaces/{branch-slug}
```

`_workspaces/project-context.md`는 보존한다 (다음 기능 개발에서 재사용 가능).

## 후속 요청 처리

| 사용자 표현      | 처리                         |
| ---------------- | ---------------------------- |
| "이어서", "계속" | 마지막 미완료 Phase부터 재개 |
| "스펙만 다시"    | Phase 2만 재실행             |
| "구현만 다시"    | Phase 3만 재실행             |
| "리뷰만"         | Phase 4만 실행               |

## 에러 핸들링

| 상황                         | 처리                              |
| ---------------------------- | --------------------------------- |
| project-context 탐색 실패    | 수동 스택 입력 요청               |
| 브랜치 생성 실패 (이미 존재) | 기존 브랜치로 체크아웃 후 진행    |
| 스펙 구현 실패 (complex)     | feature-pipeline 에러 흐름에 위임 |
| 문서 갱신 실패               | 경고 출력 후 다음 단계 진행       |
