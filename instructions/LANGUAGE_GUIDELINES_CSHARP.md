# C# / .NET 코딩 지침

**기준 버전**: C# 12 / **.NET 8** (`net8.0`, WPF 호스트는 `net8.0-windows`)
**LangVersion**: `latest`
**대상 솔루션**: `MDWidgetServer` (Core / Data / Api + WPF 호스트, ASP.NET Core + MQTTnet + Dapper/MySQL)

> .NET Core 기반(.NET 8 LTS). 최신 C# 언어 기능 적극 활용. 신규 서비스는 .NET 8 LTS 기준, 차기 LTS(.NET 10) 출시 시 PR로 검증 후 이전.

---

## 1. 프로젝트 공통 설정 (Directory.Build.props)

솔루션 루트 `Directory.Build.props`로 전 프로젝트 공통 적용:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>          <!-- 필수 -->
  <ImplicitUsings>enable</ImplicitUsings>
  <LangVersion>latest</LangVersion>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors> <!-- 신규 권장 -->
</PropertyGroup>
```

- **중앙 패키지 관리(CPM)** — `Directory.Packages.props`에 `ManagePackageVersionsCentrally=true` + `PackageVersion` 선언. 개별 `.csproj`의 `PackageReference`에는 버전 미기재
- NuGet 버전은 한 곳에서만 관리 (버전 드리프트 방지)

---

## 2. 네이밍 (Microsoft 표준)

| 대상               | 규칙                    | 예시                                       |
| ------------------ | ----------------------- | ------------------------------------------ |
| 클래스/메서드/속성 | PascalCase              | `InspectionResultApiService`, `RouteAsync` |
| 변수/파라미터      | camelCase               | `connectedClientsCount`                    |
| 인터페이스         | `I` 접두사 + PascalCase | `IMqttServer`, `IUserRepository`           |
| private 필드       | `_camelCase`            | `_mqttServer`, `_db`                       |
| 상수               | PascalCase              | `MaxRetry`                                 |
| 비동기 메서드      | `Async` 접미사          | `StartAsync`, `HandleHistAsync`            |

> TypeScript와 정반대로 C#은 인터페이스 `I` 접두사가 **표준**임에 유의.

---

## 3. 최신 C# 언어 기능 활용

- **파일 스코프 네임스페이스** (`namespace Foo;`) — 중괄호 블록 지양, 들여쓰기 절약
- **Primary constructor** (C# 12) 또는 expression-bodied 생성자로 DI 주입 간결화
- 패턴 매칭 / `switch` 식 활용 (라우팅·분기)
- Target-typed `new()` (`private static readonly JsonSerializerOptions JsonOpts = new() { ... }`)
- 컬렉션 식(`[..]`), `record` (불변 DTO), `required` 멤버

```csharp
// Good — 파일 스코프 ns + primary constructor
namespace MDWidgetServer.Services;

public class InspectionResultApiService(InspectionResultDatabaseService db)
{
    private readonly InspectionResultDatabaseService _db = db;
}
```

---

## 4. Nullable Reference Types

- `<Nullable>enable</Nullable>` 필수
- nullable 가능 멤버는 `?` 명시 (`MqttServer? _mqttServer`)
- `!` (null-forgiving 연산자) 최소화 — 가드/`ArgumentNullException.ThrowIfNull` 우선
- 이벤트·옵셔널 의존은 `?.Invoke(...)` 패턴 유지

---

## 5. 비동기 처리

- 공개 비동기 메서드는 `Async` 접미사 + `Task`/`Task<T>` 반환
- `async void` 금지 (이벤트 핸들러 예외). 핸들러는 `Task` 반환 또는 `async Task`
- **라이브러리/공유 코드는 `ConfigureAwait(false)`** (UI 컨텍스트 데드락 방지). WPF UI 코드는 예외
- 공개 비동기에 `CancellationToken` 파라미터 전파 (장기 작업·DB·네트워크)
- DB 접근은 **Dapper** 비동기 API (`QueryAsync` / `ExecuteAsync`), MySQL은 `MySqlConnector`

```csharp
// Good
public async Task<IReadOnlyList<Item>> GetItemsAsync(int id, CancellationToken ct = default)
{
    await using var conn = new MySqlConnection(_connStr);
    return (await conn.QueryAsync<Item>(Sql, new { id })).ToList();
}
```

---

## 6. 서버 로깅 (구조화 로깅 — Serilog)

> **현재 코드베이스는 `Console.WriteLine`을 사용 중이나, 이는 안티패턴.** 신규/리팩터링 코드는 아래 표준으로 전환한다.

- **`Microsoft.Extensions.Logging` 추상화 + Serilog 싱크** 사용. **log4net 신규 도입 금지**
- `Console.WriteLine` 로깅 금지 → `ILogger<T>` 주입
- **구조화 로깅** — 문자열 보간 대신 메시지 템플릿 + 파라미터 (검색·집계 가능)
- **로그 레벨** 준수: `Trace`/`Debug`(개발), `Information`(정상 흐름), `Warning`(복구 가능 이상), `Error`(처리 실패), `Critical`(시스템 중단)
- **요청 추적 ID(Correlation ID)** — 요청 단위 `LogContext.PushProperty("CorrelationId", id)` 또는 Serilog `Enrich`로 모든 로그에 부착. ASP.NET Core는 `TraceIdentifier`/`Activity.Current.Id` 활용

```csharp
// Bad — 비구조화, 보간 문자열
Console.WriteLine($"Client connected: {clientId}, Total: {count}");

// Good — 구조화 로깅 (속성으로 검색 가능)
_logger.LogInformation(
    "MQTT client connected {ClientId}, total {ClientCount}",
    clientId, count);

// 예외 로깅 — 예외 객체를 첫 인자로
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to start MQTT server on port {Port}", settings.MqttPort);
}
```

**Serilog 부트스트랩 (Program.cs / Host 빌더):**

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "MDWidgetServer")
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {CorrelationId} {Message:lj}{NewLine}{Exception}")
    .WriteTo.File("logs/mdwidget-.log", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog(); // Serilog.Extensions.Hosting
```

> 운영 환경은 JSON 포맷 싱크(`Serilog.Formatting.Compact`) 권장 — 로그 수집·검색 시스템 연동.

---

## 7. 의존성 주입 / 구조

- **생성자 주입** 표준 (`Microsoft.Extensions.DependencyInjection` / `Microsoft.Extensions.Hosting`)
- 계층 분리 준수: **Core**(도메인·서비스) → **Data**(Dapper 리포지토리·마이그레이션) → **Api**(HTTP/MQTT 라우팅). 의존 방향 역전 금지
- `JsonSerializerOptions` 등 비용 큰 객체는 `static readonly`로 재사용

## 8. DB 마이그레이션

- **dbup-mysql** 사용. 마이그레이션 SQL은 `Migrations/*.sql` 임베디드 리소스로 관리
- 신규 테이블·스키마 변경은 반드시 마이그레이션 스크립트로 — 수동 DDL 금지

## 참고

- [.NET 8 문서](https://learn.microsoft.com/dotnet/)
- [C# 코딩 컨벤션](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [Serilog](https://serilog.net/)
- [Dapper](https://github.com/DapperLib/Dapper)
