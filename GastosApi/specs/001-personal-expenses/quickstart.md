# Quickstart: Personal Expense Management

This guide describes how to validate the feature after the solution and projects
have been created. It is a design/validation guide, not implementation code.

## Prerequisites

- .NET 9 SDK and a SQL Server instance available to the developer.
- The `GastosApi.sln` solution and projects described in [plan.md](./plan.md).
- A development connection string configured as a user secret or environment
  variable; do not commit credentials.
- `CurrencyCode` set to one supported ISO 4217 code and `BusinessTimeZoneId`
  set to the intended business timezone.
- Authentication configured for the API, with an authenticated test identity
  available to API integration tests.

## Build and test

From the repository root, after project scaffolding exists:

```powershell
dotnet restore .\GastosApi.sln
dotnet build .\GastosApi.sln --no-restore
dotnet test .\GastosApi.sln --no-build
```

API endpoint tests run through the ASP.NET Core test host and SQL Server. Test
data must be isolated per run and cleaned up so repeated runs do not affect
results.

## Database setup

After the Data project defines the EF Core context and initial migration, apply
the migration to a development/test database using the Data project as the
migrations project and the API project as the startup project:

```powershell
dotnet ef database update --project .\src\GastosApi.Data --startup-project .\src\GastosApi.Api
```

Review generated migrations before applying them. Production deployments apply
reviewed migration scripts or bundles as a deployment step, not automatically
on API startup.

## End-to-end scenarios

1. Authenticate as a test user and create a valid expense with a positive amount,
   today's date, an allowed category, and optional description. Confirm the
   response is `201 Created`, includes a `Location`, and returns the configured
   currency.
2. Retrieve the returned identifier and confirm exact amount/date/category.
3. Attempt zero/negative amount, future date, unsupported category, malformed
   date, and excess currency precision. Confirm each request returns a
   validation ProblemDetails response and is not persisted.
4. Create expenses on both ends of a date range and in multiple categories.
   Query with category and inclusive date filters; confirm only matching
   caller-owned expenses are returned.
5. Request monthly totals and confirm all four categories appear, amounts are
   exact, adjacent months and other owners do not contribute, and an empty month
   reports zero totals.
6. Attempt retrieval and deletion as another authenticated user. Confirm the
   response is not found and does not disclose whether the record exists.
7. Delete an owned expense. Confirm `204 No Content` and verify it is absent from
   retrieval, listing, and monthly totals.
8. Send invalid month parameters and a reversed date range. Confirm `400` with a
   ProblemDetails response.
