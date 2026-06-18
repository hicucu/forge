# Node.js 서버 신규 프로젝트 셋업 지침 (PROJECT_SETUP_NODEJS)

> Node.js 백엔드 서버를 처음 생성할 때 Claude Code(또는 AI)가 참조하는 **프레임워크 선택·설정 지침**.
> 프레임워크 선택 → TypeScript 설정 → 미들웨어 구성 → 로깅 → 에러 핸들링까지를 다룬다.
> 코딩 규칙은 `DEVELOPMENT.md` / `LANGUAGE_GUIDELINES_TYPESCRIPT.md`, 전체 프로젝트 셋업 흐름은 `PROJECT_SETUP_GUIDE.md` 참조.

---

## 0. 셋업 진행 순서 (전체 개요)

```
1. 프레임워크 선택 (의사결정 기준)
   ↓
2. TypeScript 설정 (tsconfig, 개발 서버)
   ↓
3. 필수 미들웨어·패키지 설치
   ↓
4. 로깅 설정 (pino, Correlation ID, redact)
   ↓
5. 디렉토리 구조 생성 (프레임워크별 템플릿)
   ↓
6. 환경변수 관리 (zod 검증, 시작 시 실패 즉시 종료)
   ↓
7. 전역 에러 핸들러 구성
   ↓
8. 설정 파일 최종 점검 (tsconfig, package.json scripts)
```

> **원칙**: 프레임워크 선택이 모호하거나 팀 컨텍스트가 불명확하면 추론 전에 사용자에게 먼저 질문한다 (`AI_BEHAVIOR.md` 질문 게이트).

---

## 1. 프레임워크 선택 기준

### 1.1 의사결정 트리

```
대규모 팀 또는 엔터프라이즈 프로젝트?
├─ Yes → NestJS (DI 컨테이너, 모듈 구조)
└─ No
   ├─ 엣지/서버리스(Cloudflare Workers, Vercel Edge)?
   │   └─ Yes → Hono
   └─ No
       ├─ 고성능 요구 또는 Express 대체?
       │   └─ Yes → Fastify
       └─ 경량·단순 API, 레거시 코드베이스 호환?
           └─ Yes → Express
```

### 1.2 선택 기준 요약

| 프레임워크  | 선택 조건                                                           | 비고                                           |
| ----------- | ------------------------------------------------------------------- | ---------------------------------------------- |
| **NestJS**  | 엔터프라이즈, 대규모 팀, DI 컨테이너·모듈 구조·데코레이터 패턴 필요 | Angular 영향 받은 구조. 학습 곡선 있음         |
| **Express** | 경량, 유연성 우선, 단순 REST API, 레거시 Node.js 코드베이스 통합    | 생태계 가장 방대. 타입 지원은 `@types/express` |
| **Fastify** | 고성능 (Express 대비 2-3배), 타입 안전 라우팅, Express 대체 고려 시 | JSON 직렬화 최적화 내장. 플러그인 시스템 강점  |
| **Hono**    | 엣지/서버리스 환경 (Cloudflare Workers, Vercel Edge, Deno), 초경량  | Web API 표준 기반. Node.js + 엣지 동시 지원    |

---

## 2. TypeScript 설정

### 2.1 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

**설정값 선택 이유**

| 옵션                         | 이유                                                                |
| ---------------------------- | ------------------------------------------------------------------- |
| `target: ES2022`             | Node.js 18+ 기준, 최신 ECMAScript 기능 사용 가능                    |
| `module: Node16`             | ESM/CJS 혼용 환경에서 올바른 import 해석. Bundler는 esbuild 사용 시 |
| `moduleResolution: Node16`   | `module: Node16`과 짝. `.js` 확장자 명시 강제                       |
| `noUncheckedIndexedAccess`   | 배열/객체 인덱스 접근 시 `undefined` 가능성 타입 강제 포함          |
| `exactOptionalPropertyTypes` | `undefined` 명시 없이 optional 프로퍼티 할당 금지                   |

> Bundler(esbuild, Bun) 기반 프로젝트: `"module": "ESNext"`, `"moduleResolution": "Bundler"` 사용.

### 2.2 개발 서버 도구 선택

| 도구                | 사용 시점                                                     | 명령어 예시                          |
| ------------------- | ------------------------------------------------------------- | ------------------------------------ |
| `tsx`               | 신규 프로젝트 권장. 빠른 ESM 변환, `--watch` 내장             | `tsx watch src/main.ts`              |
| `ts-node-dev`       | 레거시 CommonJS 프로젝트, `--respawn`·`--transpile-only` 필요 | `ts-node-dev --respawn src/main.ts`  |
| `nodemon + ts-node` | 커스텀 nodemon 설정이 필요한 경우                             | `nodemon --exec ts-node src/main.ts` |

> `tsx` 우선 권장: 별도 설정 없이 ESM·CJS 모두 동작, 빠른 HMR.

### 2.3 빌드 도구

| 목적            | 도구                  | 비고                           |
| --------------- | --------------------- | ------------------------------ |
| 단순 트랜스파일 | `tsc`                 | 타입 체크 + 출력. CI 빌드 기준 |
| 빠른 번들링     | `esbuild`             | 타입 체크 생략. 속도 중요 시   |
| 모노레포 빌드   | `tsup` (esbuild 래퍼) | dual CJS/ESM 출력 간편         |

---

## 3. 필수 미들웨어 / 패키지

### 3.1 패키지 선택 기준 테이블

| 분류                | 권장 패키지                               | 대안                       | 선택 이유                                                                  |
| ------------------- | ----------------------------------------- | -------------------------- | -------------------------------------------------------------------------- |
| **로깅**            | `pino` + `pino-http`                      | `winston`                  | 구조화 JSON 기본, 성능 우수. winston은 레거시 코드베이스 통합 시           |
| **보안**            | `helmet`, `cors`, `rate-limiter-flexible` | `express-rate-limit`       | helmet: 보안 헤더 일괄 설정. rate-limiter-flexible: Redis 지원             |
| **검증**            | `zod`                                     | `class-validator` (NestJS) | zod: 런타임 + 타입 추론 동시. NestJS는 class-validator + class-transformer |
| **환경변수**        | `dotenv` + `zod`                          | `t3-env`                   | t3-env: Next.js/Vite 통합 시 유리. 순수 Node.js는 dotenv+zod               |
| **HTTP 클라이언트** | `ky` (fetch 래퍼)                         | `axios`                    | ky: 표준 fetch 기반, 경량. axios: 더 풍부한 인터셉터 필요 시               |
| **테스트**          | `vitest` + `supertest`                    | `jest` + `supertest`       | vitest: 빠른 실행, ESM 네이티브, jest와 호환 API                           |

### 3.2 설치 명령어 예시 (Express 기준)

```bash
# 코어
npm install express
npm install -D typescript tsx @types/express @types/node

# 로깅
npm install pino pino-http
npm install -D pino-pretty

# 보안
npm install helmet cors rate-limiter-flexible

# 검증 + 환경변수
npm install zod dotenv

# 테스트
npm install -D vitest supertest @types/supertest
```

---

## 4. 로깅 설정 (필수 — 상세)

### 4.1 pino 기본 설정

```typescript
// src/config/logger.ts
import pino from "pino";

const isDev = process.env.NODE_ENV !== "production";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? (isDev ? "debug" : "info"),
  // 민감 정보 제외 (redact)
  redact: {
    paths: [
      "req.headers.authorization",
      "req.headers.cookie",
      "body.password",
      "body.token",
      "body.secret",
    ],
    censor: "[REDACTED]",
  },
  // 개발 환경: pino-pretty 포맷터
  transport: isDev
    ? {
        target: "pino-pretty",
        options: { colorize: true, translateTime: "SYS:standard" },
      }
    : undefined,
  // 기본 필드
  base: {
    env: process.env.NODE_ENV,
    version: process.env.npm_package_version,
  },
  // ISO 8601 타임스탬프
  timestamp: pino.stdTimeFunctions.isoTime,
});
```

### 4.2 Correlation ID 미들웨어 패턴 (요청 추적)

AsyncLocalStorage를 사용해 요청 단위 컨텍스트를 전파한다. 외부 라이브러리 의존 없이 구현 가능.

```typescript
// src/middleware/correlation-id.middleware.ts
import { AsyncLocalStorage } from "async_hooks";
import { randomUUID } from "crypto";
import type { Request, Response, NextFunction } from "express";

interface RequestContext {
  correlationId: string;
  logger: pino.Logger;
}

export const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

export function correlationIdMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  const correlationId =
    (req.headers["x-correlation-id"] as string) ?? randomUUID();

  res.setHeader("x-correlation-id", correlationId);

  const requestLogger = logger.child({ correlationId });

  asyncLocalStorage.run({ correlationId, logger: requestLogger }, next);
}

// 어디서든 현재 요청의 logger 접근
export function getLogger(): pino.Logger {
  return asyncLocalStorage.getStore()?.logger ?? logger;
}
```

사용 예시:

```typescript
// src/services/user.service.ts
import { getLogger } from "../middleware/correlation-id.middleware.js";

export async function createUser(data: CreateUserDto) {
  const log = getLogger();
  log.info({ userId: data.email }, "사용자 생성 시작");
  // ...
}
```

### 4.3 요청/응답 자동 로깅 (pino-http)

```typescript
// src/middleware/http-logger.middleware.ts
import pinoHttp from "pino-http";
import { logger } from "../config/logger.js";

export const httpLogger = pinoHttp({
  logger,
  // 헬스체크·메트릭 경로 로그 생략
  autoLogging: {
    ignore: (req) =>
      ["/health", "/metrics", "/favicon.ico"].includes(req.url ?? ""),
  },
  // 요청/응답 커스텀 직렬화
  serializers: {
    req(req) {
      return {
        method: req.method,
        url: req.url,
        userAgent: req.headers["user-agent"],
      };
    },
    res(res) {
      return { statusCode: res.statusCode };
    },
  },
  // 요청 시작 시 로그 생략 (응답만 기록)
  quietReqLogger: true,
});
```

### 4.4 로그 레벨별 사용 기준

| 레벨    | 사용 기준                                     | 예시                                        |
| ------- | --------------------------------------------- | ------------------------------------------- |
| `debug` | 개발·디버깅 전용. 프로덕션 기본 비활성        | DB 쿼리 파라미터, 함수 진입 추적            |
| `info`  | 정상 흐름의 주요 이벤트                       | 서버 시작, 요청 처리 완료, 배치 작업 완료   |
| `warn`  | 즉각적이지 않으나 주의 필요. 서비스 중단 아님 | 재시도 발생, deprecated API 사용, 느린 쿼리 |
| `error` | 복구 가능한 오류. 요청 실패지만 서비스 계속   | 외부 API 오류, DB 일시 연결 실패            |
| `fatal` | 복구 불가 오류. 프로세스 즉시 종료 전         | DB 연결 완전 실패, 설정 파일 누락           |

### 4.5 Good / Bad 예시

```typescript
// Bad — console.log 사용, 비구조화 메시지
console.log("user created: " + userId);
console.error(error);

// Bad — 에러 객체를 문자열로 직렬화
logger.error("에러 발생: " + error.message);

// Good — 구조화 로그, 에러 객체 직접 전달
logger.info({ userId, email }, "사용자 생성 완료");
logger.error({ err: error, userId }, "사용자 생성 실패");

// Good — Correlation ID 포함 (AsyncLocalStorage 사용 시 자동)
const log = getLogger();
log.warn({ thresholdMs: 500, actualMs }, "느린 DB 쿼리 감지");
```

> `console.log` / `console.error`는 프로덕션 코드에서 **금지**. ESLint `no-console` 규칙으로 강제.

---

## 5. 디렉토리 구조 템플릿

### 5.1 NestJS 패턴

```
src/
├── main.ts                   # 진입점. 앱 부트스트랩 + 전역 설정
├── app.module.ts             # 루트 모듈
├── app.controller.ts
├── config/
│   ├── env.config.ts         # 환경변수 zod 검증·타입 정의
│   └── logger.config.ts      # pino 설정
├── common/
│   ├── decorators/           # 커스텀 데코레이터 (@CurrentUser 등)
│   ├── filters/              # 전역 예외 필터 (AllExceptionsFilter)
│   ├── guards/               # 인증·권한 가드
│   ├── interceptors/         # 로깅·응답 변환 인터셉터
│   ├── middleware/           # 미들웨어 (correlation-id 등)
│   └── pipes/                # 전역 파이프 (ValidationPipe)
└── modules/
    └── users/
        ├── users.module.ts
        ├── users.controller.ts
        ├── users.service.ts
        ├── dto/
        │   ├── create-user.dto.ts
        │   └── update-user.dto.ts
        ├── entities/
        │   └── user.entity.ts
        └── users.service.spec.ts
```

### 5.2 Express / Fastify 패턴

```
src/
├── main.ts                   # 진입점. 서버 시작
├── app.ts                    # Express/Fastify 앱 인스턴스 구성
├── config/
│   ├── env.ts                # 환경변수 zod 검증
│   └── logger.ts             # pino 인스턴스
├── middleware/
│   ├── correlation-id.ts     # Correlation ID + AsyncLocalStorage
│   ├── http-logger.ts        # pino-http 자동 요청 로깅
│   └── error-handler.ts      # 전역 에러 핸들러
├── routes/
│   ├── index.ts              # 라우터 통합 진입점
│   └── users/
│       ├── users.router.ts
│       └── users.schema.ts   # zod 스키마
├── controllers/
│   └── users.controller.ts   # 요청 처리, DTO 변환
├── services/
│   └── users.service.ts      # 비즈니스 로직
└── types/
    └── express.d.ts          # Express Request 타입 확장
```

---

## 6. 환경변수 관리

### 6.1 .env 파일 체계

| 파일              | 목적                                        | git 추적 여부                 |
| ----------------- | ------------------------------------------- | ----------------------------- |
| `.env.example`    | 필요한 변수 목록 (값 없음). 팀 공유         | 추적                          |
| `.env`            | 로컬 개발 실제 값                           | 제외 (.gitignore)             |
| `.env.test`       | 테스트 환경 고정 값                         | 추적 가능 (민감 정보 제외 시) |
| `.env.production` | 프로덕션 값 (CI/CD 시크릿 관리 도구로 주입) | **절대 추적 금지**            |

### 6.2 zod 기반 환경변수 검증 스키마

```typescript
// src/config/env.ts
import { z } from "zod";
import { config } from "dotenv";

// .env 파일 로드 (프로덕션은 CI/CD가 주입하므로 생략 가능)
config();

const envSchema = z.object({
  // 서버
  NODE_ENV: z
    .enum(["development", "test", "production"])
    .default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  HOST: z.string().default("0.0.0.0"),

  // 로깅
  LOG_LEVEL: z
    .enum(["debug", "info", "warn", "error", "fatal"])
    .default("info"),

  // 데이터베이스 (예시)
  DATABASE_URL: z.string().url(),

  // 인증 (예시)
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default("7d"),

  // 외부 API (예시 — 선택적)
  EXTERNAL_API_URL: z.string().url().optional(),
  EXTERNAL_API_KEY: z.string().optional(),
});

// 검증 실패 시 상세 오류 출력 후 즉시 종료
const result = envSchema.safeParse(process.env);

if (!result.success) {
  console.error("환경변수 검증 실패:");
  console.error(result.error.flatten().fieldErrors);
  process.exit(1);
}

export const env = result.data;

// 타입 내보내기
export type Env = z.infer<typeof envSchema>;
```

### 6.3 서버 시작 시 검증 순서

```typescript
// src/main.ts
import "./config/env.js"; // 1. 환경변수 검증 (실패 시 process.exit(1))
import { env } from "./config/env.js";
import { logger } from "./config/logger.js";
import { createApp } from "./app.js";

async function bootstrap() {
  const app = await createApp();

  app.listen(env.PORT, env.HOST, () => {
    logger.info({ port: env.PORT, env: env.NODE_ENV }, "서버 시작");
  });
}

bootstrap().catch((err) => {
  logger.fatal({ err }, "서버 시작 실패");
  process.exit(1);
});
```

> 환경변수 import를 최상단에 배치해 다른 모듈보다 먼저 검증되도록 보장.

---

## 7. 에러 핸들링

### 7.1 HTTP 에러 응답 표준 형식

모든 에러 응답은 아래 구조를 따른다.

```typescript
// src/types/error-response.type.ts
export interface ErrorResponse {
  status: number; // HTTP 상태 코드
  code: string; // 도메인 에러 코드 (예: 'USER_NOT_FOUND')
  message: string; // 사람이 읽을 수 있는 메시지
  correlationId?: string; // 요청 추적 ID
  details?: unknown; // 검증 오류 상세 (선택)
  timestamp: string; // ISO 8601
}
```

응답 예시:

```json
{
  "status": 404,
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다.",
  "correlationId": "a1b2c3d4-...",
  "timestamp": "2026-05-27T10:30:00.000Z"
}
```

### 7.2 전역 에러 핸들러 패턴 (Express)

```typescript
// src/middleware/error-handler.ts
import type { Request, Response, NextFunction } from "express";
import { ZodError } from "zod";
import { asyncLocalStorage } from "./correlation-id.middleware.js";
import { logger } from "../config/logger.js";
import type { ErrorResponse } from "../types/error-response.type.js";

// 도메인 에러 베이스 클래스
export class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code: string,
    message: string,
  ) {
    super(message);
    this.name = "AppError";
  }
}

// 전역 에러 핸들러 (Express 4-arg 시그니처 필수)
export function globalErrorHandler(
  err: unknown,
  req: Request,
  res: Response,
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  _next: NextFunction,
): void {
  const correlationId = asyncLocalStorage.getStore()?.correlationId;
  const timestamp = new Date().toISOString();

  // zod 검증 오류
  if (err instanceof ZodError) {
    res.status(422).json({
      status: 422,
      code: "VALIDATION_ERROR",
      message: "요청 데이터 검증 실패",
      correlationId,
      details: err.flatten().fieldErrors,
      timestamp,
    } satisfies ErrorResponse);
    return;
  }

  // 도메인 에러
  if (err instanceof AppError) {
    if (err.statusCode >= 500) {
      logger.error({ err, correlationId }, err.message);
    } else {
      logger.warn({ code: err.code, correlationId }, err.message);
    }

    res.status(err.statusCode).json({
      status: err.statusCode,
      code: err.code,
      message: err.message,
      correlationId,
      timestamp,
    } satisfies ErrorResponse);
    return;
  }

  // 예상치 못한 오류
  logger.error({ err, correlationId }, "처리되지 않은 예외");

  res.status(500).json({
    status: 500,
    code: "INTERNAL_SERVER_ERROR",
    message: "서버 내부 오류가 발생했습니다.",
    correlationId,
    timestamp,
  } satisfies ErrorResponse);
}
```

### 7.3 비동기 에러 전파

Express 4 이하에서는 비동기 핸들러 오류를 직접 `next(err)` 로 전달해야 한다.

```typescript
// src/utils/async-handler.ts
import type { Request, Response, NextFunction, RequestHandler } from "express";

// 비동기 라우트 핸들러 래퍼
export function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<void>,
): RequestHandler {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
}
```

사용 예시:

```typescript
// src/routes/users/users.router.ts
import { Router } from "express";
import { asyncHandler } from "../../utils/async-handler.js";
import * as controller from "../../controllers/users.controller.js";

export const usersRouter = Router();

usersRouter.get("/:id", asyncHandler(controller.getUser));
usersRouter.post("/", asyncHandler(controller.createUser));
```

> Express 5 (현재 RC): `async` 함수 오류 자동 전파 지원 — `asyncHandler` 래퍼 불필요.

---

## 8. 설정 파일 스니펫

### 8.1 package.json scripts

```json
{
  "scripts": {
    "dev": "tsx watch src/main.ts",
    "build": "tsc --project tsconfig.json",
    "start": "node dist/main.js",
    "start:prod": "NODE_ENV=production node dist/main.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix",
    "typecheck": "tsc --noEmit"
  }
}
```

### 8.2 NestJS main.ts 기본 패턴

```typescript
// src/main.ts (NestJS)
import "./config/env.js";
import { NestFactory } from "@nestjs/core";
import { ValidationPipe, VersioningType } from "@nestjs/common";
import { Logger } from "nestjs-pino";
import { AppModule } from "./app.module.js";
import { env } from "./config/env.js";

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  // pino 로거 교체
  app.useLogger(app.get(Logger));

  // 전역 검증 파이프 (class-validator)
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // DTO에 없는 필드 제거
      forbidNonWhitelisted: true, // 미허용 필드 400 에러
      transform: true, // 자동 타입 변환
    }),
  );

  // API 버저닝
  app.enableVersioning({ type: VersioningType.URI });

  // CORS
  app.enableCors({
    origin: env.NODE_ENV === "production" ? ["https://example.com"] : true,
    credentials: true,
  });

  await app.listen(env.PORT, env.HOST);
}

bootstrap().catch((err) => {
  console.error("서버 시작 실패:", err);
  process.exit(1);
});
```

### 8.3 Express app.ts 기본 패턴

```typescript
// src/app.ts (Express)
import express from "express";
import helmet from "helmet";
import cors from "cors";
import { httpLogger } from "./middleware/http-logger.js";
import { correlationIdMiddleware } from "./middleware/correlation-id.middleware.js";
import { globalErrorHandler } from "./middleware/error-handler.js";
import { usersRouter } from "./routes/users/users.router.js";
import { env } from "./config/env.js";

export function createApp() {
  const app = express();

  // 보안 헤더
  app.use(helmet());

  // CORS
  app.use(
    cors({
      origin: env.NODE_ENV === "production" ? ["https://example.com"] : true,
      credentials: true,
    }),
  );

  // JSON 파싱
  app.use(express.json({ limit: "1mb" }));
  app.use(express.urlencoded({ extended: true }));

  // Correlation ID (로깅보다 먼저 등록)
  app.use(correlationIdMiddleware);

  // HTTP 자동 로깅
  app.use(httpLogger);

  // 헬스체크
  app.get("/health", (_req, res) => res.json({ status: "ok" }));

  // 라우터
  app.use("/api/v1/users", usersRouter);

  // 404 핸들러
  app.use((_req, res) => {
    res
      .status(404)
      .json({
        status: 404,
        code: "NOT_FOUND",
        message: "리소스를 찾을 수 없습니다.",
      });
  });

  // 전역 에러 핸들러 (반드시 마지막 등록)
  app.use(globalErrorHandler);

  return app;
}
```

---

## 9. 체크리스트

신규 프로젝트 생성 후 아래 항목을 순서대로 확인한다.

```
[ ] 프레임워크 선택 완료 (NestJS / Express / Fastify / Hono)
[ ] tsconfig.json 작성 (strict: true, target: ES2022+)
[ ] tsx 또는 ts-node-dev 개발 서버 설정
[ ] pino + pino-http 설치 및 기본 설정
[ ] Correlation ID 미들웨어 등록 (AsyncLocalStorage)
[ ] helmet, cors, rate-limiter-flexible 설치
[ ] zod 환경변수 스키마 작성 + 시작 시 검증
[ ] .env.example 작성 및 .gitignore에 .env 추가
[ ] 전역 에러 핸들러 등록 (HTTP 표준 응답 형식 준수)
[ ] package.json scripts 확인 (dev / build / test / typecheck)
[ ] ESLint no-console 규칙 활성화
[ ] vitest + supertest 테스트 환경 구성
```
