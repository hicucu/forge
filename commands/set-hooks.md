---
description: hooks/ 디렉토리의 훅들을 전역 ~/.claude/hooks/ 에 복사하고 ~/.claude/settings.json 에 등록하는 설치 에이전트
---

# set-hooks 전역 설치

`hooks/` 디렉토리의 훅 파일들을 전역 Claude Code 환경에 설치한다.
이미 완료된 단계는 건너뛰므로 재실행해도 안전하다.

## 사용법

```
/set-hooks          전체 설치 (기본)
/set-hooks check    현재 설치 상태만 확인하고 종료
/set-hooks reset    이미 설치된 훅도 모두 강제 재설치
```

---

## Phase 0: 사전 환경 확인

### 0-1. 훅 소스 디렉토리 확인

현재 CWD에서 `hooks/` 디렉토리의 존재 여부를 확인한다.

```bash
ls hooks/
```

디렉토리가 없으면 아래를 출력하고 종료한다.

```
오류: hooks/ 디렉토리를 찾을 수 없습니다.
forge 저장소 루트에서 실행해야 합니다.
  현재 위치: <CWD>
  필요 위치: <forge 저장소 루트 — hooks/ 폴더가 있어야 함>
```

### 0-2. Python 확인

```bash
python --version 2>/dev/null || python3 --version 2>/dev/null
```

Python이 없으면 아래를 출력하고 종료한다.

```
오류: Python을 찾을 수 없습니다.
Python 3.x 설치 후 재실행하세요.
```

이후 `python` 명령 표기는 실제 탐지된 명령(`python` 또는 `python3`)으로 대체한다.

### 0-3. 홈 디렉토리 및 경로 설정

아래 명령으로 홈 디렉토리를 탐지하고 대상 경로를 결정한다. `expanduser` 실패 시 `USERPROFILE`(Windows) → `HOME`(Unix) 환경변수 순으로 폴백한다.

```bash
python -c "
import os
home = os.path.expanduser('~')
if not home or home == '~':
    home = os.environ.get('USERPROFILE') or os.environ.get('HOME') or ''
print(home.replace('\\\\', '/'))
"
```

이후 단계에서 사용할 경로 변수를 설정한다.

| 변수               | 값                                 |
| ------------------ | ---------------------------------- |
| `HOME_DIR`         | 탐지된 홈 디렉토리 (슬래시 정규화) |
| `GLOBAL_CLAUDE`    | `{HOME_DIR}/.claude`               |
| `GLOBAL_HOOKS_DIR` | `{HOME_DIR}/.claude/hooks`         |
| `GLOBAL_SETTINGS`  | `{HOME_DIR}/.claude/settings.json` |
| `SRC_HOOKS_DIR`    | `./hooks`                          |
| `MANIFEST`         | `./hooks/hooks-manifest.json`      |

`~/.claude/settings.json` 파일이 없으면 아래를 출력하고 종료한다.

```
오류: ~/.claude/settings.json 을 찾을 수 없습니다.
Claude Code가 설치되어 있고 최소 1회 실행된 상태여야 합니다.
```

### 0-4. 현재 설치 상태 파악

`hooks-manifest.json`을 읽어 각 훅의 설치 상태를 확인한다.

각 훅(`file` 필드)에 대해 아래 두 가지를 확인한다.

| 상태 변수           | 확인 방법                                                  | true 조건   |
| ------------------- | ---------------------------------------------------------- | ----------- |
| `copied_{name}`     | `{GLOBAL_HOOKS_DIR}/{file}` 존재 여부                      | 파일 존재   |
| `registered_{name}` | `settings.json`의 `hooks` 섹션에 `{file}` 문자열 포함 여부 | 문자열 발견 |

`check` 인수로 실행한 경우: 아래 형식으로 상태를 출력하고 종료한다.

```
set-hooks 설치 현황 — <CWD>

  전역 훅 디렉토리   ✓ 존재  /  ✗ 없음   ({GLOBAL_HOOKS_DIR})

  훅 파일 목록:
    sync-hookify.py
      파일 복사     ✓ 완료  /  ✗ 미설치
      settings 등록 ✓ 완료  /  ✗ 미등록
      설명: hookify 규칙 파일을 프로젝트에서 전역으로 동기화

  (추가 훅이 있으면 동일 형식으로 나열)
```

`reset` 인수로 실행한 경우: 모든 상태 변수를 `false`로 재설정하여 전체를 재실행한다.

---

## Phase 1: 전역 훅 디렉토리 생성

`{GLOBAL_HOOKS_DIR}`이 없으면 생성한다.

```bash
mkdir -p {GLOBAL_HOOKS_DIR}
```

성공:

```
✓ 전역 훅 디렉토리 생성: {GLOBAL_HOOKS_DIR}
```

이미 존재하면:

```
— 전역 훅 디렉토리: 이미 존재, 건너뜀
```

---

## Phase 2: 훅 파일 복사

`hooks-manifest.json`의 각 항목에 대해 `copied_{name} = false`인 경우(또는 `reset` 모드)에만 실행한다.

```bash
cp {SRC_HOOKS_DIR}/{file} {GLOBAL_HOOKS_DIR}/{file}
```

복사 후 대상 파일 존재로 성공 확인한다.

성공:

```
✓ {file} 복사 완료 → {GLOBAL_HOOKS_DIR}/{file}
```

실패 시 에러 메시지를 출력하고 해당 훅의 Phase 3 등록도 건너뛴다.

`copied_{name} = true`이면:

```
— {file}: 이미 복사됨, 건너뜀  (강제 재설치: /set-hooks reset)
```

---

## Phase 3: settings.json 훅 등록

`registered_{name} = false`이고 Phase 2 복사에 성공한 경우에만 실행한다.

### 3-1. settings.json 읽기

`{GLOBAL_SETTINGS}` 파일을 읽어 현재 JSON 구조를 파악한다.

### 3-2. 훅 항목 구성

`hooks-manifest.json`의 `command_template` 필드에서 `{hooks_dir}`을 `{GLOBAL_HOOKS_DIR}`로 치환하여 실제 커맨드를 구성한다.

예시:

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "python {GLOBAL_HOOKS_DIR}/sync-hookify.py",
      "timeout": 10
    }
  ]
}
```

### 3-3. 중복 확인

`settings.json`의 `hooks.{event}` 배열을 순회하여 이미 같은 파일명이 `command` 문자열에 포함된 항목이 있으면 등록하지 않는다.

```
— {file}: settings.json 에 이미 등록됨, 건너뜀
```

### 3-4. 항목 추가

중복이 없으면 `hooks.{event}` 배열에 3-2에서 구성한 항목을 추가한다.
`hooks` 키 또는 `hooks.{event}` 배열이 없으면 새로 생성한다.

성공:

```
✓ {file} → settings.json 등록 완료
    이벤트: {event} / 매처: {matcher} / 타임아웃: {timeout}s
```

### 3-5. settings.json 저장

수정된 JSON을 원래 파일에 쓴다 (들여쓰기 2칸 유지).
저장 실패 시 에러 메시지를 출력한다.

---

## Phase 4: 완료 요약

모든 Phase 실행 후 아래 형식으로 요약을 출력한다.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  set-hooks 설치 완료
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  전역 훅 디렉토리   <결과>
  훅 파일별 결과:
    {file}
      파일 복사     <결과>
      settings 등록 <결과>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

새 Claude Code 세션부터 훅이 자동 적용됩니다.
수동 확인:  /set-hooks check
설치 경로:  {GLOBAL_HOOKS_DIR}
```

`<결과>` 자리에 해당 단계의 실행 결과를 기입한다.

| 상황                 | 표기                    |
| -------------------- | ----------------------- |
| 이번 실행에서 완료   | `✓ 완료`                |
| 이미 설정되어 건너뜀 | `— 기존 유지`           |
| 오류 발생            | `✗ 실패 (위 오류 확인)` |

오류가 발생한 단계가 하나라도 있으면 요약 아래에 추가한다.

```
주의: 일부 단계에서 오류가 발생했습니다.
      위 오류 메시지를 확인한 후 수동으로 재실행하세요.
```

---

## 새 훅 추가 방법

1. 훅 스크립트 파일을 `hooks/` 에 추가
2. `hooks/hooks-manifest.json` 의 `hooks` 배열에 항목 추가
3. `/set-hooks` 재실행
