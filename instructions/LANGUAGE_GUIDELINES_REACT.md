# React 코딩 지침

**기준 버전**: React 19.2.x (`react` / `react-dom` 19.2)
**빌드 도구**: Vite 7 + `@vitejs/plugin-react-swc` (기본)
**라우팅**: react-router 7 / react-router-dom 7
**대상 프로젝트**: `mirabell-web-mono` (apps/MIMS, web-sender 등)

> 함수형 컴포넌트 + Hooks 기본. 클래스 컴포넌트 금지.

---

## 1. 컴포넌트

- **함수형 컴포넌트만** — 클래스 컴포넌트·라이프사이클 메서드 금지
- Props는 `interface`로 컴포넌트 바로 위에 정의 (`Props` 또는 `{Name}Props`)
- 파일 구조 고정: `imports → types/interface → component → export`
- `React.FC` 지양 — 함수 시그니처에 props 타입 직접 명시 (children은 명시적으로)
- `react/react-in-jsx-scope` off (React 19 자동 JSX 런타임) — `import React` 불필요

```tsx
// Good
interface UserCardProps {
  user: User;
  onSelect?: (id: string) => void;
}

export function UserCard({ user, onSelect }: UserCardProps) {
  return <button onClick={() => onSelect?.(user.id)}>{user.name}</button>;
}
```

```tsx
// Bad — 클래스 컴포넌트
class UserCard extends React.Component<UserCardProps> { ... }
```

---

## 2. 훅 (Hooks)

- `useEffect` 의존성 배열 **완전 명시** — `eslint-plugin-react-hooks` 강제 (`exhaustive-deps`)
- effect는 "동기화" 용도로만. 파생 값은 렌더 중 계산 또는 `useMemo`
- 커스텀 훅: 2개 이상 컴포넌트에서 재사용되는 로직만 분리 (`use` 접두사 필수)
- `useState` vs `useReducer`: 연관 상태 3개 이상 또는 전이 로직 복잡 시 reducer
- `useMemo` / `useCallback`: **실측 성능 문제 확인 시에만** 추가 (조기 최적화 금지)

```tsx
// Bad — 의존성 누락 (lint 에러)
useEffect(() => {
  load(userId);
}, []);
// Good
useEffect(() => {
  load(userId);
}, [userId]);
```

---

## 3. 상태 관리

| 범위                 | 권장 도구                                      |
| -------------------- | ---------------------------------------------- |
| 지역 상태            | `useState` / `useReducer` (우선)               |
| 서버 상태            | **TanStack Query 5** (`@tanstack/react-query`) |
| 전역 클라이언트 상태 | 3개 이상 컴포넌트 공유 시에만 도입             |
| 스키마 검증          | **Zod 4** (폼·API 경계)                        |

- **서버 상태와 클라이언트 상태 분리** — fetch 결과는 TanStack Query가 관리, `useState`로 복제 금지
- Query key는 배열 + 상수 팩토리로 관리, 캐시 무효화는 `invalidateQueries`
- HTTP는 `axios` 인스턴스 (인터셉터 기반 인증·에러 통일)

```tsx
// Good — 서버 상태는 TanStack Query
const { data, isLoading } = useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId),
});
```

---

## 4. 프레임워크 선택 기준 (Vite vs Next.js)

| 조건                                       | 선택              |
| ------------------------------------------ | ----------------- |
| SPA / 클라이언트 렌더링 (본 모노레포 앱들) | **Vite 7** (기본) |
| SSR / SSG / SEO / 서버 컴포넌트 필요       | **Next.js 15**    |

- **본 프로젝트는 Vite 기본 채택** — 위젯 sender·내부 어드민(MIMS) 등 SPA 성격. `vite-tsconfig-paths` + `vite-plugin-svgr` 사용
- SSR이 필요한 신규 앱만 Next.js로 시작 (별도 `LANGUAGE_GUIDELINES_NEXTJS.md` 작성 후 적용)
- Vite 빌드: `target: "esnext"`, `manualChunks`로 vendor 분리 (react-core / tanstack / router / prime)

---

## 5. 접근성 / 스타일

- `eslint-plugin-jsx-a11y` 적용 — `img`에 `alt` 필수 등 기본 a11y 규칙 준수
- 시맨틱 HTML 우선 (`button`, `nav`, `main`), 클릭 가능 요소는 `button`/`a`
- UI 라이브러리: **PrimeReact 11** + `@primeuix/themes`, 유틸 스타일은 **Tailwind 4** (`@tailwindcss/vite`)

---

## 6. 린트 / 디버그

- `no-console: ["warn", { allow: ["warn", "error"] }]` — `console.log` 잔류 차단, 디버그 로그는 커밋 전 제거
- `react-refresh/only-export-components` — 컴포넌트 파일은 컴포넌트만 export (HMR 보존), 상수는 `allowConstantExport`로 예외
- 빌드: `tsc --build && vite build` (타입체크 후 번들)

## 참고

- [React 공식 문서](https://react.dev/)
- [TanStack Query](https://tanstack.com/query/latest)
- [Vite](https://vite.dev/)
