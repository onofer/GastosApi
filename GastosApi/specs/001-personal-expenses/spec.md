# Feature Specification: Personal Expense Management

**Feature Branch**: `001-personal-expenses`

**Created**: 2026-11-03

**Status**: Draft

**Input**: User description: "Quiero una API para registrar gastos personales. Cada gasto tiene un importe, una fecha, una categoría (comida, transporte, ocio, otros) y una descripción opcional. Se puede crear, consultar uno, listar con filtro por categoría y rango de fechas, y eliminar. El importe debe ser mayor que cero y la fecha no puede ser futura. Quiero ver el total gastado por categoría en un mes. Moneda: una única moneda configurada para toda la API."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Record and view an expense (Priority: P1)

As a person tracking personal spending, I can register an expense and retrieve
its details so that I can keep an accurate record of what I spent.

**Why this priority**: Recording and confirming an expense is the core value of
the feature.

**Independent Test**: Create a valid expense and retrieve it by its identifier;
the returned amount, date, category, and optional description match the saved
expense.

**Acceptance Scenarios**:

1. **Given** a valid amount, non-future date, allowed category, and optional
   description, **When** the person creates an expense, **Then** the API stores
   it and returns its identifier and expense details, including the configured
   currency.
2. **Given** an existing expense owned by the caller, **When** the person
   retrieves it by identifier, **Then** the API returns its recorded details
   without changing the amount.
3. **Given** an amount that is zero or negative, **When** the person creates an
   expense, **Then** the API rejects the request and identifies the invalid
   amount.
4. **Given** a date later than the current date, **When** the person creates an
   expense, **Then** the API rejects the request and identifies the invalid
   date.

---

### User Story 2 - Find and remove expenses (Priority: P2)

As a person tracking personal spending, I can list expenses and narrow the
results by category and date range, and remove an expense I no longer want
recorded.

**Why this priority**: Finding and correcting records makes the expense history
useful beyond the moment an expense is created.

**Independent Test**: Create expenses with different categories and dates, list
them with each filter and with combined filters, then delete one and confirm it
is no longer retrievable or listed.

**Acceptance Scenarios**:

1. **Given** expenses in multiple categories and on multiple dates, **When**
   the person lists expenses with a category and inclusive date range, **Then**
   only expenses matching every supplied filter are returned.
2. **Given** no filters, **When** the person lists expenses, **Then** all
   expenses they are authorized to access are returned.
3. **Given** an existing expense owned by the caller, **When** the person
   deletes it, **Then** the API confirms deletion and subsequent retrieval does
   not return that expense.
4. **Given** a date range whose start is after its end, **When** the person
   lists expenses, **Then** the API rejects the invalid range.
5. **Given** an expense that does not exist or is not owned by the caller,
   **When** the person retrieves or deletes it, **Then** the API does not expose
   another person's data and returns a not-found response.

---

### User Story 3 - Review monthly category totals (Priority: P3)

As a person tracking personal spending, I can see how much I spent in each
category during a month so that I can understand where my money went.

**Why this priority**: Category totals provide a useful overview after
individual expenses can be recorded and retrieved.

**Independent Test**: Create expenses across categories and months, request a
monthly summary, and compare each category total with the exact sum of matching
expenses.

**Acceptance Scenarios**:

1. **Given** expenses in several categories during a month, **When** the person
   requests that month's summary, **Then** the API returns the exact total for
   each category and the configured currency.
2. **Given** expenses outside the requested month, **When** the person requests
   a monthly summary, **Then** those expenses do not affect its totals.
3. **Given** a month with no expenses, **When** the person requests its summary,
   **Then** the API returns zero for each category and the configured currency.

### Edge Cases

- A category outside `comida`, `transporte`, `ocio`, and `otros` is rejected.
- A date equal to the current date is valid; only a later date is rejected.
- A missing optional description is accepted and represented consistently as
  absent or empty.
- A filter with no matching expenses returns an empty list rather than an error.
- A monthly total includes expenses on the first and last calendar day of the
  requested month and excludes expenses from adjacent months.
- Amount handling preserves exact monetary values; unsupported precision is
  rejected rather than silently rounded.
- A caller cannot list, retrieve, delete, or aggregate expenses belonging to
  another person.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The API MUST allow an authorized caller to create an expense with
  an amount, date, category, and optional description.
- **FR-002**: The API MUST assign each created expense a unique identifier and
  retain its details so they can be retrieved later.
- **FR-003**: The API MUST reject amounts less than or equal to zero and MUST
  preserve accepted amounts without silent rounding.
- **FR-004**: The API MUST reject expense dates later than the current calendar
  date. The current date MUST be evaluated in the API's configured business
  timezone.
- **FR-005**: The API MUST accept only the categories `comida`, `transporte`,
  `ocio`, and `otros`.
- **FR-006**: The API MUST use one configured ISO 4217 currency for all
  expenses and MUST identify that currency in expense and summary responses.
- **FR-007**: The API MUST allow an authorized caller to retrieve an expense by
  its identifier.
- **FR-008**: The API MUST allow an authorized caller to list expenses, with
  optional category and inclusive start-date and end-date filters. When more
  than one filter is supplied, all filters MUST apply.
- **FR-009**: The API MUST reject unsupported categories and date ranges whose
  start is later than their end.
- **FR-010**: The API MUST allow an authorized caller to delete an expense and
  MUST exclude deleted expenses from subsequent retrieval, listing, and totals.
- **FR-011**: The API MUST provide a monthly summary with the exact total for
  each allowed category, including zero totals for categories without expenses.
  The summary MUST include the configured currency.
- **FR-012**: Expense operations and monthly summaries MUST be restricted to
  expenses the caller is authorized to access. Requests for unavailable
  identifiers MUST NOT reveal whether another person's expense exists.
- **FR-013**: Invalid requests MUST return a clear error that identifies the
  invalid input without exposing internal implementation details.

### Key Entities *(include if data involved)*

- **Expense**: A personal spending record with a unique identifier, positive
  monetary amount, date, one allowed category, and optional description.
- **Category**: One of the fixed expense groupings: `comida`, `transporte`,
  `ocio`, or `otros`.
- **Monthly Category Summary**: The exact sum of a caller's non-deleted
  expenses grouped by category for one calendar month, expressed in the
  configured currency.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of valid expense creation requests can be confirmed by
  retrieving the created expense with the same amount, date, category, and
  description.
- **SC-002**: 100% of requests with zero/negative amounts, future dates, or
  unsupported categories are rejected.
- **SC-003**: For any tested month, every returned category total equals the
  exact sum of that caller's matching, non-deleted expenses, with no
  contribution from adjacent months or other callers.
- **SC-004**: After a successful deletion, the expense is absent from retrieval,
  listing, and monthly totals.

## Assumptions

- One ISO 4217 currency is configured for the API and applies to all expenses;
  the specific currency is selected by the deployment and is not specified here.
- The calling environment supplies the authenticated caller identity. Each
  person can access only their own expenses.
- The API has a configured business timezone that determines the current
  calendar date and the boundaries of calendar months.
- Amount precision follows the configured currency's supported monetary
  precision; values exceeding it are rejected rather than rounded.
- Editing expenses, exporting records, and converting between currencies are
  outside this feature's scope.
