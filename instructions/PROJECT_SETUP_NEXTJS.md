# Next.js 셋업 지침 (PROJECT_SETUP_NEXTJS)

> Next.js **App Router** 기반 신규 프로젝트 셋업 단계별 지침.
> 공통 부트스트랩(lint/format/commitlint/husky·tsconfig·모노레포)은 `PROJECT_SETUP_GUIDE.md`, 코딩 규칙은 `LANGUAGE_GUIDELINES_REACT.md` / `LANGUAGE_GUIDELINES_TYPESCRIPT.md` 참조. 이 문서는 **Next.js App Router 스택 구성 전용**.
>
> 순수 CSR/SPA(SSR 불필요)는 이 문서 대신 `PROJECT_SETUP_REACT.md`(Vite) 참조.
> 공유 lint/format/tsconfig·TanStack Query·Zod 패턴은 사내 참조 프로젝트 `mirabell-web-mono`와 일관성을 맞춘다.

---

## 1. 언제 Next.js(App Router)를 선택하나

| 조건                                        | 적합 여부 | 비고                                              |
| ------------------------------------------- | --------- | ------------------------------------------------- |
| SSR/SEO 필요 (검색 노출·OG 메타·소셜 공유)  | 적합      | 서버 렌더 HTML로 크롤러·미리보기 대응             |
| 풀스택 (프론트 + 백엔드 한 저장소)          | 적합      | Route Handler·Server Action으로 BE 일체화         |
| ISR/SSG 활용 (정적 생성 + 주기 재생성)      | 적합      | 블로그·문서·상품 카탈로그 등                      |
| 첫 화면 렌더 속도·코어 웹 바이탈 중요       | 적합      | 서버 렌더 + 스트리밍으로 TTFB·LCP 개선            |
| 서버에서 DB·시크릿 직접 접근                | 적합      | Server Component·Server Action에서 안전하게 처리  |
| **검색 노출 불필요한 사내 어드민/대시보드** | 부적합    | → `PROJECT_SETUP_REACT.md` (Vite SPA)가 더 가벼움 |
| **정적 호스팅(S3·CDN)만으로 충분**          | 부적합    | SPA가 인프라 단순 (Next.js는 서버 런타임 필요)    |

> 판단 기준: **"서버 렌더링·SEO·서버측 데이터 접근 중 하나라도 필요한가?"** Yes면 Next.js. 전부 No면 Vite SPA. 모호하면 추론 전 사용자에게 질문 (`AI_BEHAVIOR.md` 질문 게이트).

---

## 2. 프로젝트 생성 & App Router 기본 구조

### 2.1 생성

```bash
pnpm create next-app@latest my-app --ts --app --eslint --import-alias "@/*"
# 프롬프트: App Router(Yes), src/ 디렉토리(권장 Yes), Tailwind(선택)
```

> 패키지 매니저는 `pnpm` 통일 (참조 프로젝트 정책). `--app`으로 App Router 강제, `--app` 미선택 시 구형 Pages Router가 되므로 주의.

### 2.2 App Router 특수 파일

`app/` 디렉토리 내 **약속된 파일명**이 라우트 동작을 결정한다.

| 파일            | 역할                                                              |
| --------------- | ----------------------------------------------------------------- |
| `layout.tsx`    | 공유 레이아웃 (자식 라우트 감쌈, 상태 유지). 루트 필수 (`<html>`) |
| `page.tsx`      | 해당 경로의 페이지 (라우트 공개). 이 파일이 있어야 경로 접근 가능 |
| `loading.tsx`   | Suspense 폴백 (스트리밍 로딩 UI 자동 적용)                        |
| `error.tsx`     | 에러 바운더리 (`"use client"` 필수, `reset()` 제공)               |
| `not-found.tsx` | 404 UI (`notFound()` 호출 또는 미매칭 시)                         |
| `route.ts`      | Route Handler (API 엔드포인트, §6)                                |
| `template.tsx`  | layout과 유사하나 네비게이션마다 리마운트 (애니메이션·로깅 등)    |

### 2.3 루트 레이아웃 (필수)

```tsx
// app/layout.tsx — 루트 레이아웃은 반드시 <html>/<body> 포함
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: { default: "My App", template: "%s | My App" },
  description: "...",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

### 2.4 페이지·로딩·에러 예시

```tsx
// app/page.tsx — Server Component (기본)
export default function HomePage() {
  return <main>홈</main>;
}
```

```tsx
// app/loading.tsx — 자동 Suspense 폴백
export default function Loading() {
  return <PageSkeleton />;
}
```

```tsx
// app/error.tsx — Client Component 필수
"use client";
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <p>문제가 발생했습니다.</p>
      <button onClick={reset}>다시 시도</button>
    </div>
  );
}
```

---

## 3. 렌더링 전략 — Server Component vs Client Component

### 3.1 기본값과 선택 기준

> App Router에서 **모든 컴포넌트는 기본 Server Component**다. 파일 최상단에 `"use client"`를 명시한 경우에만 Client Component가 된다.

| 기준                                   | Server Component (기본) | Client Component (`"use client"`) |
| -------------------------------------- | ----------------------- | --------------------------------- |
| 데이터 패칭 (DB·API·시크릿)            | 가능 (서버에서 직접)    | 불가 (브라우저 노출 위험)         |
| `useState`/`useEffect`/이벤트 핸들러   | 불가                    | 필수 (인터랙션은 Client)          |
| 브라우저 API (`window`·`localStorage`) | 불가                    | 가능                              |
| 번들 크기                              | JS 번들 0 (HTML만)      | 클라이언트 번들에 포함            |
| `useContext`/Zustand/TanStack Query    | 불가                    | 가능                              |

### 3.2 경계 설계 원칙

- **Server를 기본으로**, 인터랙션이 필요한 **가장 말단(leaf)에서만** `"use client"` 선언
- Client Component는 Server Component를 import할 수 없으나, **`children` prop으로 주입**받을 수 있다 (Server를 Client 안에 끼워 넣기)
- `"use client"`는 전염성 — 한 번 선언하면 그 하위 트리 전체가 클라이언트. 경계를 최대한 아래로 내릴 것

```tsx
// 권장: Server에서 데이터 패칭 → Client 컴포넌트에 props 전달
// app/users/page.tsx (Server)
import { UserTable } from "./UserTable";
export default async function Page() {
  const users = await fetchUsers(); // 서버에서 직접 패칭
  return <UserTable initialData={users} />; // 인터랙션은 Client에 위임
}
```

---

## 4. 데이터 패칭

### 4.1 세 가지 경로

| 방식               | 위치             | 사용 시점                                                    |
| ------------------ | ---------------- | ------------------------------------------------------------ |
| `fetch()` (확장)   | Server Component | 초기 렌더 데이터, 캐싱·재검증 제어                           |
| **Server Action**  | 서버 함수        | 폼 제출·뮤테이션 (DB 쓰기), `"use server"`                   |
| **TanStack Query** | Client Component | 클라이언트 측 캐시·폴링·낙관적 업데이트 (참조 프로젝트 일관) |

### 4.2 fetch 캐싱·재검증

> Next.js는 `fetch`를 확장하여 캐시 전략을 옵션으로 제어한다.

```tsx
// 정적 캐시 (기본, SSG 유사) — 빌드 시 또는 첫 요청 후 캐시
const res = await fetch(url, { cache: "force-cache" });

// 매 요청 새로 패칭 (SSR)
const res = await fetch(url, { cache: "no-store" });

// ISR — N초마다 재검증
const res = await fetch(url, { next: { revalidate: 60 } });

// 태그 기반 재검증 (revalidateTag로 무효화)
const res = await fetch(url, { next: { tags: ["users"] } });
```

> 캐싱 기본값은 Next 버전에 따라 다를 수 있어(특히 `fetch` 기본 캐시 정책 변경 이력 존재) **명시적으로 `cache`/`revalidate`를 지정**하는 것을 권장. 동작은 빌드 후 실제 응답 헤더·재검증으로 검증.

### 4.3 Server Action (뮤테이션)

```tsx
// app/actions.ts
"use server";
import { revalidatePath } from "next/cache";
import { z } from "zod";

const schema = z.object({ title: z.string().min(1) });

export async function createPost(formData: FormData) {
  const parsed = schema.safeParse({ title: formData.get("title") });
  if (!parsed.success) return { error: "유효하지 않은 입력" };

  await db.post.create({ data: parsed.data }); // 서버에서 DB 직접
  revalidatePath("/posts"); // 캐시 무효화 → 목록 갱신
}
```

```tsx
// 폼에서 직접 호출 (Client·Server 모두 가능)
<form action={createPost}>
  <input name="title" />
  <button>생성</button>
</form>
```

> Server Action 입력도 **Zod로 검증**한다 (참조 프로젝트의 `safeParse` 패턴 일관). 클라이언트 입력을 신뢰하지 말 것.

### 4.4 TanStack Query 연동 (Client)

```tsx
// app/providers.tsx — Client Provider로 QueryClient 주입
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: { staleTime: 60_000, refetchOnWindowFocus: false, retry: 1 },
          mutations: { retry: 0 },
        },
      }),
  );
  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

```tsx
// app/layout.tsx 의 <body>에서 감싸기
<body>
  <Providers>{children}</Providers>
</body>
```

> Provider는 반드시 Client Component(`"use client"`). `QueryClient`는 `useState` 초기화로 요청 간 공유를 방지(SSR 안전). 서버 데이터 프리패칭이 필요하면 `HydrationBoundary` + `dehydrate` 패턴 사용.

---

## 5. 라우팅 (App Router 고급)

### 5.1 동적 라우트

```
app/
├── posts/
│   ├── [id]/page.tsx         # /posts/123 → params.id
│   ├── [...slug]/page.tsx    # /posts/a/b/c → params.slug = ["a","b","c"]
│   └── [[...slug]]/page.tsx  # /posts 및 하위 전부 (optional catch-all)
```

```tsx
// app/posts/[id]/page.tsx — params는 Promise (await 필요)
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <article>게시글 {id}</article>;
}
```

> 최신 App Router에서 `params`·`searchParams`는 **Promise**로 전달되어 `await`가 필요할 수 있다. 프로젝트 Next 버전의 타입 시그니처를 확인하여 맞출 것.

### 5.2 Route Groups & Parallel/Intercepting Routes

| 패턴                | 문법          | 용도                                                 |
| ------------------- | ------------- | ---------------------------------------------------- |
| Route Group         | `(group)/`    | URL에 노출 안 되는 그룹핑 (레이아웃 분리·정리)       |
| Parallel Routes     | `@slot/`      | 한 레이아웃에 여러 슬롯 동시 렌더 (대시보드 패널 등) |
| Intercepting Routes | `(.)`, `(..)` | 모달 오버레이 (목록 위에 상세를 모달로)              |
| Private Folder      | `_folder/`    | 라우트화 제외 (내부 구현 모음)                       |

```
app/
├── (marketing)/             # Route Group — URL 미반영
│   ├── layout.tsx           # 마케팅 전용 레이아웃
│   └── about/page.tsx       # → /about
├── (app)/
│   ├── layout.tsx           # 인증 레이아웃
│   └── dashboard/
│       ├── @analytics/page.tsx   # Parallel slot
│       ├── @team/page.tsx        # Parallel slot
│       └── layout.tsx            # props로 analytics·team 슬롯 수신
```

---

## 6. API Routes — Route Handlers

### 6.1 `app/api/.../route.ts` 패턴

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

export async function GET(request: NextRequest) {
  const q = request.nextUrl.searchParams.get("q") ?? "";
  const users = await db.user.findMany({ where: { name: { contains: q } } });
  return NextResponse.json(users);
}

const createSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

export async function POST(request: NextRequest) {
  const body = await request.json();
  const parsed = createSchema.safeParse(body); // 입력 검증 필수
  if (!parsed.success) {
    return NextResponse.json(
      { error: parsed.error.flatten() },
      { status: 400 },
    );
  }
  const user = await db.user.create({ data: parsed.data });
  return NextResponse.json(user, { status: 201 });
}
```

### 6.2 규칙

- HTTP 메서드별로 named export (`GET`/`POST`/`PUT`/`PATCH`/`DELETE`)
- 동적 세그먼트: `app/api/users/[id]/route.ts` → `{ params }` (Promise)
- Route Handler는 기본적으로 캐시되지 않음 (GET도 `force-static` 미지정 시 동적)
- **Server Action으로 충분한 내부 뮤테이션은 Route Handler 대신 Action 권장** — 외부 공개 API·웹훅·서드파티 연동에만 Route Handler 사용

---

## 7. 환경변수

### 7.1 `NEXT_PUBLIC_` 접두사 (서버/클라이언트 분리)

```bash
# .env.local (git 제외)
DATABASE_URL=postgres://...              # 서버 전용 — 클라이언트 미노출
API_SECRET=sk_live_...                   # 서버 전용
NEXT_PUBLIC_API_BASE_URL=https://api...  # 클라이언트 노출 (번들 포함)
```

| 접두사         | 노출 범위                                             | 용도                        |
| -------------- | ----------------------------------------------------- | --------------------------- |
| 없음           | **서버 전용** (Server Component·Action·Route Handler) | DB URL·시크릿·API 키        |
| `NEXT_PUBLIC_` | 서버 + **클라이언트 번들 포함**                       | 공개 가능한 base URL·플래그 |

> **경고**: `NEXT_PUBLIC_` 값은 빌드 시 클라이언트 번들에 인라인되어 브라우저에 노출된다. 시크릿은 절대 `NEXT_PUBLIC_`로 두지 말 것. 접두사 없는 변수를 Client Component에서 읽으면 `undefined`.

### 7.2 env 파일 우선순위 & 타입

```
.env.local         # 모든 환경 (git 제외, 시크릿)
.env.development   # next dev
.env.production    # next build/start
.env               # 공통 기본값
```

```ts
// env.ts — Zod 런타임 검증 (권장)
import { z } from "zod";

const serverEnv = z.object({
  DATABASE_URL: z.string().url(),
  API_SECRET: z.string().min(1),
});
const clientEnv = z.object({
  NEXT_PUBLIC_API_BASE_URL: z.string().url(),
});

export const env = {
  ...serverEnv.parse(process.env),
  ...clientEnv.parse(process.env),
};
```

> 클라이언트 번들에는 실제 참조된 `NEXT_PUBLIC_*`만 포함되도록, 클라이언트에서 import하는 모듈에서 서버 전용 env를 참조하지 않도록 분리한다 (`server-only` 패키지로 강제 가능).

---

## 8. 성능

### 8.1 Image 최적화 (`next/image`)

```tsx
import Image from "next/image";

// 자동 최적화: WebP/AVIF 변환, 반응형 srcset, lazy load, CLS 방지
<Image src="/hero.png" alt="히어로" width={1200} height={630} priority />;
```

| 옵션           | 효과                                                        |
| -------------- | ----------------------------------------------------------- |
| `width/height` | 레이아웃 시프트(CLS) 방지 — 명시 필수 (또는 `fill`)         |
| `priority`     | LCP 이미지(첫 화면 히어로)에 지정 → 프리로드                |
| `sizes`        | 반응형 srcset 정확도 향상                                   |
| 외부 도메인    | `next.config.ts`의 `images.remotePatterns`에 허용 등록 필요 |

### 8.2 Font 최적화 (`next/font`)

```tsx
// app/layout.tsx — 셀프 호스팅·자동 서브셋·CLS 0
import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"], display: "swap" });

export default function RootLayout({ children }) {
  return (
    <html lang="ko" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

> `next/font`는 빌드 시 폰트를 셀프 호스팅하여 외부 요청·레이아웃 시프트를 제거한다. 한글 폰트(Pretendard 등)는 `next/font/local`로 로컬 등록.

### 8.3 Bundle Analyzer & 기타

```bash
pnpm add -D @next/bundle-analyzer
ANALYZE=true pnpm build   # 번들 시각화 리포트 생성
```

```ts
// next.config.ts
import withBundleAnalyzer from "@next/bundle-analyzer";
const analyzer = withBundleAnalyzer({
  enabled: process.env.ANALYZE === "true",
});
export default analyzer(nextConfig);
```

| 기법                   | 효과                                                     |
| ---------------------- | -------------------------------------------------------- |
| `next/dynamic`         | 컴포넌트 단위 lazy 로딩 (`ssr: false`로 클라이언트 전용) |
| `loading.tsx` 스트리밍 | 느린 데이터 구간만 Suspense로 스트림                     |
| Server Component 우선  | 클라이언트 JS 최소화 (가장 큰 성능 레버)                 |
| `priority`/`preload`   | LCP 자원 우선 로드                                       |

---

## 9. 디렉토리 구조 템플릿

```
my-app/
├── src/
│   ├── app/                     # App Router 루트 (라우팅 전용)
│   │   ├── (marketing)/         # Route Group
│   │   ├── (app)/               # 인증 영역 Route Group
│   │   ├── api/                 # Route Handlers
│   │   │   └── users/route.ts
│   │   ├── layout.tsx           # 루트 레이아웃 (<html>)
│   │   ├── page.tsx             # 홈
│   │   ├── loading.tsx
│   │   ├── error.tsx
│   │   ├── not-found.tsx
│   │   ├── providers.tsx        # Client Provider (QueryClient 등)
│   │   └── globals.css
│   ├── components/              # 공유 UI (ui/ + 도메인 컴포넌트)
│   ├── features/                # 기능 단위 (FSD 적용 시, 참조 프로젝트 일관)
│   ├── lib/                     # 서버/클라 유틸 (db, auth, fetch 래퍼)
│   ├── hooks/                   # 클라이언트 커스텀 훅
│   ├── schemas/                 # Zod 스키마 (검증 + 타입 원천)
│   ├── types/                   # 전역 타입
│   └── env.ts                   # 환경변수 Zod 검증
├── public/                      # 정적 자산
├── next.config.ts
├── tsconfig.json
├── eslint.config.js
├── postcss.config.mjs           # Tailwind 사용 시
└── package.json
```

> 규모가 커지면 `features/`에 FSD(shared/entities/features/widgets)를 적용해 참조 프로젝트와 레이어 규칙을 일관시킨다 (`PROJECT_SETUP_REACT.md` §7 참조). `app/`은 라우팅·레이아웃에만 집중하고 도메인 로직은 `features/`·`lib/`로 분리.

---

## 10. 설정 파일 스니펫

### 10.1 next.config.ts (기본값)

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  reactStrictMode: true,
  // 외부 이미지 도메인 허용
  images: {
    remotePatterns: [{ protocol: "https", hostname: "cdn.example.com" }],
  },
  // 모노레포에서 워크스페이스 패키지 트랜스파일 (참조 프로젝트형 공유 패키지 소비 시)
  // transpilePackages: ["@scope/shared", "@scope/features"],
  experimental: {
    // typedRoutes: true,   // 링크 타입 안전 (안정화 여부 버전 확인)
  },
};

export default nextConfig;
```

### 10.2 tsconfig.json

> strict 기본값은 `PROJECT_SETUP_GUIDE.md` §4.1 참조. Next.js는 `next dev`가 `next-env.d.ts`·플러그인을 자동 관리.

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve", // Next가 변환 담당
    "noEmit": true,
    "incremental": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] },
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"],
}
```

### 10.3 package.json scripts

```jsonc
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type": "tsc --noEmit",
  },
}
```

---

## 11. 셋업 완료 검증 체크리스트

신규 프로젝트 생성 직후 **실제 실행**하여 동작을 증명한다 (`AI_BEHAVIOR.md`: 증명 없는 완료 금지).

- [ ] `pnpm install` 성공 (`only-allow pnpm` 통과 — 정책 적용 시)
- [ ] `pnpm dev` 후 `localhost:3000` 렌더 확인
- [ ] `pnpm type` / `pnpm lint` 무오류
- [ ] `pnpm build` 성공 — 라우트별 렌더 방식(Static/SSG/Dynamic) 출력 확인
- [ ] Server Component에서 시크릿 env 접근 + 브라우저 번들에 미노출 확인 (build grep)
- [ ] `NEXT_PUBLIC_` 미접두 변수를 Client Component에서 읽으면 `undefined`인지 확인
- [ ] Route Handler(`/api/...`) 응답 + Zod 검증 실패 시 400 반환 확인
- [ ] Server Action 폼 제출 후 `revalidatePath`로 화면 갱신 확인
- [ ] `loading.tsx` 스트리밍·`error.tsx` 바운더리 동작 확인
- [ ] `ANALYZE=true pnpm build`로 번들 리포트 생성 확인
