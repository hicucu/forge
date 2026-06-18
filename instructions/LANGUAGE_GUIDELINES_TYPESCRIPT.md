# TypeScript 코딩 지침

**기준 버전**: TypeScript 5.9.x (`~5.9.3` 고정)
**strict 모드**: 필수 (협상 불가)
**대상 프로젝트**: `mirabell-web-mono` (pnpm 10 + Turborepo 모노레포)

> 모던 안정화 버전 사용. TypeScript는 패치만 자동 수용(`~5.9.3`), minor 업그레이드는 PR로 검증 후 반영.

---

## 1. 네이밍

| 대상                   | 규칙                     | 예시                         |
| ---------------------- | ------------------------ | ---------------------------- |
| 타입/인터페이스/클래스 | PascalCase               | `UserProfile`, `ApiResponse` |
| 변수/함수              | camelCase                | `fetchUser`, `isLoading`     |
| 상수(모듈 전역)        | UPPER_SNAKE_CASE         | `MAX_RETRY`, `API_BASE_URL`  |
| 파일                   | kebab-case.ts            | `user-service.ts`            |
| 컴포넌트 파일          | PascalCase.tsx           | `UserCard.tsx`               |
| Boolean                | is/has/can/should 접두사 | `isOpen`, `hasPermission`    |

- **인터페이스 `I` 접두사 금지** — ESLint `naming-convention`이 `^I[A-Z]` 매칭 차단 (`IUser` X → `User` O)
- 타입 별칭/인터페이스는 모두 PascalCase (`typeLike` 규칙)

```typescript
// Bad
interface IUserDto {
  userName: string;
}
// Good
interface UserDto {
  userName: string;
}
```

---

## 2. 타입 시스템

- **`any` 전면 금지** — ESLint `@typescript-eslint/no-explicit-any: "error"`. 불명확 입력은 `unknown` + 타입 가드
- **타입 단언 캐스팅 지양** — `consistent-type-assertions`로 객체 리터럴 단언(`{} as T`) 차단. 검증 함수/가드 사용
- 제네릭은 의미 있는 이름: `TItem`, `TResponse` (단일 문자 `T`는 단순 컨테이너 한정)
- `Readonly` / `Partial` / `Pick` / `Omit` 적극 활용, 입력 DTO는 불변 우선
- 런타임 경계(API 응답·폼)는 **Zod 4** 스키마로 파싱 후 타입 추론 (`z.infer`)

```typescript
// Bad
function parse(data: any) {
  return data.value;
}
// Good
const Schema = z.object({ value: z.string() });
function parse(data: unknown) {
  return Schema.parse(data).value;
}
```

---

## 3. tsconfig 필수 설정

모노레포 base(`packages/config/typescript-config/tsconfig.base.json`)를 상속한다. 핵심 플래그:

```jsonc
{
  "compilerOptions": {
    "strict": true, // 필수
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true, // import type 명시 강제
    "isolatedModules": true,
    "moduleResolution": "bundler", // Vite 번들러 기준
    "module": "ESNext",
    "target": "ES2022",
    "noEmit": true, // 타입체크 전용, 번들은 Vite/tsc build
    "composite": true, // 프로젝트 레퍼런스
    "incremental": true,
  },
}
```

- 신규 패키지는 base를 `extends`하고 패키지별 `include`/`paths`만 추가
- 미사용 변수·파라미터는 빌드 실패 사유 — 의도적 미사용은 `_` 접두사 (`argsIgnorePattern: "^_"`)

---

## 4. 비동기 / 에러 처리

- `async/await` 우선, 콜백·`.then()` 체인 지양
- 독립 작업 병렬화: `Promise.all()` / `Promise.allSettled()`
- `try/catch` 후 `instanceof Error`로 좁히기, 도메인 에러는 `Error` 서브클래스
- HTTP는 `axios` 인스턴스 사용 (인터셉터로 인증·에러 통일). 응답은 Zod로 검증

```typescript
// Good
try {
  const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);
} catch (e) {
  if (e instanceof ApiError) handleApi(e);
  else throw e;
}
```

---

## 5. 도구 체인 (실무 표준)

| 도구          | 버전/설정                       | 역할                                                        |
| ------------- | ------------------------------- | ----------------------------------------------------------- |
| 패키지 매니저 | **pnpm 10** (`only-allow pnpm`) | npm/yarn 사용 차단 (`preinstall` 훅)                        |
| 모노레포      | **Turborepo 2.7**               | `turbo run build/dev` 캐시·병렬                             |
| 린터          | **ESLint 9 flat config**        | `eslint.config.js`, 공유 `@mirabell-web-mono/eslint-config` |
| 포매터        | **Prettier 3.5**                | 공유 `@mirabell-web-mono/prettier-config`                   |
| 타입체크      | `tsc --noEmit` (`pnpm type`)    | 별도 `tsconfig.typecheck.json`                              |

### ESLint vs Biome 선택 기준

- **본 모노레포는 ESLint + Prettier 채택** (React/TanStack/jsx-a11y 등 풍부한 플러그인 생태계 활용)
- **Biome 채택 조건**: 신규 단독 패키지이고 React 외 플러그인 의존이 적으며 lint+format 속도가 병목일 때만. 기존 모노레포와 혼용 금지(설정 충돌)
- 핵심 ESLint 규칙: `no-explicit-any: error`, `no-console`(warn/error만 허용), `consistent-type-assertions`(객체 리터럴 단언 금지), `no-relative-import-paths`(상대경로 import 경고 → `paths` alias 사용)

### pre-commit / commit 규약 (husky 9 + lint-staged 16 + commitlint)

- `.husky/pre-commit` → `pnpm lint-staged`: 스테이징 파일만 `prettier --write` + `eslint --fix --max-warnings 0`
- `.husky/commit-msg` → `commitlint`: Conventional Commits 강제 (`@commitlint/config-conventional`)
  - 허용 type: `feat`, `fix`, `refactor`, `style`, `design`, `perf`, `docs`, `comment`, `chore`, `build`, `init`, `revert`, `etc`, `HOTFIX`
  - header 최대 100자, body 앞 빈 줄 필수

---

## 6. 모듈 / Import

- **상대경로 import 지양** — `no-relative-import-paths` 경고. `tsconfig` `paths` alias 또는 workspace 패키지명(`@mirabell-web-mono/*`) 사용
- `import type { X }` 명시 (`verbatimModuleSyntax`로 강제) — 런타임 import와 타입 import 분리
- workspace 의존은 `workspace:*` 프로토콜로 선언

## 참고

- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [typescript-eslint](https://typescript-eslint.io/)
- [Zod](https://zod.dev/)
