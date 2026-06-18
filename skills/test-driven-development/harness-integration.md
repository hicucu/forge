# harness/파이프라인 통합 — test-driven-development

**이 참조를 불러올 때:** TDD가 단독 대화가 아니라 forge 파이프라인(spec → 구현)의 한 단계로 실행될 때.

## 위치

TDD는 forge spec-driven 워크플로우의 **구현 단계 규율**이다.

```
brainstorming → writing-plans(spec) → subagent-driven-development(구현) → 코드 리뷰
                                              └─ 각 작업에서 test-driven-development 적용
```

`writing-plans`가 만든 spec을 받아, implementer가 각 작업을 RED-GREEN-REFACTOR로 구현한다.

## 입력 계약

implementer는 다음을 입력으로 받는다:

- **spec**: 구현할 작업 명세 (`writing-plans` 산출, `_workspaces/{slug}/specs/`)
- **stack-profile.json**: 테스트 실행 명령·프레임워크·파일 명명 규칙을 결정. 본문(SKILL.md)은 스택 어휘를 쓰지 않고, 구체 명령은 이 프로필에서 가져온다.

## 실행 계약 (능력)

- implementer는 **테스트를 실제 실행할 수 있어야** 한다. RED 검증(실패 직접 확인)과 GREEN 검증(통과 확인)은 명령 실행이 전제다.
- 도구가 제한되어 테스트를 실행할 수 없는 작업자에게는 TDD를 위임하지 않는다 — 실행 없는 RED/GREEN은 검증이 아니다.

## 순서 계약 (모순 해결)

- implementer는 **테스트 선행**으로 동작한다: RED(실패 테스트) → GREEN(최소 구현) → REFACTOR.
- "구현 먼저, 테스트 나중"은 **금지**한다 — 테스트 후행은 안티패턴이며, 즉시 통과하는 테스트는 아무것도 증명하지 못한다.
- 이 계약은 `subagent-driven-development`의 implementer 프롬프트와 일치해야 한다. 프롬프트가 "구현 → 테스트" 순서이면 TDD가 깨지므로, 프롬프트를 테스트 선행으로 맞춘다.

## 네임스페이스

- 파이프라인·문서·프롬프트에서 이 skill 참조는 항상 **`forge:test-driven-development`** 로 한다 (구 네임스페이스 형태 금지).

## harness 대체 메모 (권고, 침습 변경 아님)

`agent_by_deul/harness`의 `feature-pipeline`은 "TDD"를 표방하지만 실제로는 TDD 정합이 아니다:

- Phase 2 `file-developer`는 도구에 테스트 실행 수단이 없어 RED/GREEN 검증이 불가능하다 (실행 계약 위반).
- 실제 테스트는 Phase 3 `test-writer`가 **구현 이후**에 작성한다 (순서 계약 위반 = 테스트 후행).

→ 이 구조는 forge spec-driven 파이프라인으로 **대체**하면 해소된다. harness 자체(에이전트 도구·Phase 구성)는 이 작업에서 침습적으로 바꾸지 않는다.
