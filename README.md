# forge

[Superpowers](https://github.com/obra/superpowers)의 skills 철학을 기반으로, 오케스트레이터-subagent 파이프라인 아키텍처를 더한 개인 전용 AI 에이전트 시스템.

- **Skills 14개**: Superpowers의 개발 방법론(TDD, 체계적 디버깅, 브레인스토밍 등)을 계승
- **Subagent 11개**: 브레인스토밍·계획·구현·리뷰 각 단계를 전용 에이전트가 담당
- **오케스트레이터 패턴**: main agent는 복잡도를 판단하고 subagent에게 위임하는 파이프라인으로 동작

---

## 설치

Claude Code에서 플러그인으로 등록합니다:

```
/plugin install <forge 디렉토리의 절대 경로>
```

**예시:**

- Windows: `/plugin install C:/Users/yourname/forge`
- macOS/Linux: `/plugin install /home/yourname/forge`

한 번만 실행하면 skills와 hooks가 모두 자동으로 등록됩니다.

---

## 동작 흐름

세션 시작 시 자동으로 작동합니다:

```
세션 시작 (startup / clear / compact)
        ↓
SessionStart hook → using-forge SKILL.md 컨텍스트 자동 주입
        ↓
요청 유형 자동 판단 → 3방향 분기
```

| 상황             | 경로        | 시퀀스                                                                                                                                         |
| ---------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 세션 시작        | —           | `using-forge` 자동 주입                                                                                                                        |
| 기능 개발 (복잡) | 복잡 경로   | `project-context-agent` → `brainstorming-agent` → `planning-agent` → `developer-agent` × N → `review-agent` → `finishing-a-development-branch` |
| 코드 작성 (단순) | 단순 경로   | `test-driven-development` → `verification-before-completion`                                                                                   |
| 버그 / 에러      | 디버깅 경로 | `systematic-debugging` → `verification-before-completion`                                                                                      |

---

## 디렉토리 구조

```
forge/
  .claude-plugin/
    plugin.json               — 플러그인 메타데이터 + SessionStart hook 등록

  .claude/
    agents/                   — 전용 subagent 11개
      project-context-agent.md    스택·아키텍처 탐색, 24h 캐시
      brainstorming-agent.md      설계안 생성 → 사용자 승인
      planning-agent.md           specs/ + file-manifest.json 작성
      developer-agent.md          TDD로 구현 + 커밋
      review-agent.md             4개 병렬 리뷰 오케스트레이터
      architecture-reviewer-agent.md
      security-reviewer-agent.md
      performance-reviewer-agent.md
      style-reviewer-agent.md
      review-aggregator-agent.md  4개 리뷰 → review-report.md 통합
      issue-fixer-agent.md        파일별 이슈 수정

  hooks/
    run-hook.cmd              — Windows bash 래퍼
    session-start             — using-forge SKILL.md 컨텍스트 주입

  skills/
    using-forge/              — 부트스트랩 + 파이프라인 오케스트레이션 로직
    brainstorming/            — 아이디어 → 설계
    writing-plans/            — 태스크 단위 구현 계획
    subagent-driven-development/
    executing-plans/
    test-driven-development/
    systematic-debugging/
    verification-before-completion/
    requesting-code-review/
    receiving-code-review/
    dispatching-parallel-agents/
    finishing-a-development-branch/
    using-git-worktrees/
    writing-skills/

  tests/
    pressure-scenarios/       — 압박 상황에서 skill 준수 검증
    pipeline-triggering/      — 3방향 분기 정확성 검증
    agent-behavior/           — 각 subagent 동작 검증
    skill-triggering/         — skill 자동 트리거 검증 (원본)
    explicit-skill-requests/  — 명시적 요청 테스트 (원본)
```

---

## Agents (subagent 파이프라인)

### 파이프라인 에이전트

| 에이전트                | 역할                | 입력               | 출력                                               |
| ----------------------- | ------------------- | ------------------ | -------------------------------------------------- |
| `project-context-agent` | 스택·아키텍처 탐색  | 프로젝트 루트      | `_workspaces/project-context.md` (24h 캐시)        |
| `brainstorming-agent`   | 설계안 생성         | 요구사항 + context | `_workspaces/{slug}/design.md`                     |
| `planning-agent`        | 구현 계획           | design.md          | `_workspaces/{slug}/specs/` + `file-manifest.json` |
| `developer-agent`       | TDD 구현            | spec 파일          | 코드 + 커밋                                        |
| `review-agent`          | 리뷰 오케스트레이터 | 브랜치 diff        | `review-report.md`                                 |

### 리뷰 에이전트 (review-agent가 병렬 호출)

| 에이전트                      | 검토 관점                          |
| ----------------------------- | ---------------------------------- |
| `architecture-reviewer-agent` | 레이어 위반, 순환 의존성, SRP      |
| `security-reviewer-agent`     | OWASP, injection, 인증/인가        |
| `performance-reviewer-agent`  | N+1, 인덱스 누락, 동기 블로킹      |
| `style-reviewer-agent`        | 네이밍, 함수 과대, 언어 관용       |
| `review-aggregator-agent`     | 4개 리뷰 → `review-report.md` 통합 |
| `issue-fixer-agent`           | Critical/High 이슈 파일별 수정     |

---

## Skills

### 계획 및 설계

| Skill             | 설명                                                                                                  |
| ----------------- | ----------------------------------------------------------------------------------------------------- |
| **brainstorming** | 창의적 작업 전 아이디어를 설계로 발전. 소크라테스식 대화로 요구사항 구체화, 설계안 승인 후 진행.      |
| **writing-plans** | 승인된 설계를 TDD 기반 태스크 단위 구현 계획으로 분해. 정확한 파일 경로, 완전한 코드, 검증 단계 포함. |

### 구현

| Skill                           | 설명                                                                  |
| ------------------------------- | --------------------------------------------------------------------- |
| **subagent-driven-development** | 서브에이전트 기반 태스크 단위 구현. 스펙 준수 + 코드 품질 2단계 리뷰. |
| **executing-plans**             | 현재 세션 내 배치 단위 실행. 체크포인트마다 사용자 확인.              |
| **test-driven-development**     | RED → GREEN → REFACTOR 강제. 테스트 없이 작성된 코드는 삭제.          |
| **using-git-worktrees**         | git worktree로 격리된 작업 환경 구성.                                 |

### 디버깅 및 검증

| Skill                              | 설명                                                       |
| ---------------------------------- | ---------------------------------------------------------- |
| **systematic-debugging**           | 4단계 근본 원인 분석. 증상 기반 추측 대신 체계적 추적.     |
| **verification-before-completion** | 완료 선언 전 실제 동작 검증 필수. 테스트·빌드·로그로 증명. |

### 코드 리뷰

| Skill                      | 설명                                                       |
| -------------------------- | ---------------------------------------------------------- |
| **requesting-code-review** | 리뷰 요청 체크리스트. 심각도별 분류, Critical은 진행 차단. |
| **receiving-code-review**  | 피드백 수용 방법. 기술적 검토 없는 맹목적 수용 금지.       |

### 협업 및 병렬 처리

| Skill                              | 설명                                    |
| ---------------------------------- | --------------------------------------- |
| **dispatching-parallel-agents**    | 독립 태스크를 병렬 에이전트로 처리.     |
| **finishing-a-development-branch** | 테스트 검증 후 merge/PR/유지/폐기 선택. |

### 메타

| Skill              | 설명                                             |
| ------------------ | ------------------------------------------------ |
| **writing-skills** | 새로운 skill 작성. TDD 기반 테스트 방법론 포함.  |
| **using-forge**    | 부트스트랩 + 복잡도 분류 + 파이프라인 경로 안내. |

---

## 테스트

```bash
# 압박 상황 준수 검증 (Superpowers 철학)
tests/pressure-scenarios/tdd-pressure.md
tests/pressure-scenarios/debugging-pressure.md
tests/pressure-scenarios/verification-pressure.md

# 파이프라인 분류 검증 (forge 고유)
claude < tests/pipeline-triggering/prompts/simple-request.txt
claude < tests/pipeline-triggering/prompts/complex-request.txt
claude < tests/pipeline-triggering/prompts/debug-request.txt

# Subagent 동작 검증
tests/agent-behavior/developer-agent-test.md
tests/agent-behavior/review-agent-test.md
tests/agent-behavior/pipeline-state-test.md
```

---

## 철학

- **Test-Driven Development** — 테스트를 항상 먼저 작성
- **체계적 접근** — 추측 대신 프로세스
- **복잡성 최소화** — 단순함을 1순위 목표로
- **증거 기반 완료** — 주장하기 전 검증
- **오케스트레이터 우선** — 복잡한 작업은 직접 처리하지 않고 전문 에이전트에 위임

---

## 출처

| 구성 요소                                  | 출처                                                                |
| ------------------------------------------ | ------------------------------------------------------------------- |
| Skills 14개 (TDD·디버깅·브레인스토밍 철학) | [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent |
| Plugin·Hook 구조                           | [Superpowers](https://github.com/obra/superpowers)                  |
| Subagent 파이프라인 아키텍처               | 독자 설계 (harness 패턴 참고)                                       |

라이선스: MIT
