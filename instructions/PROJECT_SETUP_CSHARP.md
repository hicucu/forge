# C# / .NET Core 프로젝트 셋업 지침 (PROJECT_SETUP_CSHARP)

> 신규 C# / .NET Core 백엔드 프로젝트를 처음 생성할 때 참조하는 **언어별 부트스트랩 지침**.
> 솔루션 구조·필수 패키지·Serilog 로깅·설정 파일·`Program.cs` 패턴·코딩 스타일까지를 다룬다.
> 공통 부트스트랩 원칙은 `PROJECT_SETUP_GUIDE.md`, 코딩 규칙은 `LANGUAGE_GUIDELINES_CSHARP.md`, AI 동작은 `AI_BEHAVIOR.md`, 커밋은 `COMMIT_CONVENTION.md` 참조.

**기준 버전**: C# 12 / **.NET 8 LTS** (`net8.0`, 데스크톱 호스트는 `net8.0-windows`)
**참조 솔루션**: `MDWidgetServer` (Api / Core / Data 3계층 + WPF 호스트, Dapper + MySQL + DbUp)

> 신규 서비스는 .NET 8 LTS 기준. 차기 LTS(.NET 10) 출시 시 PR로 검증 후 이전.

---

## 0. 셋업 진행 순서 (전체 개요)

```
1. 프로젝트 유형 선택 (Web API / gRPC / Worker / WPF·MAUI)
   ↓
2. 솔루션·프로젝트 생성 (.sln + Api/Core/Data + Tests)
   ↓
3. Directory.Build.props / Directory.Packages.props 구성 (공통 속성 + CPM)
   ↓
4. 필수 NuGet 패키지 설치 (Serilog · Swashbuckle · FluentValidation · ORM · 테스트)
   ↓
5. Serilog 로깅 구성 (appsettings 기반 부트스트랩 + 요청 로그)
   ↓
6. appsettings.json / 환경별 파일 구성
   ↓
7. Program.cs 부트스트랩 (Serilog · DI · Swagger · 환경 분기)
   ↓
8. 빌드·실행으로 동작 증명 (Swagger UI 응답 확인)
```

> **원칙**: 프로젝트 유형·ORM·마이그레이션 도구가 모호하거나 선택지가 둘 이상이면 추론 전에 사용자에게 먼저 질문 (`AI_BEHAVIOR.md` 질문 게이트).

---

## 1. 프로젝트 유형 선택

먼저 무엇을 만드는지에 따라 SDK 템플릿과 호스트 모델을 확정한다.

| 유형                      | 템플릿 명령               | 호스트 모델                      | 용도                                         |
| ------------------------- | ------------------------- | -------------------------------- | -------------------------------------------- |
| **ASP.NET Core Web API**  | `dotnet new webapi`       | `WebApplication` (Kestrel)       | REST HTTP API — 기본 선택지                  |
| **gRPC 서비스**           | `dotnet new grpc`         | `WebApplication` (HTTP/2)        | 내부 서비스 간 고성능 RPC, 강타입 계약       |
| **Worker Service**        | `dotnet new worker`       | `Host` + `BackgroundService`     | 큐 소비·스케줄러·메시지 브로커 등 백그라운드 |
| **WPF / MAUI** (데스크톱) | `dotnet new wpf` / `maui` | `Application` (+ `Generic Host`) | 데스크톱 UI. 서버 로직은 별도 계층으로 분리  |

### 유형 판별 흐름

```
HTTP로 외부에 노출되는 API인가?
├─ Yes ─ 클라이언트가 브라우저/공개 API? ── Yes → ASP.NET Core Web API
│                                        └─ No, 내부 서비스 간 강타입 RPC → gRPC 서비스
└─ No ── 지속 실행 백그라운드 작업? ──────── Yes → Worker Service
         데스크톱 UI 필요? ───────────────── Yes → WPF / MAUI (+ 서버 로직은 Core/Data 분리)
```

> 참조 프로젝트 `MDWidgetServer`는 **WPF 호스트가 `Microsoft.Extensions.Hosting`으로 gRPC + MQTT 서버를 인프로세스 구동**하는 혼합형이다. 순수 서버라면 Web API 또는 Worker Service를 권장하고, WPF는 데스크톱 UI가 필수일 때만 선택.

### 단일 호스트 결정 기준

- 한 프로세스에서 HTTP API + 백그라운드 작업을 함께 운영 → Web API에 `AddHostedService<T>()`로 `BackgroundService` 등록 (별도 Worker 분리 불필요)
- API와 워커의 배포·스케일 단위가 다르면 프로젝트 분리

---

## 2. 솔루션 구조 (3계층 아키텍처)

`MDWidgetServer` 패턴을 따라 **Api / Core / Data** 3계층으로 분리한다.

### 레이어 의존 방향

```
        ┌─────────────┐
        │     Api     │  진입점·DI 조립·엔드포인트·미들웨어
        └──────┬──────┘
               │ 참조
               ▼
        ┌─────────────┐        ┌─────────────┐
        │    Core     │◄───────│    Data     │  DB 접근·마이그레이션
        │ 도메인·계약 │  참조   └─────────────┘
        └─────────────┘
```

- **Api → Core**, **Api → Data**, **Data → Core** (단방향)
- **Core는 어디에도 의존하지 않음** — 도메인 모델·인터페이스·설정 계약만 보유 (의존성 역전의 중심)
- **Data → Api 역참조 금지**, **Core → Data 역참조 금지** (순환 차단)

> 참조 프로젝트 실제 의존:
>
> - `Api.csproj` → `Core`, `Data`
> - `Data.csproj` → `Core`
> - `Core.csproj` → (참조 없음)

### 계층별 책임

| 계층      | 프로젝트           | 책임                                                                                |
| --------- | ------------------ | ----------------------------------------------------------------------------------- |
| **Api**   | `{Solution}.Api`   | 진입점, DI 컨테이너 조립, 컨트롤러/엔드포인트/gRPC 서비스, 미들웨어, 인증, Swagger  |
| **Core**  | `{Solution}.Core`  | 도메인 모델, DTO/`record`, 서비스 인터페이스, 비즈니스 규칙, 설정 모델, 도메인 예외 |
| **Data**  | `{Solution}.Data`  | DB 접근(Dapper/EF Core), 마이그레이션(DbUp/EF), 리포지토리 구현, 연결 문자열 조립   |
| **Tests** | `{Solution}.Tests` | 단위 테스트 (xUnit), Core 비즈니스 로직 + Data 리포지토리(통합 시) 검증             |

> Core에 인터페이스(`IConnGrpRepository`)를 두고 Data에서 구현하면 Api가 Core 인터페이스에만 의존 — 테스트 시 Moq로 교체 용이. 참조 프로젝트는 구체 클래스 직접 주입 방식이나, 신규 프로젝트는 **인터페이스 기반 DIP를 권장**.

---

## 3. 디렉토리 구조 템플릿

```
{Solution}/
├── {Solution}.sln
├── Directory.Build.props          # 공통 속성 (Nullable, ImplicitUsings, LangVersion, TWAE)
├── Directory.Packages.props       # 중앙 패키지 버전 관리 (CPM)
├── NuGet.Config                   # 패키지 소스 매핑 (선택)
├── .gitignore                     # dotnet new gitignore
├── src/
│   ├── {Solution}.Api/
│   │   ├── {Solution}.Api.csproj
│   │   ├── Program.cs             # 부트스트랩 (Serilog · DI · Swagger)
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json
│   │   ├── appsettings.Production.json
│   │   ├── Endpoints/             # 또는 Controllers/
│   │   ├── Middleware/            # 예외 핸들러 등
│   │   └── Protos/                # gRPC 선택 시 .proto
│   ├── {Solution}.Core/
│   │   ├── {Solution}.Core.csproj
│   │   ├── Models/                # 도메인 모델·DTO(record)
│   │   ├── Config/                # 설정 모델 (AppSettings 등)
│   │   ├── Abstractions/          # 서비스/리포지토리 인터페이스
│   │   └── Services/              # 도메인 서비스
│   └── {Solution}.Data/
│       ├── {Solution}.Data.csproj
│       ├── Migrations/            # *.sql (DbUp) — EmbeddedResource
│       └── Repositories/          # Dapper/EF 구현
└── tests/
    └── {Solution}.Tests/
        └── {Solution}.Tests.csproj
```

> 참조 프로젝트는 `src/` 없이 솔루션 루트에 프로젝트를 평면 배치하나, 신규 프로젝트는 **`src/` · `tests/` 분리를 권장** (빌드 산출물·테스트 격리 명확화).

---

## 4. Directory.Build.props 템플릿

솔루션 루트에 배치하면 하위 전 프로젝트에 자동 적용된다. 참조 프로젝트 기반 + 신규 권장값 추가:

```xml
<Project>
  <PropertyGroup>
    <!-- 참조 프로젝트(MDWidgetServer)의 실제 값 -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>

    <!-- 신규 프로젝트 권장 추가값 -->
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn> <!-- XML 주석 누락 경고만 완화 -->
  </PropertyGroup>
</Project>
```

| 속성                    | 값       | 의미                                                        |
| ----------------------- | -------- | ----------------------------------------------------------- |
| `Nullable`              | `enable` | nullable 참조 타입 활성화 — NRE 컴파일 시점 차단 (**필수**) |
| `ImplicitUsings`        | `enable` | 공통 using 자동 추가 — 보일러플레이트 감소                  |
| `LangVersion`           | `latest` | 최신 C# 언어 기능 사용                                      |
| `TreatWarningsAsErrors` | `true`   | 경고를 오류로 승격 — 기술 부채 누적 방지 (신규 권장)        |
| `EnableNETAnalyzers`    | `true`   | .NET 정적 분석기 활성화                                     |

> 참조 프로젝트는 최소 3개 속성만 사용. `TreatWarningsAsErrors`는 레거시 코드 다량 시 빌드가 막히므로, **신규 프로젝트에서 처음부터** 켜는 것이 비용 효율적.

### Directory.Packages.props 템플릿 (중앙 패키지 관리)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- 로깅 -->
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.3" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="6.0.0" />
    <PackageVersion Include="Serilog.Sinks.File" Version="6.0.0" />
    <PackageVersion Include="Serilog.Enrichers.Environment" Version="3.0.1" />
    <PackageVersion Include="Serilog.Enrichers.Thread" Version="4.0.0" />
    <!-- API 문서 -->
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.6.2" />
    <!-- 검증 -->
    <PackageVersion Include="FluentValidation.AspNetCore" Version="11.3.0" />
    <!-- DB (Dapper 선택 시) -->
    <PackageVersion Include="Dapper" Version="2.1.28" />
    <PackageVersion Include="dbup-mysql" Version="5.0.8" />
    <PackageVersion Include="MySqlConnector" Version="2.3.5" />
    <!-- 테스트 -->
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Moq" Version="4.20.70" />
    <PackageVersion Include="FluentAssertions" Version="6.12.0" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />
  </ItemGroup>
</Project>
```

- 개별 `.csproj`의 `PackageReference`에는 **버전 미기재** (`<PackageReference Include="Dapper" />`)
- 버전은 이 파일 한 곳에서만 관리 — 버전 드리프트 차단
- 위 버전은 .NET 8 기준 예시. 설치 시점에 `dotnet add` 또는 NuGet 최신 안정 버전 확인 후 반영

> 참조 프로젝트는 `Dapper 2.1.28`, `dbup-mysql 5.0.8`, `MySqlConnector 2.3.5`를 실제 사용 중. DB 외 패키지(Serilog/Swashbuckle 등)는 참조 프로젝트에 없으나 신규 표준으로 추가.

---

## 5. 필수 NuGet 패키지

### 5.1 로깅 — Serilog (필수)

| 패키지                          | 역할                                           |
| ------------------------------- | ---------------------------------------------- |
| `Serilog.AspNetCore`            | ASP.NET Core 통합 + `UseSerilogRequestLogging` |
| `Serilog.Sinks.Console`         | 콘솔 출력 sink                                 |
| `Serilog.Sinks.File`            | 파일 롤링 sink                                 |
| `Serilog.Enrichers.Environment` | 머신명·환경명 enrich                           |
| `Serilog.Enrichers.Thread`      | 스레드 ID enrich                               |

> **`log4net`, `NLog` 신규 프로젝트 사용 금지.** 구조화 로깅 표준은 **Serilog**로 통일한다. `Console.WriteLine` 직접 호출도 금지 (참조 프로젝트의 `Console.WriteLine` 패턴은 레거시 — 신규에서는 `ILogger<T>` 사용).

### 5.2 API 문서 — Swashbuckle (Web API/gRPC-Gateway 시)

- `Swashbuckle.AspNetCore` — OpenAPI/Swagger 문서 + Swagger UI 자동 생성

### 5.3 검증 — FluentValidation

- `FluentValidation.AspNetCore` — 선언적 입력 검증. DTO별 `AbstractValidator<T>` 작성, 자동 모델 검증 파이프라인 통합

### 5.4 ORM / DB — Dapper vs EF Core

| 기준                   | Dapper (경량)                 | EF Core (풀 ORM)                     |
| ---------------------- | ----------------------------- | ------------------------------------ |
| SQL 제어               | 직접 작성 (완전 제어)         | LINQ → SQL 자동 변환                 |
| 학습 곡선              | 낮음 (SQL만 알면 됨)          | 중간 (변경 추적·관계 매핑 개념 필요) |
| 성능                   | 매우 빠름 (마이크로 ORM)      | 빠름 (추적·캐싱 오버헤드 존재)       |
| 복잡 조인·레거시 DB    | **유리** (수기 SQL로 최적화)  | 불리 (복잡 매핑 시 까다로움)         |
| 마이그레이션·변경 추적 | 별도 도구 필요 (DbUp)         | 내장 (`dotnet ef migrations`)        |
| 적합 상황              | 레거시 DB·복잡 조인·성능 민감 | 신규 스키마·CRUD 중심·생산성 우선    |

> 참조 프로젝트는 **Dapper + MySqlConnector**로 레거시 다중 DB(prmdb/pacsdb) 조인을 수기 SQL로 처리. 레거시 DB 연동이면 Dapper, 신규 스키마 설계 자유도가 있으면 EF Core. 모호하면 사용자에게 질문.

### 5.5 마이그레이션 — DbUp vs EF Core Migrations

| 기준      | DbUp (SQL 스크립트)            | EF Core Migrations           |
| --------- | ------------------------------ | ---------------------------- |
| 작성 방식 | 순수 SQL 파일 (`0001_xxx.sql`) | C# 모델 변경 → 자동 생성     |
| DBA 협업  | **용이** (SQL 리뷰 가능)       | 불리 (생성된 코드 리뷰 필요) |
| ORM 종속  | 없음 (Dapper와 짝)             | EF Core 필수                 |
| 적용      | 임베디드 리소스 + 부팅 시 실행 | `dotnet ef database update`  |

> 참조 프로젝트는 **DbUp + 임베디드 SQL** 사용. `Data.csproj`에 `<EmbeddedResource Include="Migrations\*.sql" />`로 SQL을 어셈블리에 포함하고, 커스텀 `VersionedMySqlJournal`로 `schemaversions` 테이블에 버전 기록. Dapper면 DbUp, EF Core면 EF Migrations로 짝을 맞춘다.

### 5.6 CQRS — MediatR (복잡 도메인 시)

- 도메인이 복잡하고 Command/Query 분리·파이프라인(검증·로깅·트랜잭션) 횡단 관심사가 많을 때만 도입
- 단순 CRUD에는 과설계 — 직접 서비스 호출이 더 명확. **기본은 미도입**, 복잡도 임계 초과 시 사용자 합의 후 추가

### 5.7 테스트

- `xUnit` + `xunit.runner.visualstudio` + `Microsoft.NET.Test.Sdk` — 테스트 프레임워크
- `Moq` — 의존성 모킹 (인터페이스 기반 DIP 전제)
- `FluentAssertions` — 가독성 높은 단언 (`result.Should().Be(...)`)

---

## 6. Serilog 설정 (상세)

### 6.1 Program.cs 부트스트랩 패턴

부트스트랩 로거를 먼저 세워 **앱 시작 실패까지 로깅**하고, 이후 `appsettings.json` 기반 정식 로거로 교체한다.

```csharp
using Serilog;

// 1) 부트스트랩 로거 — 호스트 빌드 전 예외도 캡처
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting up");

    var builder = WebApplication.CreateBuilder(args);

    // 2) appsettings.json 의 Serilog 섹션 기반 정식 로거로 교체
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());

    var app = builder.Build();

    // 3) 요청 로그 미들웨어 (가장 먼저 등록)
    app.UseSerilogRequestLogging();

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### 6.2 appsettings.json Serilog 섹션 (전체 예시)

```jsonc
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning",
      },
    },
    "Enrich": [
      "FromLogContext",
      "WithMachineName",
      "WithThreadId",
      "WithEnvironmentName",
    ],
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}",
        },
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 31,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] [{ThreadId}] {SourceContext} {Message:lj}{NewLine}{Exception}",
        },
      },
    ],
  },
}
```

### 6.3 Enricher 설정

| Enricher              | 패키지                          | 추가 속성         |
| --------------------- | ------------------------------- | ----------------- |
| `FromLogContext`      | (기본 내장)                     | LogContext 속성   |
| `WithMachineName`     | `Serilog.Enrichers.Environment` | `MachineName`     |
| `WithEnvironmentName` | `Serilog.Enrichers.Environment` | `EnvironmentName` |
| `WithThreadId`        | `Serilog.Enrichers.Thread`      | `ThreadId`        |

**CorrelationId**: 요청 단위 추적용. 미들웨어에서 `X-Correlation-ID` 헤더(없으면 생성)를 `LogContext`에 push:

```csharp
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
                        ?? Guid.NewGuid().ToString();
    using (Serilog.Context.LogContext.PushProperty("CorrelationId", correlationId))
    {
        context.Response.Headers["X-Correlation-ID"] = correlationId;
        await next();
    }
});
```

### 6.4 ILogger<T> 주입 패턴

- Serilog를 호스트에 연결하면 표준 `ILogger<T>`가 자동으로 Serilog로 라우팅됨
- 클래스는 **`Microsoft.Extensions.Logging.ILogger<T>`를 생성자 주입** (Serilog API 직접 의존 금지 — 테스트·교체 용이)

```csharp
public sealed class OrderService(ILogger<OrderService> logger)
{
    public async Task<long> CreateAsync(CreateOrderRequest req)
    {
        var orderId = await _repository.InsertAsync(req);
        logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            orderId, req.CustomerId);
        return orderId;
    }
}
```

### 6.5 구조화 로그 — Good / Bad

```csharp
// Bad — 직접 콘솔 출력 (구조 없음, 레벨 없음, sink 우회)
Console.WriteLine("저장 완료: " + id);

// Bad — 문자열 보간/연결 (메시지 템플릿 손실, 검색·필터 불가)
_logger.LogInformation("저장 완료: " + id);
_logger.LogInformation($"Order {orderId} saved");   // 보간도 동일 — 템플릿 보존 안 됨

// Good — 메시지 템플릿 + 구조화 속성
_logger.LogInformation("Order {OrderId} saved", orderId);
_logger.LogWarning("Retry {Attempt}/{Max} for {Operation}", attempt, max, "PaymentCapture");
```

> 핵심: **중괄호 플레이스홀더는 보간 문법이 아니라 Serilog 메시지 템플릿**이다. `{OrderId}`가 구조화 속성으로 저장되어 로그 시스템에서 키 기반 검색·집계가 가능. `$"..."` 보간은 이를 깨뜨리므로 금지.

### 6.6 민감 정보 제외

- **비밀번호·토큰·연결 문자열·개인정보를 로그에 출력 금지**
- DTO를 통째로 로깅하지 말고 안전한 필드만 명시적으로 기록
- 마스킹 헬퍼 또는 Serilog `Destructure` 정책으로 민감 속성 제거

```csharp
// Bad — 요청 객체 전체 로깅 (Password·Token 노출 위험)
_logger.LogInformation("Login request {@Request}", loginRequest);

// Good — 안전한 식별자만
_logger.LogInformation("Login attempt for {UserId}", loginRequest.UserId);

// 토큰 마스킹 예시
_logger.LogDebug("Token issued {TokenPrefix}***", token[..6]);
```

---

## 7. appsettings.json 기본 구조

```jsonc
{
  "Serilog": {
    /* §6.2 참조 */
  },
  "ConnectionStrings": {
    "Default": "Server=localhost;Port=3306;Database=appdb;User=app_user;Password=__SET_IN_SECRETS__;Minimum Pool Size=1;Keepalive=60;",
  },
  "AllowedHosts": "*",
}
```

### 환경별 파일

| 파일                           | 용도                             | Git 추적       |
| ------------------------------ | -------------------------------- | -------------- |
| `appsettings.json`             | 공통 기본값                      | 추적           |
| `appsettings.Development.json` | 로컬 개발 (상세 로그 레벨 등)    | 추적           |
| `appsettings.Production.json`  | 운영 (민감값은 비밀 저장소 참조) | 추적(값 제외)  |
| `secrets.json` (User Secrets)  | 로컬 비밀값                      | **추적 안 함** |

- 환경 변수 `ASPNETCORE_ENVIRONMENT`로 분기 (`Development`/`Production`)
- **실제 비밀번호·키는 appsettings에 평문 저장 금지** — 개발은 `dotnet user-secrets`, 운영은 환경 변수 또는 비밀 관리 서비스(예: Vault, KeyVault) 사용
- `appsettings.Development.json`은 `appsettings.json`을 오버라이드 (병합)

> 참조 프로젝트는 GUI에서 `AppSettings` 모델로 설정을 관리하나, 신규 서버는 **`appsettings.json` + `IOptions<T>` 바인딩** 표준을 따른다.

---

## 8. Program.cs 기본 패턴 (Web API)

§6.1 부트스트랩을 포함한 전체 골격:

```csharp
using FluentValidation;
using FluentValidation.AspNetCore;
using Serilog;

Log.Logger = new LoggerConfiguration().WriteTo.Console().CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // 로깅
    builder.Host.UseSerilog((ctx, services, cfg) => cfg
        .ReadFrom.Configuration(ctx.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());

    // 엔드포인트·검증
    builder.Services.AddControllers();
    builder.Services.AddFluentValidationAutoValidation();
    builder.Services.AddValidatorsFromAssemblyContaining<CreateOrderRequestValidator>();

    // Swagger (Development 전용 노출 권장)
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    // 설정 바인딩 (IOptions)
    builder.Services.Configure<DatabaseOptions>(
        builder.Configuration.GetSection("ConnectionStrings"));

    // 레이어별 DI 등록 — 확장 메서드로 분리
    builder.Services.AddCoreServices();   // Core 도메인 서비스
    builder.Services.AddDataServices(builder.Configuration);  // Data 리포지토리·연결

    var app = builder.Build();

    // 미들웨어 파이프라인 (순서 중요)
    app.UseSerilogRequestLogging();

    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI();
    }

    app.UseHttpsRedirection();
    app.UseAuthentication();
    app.UseAuthorization();
    app.MapControllers();

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### 레이어별 DI 등록 패턴

각 계층에 `IServiceCollection` 확장 메서드를 두어 `Program.cs`를 간결하게 유지:

```csharp
// Core 계층 — {Solution}.Core/DependencyInjection.cs
namespace {Solution}.Core;

public static class CoreServiceCollectionExtensions
{
    public static IServiceCollection AddCoreServices(this IServiceCollection services)
    {
        services.AddScoped<IOrderService, OrderService>();
        return services;
    }
}

// Data 계층 — {Solution}.Data/DependencyInjection.cs
namespace {Solution}.Data;

public static class DataServiceCollectionExtensions
{
    public static IServiceCollection AddDataServices(
        this IServiceCollection services, IConfiguration config)
    {
        var connStr = config.GetConnectionString("Default")!;
        services.AddScoped<IOrderRepository>(_ => new OrderRepository(connStr));
        return services;
    }
}
```

### Worker Service / gRPC 분기

| 유형       | 부트스트랩 차이                                                                                                |
| ---------- | -------------------------------------------------------------------------------------------------------------- |
| Worker     | `Host.CreateApplicationBuilder` + `builder.Services.AddHostedService<Worker>()`, Swagger·HTTP 파이프라인 없음  |
| gRPC       | `builder.Services.AddGrpc()` + `app.MapGrpcService<MyService>()`, Swagger 대신 gRPC reflection(개발) 사용      |
| WPF 호스트 | `App.xaml.cs`에서 `Host.CreateApplicationBuilder` 빌드 후 `BackgroundService`로 서버 구동 (참조 프로젝트 방식) |

---

## 9. 코딩 스타일

> 세부 규칙은 `LANGUAGE_GUIDELINES_CSHARP.md` 참조. 부트스트랩 시 처음부터 적용할 핵심만 정리.

### 9.1 파일 스코프 네임스페이스 (필수)

```csharp
// Good — 파일 스코프 (들여쓰기 1단계 절약)
namespace MDWidgetServer.Services.ConnGrp;

public sealed class ConnGrpDatabaseService { /* ... */ }

// Bad — 블록 스코프 (신규 코드에서 지양)
namespace MDWidgetServer.Services.ConnGrp
{
    public class ConnGrpDatabaseService { }
}
```

### 9.2 Primary Constructor (.NET 8+)

```csharp
// Good — primary constructor로 DI 주입 간결화
public sealed class OrderService(
    IOrderRepository repository,
    ILogger<OrderService> logger)
{
    public Task<long> CreateAsync(CreateOrderRequest req) =>
        repository.InsertAsync(req);
}
```

### 9.3 record / record struct (불변 값 객체)

```csharp
// DTO·요청·응답은 record (불변, 값 동등성, with 식)
public record CreateOrderRequest(long CustomerId, string ProductCode, int Quantity);

// 작은 값 객체는 readonly record struct (힙 할당 회피)
public readonly record struct Money(decimal Amount, string Currency);
```

### 9.4 nullable 참조 타입 (Nullable=enable 전제)

- `Directory.Build.props`에서 `<Nullable>enable</Nullable>` 활성화
- nullable이면 `?` 명시 (`string? name`), 비-null 보장이면 `required` 또는 생성자 초기화
- `!` (null-forgiving)는 정말 보장될 때만 — 남용 금지

```csharp
public sealed class Customer
{
    public required string Name { get; init; }   // 필수 — 초기화 강제
    public string? Email { get; init; }            // 선택 — null 허용 명시
}
```

### 9.5 async / await 패턴

- I/O 작업은 `async`/`await`, 메서드명 `Async` 접미사
- **`CancellationToken`을 메서드 체인 끝까지 전파** (요청 취소·타임아웃 대응)
- `async void` 금지 (이벤트 핸들러 제외) — `async Task` 반환
- 라이브러리 경로에서는 `ConfigureAwait(false)` 고려 (ASP.NET Core 앱 코드에서는 불필요)

```csharp
// Good — CancellationToken 전파
public async Task<IEnumerable<OrderRow>> GetListAsync(CancellationToken ct = default)
{
    using var conn = new MySqlConnection(_connStr);
    return await conn.QueryAsync<OrderRow>(
        new CommandDefinition(Sql, cancellationToken: ct));
}

// Bad — 동기 블로킹 (.Result / .Wait()는 데드락 위험)
var rows = GetListAsync().Result;
```

---

## 10. 셋업 완료 검증

> **동작 증명 없이 완료 처리 금지** (`AI_BEHAVIOR.md` 핵심 원칙). 아래를 실제 실행으로 확인.

```powershell
# 1) 복원 + 빌드 (경고=오류 설정이라 경고도 잡힘)
dotnet restore
dotnet build

# 2) 테스트
dotnet test

# 3) API 실행 후 Swagger 응답 확인 (Web API)
dotnet run --project src/{Solution}.Api
# → https://localhost:{port}/swagger 접속 확인
```

| 검증 항목         | 통과 기준                                    |
| ----------------- | -------------------------------------------- |
| 빌드              | 경고·오류 0 (TreatWarningsAsErrors)          |
| 중앙 패키지 관리  | 개별 csproj에 버전 없이도 복원 성공          |
| Serilog           | 콘솔·파일에 구조화 로그 출력, 요청 로그 기록 |
| Swagger (Web API) | `/swagger` UI 로드, 엔드포인트 표시          |
| 테스트            | `dotnet test` 전체 통과                      |
| 레이어 의존       | Core가 Api/Data를 참조하지 않음 (순환 없음)  |

---

## 11. 빠른 부트스트랩 명령 (Web API + 3계층)

```powershell
# 솔루션·프로젝트 생성
dotnet new sln -n {Solution}
dotnet new webapi   -o src/{Solution}.Api  -n {Solution}.Api
dotnet new classlib -o src/{Solution}.Core -n {Solution}.Core
dotnet new classlib -o src/{Solution}.Data -n {Solution}.Data
dotnet new xunit    -o tests/{Solution}.Tests -n {Solution}.Tests

# 솔루션에 추가
dotnet sln add (Get-ChildItem -Recurse *.csproj)

# 프로젝트 참조 (의존 방향: Api → Core ← Data)
dotnet add src/{Solution}.Api/{Solution}.Api.csproj   reference src/{Solution}.Core/{Solution}.Core.csproj
dotnet add src/{Solution}.Api/{Solution}.Api.csproj   reference src/{Solution}.Data/{Solution}.Data.csproj
dotnet add src/{Solution}.Data/{Solution}.Data.csproj reference src/{Solution}.Core/{Solution}.Core.csproj
dotnet add tests/{Solution}.Tests/{Solution}.Tests.csproj reference src/{Solution}.Core/{Solution}.Core.csproj

# 공통 파일 생성 (Directory.Build.props / Directory.Packages.props는 §4 템플릿으로 작성)
```

이후 §4 두 props 파일 작성 → §5 패키지를 CPM에 등록 → 개별 csproj에 버전 없는 `PackageReference` 추가 → §8 `Program.cs` 작성 → §10 검증 순으로 진행.

---

## 참조 요약

| 주제                 | 참조 문서                       |
| -------------------- | ------------------------------- |
| 공통 부트스트랩 원칙 | `PROJECT_SETUP_GUIDE.md`        |
| C# 코딩 규칙 세부    | `LANGUAGE_GUIDELINES_CSHARP.md` |
| 개발 원칙(SOLID 등)  | `DEVELOPMENT.md`                |
| AI 동작·질문 게이트  | `AI_BEHAVIOR.md`                |
| 커밋 메시지          | `COMMIT_CONVENTION.md`          |
