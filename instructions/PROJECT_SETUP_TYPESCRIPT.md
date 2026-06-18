# TypeScript/JavaScript 프로젝트 셋업 지침 (PROJECT_SETUP_TYPESCRIPT)

> 신규 TS/JS 프로젝트를 처음 생성할 때 Claude Code(또는 개발자)가 참조하는 **셋업 전용 지침**.
> 패키지 매니저 → TypeScript → 린트/포매팅 → pre-commit → 환경변수 → 모노레포 구조 → 설정 스니펫 순서.
> 코딩 규칙 자체는 `LANGUAGE_GUIDELINES_TYPESCRIPT.md`, AI 동작은 `AI_BEHAVIOR.md`, 커밋은 `COMMIT_CONVENTION.md` 참조 (이 문서는 **부트스트랩 전용**).
>
> **참조 프로젝트** (실제 설정값 반영): `mirabell-web-mono` — pnpm 10 + Turborepo 모노레포, TypeScript 5.9, ESLint 9 flat config, 공유 config 패키지 3종.

---

## 0. 셋업 진행 순서 (전체 개요)

```
1. 패키지 매니저 선택 (npm / pnpm / yarn) + only-allow 강제
   ↓
2. TypeScript 설정 (tsconfig — strict 기본)
   ↓
3. 린트/포매팅 선택 (ESLint 9 flat + Prettier  vs  Biome)
   ↓
4. pre-commit 자동화 (husky + lint-staged + commitlint)
   ↓
5. 환경변수 체계 (.env 파일 + zod/t3-env 타입 검증)
   ↓
6. (모노레포 시) pnpm workspace + Turborepo + 공유 config 패키지
```

> **원칙**: 패키지 매니저·린트 도구·모노레포 여부가 모호하면 추론 전에 사용자에게 먼저 질문한다 (`AI_BEHAVIOR.md` 질문 게이트).
> 버전은 메이저를 명시하되 patch는 자동 수용(`~`), minor는 PR 검증 후 반영을 기본 정책으로 한다.

---

## 1. 패키지 매니저 선택

### 1.1 비교표

| 매니저   | 강점                                                                                 | 약점                       | 선택 기준                            |
| -------- | ------------------------------------------------------------------------------------ | -------------------------- | ------------------------------------ |
| **pnpm** | 디스크 효율(content-addressable store), 빠름, 엄격한 의존성 격리, workspace 1급 지원 | CI 캐시 설정 필요          | **모노레포·신규 프로젝트 기본 권장** |
| **npm**  | 추가 설치 불필요(Node 기본), 표준                                                    | 모노레포·디스크 효율 약함  | 단일 패키지 소규모, 외부 기여 다수   |
| **yarn** | Berry(PnP) 고급 기능, 안정적 워크스페이스                                            | PnP 호환성 이슈, 설정 복잡 | 기존 yarn 자산이 있는 경우           |

### 1.2 권장 결정 트리

```
모노레포인가?
├─ Yes → pnpm workspace + Turborepo (참조 mirabell-web-mono 채택)
└─ No ─ 단일 패키지
        ├─ 팀 표준이 npm → npm
        └─ 그 외 신규 → pnpm (디스크·속도 이점)
```

### 1.3 only-allow 강제 (필수)

선택한 매니저 외 사용을 `preinstall` 훅에서 차단하여 lock 파일 혼선을 방지한다. 참조 프로젝트는 pnpm 강제.

```jsonc
// package.json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm",
  },
  "packageManager": "pnpm@10.27.0", // Corepack 고정 — 팀 전원 동일 버전
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0",
  },
}
```

- `packageManager` 필드로 Corepack 활성화 시 매니저 버전까지 고정
- `.nvmrc`에 Node 버전 고정 (참조: `22.21.1`)

---

## 2. TypeScript 설정

### 2.1 strict 기본 템플릿 (단일 패키지)

`strict: true`는 협상 불가. 참조 프로젝트의 base 설정을 그대로 반영한다.

```jsonc
// tsconfig.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,

    // 미사용 코드 차단 (실무 권장)
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    // 모던 모듈 시스템 (번들러 기반)
    "module": "ESNext",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true, // import type 명시 강제, side-effect import 보존
    "allowImportingTsExtensions": true, // bundler 환경에서 .ts 확장자 import 허용
    "isolatedModules": true, // 파일 단위 트랜스파일 안전성 (esbuild/swc 호환)
    "resolveJsonModule": true,
    "noEmit": true, // 타입체크 전용 — 빌드는 vite/tsup 등 번들러가 담당

    // 클래스 필드·대소문자
    "useDefineForClassFields": true,
    "forceConsistentCasingInFileNames": true,

    // 타깃
    "target": "ES2022",
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"],
}
```

### 2.2 실무 권장 옵션 해설

| 옵션                   | 값          | 이유                                                                   |
| ---------------------- | ----------- | ---------------------------------------------------------------------- |
| `strict`               | `true`      | null/undefined 안전성, 암묵적 any 차단 등 8개 엄격 옵션 일괄 활성화    |
| `noUnusedLocals`       | `true`      | 미사용 지역 변수 컴파일 에러 → 죽은 코드 차단                          |
| `noUnusedParameters`   | `true`      | 미사용 매개변수 차단 (의도적 미사용은 `_` 접두사로 우회)               |
| `verbatimModuleSyntax` | `true`      | `import type` 명시 강제 → 트리쉐이킹·CJS 호환성 향상                   |
| `moduleResolution`     | `"bundler"` | Vite/esbuild/swc 등 번들러 해석 규칙. `package.json` exports 정확 반영 |
| `isolatedModules`      | `true`      | 파일 단위 트랜스파일러(swc/esbuild) 안전 보장                          |
| `noEmit`               | `true`      | tsc는 타입체크만, 실제 산출물은 번들러가 생성 (역할 분리)              |
| `skipLibCheck`         | `true`      | `.d.ts` 검사 생략으로 빌드 속도 향상 (대규모 의존성 환경 표준)         |

> Node 직접 실행(번들러 없는 CLI/서버)인 경우 `moduleResolution: "nodenext"`, `module: "nodenext"`, `noEmit: false` + `outDir` 조합 사용. `allowImportingTsExtensions`는 끈다.

### 2.3 모노레포 composite/incremental

모노레포에서는 프로젝트 참조(Project References)로 증분 빌드·타입체크를 가속한다. base 설정에 다음을 추가한다.

```jsonc
// tsconfig.base.json (공유) — §2.1에 추가
{
  "compilerOptions": {
    "composite": true, // 프로젝트 참조 대상으로 선언 (.tsbuildinfo 생성)
    "incremental": true, // 증분 컴파일 정보 저장
    "tsBuildInfoFile": "node_modules/.tmp/tsconfig.base.tsbuildinfo",
  },
}
```

루트에는 참조 목록만 가진 typecheck용 tsconfig를 둔다 (참조 프로젝트 `tsconfig.typecheck.json` 패턴).

```jsonc
// tsconfig.typecheck.json (루트)
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "files": [],
  "references": [
    { "path": "./packages/core/shared" },
    { "path": "./packages/core/features" },
    { "path": "./apps/web" },
  ],
  "compilerOptions": {
    "composite": true,
    "incremental": true,
    "tsBuildInfoFile": "tsconfig.typecheck.tsbuildinfo",
  },
}
```

- 전체 타입체크: `tsc --pretty --noEmit --project ./tsconfig.typecheck.json`
- 빌드: `tsc --build` (참조 그래프 따라 증분 빌드)
- 각 패키지/앱은 공유 base를 `extends`하고 자기 `tsBuildInfoFile`·`paths`만 추가 (§6.2)

---

## 3. ESLint vs Biome 선택 및 설정

### 3.1 선택 기준

| 항목            | **ESLint 9 (flat) + Prettier**                            | **Biome**                                |
| --------------- | --------------------------------------------------------- | ---------------------------------------- |
| 도구 수         | 2개 (lint + format 분리)                                  | 1개 (lint + format 통합)                 |
| 성능            | 보통 (Node 기반)                                          | 매우 빠름 (Rust 기반)                    |
| 플러그인 생태계 | 광범위 (react-hooks, jsx-a11y, tanstack-query 등)         | 제한적 (자체 규칙 위주, 플러그인 미성숙) |
| 설정 유연성     | 높음 (커스텀 규칙·공유 config 패키지)                     | 중간 (단일 `biome.json`)                 |
| 권장 상황       | **팀 확장성·복잡 규칙·React 생태계** (참조 프로젝트 채택) | **단일 도구·속도 우선·소규모/신규**      |

```
프로젝트 규모·팀 확장성·플러그인(jsx-a11y, react-hooks 등) 필요?
├─ Yes → ESLint 9 flat config + Prettier
└─ No, 속도·단일 설정 우선 → Biome
```

### 3.2 ESLint 9 flat config 기본 템플릿

`typescript-eslint` 헬퍼(`tseslint.config`)로 타입 안전 flat config를 구성한다. 참조 프로젝트 규칙을 반영한다.

```js
// eslint.config.js
import js from "@eslint/js";
import prettier from "eslint-config-prettier";
import tseslint from "typescript-eslint";
import globals from "globals";

export default tseslint.config([
  // 전역 무시 패턴
  {
    ignores: ["dist/", "**/dist/", "node_modules/", "**/*.d.ts"],
  },

  // 기본 JS + Prettier 충돌 규칙 비활성화
  {
    extends: [js.configs.recommended, prettier],
    languageOptions: {
      globals: { ...globals.browser, ...globals.node },
    },
  },

  // TypeScript
  {
    files: ["**/*.{ts,tsx}"],
    extends: [...tseslint.configs.recommended],
    languageOptions: {
      parser: tseslint.parser,
      parserOptions: {
        ecmaVersion: 2024,
        sourceType: "module",
        ecmaFeatures: { jsx: true },
      },
    },
    plugins: { "@typescript-eslint": tseslint.plugin },
    rules: {
      // any 전면 금지
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/explicit-function-return-type": "off",

      // _ 접두사 미사용 변수 허용
      "@typescript-eslint/no-unused-vars": [
        "warn",
        {
          argsIgnorePattern: "^_",
          caughtErrorsIgnorePattern: "^_",
          destructuredArrayIgnorePattern: "^_",
          varsIgnorePattern: "^_",
        },
      ],

      // 인터페이스 I 접두사 금지 + 타입류 PascalCase
      "@typescript-eslint/naming-convention": [
        "warn",
        {
          selector: "interface",
          format: ["PascalCase"],
          custom: { regex: "^I[A-Z]", match: false },
        },
        { selector: "typeLike", format: ["PascalCase"] },
      ],

      // 임의 객체 캐스팅({} as T) 차단 — 가드/검증 함수 유도
      "@typescript-eslint/consistent-type-assertions": [
        "warn",
        { assertionStyle: "as", objectLiteralTypeAssertions: "never" },
      ],

      // 디버그 잔류 차단
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "no-alert": "warn",
    },
  },
]);
```

React 프로젝트는 `eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`, `eslint-plugin-jsx-a11y` 블록을 추가한다 (상세는 `LANGUAGE_GUIDELINES_REACT.md`).

### 3.3 핵심 규칙 정리 (참조 프로젝트 실측)

| 규칙                                                | 수준    | 목적                                         |
| --------------------------------------------------- | ------- | -------------------------------------------- |
| `@typescript-eslint/no-explicit-any`                | `error` | `any` 사용 차단 → `unknown` + 타입 가드 유도 |
| `naming-convention` (interface `^I[A-Z]` 금지)      | `warn`  | `IUser` → `User` (I 접두사 금지)             |
| `consistent-type-assertions` (objectLiteral: never) | `warn`  | `({} as T)` 차단 → 검증 함수 사용            |
| `no-unused-vars` (`^_` 예외)                        | `warn`  | 죽은 변수 차단, 의도적 미사용은 `_` 접두사   |
| `no-console` (warn/error 허용)                      | `warn`  | `console.log` 잔류 차단                      |

> Prettier와 ESLint 역할 분리: 포매팅은 Prettier, 코드 품질은 ESLint. `eslint-config-prettier`를 마지막에 extends하여 서식 충돌 규칙을 끈다.

### 3.4 Prettier 기본값

```jsonc
// .prettierrc  (또는 prettier-config 패키지 index.js)
{
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
}
```

```
# .prettierignore
node_modules/
**/node_modules/
dist/
**/dist/
**/*.d.ts
*.svg
public/
```

### 3.5 Biome 선택 시 (대안)

단일 도구·속도 우선 시 ESLint+Prettier 대신 Biome 단독 사용.

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 80,
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": { "noExplicitAny": "error" },
    },
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "trailingCommas": "all",
      "semicolons": "always",
    },
  },
}
```

- pre-commit에서 `biome check --write` 단일 명령으로 lint+format 동시 처리
- React 전용 접근성·hooks 규칙이 필요하면 ESLint 경로 권장 (Biome 플러그인 생태계 미성숙)

---

## 4. pre-commit 자동화 (husky + lint-staged + commitlint)

### 4.1 설치 및 초기화

```bash
pnpm add -D husky lint-staged @commitlint/cli @commitlint/config-conventional
pnpm husky init   # package.json에 "prepare": "husky" 추가됨
```

### 4.2 husky 훅 (참조 프로젝트 구성)

```bash
# .husky/pre-commit
pnpm lint-staged

# .husky/commit-msg
pnpm commitlint --edit $1

# .husky/pre-push  (선택 — 무거운 검사는 push 시점으로)
pnpm run type
```

- `pre-commit`: 스테이징 파일만 포맷·린트 (빠름)
- `commit-msg`: 커밋 메시지 컨벤션 검증
- `pre-push`: 전체 타입체크 등 무거운 검사 (커밋 흐름 방해 최소화)

### 4.3 lint-staged

```jsonc
// .lintstagedrc  (또는 package.json "lint-staged")
{
  "{src,apps,packages}/**/*.{ts,tsx,js,jsx}": [
    "prettier --write",
    "eslint --fix --cache --cache-location .eslintcache --max-warnings 0 --no-warn-ignored",
  ],
  "*.{json,md,mdx,yml,yaml}": ["prettier --write"],
  "**/*.{scss,css}": ["prettier --write"],
}
```

- `--max-warnings 0`: pre-commit에서는 warning도 차단 (CI보다 엄격)
- `--cache`: 변경 파일만 재검사로 속도 확보
- Biome 사용 시 위 ts/tsx 블록을 `["biome check --write --no-errors-on-unmatched"]`로 대체

### 4.4 commitlint

```js
// commitlint.config.js
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "refactor",
        "style",
        "design",
        "perf",
        "docs",
        "comment",
        "chore",
        "build",
        "init",
        "revert",
        "etc",
        "HOTFIX",
      ],
    ],
    "type-case": [0], // type-enum이 정확한 문자열을 강제하므로 case 규칙 비활성
    "header-max-length": [2, "always", 100],
    "body-leading-blank": [2, "always"],
    "subject-case": [0],
    "footer-leading-blank": [1, "always"],
  },
};
```

> 커밋 타입·메시지 작성 규칙 상세는 `COMMIT_CONVENTION.md` 참조.

---

## 5. 환경변수 관리

### 5.1 .env 파일 체계

| 파일           | 용도                                 | 커밋 여부          |
| -------------- | ------------------------------------ | ------------------ |
| `.env.example` | 키 목록·설명 템플릿 (값은 더미/빈값) | **커밋 O**         |
| `.env`         | 기본값 (비밀 아닌 공통 설정)         | 신중히 (보통 X)    |
| `.env.local`   | 로컬 개발자별 비밀값                 | **커밋 절대 금지** |
| `.env.{mode}`  | dev/stage/live 등 환경별             | 비밀 미포함만 커밋 |

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

- 클라이언트 노출 변수는 번들러 규칙 접두사 필수 (Vite: `VITE_`, Next.js: `NEXT_PUBLIC_`)
- **절대 커밋 금지**: API 키, DB 비밀번호, 토큰, 인증서, `.env.local` 등 모든 시크릿. 커밋 시도 시 즉시 중단·경고

### 5.2 타입 안전 검증 (zod / t3-env)

런타임 시작 시 환경변수를 검증하여 누락/형식 오류를 조기 발견한다. 참조 프로젝트는 zod 4.x 채택.

```ts
// src/env.ts — zod 단독
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .default("development"),
  API_BASE_URL: z.string().url(),
  DATABASE_URL: z.string().min(1),
  PORT: z.coerce.number().default(3000),
});

export const env = envSchema.parse(process.env); // 실패 시 부팅 중단
```

```ts
// src/env.ts — @t3-oss/env-core (server/client 분리, 접두사 강제)
import { createEnv } from "@t3-oss/env-core";
import { z } from "zod";

export const env = createEnv({
  server: { DATABASE_URL: z.string().url() },
  client: { VITE_API_BASE_URL: z.string().url() },
  clientPrefix: "VITE_",
  runtimeEnv: import.meta.env,
  emptyStringAsUndefined: true,
});
```

- 단순 프로젝트: zod 단독으로 충분
- 클라이언트/서버 변수 격리·접두사 강제가 필요한 풀스택: t3-env 권장

---

## 6. 모노레포 구조 (해당 시)

### 6.1 디렉토리 구조 (pnpm workspace + Turborepo)

```
repo-root/
├── apps/                       # 배포 단위 (web, admin, api ...)
│   └── web/
├── packages/
│   ├── config/                 # 공유 설정 패키지 모음
│   │   ├── typescript-config/  # @scope/typescript-config
│   │   ├── eslint-config/      # @scope/eslint-config
│   │   └── prettier-config/    # @scope/prettier-config
│   ├── core/                   # 도메인 공유 코드 (shared, features ...)
│   └── lib/                    # 라이브러리성 패키지
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.json               # 루트 (composite/incremental만)
├── tsconfig.typecheck.json     # 전체 타입체크용 references
├── eslint.config.js            # 공유 config 재노출 (2줄)
├── commitlint.config.js
└── package.json                # 루트 — devDeps·scripts·only-allow
```

```yaml
# pnpm-workspace.yaml
packages:
  - apps/*
  - packages/**/*
  - "!**/test/**"
  - "!**/dist/**"
  - "!**/build/**"

onlyBuiltDependencies: # 빌드 스크립트 허용 의존성 명시 (pnpm 10 보안 정책)
  - esbuild
  - "@swc/core"
```

```jsonc
// turbo.json
{
  "$schema": "https://turborepo.com/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"], // 의존 패키지 먼저 빌드
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      "outputs": ["dist/**"],
    },
    "lint": {},
    "dev": { "cache": false, "persistent": true },
  },
}
```

### 6.2 공유 설정 패키지 패턴

각 config 패키지는 `private: true` + `workspace:*`로 소비되며, 실제 설정 파일을 `main`/`exports`로 노출한다.

```jsonc
// packages/config/typescript-config/package.json
{
  "name": "@scope/typescript-config",
  "version": "0.1.0",
  "private": true,
  "peerDependencies": { "typescript": "~5.9.3" },
}
```

```jsonc
// packages/config/eslint-config/package.json
{
  "name": "@scope/eslint-config",
  "main": "eslint.config.js",
  "type": "module",
  "private": true,
  "exports": { ".": "./eslint.config.js" },
  "peerDependencies": { "eslint": "^9.25.0", "typescript-eslint": "^8.52.0" },
}
```

루트·앱은 공유 config를 얇게 재노출/extends 한다.

```js
// 루트 eslint.config.js — 재노출(2줄)
import eslintConfig from "@scope/eslint-config";
export default eslintConfig;
```

```jsonc
// apps/web/tsconfig.json — 공유 base extends + 자기 paths/outDir만 추가
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@scope/typescript-config/tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "noEmit": true,
    "tsBuildInfoFile": "node_modules/.tmp/tsconfig.web.tsbuildinfo",
    "paths": {
      "@/*": ["./src/*"],
      "@packages/*": ["../../packages/*"],
    },
  },
  "include": ["src/**/*", "vite.config.ts"],
  "exclude": ["dist", "node_modules"],
}
```

### 6.3 workspace:\* 재노출 패턴

- 내부 패키지 의존은 항상 `"@scope/shared": "workspace:*"`로 선언 — 퍼블리시 시 실제 버전으로 치환
- 공유 패키지는 빌드 없이 `main`/`exports`를 소스(`./src/index.ts`)로 직접 가리켜 TS 프로젝트 참조로 즉시 소비 가능
- 루트 `package.json`이 앱·패키지를 `dependencies`에 `workspace:*`로 모아 단일 진입점 구성 (참조 프로젝트 패턴)

```jsonc
// packages/core/shared/package.json — 소스 직접 노출
{
  "name": "@scope/shared",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": { ".": "./src/index.ts", "./package.json": "./package.json" },
  "devDependencies": {
    "@scope/eslint-config": "workspace:*",
    "@scope/prettier-config": "workspace:*",
    "@scope/typescript-config": "workspace:*",
  },
}
```

---

## 7. 설정 파일 스니펫 모음

### 7.1 루트 package.json scripts 기본값

```jsonc
{
  "name": "my-app",
  "type": "module",
  "private": true,
  "packageManager": "pnpm@10.27.0",
  "scripts": {
    "preinstall": "npx only-allow pnpm",
    "prepare": "husky",
    "dev": "turbo dev", // 모노레포: turbo, 단일: vite
    "build": "turbo run build",
    "lint": "eslint . --report-unused-disable-directives --max-warnings 50 --cache --fix",
    "format": "prettier --log-level warn ./**/*.{ts,tsx,js,jsx,json,md} --cache --write",
    "type": "tsc --pretty --noEmit --project ./tsconfig.typecheck.json",
    "lint-staged": "lint-staged",
    "commitlint": "commitlint",
  },
  "engines": { "node": ">=18.0.0", "pnpm": ">=8.0.0" },
}
```

### 7.2 권장 버전 (참조 프로젝트 실측 — 2026-05 기준)

| 패키지                   | 버전      | 정책                |
| ------------------------ | --------- | ------------------- |
| `pnpm`                   | `10.27.0` | packageManager 고정 |
| node                     | `22.21.1` | `.nvmrc` 고정       |
| `typescript`             | `~5.9.3`  | patch만 자동        |
| `eslint`                 | `9.x`     | flat config         |
| `typescript-eslint`      | `^8.52.0` | minor PR 검증       |
| `eslint-config-prettier` | `^10.x`   |                     |
| `prettier`               | `^3.5.x`  |                     |
| `husky`                  | `^9.x`    |                     |
| `lint-staged`            | `^16.x`   |                     |
| `@commitlint/cli`        | `^21.x`   |                     |
| `turbo`                  | `^2.x`    | 모노레포만          |
| `zod`                    | `4.x`     | env 검증            |

### 7.3 셋업 검증 체크리스트

작업 완료 전 다음을 실제 실행하여 동작을 증명한다 (`AI_BEHAVIOR.md` — 증명 없이 완료 금지).

- [ ] `pnpm install` 성공 (only-allow가 npm/yarn 차단하는지 확인)
- [ ] `pnpm type` (또는 `tsc --build`) 타입 에러 0
- [ ] `pnpm lint` 통과
- [ ] 의도적 `any` 추가 시 ESLint error 발생 확인
- [ ] 잘못된 형식 커밋(`add stuff`) 시 commit-msg 훅이 차단하는지 확인
- [ ] `.env.local`이 `.gitignore`에 포함되어 추적되지 않는지 확인
- [ ] (모노레포) `turbo build` 의존 그래프 순서대로 빌드되는지 확인
