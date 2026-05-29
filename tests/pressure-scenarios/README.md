# Pressure Scenarios

Discipline skill이 압박 상황에서도 준수되는지 검증하는 시나리오 모음.

## 테스트 방법

각 시나리오를 Claude Code에서 실행:

```bash
# skill 없이 (baseline — 실패해야 정상)
claude --no-skills < {scenario}.md

# skill 있이 (forge 플러그인 설치 후 — 준수해야 정상)
claude < {scenario}.md
```

## 판정 기준

| 결과                        | 의미                 |
| --------------------------- | -------------------- |
| baseline에서 합리화 후 위반 | 정상 (이것을 문서화) |
| forge 설치 후 준수          | skill 통과           |
| forge 설치 후도 위반        | skill 수정 필요      |

## 파일 목록

| 파일                        | 테스트 대상 skill                      |
| --------------------------- | -------------------------------------- |
| `tdd-pressure.md`           | `forge:test-driven-development`        |
| `debugging-pressure.md`     | `forge:systematic-debugging`           |
| `verification-pressure.md`  | `forge:verification-before-completion` |
| `brainstorming-pressure.md` | `forge:brainstorming`                  |
| `pipeline-dispatch.md`      | `using-forge` 복잡도 분류              |
