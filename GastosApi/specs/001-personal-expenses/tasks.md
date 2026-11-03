# Tasks: Personal Expense Management

**Input**: Design documents from `/specs/001-personal-expenses/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Tests**: Required by the feature specification and constitution. All endpoint
tests use MSTest and exercise the ASP.NET Core HTTP pipeline. SQL Server-backed
tests verify persistence and query behavior.

**Organization**: Tasks are grouped by user story so each increment can be
implemented and validated independently after the shared foundation.

## Phase 1: Setup

**Purpose**: Create the solution and the three production layers plus MSTest projects.

- [ ] T001 Create `GastosApi.sln` and the `src/GastosApi.Api`, `src/GastosApi.Application`, `src/GastosApi.Data`, `tests/GastosApi.Application.Tests`, and `tests/GastosApi.Api.Tests` projects targeting .NET 9.
- [ ] T002 Add project references so API composes Application and Data, Data implements Application abstractions, and Application references neither API nor Data; update each `.csproj` and `GastosApi.sln`.
- [ ] T003 Add EF Core SQL Server and Design packages to `src/GastosApi.Data/GastosApi.Data.csproj`.
- [ ] T004 Add MSTest, Microsoft.AspNetCore.Mvc.Testing, and required test SDK packages to `tests/GastosApi.Application.Tests/GastosApi.Application.Tests.csproj` and `tests/GastosApi.Api.Tests/GastosApi.Api.Tests.csproj`.
- [ ] T005 [P] Add nullable reference types, implicit usings, and consistent .NET 9 build properties to `Directory.Build.props`.

---

## Phase 2: Foundational

**Purpose**: Establish secure persistence, cross-cutting HTTP behavior, and test infrastructure required by every story.

**⚠️ CRITICAL**: Complete this phase before implementing any user story.

- [ ] T006 Define validated `CurrencyCode` and `BusinessTimeZoneId` options in `src/GastosApi.Application/Configuration/ExpenseOptions.cs`; fail application startup with a clear configuration error when either is missing or invalid.
- [ ] T007 Define the shared `Expense` entity shape, internal category enum, persistence abstraction, and authenticated owner context in `src/GastosApi.Application/Expenses/Models/Expense.cs`, `src/GastosApi.Application/Expenses/Models/ExpenseCategory.cs`, `src/GastosApi.Application/Abstractions/IExpenseRepository.cs`, and `src/GastosApi.Application/Abstractions/ICurrentUser.cs`; include `Guid` ID, owner ID, `decimal` amount, currency code, `DateOnly` date, category, and optional description.
- [ ] T008 Configure SQL Server `GastosDbContext`, Expense entity mapping, decimal precision, and owner/date and owner/category/date indexes in `src/GastosApi.Data/Persistence/GastosDbContext.cs` and `src/GastosApi.Data/Persistence/Configurations/ExpenseConfiguration.cs`.
- [ ] T009 Implement repository queries that always require owner scope in `src/GastosApi.Data/Expenses/ExpenseRepository.cs`; ensure unavailable and other-owner identifiers resolve identically.
- [ ] T010 Register Data and Application services and load `ConnectionStrings:Gastos` and expense options in `src/GastosApi.Api/Program.cs` and `src/GastosApi.Api/appsettings.json`; do not commit credentials.
- [ ] T011 Require an authenticated principal and register the owner-access policy in `src/GastosApi.Api/Program.cs`; bind bearer authentication supplied by the hosting environment and replace it only with a test authentication handler in `tests/GastosApi.Api.Tests/ExpenseApiFactory.cs`.
- [ ] T012 Configure boundary validation and consistent ProblemDetails for model-validation failures, 400/401/404 responses, and unhandled exceptions in `src/GastosApi.Api/Program.cs` and `src/GastosApi.Api/ProblemDetails/ApiExceptionHandler.cs`.
- [ ] T013 Create the initial EF Core migration in `src/GastosApi.Data/Persistence/Migrations/` and verify it creates the Expense table, constraints, and indexes from `data-model.md`.
- [ ] T014 [P] Add an Application test builder and fixed clock/timezone test helpers in `tests/GastosApi.Application.Tests/TestSupport/ExpenseTestSupport.cs`.
- [ ] T015 [P] Add SQL Server test database configuration, isolated database lifecycle, and a test-host factory in `tests/GastosApi.Api.Tests/ExpenseApiFactory.cs`; tests must fail clearly when the required SQL Server test connection is unavailable.

**Checkpoint**: Solution builds, migrations apply to a disposable SQL Server database, an unauthenticated request is rejected, and an API error is serialized as ProblemDetails.

---

## Phase 3: User Story 1 - Record and view an expense (Priority: P1) 🎯 MVP

**Goal**: An authenticated person can create a valid expense and retrieve it without loss of monetary precision.

**Independent Test**: Through the HTTP test host backed by SQL Server, create an expense, retrieve its returned identifier, and verify all returned fields and configured currency; verify invalid amount and future date produce ProblemDetails and do not persist.

### Tests for User Story 1

- [ ] T016 [P] [US1] Add MSTest API integration tests for valid expense creation, 201 response, Location header, exact returned fields, and retrieval in `tests/GastosApi.Api.Tests/Expenses/CreateAndGetExpenseTests.cs`.
- [ ] T017 [P] [US1] Add MSTest API integration tests for zero/negative amount, future date, unsupported category, excess currency precision, and missing authentication in `tests/GastosApi.Api.Tests/Expenses/CreateExpenseValidationTests.cs`.
- [ ] T018 [P] [US1] Add MSTest Application tests for positive amount, currency precision, category mapping, and business-timezone date rules in `tests/GastosApi.Application.Tests/Expenses/CreateExpenseTests.cs`.

### Implementation for User Story 1

- [ ] T019 [US1] Implement `Expense` creation invariants in `src/GastosApi.Application/Expenses/Models/Expense.cs`; enforce positive exact `decimal` amount, configured-currency precision, supported category, required owner, date no later than the supplied business date, and optional description.
- [ ] T020 [P] [US1] Map the API category values `comida`, `transporte`, `ocio`, and `otros` to the existing English internal enum in `src/GastosApi.Api/Contracts/CategoryMapping.cs`.
- [ ] T021 [US1] Implement create and get use cases in `src/GastosApi.Application/Expenses/Commands/CreateExpense.cs` and `src/GastosApi.Application/Expenses/Queries/GetExpense.cs`, assigning the configured currency and enforcing owner scope.
- [ ] T022 [US1] Add request/response DTOs and API-boundary validators in `src/GastosApi.Api/Contracts/Requests/CreateExpenseRequest.cs`, `src/GastosApi.Api/Contracts/Responses/ExpenseResponse.cs`, and `src/GastosApi.Api/Validation/CreateExpenseRequestValidator.cs`; enforce required amount/date/category, supported category, configured currency precision, and future-date rejection.
- [ ] T023 [US1] Add create and get routes that only bind, delegate, and map results in `src/GastosApi.Api/Controllers/ExpensesController.cs`; return 201 with Location and expense body, 200 on retrieval, and not-found ProblemDetails for unavailable IDs.
- [ ] T024 [US1] Verify created expenses are persisted and retrievable with SQL Server and exact decimal values using `tests/GastosApi.Api.Tests/Expenses/CreateAndGetExpenseTests.cs`.

**Checkpoint**: User Story 1 is independently testable and delivers the MVP.

---

## Phase 4: User Story 2 - Find and remove expenses (Priority: P2)

**Goal**: An authenticated person can list expenses with combined filters and delete only their own records.

**Independent Test**: Seed records for two owners across categories and date boundaries; verify unfiltered, category-only, range-only, and combined lists; delete an owned record and verify it is absent from get, list, and totals; verify another owner receives not-found.

### Tests for User Story 2

- [ ] T025 [P] [US2] Add MSTest API integration tests for unfiltered listing, category/date filtering, inclusive boundaries, empty results, reversed ranges, and invalid categories in `tests/GastosApi.Api.Tests/Expenses/ListExpensesTests.cs`.
- [ ] T026 [P] [US2] Add MSTest API integration tests for successful deletion, repeat deletion, foreign-owner retrieval/deletion, and not-found ProblemDetails in `tests/GastosApi.Api.Tests/Expenses/DeleteExpenseTests.cs`.
- [ ] T027 [P] [US2] Add MSTest Application tests for filter composition, date-range validation, owner scoping, and deletion behavior in `tests/GastosApi.Application.Tests/Expenses/FindAndDeleteExpenseTests.cs`.

### Implementation for User Story 2

- [ ] T028 [US2] Implement owner-scoped list and delete use cases in `src/GastosApi.Application/Expenses/Queries/ListExpenses.cs` and `src/GastosApi.Application/Expenses/Commands/DeleteExpense.cs`; apply every supplied filter and reject start dates later than end dates.
- [ ] T029 [US2] Extend the expense repository in `src/GastosApi.Application/Abstractions/IExpenseRepository.cs` and `src/GastosApi.Data/Expenses/ExpenseRepository.cs` for owner-scoped optional category/date filters and deletion.
- [ ] T030 [US2] Add boundary validation for category, ISO date values, and reversed ranges in `src/GastosApi.Api/Validation/ExpenseListQueryValidator.cs`; return ProblemDetails for invalid filters.
- [ ] T031 [US2] Add list and delete actions to `src/GastosApi.Api/Controllers/ExpensesController.cs`; return an empty array for no matches and 204 for successful deletion.
- [ ] T032 [US2] Verify SQL Server owner/date/category query behavior and deleted-record exclusion in `tests/GastosApi.Api.Tests/Expenses/ListExpensesTests.cs` and `tests/GastosApi.Api.Tests/Expenses/DeleteExpenseTests.cs`.

**Checkpoint**: User Stories 1 and 2 work independently; all queries and mutations remain owner-scoped.

---

## Phase 5: User Story 3 - Review monthly category totals (Priority: P3)

**Goal**: An authenticated person can see exact monthly spending totals for every category.

**Independent Test**: Seed SQL Server with expenses for the caller and another owner across month boundaries and categories; compare summary values to exact sums and verify all four categories are present, including zero totals.

### Tests for User Story 3

- [ ] T033 [P] [US3] Add MSTest API integration tests for exact totals, all four categories, empty-month zero totals, owner isolation, month boundaries, and currency in `tests/GastosApi.Api.Tests/Expenses/MonthlyExpenseSummaryTests.cs`.
- [ ] T034 [P] [US3] Add MSTest Application tests for month/year validation, timezone month boundaries, exact category aggregation, and zero-filled categories in `tests/GastosApi.Application.Tests/Expenses/MonthlyExpenseSummaryTests.cs`.

### Implementation for User Story 3

- [ ] T035 [US3] Define the monthly category summary result model in `src/GastosApi.Application/Expenses/Models/MonthlyExpenseSummary.cs`, including year, month, configured currency, and all four category totals.
- [ ] T036 [US3] Implement owner-scoped monthly aggregation in `src/GastosApi.Application/Expenses/Queries/GetMonthlyExpenseSummary.cs` and `src/GastosApi.Data/Expenses/ExpenseRepository.cs`; use inclusive calendar-month boundaries in the configured business timezone and exclude deleted records.
- [ ] T037 [US3] Add year/month query validation and the monthly summary response DTO in `src/GastosApi.Api/Validation/MonthlySummaryQueryValidator.cs` and `src/GastosApi.Api/Contracts/Responses/MonthlyExpenseSummaryResponse.cs`; reject invalid month/year with ProblemDetails.
- [ ] T038 [US3] Add the monthly summary endpoint to `src/GastosApi.Api/Controllers/ExpensesController.cs`; return every supported category and configured currency.
- [ ] T039 [US3] Verify aggregate query precision and SQL Server translation in `tests/GastosApi.Api.Tests/Expenses/MonthlyExpenseSummaryTests.cs`.

**Checkpoint**: All user stories meet their independent test criteria.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Validate the complete contract, configuration, and delivery readiness.

- [ ] T040 [P] Compare implemented routes, status codes, validation, and ProblemDetails schemas with `specs/001-personal-expenses/contracts/openapi.yaml`; update the contract if reviewed implementation decisions change it.
- [ ] T041 [P] Document configuration keys, authentication assumptions, SQL Server setup, migration procedure, currency precision, and .NET 9 support-end date in `README.md`.
- [ ] T042 Verify logs and ProblemDetails responses exclude secrets, stack traces, and sensitive expense values in `src/GastosApi.Api/ProblemDetails/ApiExceptionHandler.cs` and `tests/GastosApi.Api.Tests/ProblemDetails/ProblemDetailsSafetyTests.cs`.
- [ ] T043 Run the complete quickstart validation from `specs/001-personal-expenses/quickstart.md` and record/fix any failures.
- [ ] T044 Review the .NET 9 support lifecycle before delivery; document and approve a target-framework upgrade if delivery occurs after 2026-11-10 in `README.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies; establish the solution and project references first.
- **Foundational (Phase 2)**: Depends on Setup; blocks all story work.
- **User Story 1 (Phase 3)**: Depends on the shared foundation; first independently deliverable increment.
- **User Story 2 (Phase 4)**: Depends on the shared Expense model and persistence contract from the foundation and US1; US1 creates the records needed to demonstrate list/delete behavior.
- **User Story 3 (Phase 5)**: Depends on the shared Expense model and persisted data from US1; deleted-record exclusion also uses US2 delete semantics.
- **Polish (Phase 6)**: Depends on all stories selected for delivery.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2; no dependency on another story.
- **US2 (P2)**: Starts after US1's Expense model and repository contract are available; needs existing expenses to validate listing and deletion.
- **US3 (P3)**: Starts after US1's persisted Expense records and US2's deletion semantics; can implement its summary query independently of list filtering.

### Within Each User Story

- Write and run the story's MSTest cases before implementation; they should fail for the missing behavior and pass when complete.
- Define or extend Application models and use cases before API endpoint wiring.
- Keep external-input validation in API; retain domain invariants in Application.
- Keep controllers limited to HTTP binding, delegation, and response mapping.
- Verify SQL Server-backed behavior before marking a story complete.

## Parallel Opportunities

- Complete T001 before project configuration; T002 establishes project references, then T003 and T004 add package dependencies. T005 can proceed independently after the solution scaffold exists.
- Within Phase 2, T008, T014, and T015 can be prepared in parallel after the project skeleton exists; DB-context work and test-host work both depend on their respective packages.
- In US1, author T016, T017, and T018 in parallel before implementation; then implement T019, T020, and the dependent use-case and endpoint tasks.
- In US2, T025, T026, and T027 can be authored in parallel; list/delete use cases and repository changes must be coordinated through the shared repository contract.
- In US3, T033 and T034 can be authored in parallel; summary model, aggregation, and endpoint tasks follow in order.
- Different stories are not fully parallel in this plan because US2 and US3 rely on records created by US1, and US3 verification includes deletion semantics.

## Parallel Example: User Story 1

```text
After Phase 2 is complete:
- T016: API create/get integration tests
- T017: API validation integration tests
- T018: Application expense invariant tests

After those tests are written and fail:
- T019: Expense creation invariants
- T020: Category mapping
```

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 and Phase 2.
2. Implement US1 creation and retrieval with validation and SQL Server-backed MSTest coverage.
3. Run US1 tests independently and verify exact values and ProblemDetails errors.
4. Deliver/demo the recording and retrieval slice before starting additional stories.

### Incremental Delivery

1. Add US2 filtering and deletion; verify owner scoping and deleted-record behavior.
2. Add US3 monthly category totals; verify exact sums, all category zeros, and month boundaries.
3. Complete cross-cutting contract, security, configuration, and quickstart checks.

## Independent Test Criteria

- **US1**: Create a valid expense over HTTP, retrieve it by ID, and compare amount, date, category, optional description, and currency; invalid amount/date requests return ProblemDetails and persist nothing.
- **US2**: List records with no filter and with each/combined filters, verify inclusive dates and empty result behavior, delete an owned expense, and confirm a foreign owner receives not found.
- **US3**: Request a month with caller-owned expenses across all categories, compare exact totals, verify zero-filled categories, and prove adjacent months, deleted records, and another owner do not affect totals.
