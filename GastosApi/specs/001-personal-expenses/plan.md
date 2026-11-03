# Implementation Plan: Personal Expense Management

**Branch**: `001-personal-expenses` | **Date**: 2026-11-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-personal-expenses/spec.md`

## Summary

Deliver an authenticated ASP.NET Core Web API for creating, retrieving,
filtering, deleting, and summarizing personal expenses. Use three layers:
`GastosApi.Api` for HTTP and boundary validation, `GastosApi.Application` for
use cases and business rules, and `GastosApi.Data` for EF Core persistence to
SQL Server. Persist exact decimal amounts and an ISO 4217 currency code on each
expense, return consistent ProblemDetails errors, and verify every endpoint
through MSTest integration tests using SQL Server.

## Technical Context

**Language/Version**: C# / .NET 9

**Primary Dependencies**: ASP.NET Core Web API; Entity Framework Core 9 SQL
Server provider; MSTest; ASP.NET Core integration-test host

**Storage**: SQL Server; EF Core migrations owned by the Data project

**Testing**: MSTest unit and API integration tests; SQL Server-backed tests for
relational behavior

**Target Platform**: ASP.NET Core on a supported .NET 9 runtime platform

**Project Type**: REST web API, organized into API, Application, and Data
projects

**Performance Goals**: No numeric SLA is specified. Queries and summaries are
scoped to the authenticated owner; database indexes support owner/date and
owner/category/date filtering.

**Constraints**: One configured ISO 4217 currency; store positive amounts
without binary floating-point or silent rounding; use the configured business
timezone for date validation and month boundaries; protect all expense
operations by authenticated owner; use ProblemDetails for errors; no business
logic in controllers; validate external input at the API boundary.

**Scale/Scope**: Personal expense records per authenticated caller; create,
retrieve, list with optional category/date filters, delete, and monthly
category totals. Editing, exports, currency conversion, and pagination are out
of scope for this version.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / constraint | Gate result | Plan evidence |
|---|---|---|
| I. Financial Data Integrity | PASS | Use `decimal`, enforce currency precision, persist the configured currency code with each expense, and aggregate exact values. |
| II. English Naming and Clear Boundaries | PASS | C# identifiers, source names, and technical names are English; controllers only bind, delegate, and map results. |
| III. Explicit API Contracts and Boundary Validation | PASS | OpenAPI contract; validate DTOs and query inputs before application use cases; document stable error and success responses. |
| IV. Endpoint Test Coverage | PASS | MSTest integration coverage for every endpoint, including success, validation, authorization, and not-found outcomes. |
| V. Security, Simplicity, and Operability | PASS | Require authenticated owner context, scope data access to owner, standardize safe errors, and avoid unnecessary layers or dependencies. |
| C# / .NET 9 / MSTest | PASS | All three are explicit stack requirements. |
| API, Application, Data layers | PASS | API is the composition root; Application owns use cases and abstractions; Data implements persistence. |
| ProblemDetails | PASS | Configure ProblemDetails for validation, not-found, other HTTP errors, and unhandled exceptions without leaking internals. |

**Lifecycle risk**: .NET 9 support ends on 2026-11-10, shortly after this plan
date. The requested target is retained, but implementation and deployment beyond
that date require an explicit decision to upgrade to a supported .NET release.

## Project Structure

### Documentation (this feature)

```text
specs/001-personal-expenses/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── openapi.yaml
└── tasks.md
```

### Source Code (repository root)

```text
GastosApi.sln
src/
├── GastosApi.Api/
│   ├── Controllers/
│   ├── Contracts/
│   │   ├── Requests/
│   │   └── Responses/
│   ├── Validation/
│   ├── ProblemDetails/
│   └── Program.cs
├── GastosApi.Application/
│   ├── Expenses/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── Models/
│   └── Abstractions/
└── GastosApi.Data/
    ├── Persistence/
    │   ├── Configurations/
    │   └── Migrations/
    └── Expenses/
        └── Repositories/
tests/
├── GastosApi.Application.Tests/
└── GastosApi.Api.Tests/
```

**Structure Decision**: Use three production projects and two MSTest projects.
`GastosApi.Api` references Application and Data only for composition-root
registration; `GastosApi.Data` references Application to implement its
persistence abstractions; Application does not reference API or Data. The API
project owns HTTP request/response models and validation. Domain entities,
invariants, and use cases reside in Application. SQL Server mapping and
migrations reside in Data. API integration tests exercise the real HTTP pipeline
and SQL Server provider.

## Complexity Tracking

No constitution violations or additional architectural projects are proposed.
