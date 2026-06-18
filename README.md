# forge — AI 에이전트 Skills 플러그인

**forge**는 개인 전용 AI 에이전트 시스템입니다 (TDD·체계적 디버깅·브레인스토밍·오케스트레이터 파이프라인). 오케스트레이터-서브에이전트 파이프라인 아키텍처로 복잡도에 따라 요청을 자동 분류·처리합니다.

## 설치

### 1. 마켓플레이스 등록 (최초 1회)

```
/plugin marketplace add hicucu/forge
```

### 2. 플러그인 설치

```
/plugin install forge@hicucu
/reload-plugins
```

### 3. SessionStart 훅 수동 등록

플러그인 설치 후, `~/.claude/settings.json`의 `hooks` 섹션에 아래를 추가합니다:

```json
"SessionStart": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "bash \"${CLAUDE_PLUGIN_ROOT}/hooks/session-start\"",
        "timeout": 10000
      }
    ]
  }
]
```

훅이 등록되면 매 세션 시작(`/clear`, `/compact`, 신규 세션)마다 `using-forge` 스킬 컨텍스트가 자동 주입됩니다.

## 핵심 구성

- **20개 스킬**: TDD, 체계적 디버깅, 브레인스토밍, 계획 수립, 코드 리뷰, 멀티에이전트 파이프라인, 문서 동기화 등
- **20개 서브에이전트**: project-context, docs-suite(10개), feature-pipeline(9개) — 단계별 전담
- **멀티에이전트 파이프라인** (`feature-pipeline`): 계획 → 병렬 개발 → 테스트 → 4종 병렬 리뷰 → 이슈 수정의 9개 에이전트 Phase 흐름
- **docs-suite** (10개): CLAUDE.md 지침 생성(`generate-claude-instructions`)과 diff 기반 문서 동기화(`sync-docs-from-diff`) 전담
- **13개 instruction 문서**: 개발 원칙·언어 가이드라인·프로젝트 셋업 등 재사용 지침
- **오케스트레이터 패턴**: 복잡도 판단 후 적절한 전문 에이전트로 위임

## 요청 분류 및 처리 경로

| 경로       | 조건                       | 파이프라인                                  |
| ---------- | -------------------------- | ------------------------------------------- |
| **DEBUG**  | 버그·에러·예상치 못한 동작 | systematic-debugging → verification         |
| **SIMPLE** | 파일 1-2개, 10분 이내      | TDD → verification                          |
| **FULL**   | 신규 기능, 복잡한 변경     | brainstorming → planning → 병렬 개발 → 리뷰 |

## 플러그인 구조

```
hicucu/forge/
├── .claude-plugin/
│   ├── marketplace.json   # 마켓플레이스 매니페스트
│   └── plugin.json        # 플러그인 메타데이터
├── hooks/
│   ├── hooks.json         # SessionStart 훅 정의
│   ├── run-hook.cmd       # 크로스플랫폼 bash 래퍼 (폴리글랏: Windows cmd.exe + Unix bash)
│   └── session-start      # using-forge 컨텍스트 주입 스크립트
├── skills/                # 20개 스킬 디렉토리
├── agents/                # 20개 서브에이전트 (project-context + docs-suite 10 + feature-pipeline 9)
├── commands/              # 슬래시 커맨드 (forge·setup·codex-setup·commands 카탈로그 + 세부 커맨드 9개)
└── instructions/          # 13개 재사용 지침 문서 (개발 원칙·언어 가이드·프로젝트 셋업)
```

## 철학

테스트 우선 개발, 추측 대신 체계적 프로세스, 증거 기반 완료 검증, 복잡성을 전문 에이전트에 위임.

**라이선스:** MIT — 컨셉 차용 출처 및 저작권 고지는 LICENSE / NOTICE 참조
