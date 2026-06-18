# AGENTS.md — forge 에이전트 명세

이 문서는 forge에 포함된 20개 에이전트의 역할·입출력 계약·호출 관계를 기술한다.
사용법(스킬 트리거, 커맨드)은 `CLAUDE.md` 참조.

---

## 에이전트 풀 개요

| 그룹             | 에이전트 수 | 소속 스킬                                             | 호출 방식                                                  |
| ---------------- | ----------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| 공통 인프라      | 1개         | `feature-dev`                                         | setup-all, feature-dev Phase 0, 신규 요구사항 수신 시 호출 |
| docs-suite       | 10개        | `generate-claude-instructions`, `sync-docs-from-diff` | 스킬 오케스트레이터 → 병렬 fan-out                         |
| feature-pipeline | 9개         | `feature-pipeline`                                    | Phase 1→5 순차 (Phase 내 일부 병렬)                        |

전체 합계: **1 + 10 + 9 = 20개**. 모든 에이전트 정의는 `agents/*.md`에 위치한다.

---

## 공통 인프라 에이전트 (1개)

### project-context.md

모든 기능개발 시작 전 프로젝트 상태를 캡처한다.

**호출 시점**: setup-all 실행, feature-dev Phase 0, 신규 요구사항 수신
**입력**: `CWD`, `mode` (full | stack-only), `output` 경로
**출력**: `_workspaces/project-context.md`

```
탐색 범위 (mode별):
  full:       스택 감지 + 구조 트리 + 진입점 + 컨벤션 + 최근 커밋 5개
  stack-only: 마커 파일 재스캔 (스택·의존성 변경 감지)
```

**완료 조건**: `_workspaces/project-context.md` 생성 완료 후 오케스트레이터에 경로 보고

---

## docs-suite 에이전트 그룹 (10개)

### generate-claude-instructions 서브그룹 (5개)

스킬이 Phase 1에서 4개를 병렬 호출하고, Phase 2에서 composer를 순차 호출한다.

```
Phase 1 (병렬, run_in_background: true)
├── dev-principles.md       → DEVELOPMENT.md
├── language-guidelines.md  → LANGUAGE_GUIDELINES.md
├── ai-behavior.md          → AI_BEHAVIOR.md
└── commit-convention.md    → COMMIT_CONVENTION.md
         ↓ (모두 완료 대기)
Phase 2 (순차)
└── claude-md-composer.md   → CLAUDE.md (위 4개 합성)
```

**공통 입력 프로토콜** (오케스트레이터가 각 에이전트 프롬프트에 포함):

| 필드           | 설명                                          |
| -------------- | --------------------------------------------- |
| `{output_dir}` | 산출물 절대 경로 (`{CWD}/instruction/`)       |
| `{input_refs}` | 파일/디렉토리 경로 목록 또는 `"없음"`         |
| `{mode}`       | `초기` / `전체` / `부분`                      |
| `{팀_위치}`    | forge 플러그인의 `agents/` 디렉토리 절대 경로 |

**산출물 경로**: `{CWD}/instruction/`

| 에이전트            | 산출물 파일              | 모델  |
| ------------------- | ------------------------ | ----- |
| dev-principles      | `DEVELOPMENT.md`         | opus  |
| language-guidelines | `LANGUAGE_GUIDELINES.md` | opus  |
| ai-behavior         | `AI_BEHAVIOR.md`         | haiku |
| commit-convention   | `COMMIT_CONVENTION.md`   | haiku |
| claude-md-composer  | `CLAUDE.md`              | haiku |

---

### sync-docs-from-diff 서브그룹 (5개)

```
Phase 1 (순차, opus)
└── change-analyzer.md
        → _workspaces/01_change_analysis.json
        → _workspaces/01_change_analysis.md

Phase 2 (병렬, haiku, 파일 수정 없음)
├── readme-updater.md      → _workspaces/proposals/readme/
├── docs-updater.md        → _workspaces/proposals/docs/
└── inline-doc-updater.md  → _workspaces/proposals/inline/

Phase 3 (오케스트레이터 직접 수행 — 에이전트 없음)
    사용자 승인 후 Edit으로 패치 적용
    → _workspaces/03_apply_log.md

Phase 4 (순차, opus)
└── doc-sync-validator.md  → _workspaces/02_validation_report.md
```

**에이전트별 입력 계약**:

| 에이전트           | 입력                                                    | 출력                                        |
| ------------------ | ------------------------------------------------------- | ------------------------------------------- |
| change-analyzer    | `BASE_BRANCH`, `WORKSPACE_DIR`, `PROJECT_ROOT`          | `01_change_analysis.{json,md}`              |
| readme-updater     | `WORKSPACE_DIR`, `PROJECT_ROOT`, `01_change_analysis.*` | `proposals/readme/*.patch.md` + `_index.md` |
| docs-updater       | 위와 동일                                               | `proposals/docs/*.patch.md` + `_index.md`   |
| inline-doc-updater | 위와 동일                                               | `proposals/inline/*.patch.md` + `_index.md` |
| doc-sync-validator | `WORKSPACE_DIR`, `PROJECT_ROOT`                         | `02_validation_report.md`                   |

**영역 분리 원칙** (각 에이전트가 담당하는 파일 범위):

| 에이전트           | 담당 영역                                             | 제외                |
| ------------------ | ----------------------------------------------------- | ------------------- |
| readme-updater     | `<PROJECT_ROOT>/README.md` 단일 파일                  | docs/, 인라인       |
| docs-updater       | `<PROJECT_ROOT>/docs/` 하위 전체 (locale README 포함) | 루트 README, 인라인 |
| inline-doc-updater | 변경 파일의 부모 dir 1~2단계 `*.md`                   | 루트 README, docs/  |

---

## feature-pipeline 에이전트 그룹 (9개)

### Phase 흐름

```
Phase 1 (순차)
└── feature-planner.md
    입력: 사용자 기능 요구사항
    출력: stack-profile.json, plan.md, file-manifest.json
    위치: _workspaces/{workspaceName}/
         ↓ 사용자 승인 게이트
Phase 2 (파일별 병렬, run_in_background: true)
└── file-developer.md × N
    입력: stack-profile.json, file-manifest.json의 각 파일 항목
    출력: 구현된 소스 파일
         ↓
Phase 3 (순차)
└── test-writer.md
    입력: stack-profile.json, file-manifest.json의 businessLogicFiles
    출력: 단위테스트 파일
         ↓
Phase 4a (4개 병렬, run_in_background: true)
├── architecture-reviewer.md  → _workspaces/review-{branch-slug}/reviews/architecture.md
├── security-reviewer.md      → _workspaces/review-{branch-slug}/reviews/security.md
├── performance-reviewer.md   → _workspaces/review-{branch-slug}/reviews/performance.md
└── style-reviewer.md         → _workspaces/review-{branch-slug}/reviews/style.md
         ↓
Phase 4b (순차)
└── review-aggregator.md      → _workspaces/review-{branch-slug}/review-report.md
         ↓ 사용자 확인 게이트
Phase 5 (파일별 병렬, run_in_background: true)
└── issue-fixer.md × M
    입력: review-report.md의 파일별 이슈 + stack-profile.json
    출력: 수정된 소스 파일
```

### 에이전트별 입력·출력 계약

| 에이전트              | 주요 입력                                      | 주요 출력                                       | 모델   |
| --------------------- | ---------------------------------------------- | ----------------------------------------------- | ------ |
| feature-planner       | 사용자 요구사항, CWD                           | stack-profile.json, plan.md, file-manifest.json | opus   |
| file-developer        | stack-profile.json, 파일 항목 1개              | 구현 파일                                       | haiku  |
| test-writer           | stack-profile.json, businessLogicFiles         | 테스트 파일                                     | haiku  |
| architecture-reviewer | 변경 파일, stack-profile.json                  | reviews/architecture.md                         | opus   |
| security-reviewer     | 변경 파일, stack-profile.json                  | reviews/security.md                             | sonnet |
| performance-reviewer  | 변경 파일, stack-profile.json                  | reviews/performance.md                          | opus   |
| style-reviewer        | 변경 파일, stack-profile.json                  | reviews/style.md                                | sonnet |
| review-aggregator     | 4개 reviews/\*.md                              | review-report.md                                | opus   |
| issue-fixer           | review-report.md 파일 섹션, stack-profile.json | 수정된 파일                                     | haiku  |

### 산출물 경로 정책

| 용도             | 경로                               |
| ---------------- | ---------------------------------- |
| 기능 개발 산출물 | `_workspaces/{workspaceName}/`      |
| 세션 인계 문서   | `_workspaces/{branch-slug}/HANDOFF.md` |
| 리뷰 산출물      | `_workspaces/review-{branch-slug}/` |
| 절대 경로 사용   | 금지 (`~/`, `/` 시작 경로)         |

---

## 에이전트 간 의존성

```
그룹 간 호출: 없음 (docs-suite ↔ feature-pipeline 직접 참조 금지)
그룹 내 호출: 스킬 오케스트레이터만 호출, 에이전트 간 직접 호출 없음
외부 의존성: git CLI (모든 에이전트), pip/npm (project-setup 커맨드만)
```

---

## 공통 에이전트 작성 규칙

새 에이전트 추가 시 준수해야 할 불변식:

1. **단일 책임**: 에이전트 1개 = 산출물 타입 1개
2. **경로**: CWD 기준 상대 경로만 사용
3. **하드코딩 금지**: 레포 경로·사용자 환경 경로 미포함
4. **파일 수정 금지**: sync-docs-from-diff 그룹의 Phase 2 에이전트는 proposals/ 에 패치만 생성, 원본 파일 직접 수정 금지
5. **항상 exit 0**: 에이전트 오류가 파이프라인 전체를 차단하지 않도록 1회 재시도 후 실패 보고
