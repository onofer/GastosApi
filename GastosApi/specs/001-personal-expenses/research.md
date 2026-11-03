# Research: Personal Expense Management

## Decisions

### Three-layer dependency direction

- **Decision**: Keep HTTP concerns in API, use cases and persistence abstractions
  in Application, and EF Core/SQL Server implementations in Data. API is the
  composition root and wires Data implementations into Application abstractions.
  Application does not reference Data.
- **Rationale**: Business behavior remains testable without binding it to
  persistence details. This honors the requested three layers while keeping
  dependencies directed inward.
- **Alternatives considered**: Application directly referencing Data is a
  simpler compile-time graph but couples use cases to EF Core and SQL Server.
- **Source**: [Microsoft: Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture)

### Request validation and ProblemDetails

- **Decision**: Use controller request models and API-boundary validation;
  `[ApiController]` handles invalid model state as a 400 response. Configure
  ProblemDetails services, exception handling, and status-code pages so
  validation and HTTP errors have a consistent public format. Keep application
  invariants as a second line of defense, not as a substitute for boundary
  validation.
- **Rationale**: Invalid external input is rejected before use cases execute,
  while invariants remain protected if application operations are called through
  another entry point. Framework middleware provides standardized errors without
  exposing exception details.
- **Alternatives considered**: Hand-building each controller error response
  duplicates policy; unhandled exceptions alone do not ensure all error statuses
  have a response body.
- **Sources**:
  - [ASP.NET Core Web API behavior](https://learn.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-9.0#apicontroller-attribute)
  - [ASP.NET Core API error handling and ProblemDetails](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling-api?view=aspnetcore-9.0#problem-details)

### Endpoint and persistence testing

- **Decision**: Use MSTest integration tests with the ASP.NET Core test host
  for the endpoint request/response pipeline. Use SQL Server for persistence
  integration tests and isolate test data per run.
- **Rationale**: Tests through the real request pipeline validate routing,
  binding, boundary validation, authorization, error serialization, and
  persistence together. EF providers differ in SQL translation, collation, and
  behavior; SQL Server tests catch provider-specific issues.
- **Alternatives considered**: EF Core InMemory and SQLite are faster for some
  tests but are not behaviorally equivalent to SQL Server. Use them only for
  tests that do not claim to verify SQL Server behavior.
- **Sources**:
  - [ASP.NET Core integration tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-9.0)
  - [EF Core: Choosing a testing strategy](https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy)
  - [EF Core: Testing against the database](https://learn.microsoft.com/en-us/ef/core/testing/testing-with-the-database)

### EF Core registration, configuration, and migrations

- **Decision**: Register the Data `DbContext` with the SQL Server provider through
  dependency injection using a request-scoped lifetime. Read the connection
  string from configuration/secrets. Keep migrations with the Data project;
  review generated migrations and apply them as a deployment step, not
  automatically at production startup.
- **Rationale**: This aligns context lifetime with a web request, keeps secrets
  out of source, and makes database changes reviewable and operationally
  controlled.
- **Alternatives considered**: A separate migrations project is unnecessary for
  this single provider/project layout. Startup migration can complicate
  production permissions and concurrent deployments.
- **Sources**:
  - [EF Core DbContext configuration](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#dbcontext-in-dependency-injection-for-aspnet-core)
  - [EF Core migrations overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
  - [Applying EF Core migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)
  - [ASP.NET Core safe storage of app secrets](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-9.0)

## Operational Risk

- **Decision**: Keep the explicitly requested .NET 9 target for this feature.
- **Rationale**: The .NET support policy lists November 10, 2026 as .NET 9's
  end-of-support date; this plan is dated November 3, 2026. A supported target
  must be selected before development/deployment that continues past that date.
- **Source**: [.NET and .NET Core support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)

## Resolved Design Assumptions

- The hosting environment configures ASP.NET Core bearer authentication and
  supplies a stable user identifier as a claim, matching the API contract.
  Token issuance and identity-provider choice are outside this feature's scope.
- `CurrencyCode` is fixed in configuration for the lifetime of stored expenses.
  Each stored record also snapshots the code so persisted amounts remain
  interpretable; changing currency with existing data requires a separately
  reviewed migration/conversion.
- The configured business timezone determines today's date and monthly summary
  boundaries.
- SQL Server tests use an isolated database or equivalent isolated test data;
  production connection strings and credentials are not committed.
