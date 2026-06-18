# React SPA 셋업 지침 (PROJECT_SETUP_REACT)

> Vite 기반 React SPA(Single Page Application) 신규 프로젝트 셋업 단계별 지침.
> 공통 부트스트랩(lint/format/commitlint/husky·tsconfig·모노레포)은 `PROJECT_SETUP_GUIDE.md`, 코딩 규칙은 `LANGUAGE_GUIDELINES_REACT.md` / `LANGUAGE_GUIDELINES_TYPESCRIPT.md` 참조. 이 문서는 **SPA 스택 선택·구성 전용**.
>
> 실제 패턴은 사내 참조 프로젝트 `mirabell-web-mono`(React 19 + Vite 7 + react-router 7 + TanStack Query 5 + Zod 4 + Tailwind 4 + PrimeReact 11)를 분석·반영했다.

---

## 1. 언제 React SPA(Vite)를 선택하나

| 조건                                  | 적합 여부 | 비고                                        |
| ------------------------------------- | --------- | ------------------------------------------- |
| SSR/SEO 불필요 (검색 노출 대상 아님)  | 적합      | 로그인 뒤 화면, 사내 도구는 SEO 무관        |
| CSR/SPA 형태 (앱 같은 인터랙션)       | 적합      | 클라이언트 라우팅·상태 중심                 |
| 관리자 대시보드 / 어드민 / 백오피스   | 적합      | 참조 프로젝트 `MIMS`가 이 유형              |
| 내부 운영 도구 / 모니터링 콘솔        | 적합      | 인증 게이트 뒤 화면, 인덱싱 불필요          |
| 임베드 위젯 / 정적 호스팅(S3·CDN)     | 적합      | 단일 번들 정적 배포                         |
| **SEO·OG 메타·초기 렌더 속도가 핵심** | 부적합    | → `PROJECT_SETUP_NEXTJS.md` (SSR/SSG) 선택  |
| **서버에서 직접 DB·시크릿 접근 필요** | 부적합    | → Next.js(풀스택) 또는 별도 백엔드 API 분리 |

> 판단 기준: **"검색엔진이 이 페이지를 읽어야 하는가?"** + **"첫 화면을 서버가 렌더해야 하는가?"** 둘 다 No면 SPA. 하나라도 Yes면 Next.js 검토. 모호하면 추론 전 사용자에게 질문 (`AI_BEHAVIOR.md` 질문 게이트).

---

## 2. Vite 초기 설정

### 2.1 프로젝트 생성

```bash
# pnpm 권장 (참조 프로젝트 packageManager: pnpm)
pnpm create vite my-spa --template react-swc-ts
cd my-spa
pnpm install
```

| 템플릿         | 사용 시점                                                    |
| -------------- | ------------------------------------------------------------ |
| `react-swc-ts` | **기본 권장** — SWC 컴파일러로 빠른 HMR (참조 프로젝트 채택) |
| `react-ts`     | Babel 기반 — SWC 미지원 플러그인(특정 babel-macro) 필요 시   |

### 2.2 plugin-react-swc vs plugin-react

| 항목     | `@vitejs/plugin-react-swc`    | `@vitejs/plugin-react` (Babel)          |
| -------- | ----------------------------- | --------------------------------------- |
| 컴파일러 | SWC (Rust)                    | Babel                                   |
| 속도     | 빠름 (대형 프로젝트 HMR 우위) | 상대적으로 느림                         |
| 플러그인 | SWC 플러그인 생태계 (제한적)  | Babel 플러그인 전체 (예: emotion macro) |
| 권장     | **기본 선택** (참조 프로젝트) | Babel 전용 변환 필요 시에만             |

> 결론: 특별한 Babel 매크로/변환이 없으면 `react-swc`가 기본. 참조 프로젝트의 `apps/MIMS`는 `@vitejs/plugin-react-swc@^4` 사용.

### 2.3 추가 Vite 플러그인 (참조 프로젝트 채택)

| 플러그인                   | 용도                                                           |
| -------------------------- | -------------------------------------------------------------- |
| `vite-tsconfig-paths`      | tsconfig `paths` 별칭을 Vite에 자동 반영 (별칭 중복 정의 제거) |
| `vite-plugin-svgr`         | SVG를 React 컴포넌트로 import (`import { ReactComponent }`)    |
| `rollup-plugin-visualizer` | 번들 분석 (조건부 로딩, §9 참조)                               |

---

## 3. 라우팅 — react-router v7 (Data Router)

### 3.1 두 가지 사용 방식

react-router v7은 **Data Router**(`createBrowserRouter` + loader/action) 와 전통 **Component Router**(`<BrowserRouter><Routes>`)를 모두 지원한다.

| 방식                 | 사용 시점                                                          |
| -------------------- | ------------------------------------------------------------------ |
| **Data Router**      | loader/action으로 라우트 단위 데이터 패칭·뮤테이션, 권장 (신규)    |
| **Component Router** | 단순 SPA, 데이터 패칭을 TanStack Query에 일임 (참조 프로젝트 채택) |

> 참조 `MIMS`는 `<BrowserRouter>` + `<Routes>` 컴포넌트 라우팅을 쓰고, 데이터는 TanStack Query가 전담한다 (라우터 loader 미사용). 신규 프로젝트는 loader 기반 데이터 프리패칭이 필요한지에 따라 선택.

### 3.2 Data Router 기본 패턴

```tsx
// src/app/router.tsx
import { createBrowserRouter } from "react-router";
import { RootLayout } from "@/app/RootLayout";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      { index: true, lazy: () => import("@/pages/home/HomePage") },
      {
        path: "statistics",
        lazy: () => import("@/pages/statistics/StatisticsPage"),
      },
      { path: "*", element: <NotFound /> },
    ],
  },
]);
```

```tsx
// src/main.tsx
import { RouterProvider } from "react-router";
import { router } from "@/app/router";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

### 3.3 Component Router 패턴 (참조 프로젝트)

```tsx
// src/main.tsx — 참조 프로젝트 그대로
import { BrowserRouter } from "react-router-dom";

createRoot(document.getElementById("root")!).render(
  <BrowserRouter>
    <StrictMode>
      <App />
    </StrictMode>
  </BrowserRouter>,
);
```

```tsx
// src/routes/routes.tsx
import { Route, Routes } from "react-router-dom";

const Routers = () => (
  <Routes>
    <Route path="/statistics" element={<Statistics />} />
    <Route path="/" element={<Home />} />
    <Route path="*" element={<NotFound />} />
  </Routes>
);
```

### 3.4 코드 스플리팅 (라우트 단위)

```tsx
// React.lazy + Suspense — 라우트별 청크 분리
import { lazy, Suspense } from "react";
const StatisticsPage = lazy(() => import("@/pages/statistics/StatisticsPage"));

<Suspense fallback={<PageSkeleton />}>
  <StatisticsPage />
</Suspense>;
```

> Data Router는 라우트 객체의 `lazy` 옵션이 더 간결(loader까지 함께 lazy). 둘 다 라우트 단위 청크를 생성해 초기 번들을 줄인다.

---

## 4. 상태관리

### 4.1 상태 종류별 도구 매핑

| 상태 종류           | 도구                         | 비고                                               |
| ------------------- | ---------------------------- | -------------------------------------------------- |
| **서버 상태**       | **TanStack Query v5**        | API 캐시·재요청·무효화. 참조 프로젝트 채택         |
| **클라이언트 전역** | **Zustand**                  | 가벼운 전역 스토어 (테마·UI 토글·세션 등)          |
| **소수 전역 값**    | React Context                | 거의 안 변하는 값(테마·로케일). 빈번 변경엔 부적합 |
| **로컬/폼 상태**    | `useState` / React Hook Form | 컴포넌트 내부 상태, 폼은 §5                        |

> 원칙: **서버에서 온 데이터는 Context/Zustand에 복제하지 말 것.** 서버 상태는 TanStack Query 캐시가 단일 출처(SSOT). 전역 스토어는 순수 클라이언트 상태만.

### 4.2 TanStack Query 설정 (참조 프로젝트)

```tsx
// src/app/App.tsx — 참조 프로젝트 QueryClient 기본값
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000, // 1분간 fresh
      gcTime: 5 * 60_000, // 5분 후 GC
      refetchOnWindowFocus: false,
      retry: 1,
    },
    mutations: { retry: 0 },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AppLayout>
        <Routers />
      </AppLayout>
      {import.meta.env.DEV && <ReactQueryDevtools initialIsOpen={false} />}
    </QueryClientProvider>
  );
}
```

### 4.3 query key 컨벤션

```ts
// queryKey는 배열로 계층화 — 부분 무효화 용이
useQuery({ queryKey: ["users", filters], queryFn: () => fetchUsers(filters) });
queryClient.invalidateQueries({ queryKey: ["users"] }); // 하위 전부 무효화
```

### 4.4 Zustand 기본 패턴

```ts
// src/shared/store/uiStore.ts
import { create } from "zustand";

interface UiState {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
}

export const useUiStore = create<UiState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));
```

---

## 5. 폼 & 검증 — React Hook Form + Zod

### 5.1 연동 패턴

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

// 1) Zod 스키마로 검증 규칙 + 타입 단일 정의
const loginSchema = z.object({
  email: z.string().email("이메일 형식이 올바르지 않습니다."),
  password: z.string().min(8, "비밀번호는 8자 이상입니다."),
});
type LoginForm = z.infer<typeof loginSchema>; // 타입 자동 추론

// 2) zodResolver로 RHF에 연결
function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginForm>({ resolver: zodResolver(loginSchema) });

  const onSubmit = handleSubmit(async (data) => {
    // data는 LoginForm 타입으로 검증 통과된 값
    await login(data);
  });

  return (
    <form onSubmit={onSubmit}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}
      <button disabled={isSubmitting}>로그인</button>
    </form>
  );
}
```

> 핵심: **Zod 스키마 하나로 런타임 검증 + 정적 타입(`z.infer`)을 동시 확보**. 스키마는 API 요청/응답 검증에도 재사용 (참조 프로젝트 `features/url-shortener/schemas.ts`처럼 `z.object` + `safeParse`).

### 5.2 설치 패키지

```bash
pnpm add react-hook-form zod @hookform/resolvers
```

---

## 6. 스타일링 — Tailwind CSS v4 vs CSS Modules

### 6.1 선택 기준

| 조건                                              | 선택                                      |
| ------------------------------------------------- | ----------------------------------------- |
| 유틸리티 우선·빠른 프로토타이핑·디자인 토큰       | **Tailwind CSS v4** (참조 프로젝트 채택)  |
| 컴포넌트 단위 캡슐화·복잡한 셀렉터·기존 SCSS 자산 | **CSS Modules** (`*.module.scss`)         |
| UI 컴포넌트 라이브러리 사용                       | PrimeReact + `tailwindcss-primeui` (참조) |

> 참조 프로젝트는 **Tailwind 4 + PrimeReact + `tailwindcss-primeui`** 조합과 **CSS Modules(`Nav.module.scss`)** 를 혼용한다. 레이아웃·유틸은 Tailwind, 컴포넌트 고유 스타일은 CSS Modules로 분리.

### 6.2 Tailwind v4 설정 (Vite 플러그인 방식)

> Tailwind v4는 `@tailwindcss/vite` 플러그인으로 PostCSS 설정 없이 동작. `tailwind.config.js`가 선택사항이 되고 CSS에서 `@theme`로 토큰 정의.

```ts
// vite.config.ts
import tailwindcss from "@tailwindcss/vite";
export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

```css
/* src/index.css */
@import "tailwindcss";

/* PrimeReact 연동 시 */
@import "tailwindcss-primeui";

/* 디자인 토큰 (v4 @theme) */
@theme {
  --color-brand: #2563eb;
  --font-sans: "Pretendard", sans-serif;
}
```

### 6.3 CSS Modules 패턴 (참조 프로젝트)

```tsx
// Nav.tsx
import styles from "./Nav.module.scss";
<nav className={styles.nav}>...</nav>;
```

> SCSS 사용 시 `sass-embedded`를 devDependency로 추가 (참조 프로젝트).

---

## 7. 디렉토리 구조 — FSD(Feature-Sliced Design)

### 7.1 레이어 계층 (단방향 의존)

```
app → widgets → features → entities → shared
(상위 레이어만 하위 레이어를 import. 역방향·동일 레이어 간 import 금지)
```

| 레이어     | 책임                                                            |
| ---------- | --------------------------------------------------------------- |
| `shared`   | 프레임워크 무관 공용 (UI 키트, api 클라이언트, 유틸, 상수)      |
| `entities` | 비즈니스 엔티티 (User, Product 등 도메인 모델·기본 CRUD)        |
| `features` | 사용자 행동 단위 기능 (로그인, 단축URL 생성 등 — 참조 프로젝트) |
| `widgets`  | 여러 feature/entity를 조합한 독립 UI 블록 (Header, Sidebar)     |
| `app`      | 앱 초기화 (Provider, 라우터, 전역 스타일 — 진입점)              |

### 7.2 슬라이스 내부 세그먼트

각 feature/entity 슬라이스는 동일한 세그먼트 구조를 가진다 (참조 `features/url-shortener/`):

```
features/url-shortener/
├── api.ts        # API 호출 (axios + zod safeParse)
├── schemas.ts    # Zod 스키마 (검증 + 타입 원천)
├── types.ts      # 타입 정의
├── ui/           # 컴포넌트 (해당 feature 전용)
├── model/        # 상태·훅 (useXxx)
└── index.ts      # public API (배럴 — 외부는 이것만 import)
```

> **public API 규칙**: 슬라이스 외부에서는 `index.ts`가 export한 것만 import. 내부 파일 직접 import 금지 (캡슐화). 참조 프로젝트는 `eslint-plugin-no-relative-import-paths`로 상대경로 import를 차단.

### 7.3 단일 앱 디렉토리 템플릿

```
my-spa/
├── src/
│   ├── app/                 # 진입점·Provider·라우터·전역 스타일
│   │   ├── App.tsx
│   │   ├── router.tsx
│   │   └── providers/
│   ├── pages/               # 라우트별 페이지 (FSD pages 레이어)
│   ├── widgets/             # 조합 UI 블록
│   ├── features/            # 기능 슬라이스
│   ├── entities/            # 도메인 엔티티
│   ├── shared/
│   │   ├── api/             # axios 인스턴스, 공통 config
│   │   ├── ui/              # 공용 컴포넌트
│   │   ├── lib/             # 유틸
│   │   └── config/          # 상수·env 래퍼
│   ├── main.tsx
│   └── vite-env.d.ts
├── public/
├── index.html
├── vite.config.ts
├── eslint.config.js
├── tsconfig.json
└── package.json
```

### 7.4 모노레포 시 FSD 패키지 분리 (참조 프로젝트)

> 참조 `mirabell-web-mono`는 FSD 레이어를 **워크스페이스 패키지**로 분리하여 여러 앱이 공유한다.

```
packages/core/
├── shared/      # @scope/shared    — API_ENDPOINTS, ApiCommonConfig 등
├── entities/    # @scope/entities
├── features/    # @scope/features  — url-shortener 등 슬라이스
└── widgets/     # @scope/widgets
apps/
├── MIMS/        # 어드민 앱 — @scope/shared, @scope/features 를 workspace:* 로 소비
└── ...
```

---

## 8. 환경변수

### 8.1 `VITE_` 접두사 규칙

```bash
# .env / .env.development / .env.production
VITE_API_BASE_URL=https://api.example.com   # 클라이언트 노출 (번들에 포함됨)
DB_PASSWORD=secret                            # 접두사 없음 → 클라이언트 미노출, 빌드에서 무시
```

> **경고**: `VITE_` 접두사가 붙은 값은 **빌드 결과물(JS 번들)에 그대로 박혀 브라우저에 노출**된다. 시크릿(DB 비밀번호·API secret key)은 절대 `VITE_`로 넣지 말 것. SPA에는 본질적으로 서버 시크릿을 둘 수 없음.

### 8.2 타입 안전한 env (`import.meta.env`)

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_FEATURE_FLAG_BETA?: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

```ts
// 사용
const baseURL = import.meta.env.VITE_API_BASE_URL;
if (import.meta.env.DEV) {
  /* 개발 전용 */
}
```

> 참조 프로젝트는 `vite.config.ts`의 `envDir`로 env 파일 위치를 커스텀(`envDir: "../../env"`)하여 모노레포 공용 env를 한곳에 모은다.

### 8.3 런타임 검증 (선택, 권장)

```ts
// src/shared/config/env.ts — Zod로 env 스키마 검증
import { z } from "zod";

const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
});
export const env = envSchema.parse(import.meta.env); // 누락 시 부팅 실패로 조기 발견
```

---

## 9. 빌드 최적화

### 9.1 manualChunks로 vendor 분리 (참조 프로젝트)

> 대형 의존성을 별도 청크로 분리해 캐시 효율을 높이고 초기 로딩을 줄인다. 참조 `MIMS/vite.config.ts` 그대로:

```ts
// vite.config.ts — build.rollupOptions.output
manualChunks(id) {
  if (!id.includes("node_modules")) return;
  if (/[\\/]node_modules[\\/](react|react-dom|scheduler)[\\/]/.test(id))
    return "react-core";
  if (/[\\/]node_modules[\\/]@tanstack[\\/]/.test(id)) return "tanstack";
  if (/[\\/]node_modules[\\/]react-router[\\/]/.test(id)) return "router";
  if (/[\\/]node_modules[\\/](primereact|@primereact|@primeuix|primeicons)[\\/]/.test(id))
    return "prime";
  if (/[\\/]node_modules[\\/]zod[\\/]/.test(id)) return "zod";
  return "vendor";
}
```

### 9.2 번들 분석 (조건부 visualizer)

```ts
// ANALYZE=1 일 때만 visualizer 활성화 — 평소 빌드 영향 없음
...(process.env.ANALYZE
  ? [visualizer({ filename: "dist/stats.html", open: true, gzipSize: true, brotliSize: true })]
  : []),
```

```bash
ANALYZE=1 pnpm build   # dist/stats.html 생성·자동 오픈
```

### 9.3 기타 빌드 옵션 (참조 프로젝트)

| 옵션                             | 값                 | 효과                                    |
| -------------------------------- | ------------------ | --------------------------------------- |
| `build.target`                   | `"esnext"`         | 최신 문법 유지, 다운레벨 변환 최소화    |
| `build.minify`                   | `"esbuild"`        | 빠른 압축                               |
| `build.sourcemap`                | `false`            | 프로덕션 소스맵 비공개 (필요 시 hidden) |
| `optimizeDeps.holdUntilCrawlEnd` | `true`             | 의존성 사전 번들링 안정화               |
| 파일명 해시                      | `[name]-[hash].js` | 캐시 무효화 (불변 캐시 + 해시)          |

---

## 10. 설정 파일 스니펫

### 10.1 vite.config.ts (참조 프로젝트 전체 골격)

```ts
import react from "@vitejs/plugin-react-swc";
import { visualizer } from "rollup-plugin-visualizer";
import { defineConfig } from "vite";
import svgr from "vite-plugin-svgr";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  // 모노레포 공용 env (단일 앱은 생략 가능)
  // envDir: "../../env",
  plugins: [
    svgr(),
    react(),
    tsconfigPaths(),
    ...(process.env.ANALYZE
      ? [
          visualizer({
            filename: "dist/stats.html",
            open: true,
            gzipSize: true,
            brotliSize: true,
          }),
        ]
      : []),
  ],

  // 모노레포 HMR (상위 디렉토리 접근 허용) — 단일 앱은 생략
  server: {
    fs: { allow: ["../.."] },
  },

  build: {
    target: "esnext",
    minify: "esbuild",
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (!id.includes("node_modules")) return;
          if (/[\\/]node_modules[\\/](react|react-dom|scheduler)[\\/]/.test(id))
            return "react-core";
          if (/[\\/]node_modules[\\/]@tanstack[\\/]/.test(id))
            return "tanstack";
          if (/[\\/]node_modules[\\/]react-router[\\/]/.test(id))
            return "router";
          if (/[\\/]node_modules[\\/]zod[\\/]/.test(id)) return "zod";
          return "vendor";
        },
        chunkFileNames: "[name]-[hash].js",
        entryFileNames: "[name]-[hash].js",
        assetFileNames: "[ext]/[name]-[hash].[ext]",
      },
    },
  },

  optimizeDeps: {
    include: ["react", "react-dom"],
    holdUntilCrawlEnd: true,
  },
});
```

### 10.2 tsconfig.json (앱 단위, base extends)

> strict 기본값은 `PROJECT_SETUP_GUIDE.md` §4.1 참조. SPA 앱은 base를 extends하고 path 별칭만 추가.

```jsonc
{
  "extends": "@scope/typescript-config/tsconfig.base.json", // 모노레포
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }, // vite-tsconfig-paths가 Vite에 자동 반영
    "types": ["vite/client"],
  },
  "include": ["src"],
}
```

### 10.3 package.json scripts (참조 프로젝트 앱 단위)

```jsonc
{
  "scripts": {
    "dev": "vite",
    "build": "tsc --build && vite build", // 타입체크 후 빌드
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --max-warnings 0 --cache --fix",
    "type": "tsc --pretty --noEmit --incremental",
  },
}
```

### 10.4 기본 진입 파일

```tsx
// src/main.tsx
import "@/app/index.css"; // 전역 스타일·Tailwind
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "@/app/App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

```html
<!-- index.html -->
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My SPA</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## 11. 셋업 완료 검증 체크리스트

신규 SPA 생성 직후 **실제 실행**하여 동작을 증명한다 (`AI_BEHAVIOR.md`: 증명 없는 완료 금지).

- [ ] `pnpm install` 성공 (`only-allow pnpm` 통과)
- [ ] `pnpm dev` 후 브라우저에서 `localhost:5173` 렌더 확인
- [ ] `pnpm type` / `pnpm lint` 무오류
- [ ] `pnpm build` 성공 + `dist/` 생성, 청크 분리(`react-core`·`vendor` 등) 확인
- [ ] `ANALYZE=1 pnpm build`로 번들 리포트 생성 확인
- [ ] 라우트 이동 시 코드 스플리팅 청크가 lazy 로드되는지 네트워크 탭 확인
- [ ] `VITE_` 미접두 시크릿이 번들에 포함되지 않는지 확인 (dist grep)
- [ ] (참조 프로젝트 형) TanStack Query Devtools가 DEV에서만 노출되는지 확인
