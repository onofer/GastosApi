<!--
Sync Impact Report
Version change: 1.0.0 → 1.1.0
Modified principles:
  II. Explicit and Stable API Contracts → II. English Naming and Clear Boundaries
  IV. Automated Verification → IV. Endpoint Test Coverage
Added sections: Technology Constraints
Removed sections: none
Follow-up TODOs: none
-->

# GastosApi Constitution

## Core Principles

### I. Financial Data Integrity
Expense amounts MUST retain exact decimal precision and MUST be associated with an
explicit currency. Calculations, persistence, and API serialization MUST preserve
the entered value without silent rounding or currency assumptions. Invalid or
inconsistent financial data MUST be rejected explicitly.

### II. English Naming and Clear Boundaries
Source code, identifiers, and technical names MUST be written in English.
Controllers MUST NOT contain business logic; they MUST delegate business
operations to appropriate application services. This separation keeps the
behavior independently testable and the API layer focused on HTTP concerns.

### III. Explicit API Contracts and Boundary Validation
Every endpoint MUST have a clear request, response, and error contract. All
external input MUST be validated at the API boundary before it enters application
or domain logic. Changes to public behavior MUST remain backward compatible or be
introduced as an explicitly versioned breaking change. Validation failures MUST
produce actionable, consistent error responses without exposing internal
implementation details.

### IV. Endpoint Test Coverage
Every API endpoint MUST have automated tests written with MSTest. Tests MUST
cover successful behavior and relevant validation and error outcomes, including
the endpoint's public contract. Financial calculations, authorization, and
compatibility changes MUST also have focused tests. The relevant tests and build
MUST pass before a change is considered complete.

### V. Security, Simplicity, and Operability
Access to expense data MUST be authorized for the relevant user or role, and
authorization MUST be enforced on every applicable operation. Secrets and
sensitive personal or financial data MUST NOT be written to logs. Implementations
MUST prefer the simplest design that satisfies requirements; new dependencies and
abstractions MUST have a clear benefit. Operational failures MUST be observable
through useful, non-sensitive diagnostics, and errors MUST NOT be silently
converted into successful results.

## Technology Constraints

- The project MUST use C# and target .NET 9.
- Endpoint tests MUST use MSTest.
- Monetary values MUST use a representation that avoids binary floating-point
  rounding errors. Currency MUST be explicit wherever amounts are stored,
  exchanged, or compared.
- Responses and logs MUST exclude secrets and unnecessary personal or financial
  details. Data access MUST be restricted to the authorized scope.
- API changes MUST document any compatibility impact and provide a migration path
  for breaking changes.

## Development Workflow

- Before implementation, identify the affected API contract and relevant data,
  validation, and authorization rules.
- Validate external input at the API boundary; keep business rules outside
  controllers.
- Add or update MSTest coverage for every endpoint affected by a behavior change.
  Prioritize exact-value financial cases, invalid input, unauthorized access, and
  compatibility.
- Run the smallest relevant test set during development and the project build
  and applicable full tests before completion.
- Review changes for data exposure, error handling, backward compatibility, and
  adherence to these principles.

## Governance

This constitution governs project design and delivery. A change that conflicts
with it MUST either be revised to comply or be accompanied by an explicit
constitution amendment. Amendments MUST be reviewed, explain their rationale, and
update this document's version and amendment date.

Versioning follows semantic versioning: MAJOR for backward-incompatible
governance changes, MINOR for new principles or materially expanded requirements,
and PATCH for clarifications that do not change obligations. Reviews MUST assess
compliance with the principles, and any justified exception MUST be documented
with its scope and rationale.

**Version**: 1.1.0 | **Ratified**: 2026-11-03 | **Last Amended**: 2026-11-03
