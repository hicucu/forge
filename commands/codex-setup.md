---
description: forge 에이전트와 multi-agent 기능을 ~/.codex/config.toml에 등록
allowed-tools: [Read, Edit, Write, Bash]
---

# forge:codex-setup — Codex 설정 등록

forge 플러그인의 에이전트와 multi-agent 기능을 `~/.codex/config.toml`에 등록합니다.

## 실행 절차

**1단계 — forge 설치 경로 탐지**

이 커맨드 파일이 위치한 디렉토리의 부모가 forge 루트입니다. Bash로 확인:

```bash
FORGE_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
echo "Forge root: $FORGE_ROOT"
```

**2단계 — ~/.codex/config.toml 확인**

`~/.codex/config.toml` 파일을 읽어 `[features]` 섹션에 `multi_agent = true`가 이미 있는지 확인합니다.

이미 있으면 "multi_agent 이미 활성화됨" 메시지를 출력하고 3단계로 건너뜁니다.

**3단계 — multi_agent 기능 활성화**

파일이 없거나 `[features]` 섹션이 없으면 파일 끝에 추가:

```toml
[features]
multi_agent = true
```

`[features]` 섹션은 있지만 `multi_agent`가 없으면 해당 섹션에 `multi_agent = true` 추가.

**4단계 — 에이전트 전역 등록**

forge 루트의 `.codex/agents/` 디렉토리에서 `*.toml` 파일 목록을 확인합니다.

`~/.codex/agents/` 디렉토리가 없으면 생성합니다.

각 `.toml` 파일을 `~/.codex/agents/forge-{파일명}`으로 복사합니다.
(`forge-` 접두사로 다른 플러그인과 충돌 방지)

**5단계 — 설정 검증**

```bash
python3 -c "
import tomllib, pathlib
cfg = pathlib.Path.home() / '.codex' / 'config.toml'
if cfg.exists():
    data = tomllib.loads(cfg.read_text())
    assert data.get('features', {}).get('multi_agent') == True, 'multi_agent not set'
    print('config.toml valid: multi_agent =', data['features']['multi_agent'])
else:
    print('WARNING: ~/.codex/config.toml not found')
"
```

**6단계 — 완료 메시지**

등록된 에이전트 수를 출력합니다:

```
forge Codex 설정 완료.
- ~/.codex/config.toml: multi_agent = true
- ~/.codex/agents/: forge 에이전트 {N}개 등록

codex 명령 실행 시 forge 에이전트가 활성화됩니다.
서브에이전트 스킬(dispatching-parallel-agents 등) 사용을 위해 multi_agent 기능이 필요합니다.
```
