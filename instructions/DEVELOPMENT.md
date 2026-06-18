# DEVELOPMENT.md — 개발원칙 및 개발론

> CLAUDE.md에서 참조하는 개발 원칙 문서.
> 코드 작성·리뷰·아키텍처 설계·테스트·Git 워크플로우 시 적용.

이 문서는 두 실제 프로젝트를 분석하여 맞춤 작성됐다.

| 프로젝트            | 스택                                                       | 비고                                                           |
| ------------------- | ---------------------------------------------------------- | -------------------------------------------------------------- |
| `MDWidgetServer`    | .NET 8 / C# (WPF 호스트 + HttpListener API + Dapper/MySQL) | 중앙 패키지 관리, DbUp 마이그레이션, 3계층(Api/Core/Data)      |
| `mirabell-web-mono` | pnpm + turbo 모노레포 / React 19 + TS 5.9 + Vite 7         | FSD 구조(entities/features/widgets/shared), ESLint flat config |

---

## 0. 프레임워크·런타임 버전 정책 (필독)

1. **모던 프레임워크 최신 안정화 버전 사용** — 신규 작업·의존성 추가 시 LTS 또는 최신 stable 채택. 보안 패치가 포함된 패치 버전을 우선 적용하고, 알려진 CVE가 있는 버전 고정 금지.
2. **React 프레임워크 선택 기준**
   - **SSR/SEO/메타데이터/서버 렌더링 필요** → **Next.js**(App Router) 기본
   - **SPA·사내 도구·위젯·관리자 화면 등 SSR 불필요** → **Vite** 기본 (현 `mirabell-web-mono` 전 앱이 Vite 7 사용)
   - 조건이 명확히 다르면 앱별로 다른 프레임워크 허용 (모노레포 내 혼용 가능)
3. **C# 는 .NET (Core 계열) 기반** — `net8.0`(현재) 이상. .NET Framework 4.x 레거시 신규 도입 금지. UI 호스트가 WPF여도 타깃은 `net8.0-windows`로 유지.
4. **실무 업계 표준 스펙 일괄 설치** — 신규 프로젝트/패키지는 Linter + Formatter + pre-commit hook + commit 규칙 검증을 처음부터 구성 (§5, §10 참조).
5. **alpha/beta/rc 의존성** — 불가피하게 prerelease(예: `@primereact 11.0.0-alpha`)를 쓸 때는 `package.json`/`Directory.Packages.props`에 버전 고정 + 사유 주석. stable 출시 시 즉시 교체 대상으로 추적.

---

## 1. 소프트웨어 공학 원칙

### SOLID

| 원칙    | 핵심                                 | 위반 사례                                                     |
| ------- | ------------------------------------ | ------------------------------------------------------------- |
| **SRP** | 한 클래스/모듈은 변경 이유가 하나    | `PrescriptionApiService`가 HTTP 파싱 + DB + 파일삭제까지 담당 |
| **OCP** | 확장에 열림, 수정에 닫힘             | 새 엔드포인트마다 거대한 `switch` 분기 직접 추가              |
| **LSP** | 하위 타입은 상위 타입 계약 위반 금지 | 예외를 던지도록 오버라이드한 파생 클래스                      |
| **ISP** | 비대한 인터페이스 분리               | 한 인터페이스에 미사용 메서드 다수                            |
| **DIP** | 추상에 의존, 구체 구현에 의존 금지   | 서비스가 `new MySqlConnection()` 직접 생성 → 팩토리/DI로 주입 |

> 현 `MDWidgetServer`의 `*ApiService`는 라우팅·검증·DB·파일I/O가 한 메서드에 섞여 있다. 신규 코드는 핸들러 ↔ 도메인 서비스 ↔ 데이터 접근을 분리할 것.

### DRY / KISS / YAGNI

- **DRY**: 중복 로직은 추출. 단, 우연한 중복(서로 다른 이유로 같은 코드)까지 묶지 말 것 — 과도한 추상화는 KISS 위반.
- **KISS**: `mirabell-web-mono`의 `consistent-type-assertions: never` 규칙처럼 단순·명시적 코드 우선. 임의 캐스팅보다 가드/검증 함수.
- **YAGNI**: "나중에 쓸지도" 코드 작성 금지. 요청 범위 외 기능 추가 금지.

### 관심사 분리 (SoC) & 함수 크기

- 레이어 분리: `Api`(전송/라우팅) ↔ `Core`(도메인 모델·설정) ↔ `Data`(영속성). 프론트는 FSD 레이어로 분리(§2).
- **함수는 20–30줄 권장**. 한 화면에 안 들어오면 분해 신호. 한 함수 = 한 추상화 레벨.

---

## 2. 아키텍처 원칙

### 클린 아키텍처 — 의존성 방향

- 의존성은 **항상 바깥(인프라) → 안(도메인)** 한 방향. 도메인(`Core`)은 `Api`/`Data`를 알지 못함.
- 현 .NET 솔루션 의존: `Api → Core, Data` / `Data → Core` / `WPF Host → Core, Api`. `Core`는 어디에도 의존하지 않음 (도메인 코어 유지).

### 프론트엔드 — FSD (Feature-Sliced Design)

`mirabell-web-mono`는 FSD를 채택했다. 레이어 의존 방향을 **상위→하위 단방향**으로 강제한다 (`dependency-cruiser`로 검증).

```
app  →  widgets  →  features  →  entities  →  shared
(상위 레이어가 하위만 import. 역방향·동일레이어 횡단 import 금지)
```

| 레이어     | 패키지                                    | 역할                             |
| ---------- | ----------------------------------------- | -------------------------------- |
| `shared`   | `packages/core/shared`                    | 범용 유틸·UI 프리미티브·설정     |
| `entities` | `packages/core/entities`                  | 도메인 모델·기본 비즈니스 엔티티 |
| `features` | `packages/core/features`                  | 사용자 시나리오 단위 기능        |
| `widgets`  | `packages/core/widgets`                   | 조합 UI 블록                     |
| `app`      | `apps/*` (MIMS, web-sender, dynamic-link) | 진입점·라우팅·조립               |

- **상대경로 import 금지** (`no-relative-import-paths`): 슬라이스 경계는 `@mirabell-web-mono/*` 별칭으로 import. 같은 폴더만 상대경로 허용.
- 신규 코드는 올바른 레이어에 배치. `dep:check`(`depcruise`)로 위반 차단.

### 모듈 응집도·결합도

- 높은 응집(관련 기능 한곳) + 낮은 결합(모듈 간 최소 의존). 슬라이스 간 통신은 공개 API(`index.ts` barrel) 경유.
- 도메인 모델링: 빈약한 모델(데이터 백)보다 행위를 가진 모델 지향. DTO와 도메인 모델은 구분.

---

## 3. 테스트 방법론

> 현 두 저장소 모두 테스트 인프라가 미비하다. 신규 기능은 아래 기준으로 테스트를 함께 작성한다.

### TDD 사이클

**Red**(실패 테스트) → **Green**(최소 통과 구현) → **Refactor**(중복 제거·구조 개선). 핵심 비즈니스 로직·버그 수정은 TDD 우선.

### 테스트 피라미드

| 구분 | 비중 | 대상                                     |
| ---- | ---- | ---------------------------------------- |
| 단위 | 70%  | 도메인 로직·순수 함수 (DB/네트워크 모킹) |
| 통합 | 20%  | 레이어 결합·DB 쿼리·API 핸들러           |
| E2E  | 10%  | 핵심 사용자 흐름                         |

### 프레임워크 (스택 권장)

- **TS/React**: **Vitest** + **React Testing Library** (Vite 환경과 호환). E2E는 Cypress (현 `eslint-plugin-cypress` 설치됨).
- **.NET**: **xUnit** + **FluentAssertions**, 모킹은 **NSubstitute** 또는 **Moq**. DB 통합은 Testcontainers(MySQL) 권장.

### 규칙

- **명명**: `메서드명_시나리오_기대결과` (예: `LoginAsync_InvalidPassword_ReturnsFail`). AAA(Arrange-Act-Assert) 구조.
- **모킹**: 외부 경계(DB·HTTP·파일시스템)만 모킹. 도메인 로직은 실제 구현으로 테스트.
- **커버리지**: 전체 최소 80%, 핵심 비즈니스 로직·결제/인증 경로 100%. 커버리지 수치 자체가 목표가 되지 않게 — 의미 있는 검증 우선.

---

## 4. 코드 품질

### 코드 리뷰 체크리스트

- **기능**: 요구사항 충족? 엣지케이스(빈 입력·null·동시성)?
- **보안**: 입력 검증, 파라미터화 쿼리, 시크릿 노출 없음 (§6)
- **성능**: N+1, 불필요 반복/할당, 큰 페이로드 (§7)
- **가독성**: 의미 있는 이름, 적정 함수 크기, 죽은 코드 없음
- **테스트**: 신규 로직에 테스트 동반? 실패 케이스 포함?

### 네이밍

- **의미 있는 이름**, 약어 금지(`usr`→`user`, `cnt`→`count`). 검색 가능한 이름.
- TS: 변수/함수 `camelCase`, 타입/컴포넌트 `PascalCase`, 상수 `UPPER_SNAKE`. **인터페이스에 `I` 접두사 금지**(ESLint `naming-convention`으로 강제).
- C#: 타입/메서드/프로퍼티 `PascalCase`, 지역변수/매개변수 `camelCase`, private 필드 `_camelCase`(현 코드 컨벤션).

### 리팩터링 & 기술 부채

- **보이스카우트 규칙**: 만진 코드는 조금 더 깨끗하게. 단 요청 범위 외 대규모 리팩터링은 별도 작업으로 분리(회귀 위험 차단).
- 임시방편 발견 시 근본 원인 해결로 재구현. `// TODO`/`// HACK`은 추적 가능하게 사유·티켓 명시.

---

## 5. 도구·린트·포맷·훅 (실무 표준 일괄 적용)

### JavaScript / TypeScript (필수)

**TypeScript + ESLint + Prettier 필수.** (팀 합의 시 ESLint+Prettier를 **Biome**로 대체 가능 — 단일 도구로 lint+format 통합)

현 `mirabell-web-mono` 구성 (그대로 따른다):

- **ESLint flat config** (`eslint.config.js`, ESLint 9). 공유 설정 `@mirabell-web-mono/eslint-config` 패키지로 중앙화.
  - `@typescript-eslint/no-explicit-any`: **error** (any 금지)
  - `@typescript-eslint/no-unused-vars`: `_` 접두사 예외
  - `naming-convention`: 인터페이스 `I` 접두사 금지, 타입 PascalCase
  - `no-console`: warn (`warn`/`error`만 허용)
  - `consistent-type-assertions`: 객체 리터럴 캐스팅(`as`) 금지 → 가드 함수 유도
  - `no-relative-import-paths`: 슬라이스 경계 상대경로 금지 (FSD 강제)
- **Prettier** 공유 설정 `@mirabell-web-mono/prettier-config`. `format` 스크립트로 일괄 적용.
- **TypeScript strict** 권장. `tsc --noEmit`(`type` 스크립트)로 타입 게이트.

### C# / .NET (실무 표준)

- **`Directory.Build.props`로 공통 설정 중앙화** (현 적용): `Nullable=enable`(NRT 활성), `ImplicitUsings=enable`, `LangVersion=latest`.
- **중앙 패키지 버전 관리** `Directory.Packages.props` (`ManagePackageVersionsCentrally=true`) — 모든 프로젝트 버전 단일 소스.
- **포맷**: `dotnet format` + `.editorconfig`로 스타일 강제. 분석기는 `<AnalysisLevel>latest</AnalysisLevel>` + `<TreatWarningsAsErrors>` 권장(신규).
- **Roslyn 분석기**: `Microsoft.CodeAnalysis.NetAnalyzers` 기본 + 보안 분석기 추가 권장.

### pre-commit / commit 게이트

현 모노레포 구성 (신규 프로젝트도 동일 적용):

- **husky** (`prepare: husky`) + **lint-staged**: 스테이징 파일에 ESLint+Prettier 자동 실행.
- **commitlint** (`@commitlint/config-conventional`): commit 메시지 컨벤션 강제 (§8, `COMMIT_CONVENTION.md` 참조).
- .NET 저장소도 pre-commit 훅으로 `dotnet format --verify-no-changes` + 빌드 검증 적용 권장.

---

## 6. 보안 원칙

### OWASP Top 10 핵심 방어

- **인젝션(SQLi)**: **항상 파라미터화 쿼리**. 현 `ConnGrpDatabaseService`는 Dapper 익명 객체 바인딩(`@ConnGrpNm`) 사용 — 올바름. 문자열 연결로 SQL 조립 금지. ID 목록 삭제 시 `long.TryParse` 후 Dapper list expansion 사용(현 `deletePrescriptionHist` 패턴).
- **입력 검증**: **경계에서 검증, 내부는 신뢰**. TS는 **Zod** 스키마로 외부 입력(API 응답·폼) 파싱(현 `zod` 4.x 사용). C#는 DTO 역직렬화 후 null/범위 검증.
- **인증/암호화**: `LoginService`의 XOR+Base64는 레거시 호환용 — 신규 인증에 자체 암호화 금지. 표준 라이브러리(BCrypt/Argon2 해시, TLS) 사용.
- **XSS/CSRF**: React는 기본 escape. `dangerouslySetInnerHTML` 금지. 외부 HTML은 sanitize.

### 시크릿 관리

- **자격증명 하드코딩 금지**. DB 접속정보·키는 환경변수/시크릿 매니저. `.env`·`appsettings.*.json`(시크릿)은 `.gitignore`.
- 현 코드의 connection string 조립은 설정값 주입 경로 유지 — 소스에 평문 비밀번호 박지 말 것.

### 최소 권한 & 전송

- DB 계정·서비스 권한 최소화. 외부 통신은 HTTPS/TLS. 파일 다운로드 시 경로 traversal 방지(`Path.GetFileName` 검증).

---

## 7. 성능 원칙

- **측정 후 최적화** — 조기 최적화 금지. 프로파일러/벤치마크로 병목 확인 후 개선.
- **N+1 쿼리 방지**: 루프 안 쿼리 금지. JOIN·배치 조회·`IN (...)` 일괄 조회. 현 cross-DB(pacsdb) 매핑은 특히 주의.
- **알고리즘 복잡도 인식**: 대용량 컬렉션에 중첩 루프(O(n²)) 회피. 적절한 자료구조(Dictionary/Set) 선택.
- **비동기 I/O**: .NET은 `async/await` 일관 적용(현 코드 준수), `.Result`/`.Wait()` 블로킹 금지. React는 TanStack Query로 서버 상태 캐싱·중복요청 제거.
- **캐싱**: **무효화 전략까지 함께 설계**. 캐시 키·TTL·무효화 트리거 명시. 무효화 없는 캐시 금지.
- **프론트 번들**: 코드 스플리팅·lazy import. Vite `rollup-plugin-visualizer`로 번들 분석(현 설치됨).

---

## 8. Git 워크플로우

> 상세 규칙은 `COMMIT_CONVENTION.md` 참조. commitlint(husky pre-commit)로 자동 검증됨.

### Conventional Commits

`<type>(<scope>): <subject>` 형식. 현 `commitlint.config.js` 허용 type:

```
feat fix refactor style design perf docs comment chore build init revert etc HOTFIX
```

- 표준 명세: `feat`(기능), `fix`(버그), `docs`, `style`(포맷), `refactor`, `perf`, `test`, `chore`, `ci`/`build`.
- **Breaking change**: `feat!:` 또는 footer `BREAKING CHANGE:` 명시.
- `header-max-length: 100`, body 앞 빈 줄 필수.

### 브랜치 전략 & 커밋 단위

- `main`(배포 가능) / `feat/*` / `fix/*` / `hotfix/*`.
- **커밋은 논리적 단위 하나** — 한 커밋 = 한 의도. 무관한 변경 섞지 말 것.
- **PR 최대 ~400줄** 권장. 제출 전 **자기 리뷰** 먼저. PR 본문에 변경 의도·테스트 방법 기재.

---

## 9. 데이터베이스 원칙

### 마이그레이션 패턴 (감지됨: DbUp)

`MDWidgetServer.Data`는 **DbUp** (`dbup-mysql`)을 사용한다. **신규 테이블·스키마 변경 시 반드시 이 패턴을 따른다.**

- **마이그레이션 파일 위치**: `MDWidgetServer.Data/Migrations/*.sql`, `EmbeddedResource`로 어셈블리에 포함.
- **파일명 규칙**: `NNNN_설명.sql` (예: `0001_create_conn_grp_tables.sql`). 버전은 파일명 숫자에서 자동 추출(`VersionedMySqlJournal`).
- **이력 추적**: `schemaversions` 테이블에 적용 이력 기록. 트랜잭션(`WithTransaction`) 단위 적용.
- 파일 상단에 **변경 이력 주석**(버전·날짜·설명) 기재 — 현 `0001` 파일 컨벤션 준수.
- TS/Node 측에서 DB 작업이 추가될 경우 별도 마이그레이션 도구(Prisma Migrate / Drizzle / node-pg-migrate 등) 도입을 먼저 확인 후 결정.

### 운영·스키마 규칙

- **운영 DB에 직접 DDL 금지** — 반드시 마이그레이션 파이프라인 경유.
- **롤백 시나리오 고려**: DbUp은 forward-only이므로, 파괴적 변경 전 백업 + 보상 마이그레이션 스크립트 준비.
- **하위 호환 유지** — 컬럼 삭제는 **2단계**: ① 앱 코드에서 사용 제거 배포 → ② 후속 마이그레이션에서 스키마 제거.
- **CASCADE·제약**: 현 스키마는 `ON DELETE CASCADE` + `UNIQUE KEY`로 정합성 보장. cross-DB(pacsdb) 참조는 FK 불가 → 앱 레벨 정합성 검증.

---

## 10. 서버 구조화 로깅 (Structured Logging) — 필수

> **서버 개발 시 구조화 로깅을 필수 적용한다.** 현 `MDWidgetServer`는 `Console.WriteLine`/`Console.Error` 기반(87곳) — **신규/리팩터링 코드는 아래 구조화 로깅으로 전환**한다.

### 원칙

- 평문 문자열 보간 대신 **구조화된 속성**(key-value)으로 기록 → 쿼리·필터·집계 가능.
- 로그 레벨 구분: `Trace`/`Debug`(개발) · `Information`(흐름) · `Warning`(이상 징후) · `Error`(실패) · `Fatal`(중단).
- **민감정보(비밀번호·토큰·개인정보) 로깅 금지**. 요청 단위 상관관계 ID(correlation/request id) 부여.

### C# — Serilog 우선 (log4net 지양)

**Serilog 채택.** log4net은 신규 도입 금지. sink + enricher 구성 예시:

```csharp
// 부트스트랩 (WPF App.xaml.cs OnStartup 또는 호스트 빌더)
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()                    // 스코프 속성 전파
    .Enrich.WithMachineName()                   // Serilog.Enrichers.Environment
    .Enrich.WithThreadId()                      // Serilog.Enrichers.Thread
    .Enrich.WithProperty("App", "MDWidgetServer")
    .WriteTo.Console(                            // 콘솔 sink (개발)
        outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(                              // 롤링 파일 sink (운영)
        path: "logs/mdwidget-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,
        formatter: new Serilog.Formatting.Compact.CompactJsonFormatter()) // 구조화 JSON
    .CreateLogger();
```

```csharp
// 구조화 호출 — 문자열 보간(❌)이 아닌 메시지 템플릿(✅)
// ❌ Console.WriteLine($"[Prescription] 삭제 실패 {id}: {ex.Message}");
// ✅
_logger.Error(ex, "처방전 삭제 실패 {PrescriptionId} (요청 {RequestId})", id, requestId);
_logger.Information("처방전 목록 조회 {Count}건 (ifKey={IfKey})", rows.Count, ifKey);

// 요청 스코프 상관관계
using (LogContext.PushProperty("RequestId", Guid.NewGuid()))
{
    // 이 블록 내 모든 로그에 RequestId 자동 첨부
}
```

- 필요 패키지: `Serilog`, `Serilog.Sinks.Console`, `Serilog.Sinks.File`, `Serilog.Formatting.Compact`, `Serilog.Enrichers.Environment`, `Serilog.Enrichers.Thread`. (ASP.NET Core 도입 시 `Serilog.AspNetCore` + `Microsoft.Extensions.Logging` 추상화로 DI).
- DI 환경에서는 `ILogger<T>` 주입으로 사용하고, 부트스트랩만 Serilog로 구성.

### TypeScript/Node 서버

- 구조화 JSON 로거 사용: **pino**(권장, 고성능) 또는 **winston**. 브라우저 코드는 `no-console` 규칙(warn/error만) 준수.

```ts
import pino from "pino";
const logger = pino({ level: "info" });
logger.info({ requestId, userId }, "주문 생성 완료");
logger.error({ err, orderId }, "주문 처리 실패");
```

---

## 11. 접근성 (프론트엔드)

- **WCAG 2.1/2.2 AA** 기준 준수. 현 모노레포는 `eslint-plugin-jsx-a11y` 적용(`alt-text` 등).
- **시맨틱 HTML**: `div` 남용 대신 `button`/`nav`/`main`/`header` 등 의미 태그.
- **키보드 네비게이션**: 모든 인터랙티브 요소 Tab 접근·포커스 표시. `onClick`만 있는 `div` 금지.
- **스크린 리더**: 이미지 `alt`, 폼 `label` 연결, 동적 영역 `aria-live`, 적절한 `aria-*` 속성.
- 색상 대비 4.5:1(본문) 이상. 색상만으로 정보 전달 금지.

---

## 부록 — 프로젝트별 빠른 참조

| 항목          | `mirabell-web-mono`                        | `MDWidgetServer`                           |
| ------------- | ------------------------------------------ | ------------------------------------------ |
| 패키지 매니저 | pnpm 10 (`only-allow pnpm` 강제)           | NuGet (중앙 버전 관리)                     |
| 빌드/태스크   | turbo                                      | dotnet / MSBuild                           |
| 언어 버전     | TypeScript 5.9, React 19                   | C# latest, .NET 8                          |
| Lint/Format   | ESLint 9 flat + Prettier (공유 패키지)     | dotnet format + .editorconfig (권장)       |
| 테스트        | Vitest + RTL (권장 도입)                   | xUnit + FluentAssertions (권장 도입)       |
| 로깅          | pino/winston (서버), no-console(브라우저)  | **Serilog** (Console→Serilog 전환 대상)    |
| DB            | —                                          | Dapper + MySqlConnector, DbUp 마이그레이션 |
| 아키텍처      | FSD (shared→entities→features→widgets→app) | 3계층 (Api→Core←Data)                      |
| 커밋 검증     | husky + lint-staged + commitlint           | pre-commit 훅 (권장 도입)                  |
