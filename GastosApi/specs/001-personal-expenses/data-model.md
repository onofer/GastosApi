# Data Model: Personal Expense Management

## Expense

Represents one personal expense owned by one authenticated user.

| Field | Type / representation | Rules |
|---|---|---|
| `Id` | `Guid` | Required, unique, generated on creation. |
| `OwnerId` | String | Required stable identifier from the authenticated principal; every read, delete, list, and summary is scoped to it. |
| `Amount` | `decimal` / SQL `decimal(19,4)` | Required and greater than zero. Reject values exceeding configured currency precision; never silently round. |
| `CurrencyCode` | ISO 4217 string, SQL `char(3)` | Required; captured from the single configured currency on creation. Currency configuration must not change while records use it without an explicit conversion/migration. |
| `Date` | `DateOnly` / SQL `date` | Required; may not be later than today in the configured business timezone. |
| `Category` | English internal enum / SQL string | Required; allowed values map to the API's `comida`, `transporte`, `ocio`, and `otros`. |
| `Description` | Nullable string | Optional; trim surrounding whitespace; empty/whitespace input is treated as absent. |

### Invariants

- Amount is positive and representable at the configured currency's minor-unit
  precision.
- Currency code is a valid, configured ISO 4217 code.
- Owner identifier is present.
- Date is not in the future according to the configured business timezone.
- Category maps to one of the four supported external category values.
- A deleted expense is unavailable to all subsequent queries and aggregates.

## Monthly Category Summary

Read-only result, not persisted as an entity.

| Field | Type / representation | Rules |
|---|---|---|
| `Year` | Integer | Valid calendar year supplied by the caller. |
| `Month` | Integer | 1 through 12. |
| `CurrencyCode` | ISO 4217 string | The configured currency for the expense set. |
| `Totals` | Collection of category totals | Contains all four categories, including zero totals. |
| `Totals[].Category` | API category string | One of `comida`, `transporte`, `ocio`, `otros`. |
| `Totals[].Amount` | `decimal` | Exact sum of non-deleted expenses owned by the caller within the requested calendar month. |

## Relationships and Indexes

- One owner has zero or more expenses; each expense belongs to exactly one
  owner.
- Add an index on `(OwnerId, Date)` for owner-scoped date-range queries and
  monthly totals.
- Add an index on `(OwnerId, Category, Date)` for combined category/date filters.
- Use a stable primary key on `Id`; owner scope is applied in application/data
  queries so another owner's record is returned as not found.
