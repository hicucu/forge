---
description: forge 요청 처리 진입점 — setup 없이도 사용 가능
argument-hint: [요청 내용]
allowed-tools: [Read, Edit, Write, Bash, Glob, Grep]
---

# /forge — forge 시스템 진입점

forge 컨텍스트를 수동으로 활성화하고 요청을 처리합니다.
SessionStart 훅이 등록되지 않은 환경에서도 동작합니다.

## 실행 방법

**Step 1 — using-forge 스킬 호출**

`forge:using-forge` 스킬을 즉시 호출해 forge 컨텍스트와 요청 분류 파이프라인을 활성화합니다.

**Step 2 — 요청 처리**

`$ARGUMENTS`가 있으면 해당 내용을 forge 파이프라인으로 처리합니다.
인자가 없으면 사용자에게 무엇을 도와드릴지 묻습니다.

**Step 3 — setup 미완료 감지 (선택)**

SessionStart 훅이 등록되지 않은 것 같으면 (매 세션마다 이 커맨드를 직접 실행해야 하는 경우),
다음 안내를 덧붙입니다:

> 💡 `/forge:setup` 을 실행하면 매 세션 시작 시 forge가 자동 활성화됩니다.
