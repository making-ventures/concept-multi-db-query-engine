← [Back to README](./README.md)

# @mkven/multi-db-query — HTTP Contract Test Suite

This document defines the **full contract test suite** for any implementation of the `@mkven/multi-db-query` HTTP API. The contract verifies behavioral correctness through four endpoints:

- `POST /query` — accepts `{ definition, context }`, returns `QueryResult`
- `GET /health` — returns `HealthCheckResult`
- `POST /validate/query` — accepts `{ definition, context }`, returns `{ valid: true }` or throws `ValidationError` (400)
- `POST /validate/config` — accepts `{ metadata, roles }`, returns `{ valid: true }` or throws `ConfigError` (400)

The validation endpoints require **no database connections** — they run pure validation logic only. This means all validation tests (~100 tests in sections 12 and 17) can run without a live database, enabling fast feedback during implementation.

Any server (TypeScript, Go, Rust, Java, etc.) that wraps a multi-db query engine must pass all tests described here to be considered a conforming implementation.

---

## How It Works

### QueryContract Interface

Every implementation must satisfy this interface:

```ts
interface QueryContract {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>
}
```

The `@mkven/multi-db-contract` package exports 7 parameterized test suites: `describeQueryContract()`, `describeValidationContract()`, `describeErrorContract()`, `describeEdgeCaseContract()`, `describeHealthLifecycleContract()`, `describeInjectionContract()`, and `describeExecutorContract()`. Each implementation provides a factory function:

```ts
import { describeQueryContract } from '@mkven/multi-db-contract'

// In-process (TypeScript reference)
describeQueryContract('direct', async () => {
  const multiDb = await createMultiDb({ ... })
  return multiDb
})

// HTTP client (any implementation)
describeQueryContract('http-client', async () => {
  const client = createMultiDbClient({ baseUrl: 'http://localhost:3000' })
  return client
})
```

### ValidationContract Interface

Validation endpoints have a separate contract:

```ts
interface ValidationContract {
  validateQuery(input: ValidateQueryInput): Promise<ValidateResult>
  validateConfig(input: ValidateConfigInput): Promise<ValidateResult>
}
```

These tests require zero database I/O. `describeValidationContract()` accepts the contract, plus the fixture metadata and roles:

```ts
import { describeValidationContract } from '@mkven/multi-db-contract'
describeValidationContract(
  'direct',
  async () => { /* return ValidationContract impl */ },
  metadata,
  roles,
)
```

### Test Fixture Requirements

The server under test **must** be configured with the metadata, roles, and seed data described in the [Fixture](#fixture) section before running the suite. The fixture is deterministic and self-contained — no external dependencies.

The private `@mkven/multi-db-contract-tests` package wires these suites to real executors (Postgres, ClickHouse, Trino) for integration testing.

### Assertions

Tests assert on:
1. **Response shape** — `kind` discriminant, required fields
2. **Meta structure** — `meta.columns`, `meta.strategy`, `meta.timing`, `meta.tablesUsed`
3. **Error types** — correct error class (`ValidationError`, `PlannerError`, `ExecutionError`), correct `code`, `errors[]` contents
4. **Data correctness** — row counts, column values, masking applied
5. **HTTP status codes** — `400`, `422`, `500`, `503` for errors

---

## Fixture

### Databases

| id | engine | trinoCatalog |
|---|---|---|
| `pg-main` | postgres | `pg_main` |
| `ch-analytics` | clickhouse | `ch_analytics` |

Trino: `{ enabled: true }`

### Tables

#### orders (pg-main)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | int | false | — |
| customerId | customer_id | uuid | false | uuid |
| productId | product_id | uuid | true | — |
| total | total_amount | decimal | false | number |
| discount | discount | decimal | true | — |
| status | order_status | string | false | — |
| internalNote | internal_note | string | true | full |
| createdAt | created_at | timestamp | false | date |
| quantity | quantity | int | false | — |
| isPaid | is_paid | boolean | true | — |
| priorities | priorities | int[] | true | — |

- Primary key: `[id]`
- Relations:
  - `customerId → users.id` (many-to-one)
  - `productId → products.id` (many-to-one)

#### products (pg-main)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | uuid | false | — |
| name | name | string | false | — |
| category | category | string | false | — |
| price | price | decimal | false | number |
| labels | labels | string[] | true | — |

- Primary key: `[id]`
- Relations: none

#### users (pg-main)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | uuid | false | — |
| email | email | string | false | email |
| phone | phone | string | true | phone |
| firstName | first_name | string | false | name |
| lastName | last_name | string | false | name |
| role | role | string | false | — |
| age | age | int | true | — |
| managerId | manager_id | uuid | true | — |
| createdAt | created_at | timestamp | false | — |

- Primary key: `[id]`
- Relations:
  - `managerId → users.id` (many-to-one, self-referencing)

#### invoices (pg-main)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | uuid | false | — |
| orderId | order_id | int | true | — |
| amount | amount | decimal | false | number |
| status | status | string | false | — |
| issuedAt | issued_at | timestamp | false | — |
| paidAt | paid_at | timestamp | true | — |
| dueDate | due_date | date | true | — |

- Primary key: `[id]`
- Relations:
  - `orderId → orders.id` (many-to-one)

#### events (ch-analytics)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | uuid | false | — |
| type | event_type | string | false | — |
| userId | user_id | uuid | false | — |
| orderId | order_id | int | true | — |
| payload | payload | string | true | full |
| tags | tags | string[] | true | — |
| timestamp | event_ts | timestamp | false | — |

- Primary key: `[id]`
- Relations:
  - `userId → users.id` (many-to-one)
  - `orderId → orders.id` (many-to-one)

#### orderItems (pg-main)

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| orderId | order_id | int | false | — |
| productId | product_id | uuid | false | — |
| quantity | quantity | int | false | — |
| unitPrice | unit_price | decimal | false | — |

- Primary key: `[orderId, productId]` *(composite)*
- Relations:
  - `orderId → orders.id` (many-to-one)
  - `productId → products.id` (many-to-one)

#### samples (pg-main) / chSamples (ch-analytics)

Mirror tables for dialect-parameterized testing. **Identical schema and seed data** exist in both pg-main (`samples`) and ch-analytics (`chSamples`), enabling the same test to run against PostgreSQL, ClickHouse, and Trino with identical assertions (see [Parameterization](#parameterization)).

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | int | false | — |
| name | name | string | false | — |
| email | email | string | false | — |
| category | category | string | false | — |
| amount | amount | decimal | false | — |
| discount | discount | decimal | true | — |
| status | status | string | false | — |
| tags | tags | string[] | true | — |
| scores | scores | int[] | true | — |
| isActive | is_active | boolean | true | — |
| note | note | string | true | — |
| createdAt | created_at | timestamp | false | — |
| dueDate | due_date | date | true | — |
| externalId | external_id | uuid | false | — |
| managerId | manager_id | int | true | — |

- Primary key: `[id]`
- Relations (samples): `managerId → samples.id` (self-referencing, nullable)
- Relations (chSamples): `id → samples.id` (cross-DB mirror — enables Trino routing), `managerId → chSamples.id` (self-referencing, nullable)

#### sampleItems (pg-main) / chSampleItems (ch-analytics)

Child table for `samples`/`chSamples`. Identical schema and seed data in both databases.

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | int | false | — |
| sampleId | sample_id | int | false | — |
| label | label | string | false | — |
| category | category | string | false | — |
| amount | amount | decimal | false | — |
| quantity | quantity | int | false | — |
| status | status | string | false | — |

- Primary key: `[id]`
- Relations (sampleItems): `sampleId → samples.id`
- Relations (chSampleItems): `sampleId → chSamples.id`

#### sampleDetails (pg-main) / chSampleDetails (ch-analytics)

Grandchild table for `sampleItems`/`chSampleItems`. Enables 3-table join and nested EXISTS tests.

| apiName | physicalName | type | nullable | maskingFn |
|---|---|---|---|---|
| id | id | int | false | — |
| sampleItemId | sample_item_id | int | false | — |
| info | info | string | true | — |

- Primary key: `[id]`
- Relations (sampleDetails): `sampleItemId → sampleItems.id`
- Relations (chSampleDetails): `sampleItemId → chSampleItems.id`

### External Syncs (Debezium)

| Source | Target DB | Target Physical | Lag |
|---|---|---|---|
| orders | ch-analytics | default.orders_replica | seconds |

### Caches

| id | engine | tables |
|---|---|---|
| `redis-main` | redis | `users` (keyPattern: `users:{id}`, all columns) |

The `redis-main` cache provider enables `byIds` queries on `users` to be served from cache (strategy `'cache'`). Health check tests (C1300) verify the `cacheProviders` key includes `redis-main`.

### Roles

| Role ID | Tables |
|---|---|
| `admin` | `'*'` (all tables, all columns, no masking) |
| `tenant-user` | orders: `[id, total, status, createdAt]`, maskedColumns: `[total]`; users: `[id, firstName, lastName, email]`, maskedColumns: `[email]`; products: `[id, name, category, price]` |
| `analyst` | orders: `[id, total, status, internalNote, createdAt, customerId]`, maskedColumns: `[internalNote, createdAt, customerId]`; users: `[id, firstName, lastName, email, phone]`, maskedColumns: `[phone, firstName, lastName]`; products: `[id, name, category, price]`, maskedColumns: `[price]`; invoices: `[id, orderId, amount, status]`, maskedColumns: `[amount]` |
| `viewer` | orders: `[id, status, createdAt, quantity]`; users: `[id, firstName]` |
| `no-access` | `[]` (empty — zero permissions) |
| `orders-service` | orders: `'*'`; products: `'*'`; users: `[id, firstName, lastName]` |
| `reporting-service` | orders: `[id, total, status, createdAt]`, maskedColumns: `[total]`; products: `'*'` |

### Seed Data

The implementation must populate tables with deterministic data so assertions on row counts and values are reliable.

**orders** (minimum 5 rows):

| id | customerId | productId | total | discount | status | internalNote | createdAt | quantity | isPaid | priorities |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | uuid-c1 | uuid-p1 | 100.00 | 10.00 | active | internal-1 | 2024-01-15T10:00:00Z | 2 | true | [1, 2] |
| 2 | uuid-c2 | uuid-p2 | 200.00 | null | paid | null | 2024-02-20T14:30:00Z | 5 | true | [3] |
| 3 | uuid-c1 | uuid-p1 | 50.00 | 5.00 | cancelled | internal-3 | 2024-03-10T08:15:00Z | 1 | false | null |
| 4 | uuid-c3 | null | 300.00 | null | active | null | 2024-04-05T16:45:00Z | 10 | null | [] |
| 5 | uuid-c2 | uuid-p3 | 150.00 | 0.00 | shipped | internal-5 | 2024-05-12T12:00:00Z | 3 | true | [1, 2, 3] |

**products** (minimum 3 rows):

| id | name | category | price | labels |
|---|---|---|---|---|
| uuid-p1 | Widget A | electronics | 25.00 | ["sale", "new"] |
| uuid-p2 | Widget B | clothing | 40.00 | ["clearance"] |
| uuid-p3 | Widget C | electronics | 15.00 | null |

**users** (minimum 3 rows):

| id | email | phone | firstName | lastName | role | age | managerId | createdAt |
|---|---|---|---|---|---|---|---|---|
| uuid-c1 | alice@example.com | +1234567890 | Alice | Smith | admin | 30 | null | 2023-01-01T00:00:00Z |
| uuid-c2 | bob@example.com | null | Bob | Jones | viewer | 25 | uuid-c1 | 2023-06-15T00:00:00Z |
| uuid-c3 | carol@example.com | +9876543210 | Carol | Williams | viewer | null | uuid-c1 | 2024-01-01T00:00:00Z |

**invoices** (minimum 3 rows):

| id | orderId | amount | status | issuedAt | paidAt | dueDate |
|---|---|---|---|---|---|---|
| uuid-i1 | 1 | 100.00 | paid | 2024-01-20T00:00:00Z | 2024-01-25T00:00:00Z | 2024-02-20 |
| uuid-i2 | 2 | 200.00 | pending | 2024-02-25T00:00:00Z | null | 2024-03-25 |
| uuid-i3 | 1 | 50.00 | paid | 2024-01-22T00:00:00Z | 2024-01-28T00:00:00Z | null |

**events** (ch-analytics, minimum 3 rows):

| id | type | userId | orderId | payload | tags | timestamp |
|---|---|---|---|---|---|---|
| uuid-e1 | purchase | uuid-c1 | 1 | {"action":"buy"} | ["urgent", "vip"] | 2024-01-15T10:05:00Z |
| uuid-e2 | view | uuid-c2 | null | null | null | 2024-02-20T14:00:00Z |
| uuid-e3 | purchase | uuid-c1 | 3 | {"action":"buy"} | ["urgent"] | 2024-03-10T08:20:00Z |

**orderItems** (pg-main, minimum 4 rows):

| orderId | productId | quantity | unitPrice |
|---|---|---|---|
| 1 | uuid-p1 | 2 | 25.00 |
| 1 | uuid-p2 | 1 | 40.00 |
| 2 | uuid-p2 | 5 | 40.00 |
| 5 | uuid-p3 | 3 | 15.00 |

**samples** (pg-main, minimum 5 rows) and **chSamples** (ch-analytics, **identical** data):

| id | name | email | category | amount | discount | status | tags | scores | isActive | note | createdAt | dueDate | externalId | managerId |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Alpha | alpha@test.com | electronics | 100.00 | 10.00 | active | ["fast","new"] | [1,2] | true | note-1 | 2024-01-15T10:00:00Z | 2024-02-20 | uuid-s1 | null |
| 2 | Beta | beta@test.com | clothing | 200.00 | null | paid | ["slow"] | [3] | true | null | 2024-02-20T14:30:00Z | 2024-03-25 | uuid-s2 | 1 |
| 3 | Gamma | gamma@test.com | electronics | 50.00 | 5.00 | cancelled | ["fast"] | null | false | note-3 | 2024-03-10T08:15:00Z | null | uuid-s3 | 1 |
| 4 | Delta | delta@test.com | food | 300.00 | null | active | null | [] | null | null | 2024-04-05T16:45:00Z | 2024-05-01 | uuid-s4 | null |
| 5 | Epsilon | epsilon@test.com | electronics | 150.00 | 0.00 | shipped | ["fast","slow","new"] | [1,2,3] | true | note-5 | 2024-05-12T12:00:00Z | 2024-06-15 | uuid-s5 | 2 |

**sampleItems** (pg-main, minimum 6 rows) and **chSampleItems** (ch-analytics, **identical** data):

| id | sampleId | label | category | amount | quantity | status |
|---|---|---|---|---|---|---|
| 1 | 1 | item-A | electronics | 25.00 | 2 | active |
| 2 | 1 | item-B | clothing | 120.00 | 1 | active |
| 3 | 2 | item-C | clothing | 40.00 | 5 | paid |
| 4 | 3 | item-D | electronics | 60.00 | 3 | cancelled |
| 5 | 5 | item-E | food | 10.00 | 1 | active |
| 6 | 5 | item-F | electronics | 20.00 | 2 | paid |

> Note: sample 4 (Delta) has **no** items — used for NOT EXISTS tests.

**sampleDetails** (pg-main, minimum 4 rows) and **chSampleDetails** (ch-analytics, **identical** data):

| id | sampleItemId | info |
|---|---|---|
| 1 | 1 | detail-1 |
| 2 | 2 | null |
| 3 | 3 | detail-3 |
| 4 | 5 | detail-4 |

> Note: sampleItems 4 and 6 have **no** details — used for nested EXISTS tests.

---

## Test Categories

Tests are organized into categories. Each test has a unique ID for traceability. Tests marked with **(negative)** expect errors.

> **Default role:** Unless stated otherwise, all positive tests use the `admin` role (full access, no masking). Tests that exercise ACL or masking explicitly specify the role.

---

## 1. Execute Modes

### 1.1 Data Mode (default)

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C001 | Default execute mode returns data | `{ from: 'orders', columns: ['id', 'status'] }`, admin | `kind === 'data'`; `data` is array; `data.length >= 1`; each row has `id` and `status` keys |
| C002 | Data result includes correct meta.columns | `{ from: 'orders', columns: ['id', 'status'] }`, admin | `meta.columns` has 2 entries; each has `apiName`, `type`, `nullable`, `fromTable`, `masked` (all `false` for admin) |
| C003 | Data result includes meta.timing | same as C001 | `meta.timing.planningMs >= 0`; `meta.timing.generationMs >= 0`; `meta.timing.executionMs >= 0` |
| C004 | Data result includes meta.strategy | same as C001 | `meta.strategy` is one of `'direct'`, `'cache'`, `'materialized'`, `'trino-cross-db'` |
| C005 | Data result includes meta.tablesUsed | same as C001 | `meta.tablesUsed` is array with at least 1 entry; each entry has `tableId`, `source`, `database`, `physicalName` |
| C006 | Omitting columns returns all allowed | `{ from: 'orders' }`, admin | `meta.columns.length` matches total orders column count; all column apiNames present |
| C007 | No debugLog when debug is omitted | same as C001 | `result.debugLog` is `undefined` |

### 1.2 SQL-Only Mode

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C010 | sql-only returns SqlResult | `{ from: 'orders', columns: ['id'], executeMode: 'sql-only' }`, admin | `kind === 'sql'`; `sql` contains `'SELECT'`; `params` is array |
| C011 | sql-only has no data field | same as C010 | `result.data` is `undefined` |
| C012 | sql-only includes meta.columns | same as C010 | `meta.columns` has 1 entry with `apiName: 'id'` |
| C013 | sql-only has no executionMs | same as C010 | `meta.timing.executionMs` is `undefined`; `planningMs` and `generationMs` are present |
| C014 | sql-only with filters produces parameterized SQL | `{ from: 'orders', columns: ['id'], filters: [{ column: 'status', operator: '=', value: 'active' }], executeMode: 'sql-only' }`, admin | `params.length >= 1`; `sql` contains parameter placeholder |
| C015 | sql-only masking reported in meta | `{ from: 'orders', columns: ['id', 'total'], executeMode: 'sql-only' }`, tenant-user | `meta.columns.find(c => c.apiName === 'total').masked === true` — masking intent reported even without execution |
| C016 | sql-only with join | `{ from: 'orders', joins: [{ table: 'products' }], columns: ['id'], executeMode: 'sql-only' }`, admin | `kind === 'sql'`; `sql` contains `'JOIN'`; `meta.tablesUsed.length === 2` |

### 1.3 Count Mode

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C020 | count mode returns CountResult | `{ from: 'orders', executeMode: 'count' }`, admin | `kind === 'count'`; `typeof count === 'number'`; `count >= 5` (seed data) |
| C021 | count mode has empty meta.columns | same as C020 | `meta.columns` is empty array |
| C022 | count with filter | `{ from: 'orders', filters: [{ column: 'status', operator: '=', value: 'active' }], executeMode: 'count' }`, admin | `kind === 'count'`; `count >= 2` (seed: orders 1,4 are active) |
| C023 | count ignores groupBy/aggregations | `{ from: 'orders', groupBy: [{ column: 'status' }], aggregations: [{ column: 'total', fn: 'sum', alias: 'totalSum' }], executeMode: 'count' }`, admin | `kind === 'count'`; returns scalar count (not grouped) |
| C024 | count ignores orderBy, limit, offset | `{ from: 'orders', orderBy: [{ column: 'id', direction: 'asc' }], limit: 2, offset: 1, executeMode: 'count' }`, admin | `kind === 'count'`; `count >= 5` (limit/offset not applied to count) |
| C025 | count with join | `{ from: 'orders', joins: [{ table: 'products' }], executeMode: 'count' }`, admin | `kind === 'count'`; count reflects joined result set |
| C026 | count with restricted role | `{ from: 'orders', executeMode: 'count' }`, tenant-user | `kind === 'count'`; ACL applies — restricted role can count rows on an allowed table |
| C027 | count with zero matching rows | `{ from: 'orders', filters: [{ column: 'status', operator: '=', value: 'nonexistent' }], executeMode: 'count' }` | `kind === 'count'`; `count === 0` |

---

## 2. Debug Mode

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C030 | debug: true includes debugLog | `{ from: 'orders', columns: ['id'], debug: true }`, admin | `result.debugLog` is array; `length > 0` |
| C031 | debugLog entries have required fields | same as C030 | each entry: `typeof timestamp === 'number'`; `typeof phase === 'string'`; `typeof message === 'string'` |
| C032 | debugLog covers pipeline phases | same as C030 | phases include at least `'validation'`, `'access-control'`, `'planning'`, `'name-resolution'`, `'sql-generation'` (7 total phases: + `'cache'`, `'execution'`) |
| C033 | debug works with sql-only | `{ from: 'orders', executeMode: 'sql-only', debug: true }`, admin | `kind === 'sql'`; `debugLog` array present |
| C034 | debug works with count | `{ from: 'orders', executeMode: 'count', debug: true }`, admin | `kind === 'count'`; `debugLog` array present |

---

## 3. Filtering

<a id="parameterization"></a>

> **Parameterization — all dialects:** Every test in sections 3–9 is executed **three times**, once per SQL dialect. Tests are written using pg-main table names for readability. The runner substitutes tables per variant:
>
> | Variant | `samples` → | `sampleItems` → | `sampleDetails` → | Routing trigger | Engine |
> |---|---|---|---|---|---|
> | **pg** | `samples` | `sampleItems` | `sampleDetails` | — | PostgreSQL |
> | **ch** | `chSamples` | `chSampleItems` | `chSampleDetails` | — | ClickHouse |
> | **trino** | `chSamples` | `chSampleItems` | `chSampleDetails` | transparent `JOIN samples ON chSamples.id = samples.id` added (pg-main table → cross-DB → Trino) | Trino |
>
> All mirror tables share **identical schema and seed data**, so expected results are the same across all three variants.
> The only exception is **C505** (composite PK rejection) which uses `orderItems` and runs once (pg-only).

### 3.1 Comparison Operators

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C100 | `=` filter | samples, `status = 'active'` | 2 rows (ids 1, 4) |
| C101 | `!=` filter | samples, `status != 'cancelled'` | 4 rows (ids 1, 2, 4, 5) |
| C102 | `>` filter | samples, `amount > 100` | 3 rows (ids 2, 4, 5) |
| C103 | `<` filter | samples, `amount < 200` | 3 rows (ids 1, 3, 5) |
| C104 | `>=` filter | samples, `amount >= 150` | 3 rows (ids 2, 4, 5) |
| C105 | `<=` filter | samples, `amount <= 100` | 2 rows (ids 1, 3) |
| C106 | `=` on boolean column | samples, `isActive = true` | 3 rows (ids 1, 2, 5) |
| C107 | `!=` on boolean column | samples, `isActive != true` | 2 rows (ids 3, 4 — `false` and `null`) |
| C108 | `=` on uuid column | samples, `externalId = 'uuid-s1'` | exactly 1 row (id 1) |

### 3.2 Pattern Operators (string)

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C110 | `like` filter | samples, `email like '%@test%'` | all 5 rows match |
| C111 | `notLike` filter | samples, `email notLike '%alpha%'` | 4 rows (not id 1) |
| C112 | `ilike` filter | samples, `email ilike '%TEST%'` | all 5 rows — case-insensitive match |
| C113 | `notIlike` filter | samples, `email notIlike '%ALPHA%'` | 4 rows (not id 1) |
| C114 | `contains` filter | samples, `email contains 'alpha'` | 1 row (id 1) |
| C115 | `icontains` filter | samples, `email icontains 'ALPHA'` | 1 row (id 1) |
| C116 | `notContains` filter | samples, `email notContains 'alpha'` | 4 rows |
| C117 | `notIcontains` filter | samples, `email notIcontains 'ALPHA'` | 4 rows |
| C118 | `startsWith` filter | samples, `name startsWith 'Al'` | 1 row (id 1, Alpha) |
| C119 | `istartsWith` filter | samples, `name istartsWith 'AL'` | 1 row (id 1) |
| C120 | `endsWith` filter | samples, `email endsWith '@test.com'` | all 5 rows |
| C121 | `iendsWith` filter | samples, `email iendsWith '@TEST.COM'` | all 5 rows |
| C122 | `contains` with wildcard escaping | samples, `name contains 'Al%ha'` | 0 rows — `%` is treated literally, not as wildcard |
| C123 | `contains` with underscore escaping | samples, `name contains 'Al_ha'` | 0 rows — `_` is treated literally |

### 3.3 Range Operators

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C130 | `between` filter (decimal) | samples, `amount between { from: 100, to: 200 }` | 3 rows (ids 1, 2, 5) |
| C131 | `notBetween` filter | samples, `amount notBetween { from: 100, to: 200 }` | 2 rows (ids 3, 4) |
| C132 | `between` on int | samples, `id between { from: 2, to: 4 }` | 3 rows (ids 2, 3, 4) |
| C133 | `between` on timestamp | samples, `createdAt between { from: '2024-01-01T00:00:00Z', to: '2024-03-31T23:59:59Z' }` | 3 rows (ids 1, 2, 3) |
| C134 | `between` on date | samples, `dueDate between { from: '2024-02-01', to: '2024-05-01' }` | 3 rows (ids 1, 2, 4) |
| C135 | `notBetween` on int | samples, `id notBetween { from: 2, to: 4 }` | 2 rows (ids 1, 5) |

### 3.4 Set Operators

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C140 | `in` filter | samples, `status in ['active', 'paid']` | 3 rows (ids 1, 2, 4) |
| C141 | `notIn` filter | samples, `status notIn ['cancelled']` | 4 rows |
| C142 | `in` on int column | samples, `id in [1, 3, 5]` | 3 rows |
| C143 | `in` on uuid column | samples, `externalId in ['uuid-s1', 'uuid-s2']` | 2 rows (ids 1, 2) |
| C144 | `in` on decimal column | samples, `amount in [100.00, 200.00]` | 2 rows (ids 1, 2) |

### 3.5 Null Operators

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C150 | `isNull` filter | samples, `discount isNull` | 2 rows (ids 2, 4) |
| C151 | `isNotNull` filter | samples, `discount isNotNull` | 3 rows (ids 1, 3, 5) |
| C152 | `isNull` on array column | samples, `tags isNull` | 1 row (id 4) |
| C153 | `isNotNull` on array column | samples, `tags isNotNull` | 4 rows (ids 1, 2, 3, 5) |

### 3.6 Levenshtein

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C160 | `levenshteinLte` filter | samples, `name levenshteinLte { text: 'Alphb', maxDistance: 2 }` | 1 row (id 1, 'Alpha', distance 1) |

### 3.7 Array Operators

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C170 | `arrayContains` (int[]) | samples, `scores arrayContains 1` | 2 rows (ids 1, 5) |
| C171 | `arrayContainsAll` (string[]) | samples, `tags arrayContainsAll ['fast', 'new']` | 2 rows (ids 1, 5) |
| C172 | `arrayContainsAny` (string[]) | samples, `tags arrayContainsAny ['slow', 'new']` | 3 rows (ids 1, 2, 5) |
| C173 | `arrayIsEmpty` | samples, `scores arrayIsEmpty` | 1 row (id 4) — has `[]` |
| C174 | `arrayIsNotEmpty` | samples, `scores arrayIsNotEmpty` | 3 rows (ids 1, 2, 5) |
| C175 | `arrayContainsAll` single element | samples, `tags arrayContainsAll ['fast']` | 3 rows (ids 1, 3, 5) |
| C176 | `arrayContains` on string[] | samples, `tags arrayContains 'fast'` | 3 rows (ids 1, 3, 5) |

### 3.8 Column-vs-Column Filters

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C180 | same-table column filter | samples, `amount > discount` (QueryColumnFilter) | 3 rows (ids 1, 3, 5 — rows with non-null discount where amount > discount) |
| C181 | cross-table column filter | samples JOIN sampleItems, `samples.amount > sampleItems.amount` (QueryColumnFilter) | rows where sample amount exceeds item amount |

### 3.9 Filter Groups (AND/OR/NOT)

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C190 | OR filter group | samples, `(status = 'active' OR status = 'paid')` | 3 rows (ids 1, 2, 4) |
| C191 | AND filter group | samples, `(status = 'active' AND amount > 100)` | 1 row (id 4) |
| C192 | NOT filter group | samples, `NOT (status = 'cancelled')` | 4 rows |
| C193 | Nested filter groups | samples, `(status = 'active' OR (amount > 100 AND isActive = true))` | 4 rows (ids 1, 2, 4, 5) — correct logical evaluation |
| C194 | Deeply nested (3 levels) | samples, `((status = 'active' AND amount > 50) OR (status = 'paid' AND NOT (amount < 100)))` | 3 rows (ids 1, 2, 4) — correct deep nesting |

### 3.10 Filter with Table Qualifier

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C195 | Top-level filter on joined column | samples JOIN sampleItems, filter: `{ column: 'category', table: 'sampleItems', operator: '=', value: 'electronics' }` | returned items are electronics |
| C196 | Explicit from-table reference | samples, filter: `{ column: 'status', table: 'samples', operator: '=', value: 'active' }` | same as omitting `table` |

---

## 4. Joins

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C200 | LEFT JOIN (default) | samples JOIN sampleItems | rows include sampleItem columns (null for sample 4 which has no items) |
| C201 | INNER JOIN | samples JOIN sampleItems `type: 'inner'` | only samples that have items (no sample 4) |
| C202 | Multi-table join (3 tables, transitive) | samples JOIN sampleItems JOIN sampleDetails | sampleDetails has relation to sampleItems (already joined), not to samples — transitive resolution; columns from all 3 tables present |
| C203 | Join with column selection | samples JOIN sampleItems `columns: ['label']` | only `label` from sampleItems in result |
| C204 | Join with `columns: []` | samples, JOIN sampleItems `columns: []`, groupBy sampleItems.category | join used for groupBy only — no sampleItem columns in SELECT |
| C205 | Join-scoped filter | samples JOIN sampleItems, `sampleItems.filters: [{ column: 'category', operator: '=', value: 'electronics' }]` | only electronics items matched |
| C206 | Column collision on join | samples JOIN sampleItems (both have `id`, `category`, `amount`, `status`) | result keys are qualified: `samples.id`, `sampleItems.id`, etc.; `meta.columns[].apiName` reflects qualification |
| C207 | Join filter at top level vs QueryJoin.filters | same filter on sampleItems, placed in top-level `filters` with `table` qualifier vs `QueryJoin.filters` | identical results |

---

## 5. Aggregations

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C300 | COUNT(*) | samples, `aggregations: [{ column: '*', fn: 'count', alias: 'total' }]` | `kind === 'data'`; single row; `total >= 5` |
| C301 | SUM | samples GROUP BY status, `SUM(amount) as totalAmt` | grouped results with sum per status |
| C302 | AVG | samples, `AVG(amount) as avgAmt` | result type is decimal |
| C303 | MIN | samples, `MIN(createdAt) as earliest` | type preserved as timestamp |
| C304 | MAX | samples, `MAX(amount) as maxAmt` | correct max value (300.00) |
| C305 | COUNT(column) | samples, `COUNT(discount) as discountCount` | counts non-NULL discount values only (3: ids 1, 3, 5) |
| C306 | Multiple aggregations | samples GROUP BY status, `SUM(amount) as totalAmt, COUNT(*) as cnt` | both aggregation aliases in result |
| C307 | Aggregation on joined column | samples JOIN sampleItems, `SUM(sampleItems.amount) as totalItemAmt` | aggregation references joined table |
| C308 | Aggregation-only (`columns: []`) | samples `columns: []`, `SUM(amount) as totalAmt` | only aggregation alias in result — no regular columns |
| C309 | `columns: undefined` + aggregations + groupBy | samples, groupBy: [status], `SUM(amount) as totalAmt` (columns omitted) | result includes `status` (from groupBy) + `totalAmt`; omitted columns defers to groupBy columns only |
| C310 | SUM on nullable column | samples, `SUM(discount) as discountSum` | NULLs are skipped; result is sum of non-null discounts (10.00 + 5.00 + 0.00 = 15.00) |

---

## 6. GROUP BY & HAVING

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C320 | GROUP BY single column | samples, groupBy: [status], columns: [status] | 4 distinct statuses (active, paid, cancelled, shipped) |
| C321 | GROUP BY with multi-column | samples, groupBy: [status, isActive], columns: [status, isActive], COUNT(*) | grouped by both |
| C322 | HAVING single condition | samples GROUP BY status, SUM(amount) as totalAmt, `having: [{ column: 'totalAmt', operator: '>', value: 100 }]` | 3 groups: active (400), paid (200), shipped (150) |
| C323 | HAVING with OR group | samples GROUP BY status, HAVING `(SUM(amount) > 250 OR AVG(amount) > 150)` | 2 groups: active (SUM 400 > 250) and paid (AVG 200 > 150) |
| C324 | HAVING with BETWEEN | samples GROUP BY status, SUM(amount) as totalAmt, HAVING `totalAmt between { from: 100, to: 300 }` | 2 groups: paid (200) and shipped (150) |
| C325 | HAVING with NOT BETWEEN | samples GROUP BY status, SUM(amount) as totalAmt, HAVING `totalAmt notBetween { from: 100, to: 300 }` | 2 groups: active (400) and cancelled (50) |
| C326 | HAVING with IS NULL | samples GROUP BY status, SUM(discount) as discountSum, HAVING `discountSum isNull` | 1 group: paid (all discounts null → SUM is null) |
| C327 | NOT in HAVING group | samples GROUP BY status, HAVING `NOT (SUM(amount) > 100 OR COUNT(*) > 1)` | 1 group: cancelled (SUM 50, COUNT 1) |
| C328 | ORDER BY aggregation alias | samples GROUP BY status, SUM(amount) as totalAmt, orderBy totalAmt desc | results ordered by totalAmt descending |
| C329 | GROUP BY joined column | samples JOIN sampleItems, groupBy: [{ column: 'category', table: 'sampleItems' }], columns: [], COUNT(*) as cnt | grouped by item category; one row per category |

---

## 7. ORDER BY, LIMIT, OFFSET, DISTINCT

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C400 | ORDER BY single column asc | samples, orderBy: [{ column: 'amount', direction: 'asc' }] | rows in ascending amount order |
| C401 | ORDER BY single column desc | samples, orderBy: [{ column: 'amount', direction: 'desc' }] | rows in descending amount order |
| C402 | ORDER BY multiple columns | samples, orderBy: [status asc, amount desc] | multi-column ordering |
| C403 | ORDER BY joined column | samples JOIN sampleItems, orderBy: [{ column: 'category', table: 'sampleItems', direction: 'asc' }] | ordered by joined-table column |
| C404 | LIMIT | samples, limit: 2 | `data.length <= 2` |
| C405 | LIMIT + OFFSET | samples, limit: 2, offset: 2 | skips first 2 rows |
| C406 | DISTINCT | samples columns: [status], distinct: true | 4 unique status values only |
| C407 | DISTINCT + GROUP BY | samples, distinct: true, groupBy: [status], columns: [status], SUM(amount) | valid SQL — DISTINCT has no effect when GROUP BY present |

---

## 8. byIds

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C500 | byIds returns matching rows | samples, `byIds: [1, 2]` | exactly 2 rows; ids are 1 and 2 |
| C501 | byIds with non-existent IDs | samples, `byIds: [1, 999]` | returns only existing row (id=1) |
| C502 | byIds with count mode | samples, `byIds: [1, 2, 3]`, `executeMode: 'count'` | `kind === 'count'`; `count === 3` |
| C503 | byIds with join | samples, `byIds: [1, 2]`, join sampleItems | id + sampleItem data returned |
| C504 | byIds with column selection | samples, `byIds: [1]`, columns: [id, status] | only selected columns returned |
| C505 | byIds rejects composite PK **(negative, pg-only)** | orderItems, `byIds: [1, 2]`, admin | `ValidationError` with `INVALID_BY_IDS` — byIds requires a single-column primary key; **not parameterized** |
| C506 | byIds with filter | samples, `byIds: [1, 2, 3]`, `filters: [{ column: 'status', operator: '=', value: 'active' }]`, admin | returns only id=1 (active); id=2 is 'paid', id=3 is 'cancelled' |
| C507 | byIds with sql-only | samples, `byIds: [1, 2]`, `executeMode: 'sql-only'`, admin | `kind === 'sql'`; `sql` contains `'WHERE'`; `params` includes primary key values |

---

## 9. EXISTS / NOT EXISTS

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C600 | EXISTS filter | samples WHERE EXISTS sampleItems | only samples that have items (ids 1, 2, 3, 5) |
| C601 | NOT EXISTS filter | samples WHERE NOT EXISTS sampleItems (`exists: false`) | only samples without items (id 4) |
| C602 | EXISTS with subquery filter | samples WHERE EXISTS sampleItems(status = 'paid') | only samples with paid items (ids 2, 5) |
| C603 | EXISTS inside OR group | samples WHERE `(status = 'cancelled' OR EXISTS sampleItems)` | 4 rows (ids 1, 2, 3, 5 — all four have items; id 3 also matches via status; id 4 excluded: not cancelled and no items) |
| C604 | Nested EXISTS | samples WHERE EXISTS sampleItems WHERE EXISTS sampleDetails (2-hop chain) | 3 rows (ids 1, 2, 5 — samples whose items have details) |
| C605 | Counted EXISTS (>=) | samples WHERE EXISTS sampleItems `count: { operator: '>=', value: 2 }` | 2 rows (ids 1, 5 — each has 2 items) |
| C606 | Counted EXISTS (=) | samples WHERE EXISTS sampleItems `count: { operator: '=', value: 1 }` | 2 rows (ids 2, 3 — each has exactly 1 item) |
| C607 | Counted EXISTS ignores `exists` field | samples, `filters: [{ table: sampleItems, exists: false, count: { operator: '>=', value: 1 } }]` | `exists` is ignored when `count` present — counted subquery decides; 4 rows (ids 1,2,3,5) |
| C608 | Self-referencing EXISTS | samples WHERE EXISTS samples (via managerId → samples.id) | 2 rows (ids 1, 2 — they manage other samples) |
| C609 | EXISTS with join | samples JOIN sampleItems WHERE EXISTS samples (via managerId) | samples that manage others, with item data included |
| C610 | Counted EXISTS (>) | samples WHERE EXISTS sampleItems `count: { operator: '>', value: 1 }` | 2 rows (ids 1, 5 — each has > 1 items) |
| C611 | Counted EXISTS (<) | samples WHERE EXISTS sampleItems `count: { operator: '<', value: 2 }` | 3 rows (ids 2, 3, 4 — scalar subquery: COUNT 1, 1, 0 all < 2) |
| C612 | Counted EXISTS (!=) | samples WHERE EXISTS sampleItems `count: { operator: '!=', value: 0 }` | 4 rows (ids 1, 2, 3, 5 — have non-zero items) |
| C613 | Counted EXISTS (<=) | samples WHERE EXISTS sampleItems `count: { operator: '<=', value: 1 }` | 3 rows (ids 2, 3, 4 — 0 or 1 items) |

---

## 10. Access Control

### 10.1 Role-Based Permissions

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C700 | Admin sees all columns | orders (admin) | `meta.columns.length` equals total orders columns |
| C701 | Restricted role sees subset | orders columns: [id, total, status] (tenant-user) | `meta.columns.length === 3`; only allowed columns |
| C702 | Omitting columns uses role-allowed set | orders (tenant-user, no columns specified) | only `[id, total, status, createdAt]` returned |
| C703 | Access denied on table **(negative)** | events (tenant-user) | `ValidationError` with `ACCESS_DENIED` |
| C704 | Access denied on column **(negative)** | orders columns: [id, internalNote] (tenant-user) | `ValidationError` with `ACCESS_DENIED` |
| C705 | No-access role **(negative)** | orders (no-access) | `ValidationError` with `ACCESS_DENIED` |
| C706 | Empty roles array **(negative)** | orders, context: `{ roles: { user: [] } }` | `ValidationError` with `ACCESS_DENIED` (zero roles → zero permissions) |
| C707 | Access denied on joined table **(negative)** | orders JOIN events (tenant-user) | `ValidationError` with `ACCESS_DENIED` — tenant-user has no access to events table |

### 10.2 Multi-Role (Union Within Scope)

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C710 | Union of two user roles | orders, `user: ['tenant-user', 'admin']` | all columns visible (admin overrides) |
| C711 | Union adds permissions | orders, `user: ['tenant-user', 'viewer']` | effective columns = `[id, total, status, createdAt, quantity]` (tenant-user: id,total,status,createdAt + viewer: id,status,createdAt,quantity) |

### 10.3 Cross-Scope (Intersection)

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C720 | Admin user + service restriction | orders JOIN users, `user: ['admin'], service: ['orders-service']` | orders: all columns (admin ∩ orders-service `'*'` = all); users: restricted to `[id, firstName, lastName]` (admin ∩ orders-service) |
| C721 | Empty scope intersection **(negative)** | events, `user: ['tenant-user'], service: ['orders-service']` | tenant-user has no events access; orders-service has no events access → `ACCESS_DENIED` |
| C722 | Omitted scope = no restriction | orders, `user: ['admin']` (no service scope) | full admin access; service scope imposes no restriction |
| C723 | One scope with zero roles **(negative)** | orders, `user: [], service: ['orders-service']` | user scope has zero roles → zero permissions → `ACCESS_DENIED` regardless of service |
| C724 | Multi-scope disjoint columns **(negative)** | users, `user: [scope-a: id+firstName], service: [scope-b: email+age]` | intersection of disjoint column sets → empty → `ACCESS_DENIED` |
| C725 | Multi-scope wildcard ∩ specific | users, `user: ['admin'], service: [restricted: id+firstName]` | wildcard intersected with specific → restricted to id+firstName; requesting 'email' → `ACCESS_DENIED` |

---

## 11. Column Masking

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C800 | Masked column reported in meta | orders columns: [id, total] (tenant-user) | `meta.columns.find(c => c.apiName === 'total').masked === true`; id: `false` |
| C801 | Admin sees unmasked | orders columns: [id, total] (admin) | `meta.columns.find(c => c.apiName === 'total').masked === false` |
| C802 | Masked value is obfuscated (number) | orders columns: [total] (tenant-user) | `data[0].total === 0` (number masking → replace with 0) |
| C803 | Masked value (full) | orders columns: [id, internalNote] (analyst) | `data[0].internalNote === '***'` (analyst has internalNote with maskingFn `full`) |
| C804 | Masking on email column | users columns: [email] (tenant-user) | email is masked: first char + domain hint (e.g. `a***@***.com`) |
| C805 | Aggregation alias never masked | orders (tenant-user), GROUP BY status, SUM(total) as totalSum | `meta.columns.find(c => c.apiName === 'totalSum').masked === false` — aggregation aliases are never masked |
| C806 | sql-only reports masking intent | orders columns: [id, total], executeMode: 'sql-only' (tenant-user) | `meta.columns.find(c => c.apiName === 'total').masked === true` — no data, but intent reported |
| C807 | Multi-role masking (union unmasks) | orders, `user: ['tenant-user', 'admin']` | admin provides unmasked access → `total.masked === false` (union within scope: most permissive wins) |
| C808 | Cross-scope masking preserved | orders columns: [id, total], `user: ['admin'], service: ['reporting-service']` | `total.masked === true` (admin unmasks, reporting-service masks total → intersection: stays masked) |
| C809 | Masked value (phone) | users columns: [id, phone] (analyst) | phone is masked: e.g. `+1***890` (phone masking → country code + `***` + last 3 digits) |
| C810 | Masked value (name) | users columns: [id, firstName, lastName] (analyst) | firstName/lastName masked: e.g. `A***e` / `S***h` (name masking → first char + stars + last char) |
| C811 | Masked value (number on price) | products columns: [id, price] (analyst) | `data[0].price === 0` (number masking on price) |
| C812 | Masked value (number on amount) | invoices columns: [id, amount] (analyst) | `data[0].amount === 0` (number masking on amount) |
| C813 | Multiple masking functions in one query | users columns: [id, email, phone, firstName] (analyst) | email: `false` (analyst has no email masking); phone: `true`; firstName: `true` — different functions on different columns |
| C814 | Masked value (date) | orders columns: [id, createdAt] (analyst) | `createdAt` is masked: e.g. `'2024-01-01'` (date masking → year + `-01-01`, no time component) |
| C815 | Masking on null value | orders columns: [id, internalNote] (analyst) | rows where `internalNote` is null (orders 2, 4) remain `null` — masking (`full`) is skipped for null/undefined values |
| C816 | Masked value (uuid) | orders columns: [id, customerId] (analyst) | `customerId` masked: e.g. `a1b2****` (first 4 hex chars + `****`) |

---

## 12. Validation Errors (negative tests)

All tests in this section expect the query to throw `ValidationError` with `code: 'VALIDATION_FAILED'`.

### 12.1 Table & Column

| ID | Test | Expected error code |
|---|---|---|
| C900 | Unknown table: `from: 'nonExistentTable'` | `UNKNOWN_TABLE` |
| C901 | Unknown column: `orders columns: ['nonexistent']` | `UNKNOWN_COLUMN` |
| C902 | Unknown column in filter: `orders filter: { column: 'nonexistent', ... }` | `UNKNOWN_COLUMN` |
| C903 | Unknown column on joined table: orders JOIN products, filter `products.nonexistent` | `UNKNOWN_COLUMN` |

### 12.2 Filter Validity

| ID | Test | Expected error code |
|---|---|---|
| C910 | `>` on uuid column | `INVALID_FILTER` |
| C911 | `>` on boolean column | `INVALID_FILTER` |
| C912 | `in` on boolean column | `INVALID_FILTER` |
| C913 | `in` on date column | `INVALID_FILTER` |
| C914 | `in` on timestamp column | `INVALID_FILTER` |
| C915 | `notIn` on date column | `INVALID_FILTER` |
| C916 | `notIn` on boolean column | `INVALID_FILTER` |
| C917 | `like` on int column | `INVALID_FILTER` |
| C918 | `contains` on decimal column | `INVALID_FILTER` |
| C919 | `levenshteinLte` on decimal column | `INVALID_FILTER` |
| C920 | `between` on boolean column | `INVALID_FILTER` |
| C921 | `between` on uuid column | `INVALID_FILTER` |
| C922 | `notBetween` on boolean column | `INVALID_FILTER` |
| C923 | `notBetween` on uuid column | `INVALID_FILTER` |
| C924 | `isNull` on non-nullable column | `INVALID_FILTER` |
| C925 | `isNotNull` on non-nullable column | `INVALID_FILTER` |
| C926 | `arrayContains` on scalar column | `INVALID_FILTER` |
| C927 | Scalar operator on array column (e.g. `= 'x'` on `string[]`) | `INVALID_FILTER` |
| C928 | Filter `table` references non-joined table | `INVALID_FILTER` |
| C929 | Filter on access-denied column (tenant-user filtering on `internalNote`) | `ACCESS_DENIED` |

### 12.3 Value Validity

| ID | Test | Expected error code |
|---|---|---|
| C930 | `between` missing `to` | `INVALID_VALUE` |
| C931 | `notBetween` missing `to` | `INVALID_VALUE` |
| C932 | `levenshteinLte` negative maxDistance | `INVALID_VALUE` |
| C933 | `levenshteinLte` fractional maxDistance | `INVALID_VALUE` |
| C934 | `in` with empty array | `INVALID_VALUE` |
| C935 | `in` with type-mismatched elements | `INVALID_VALUE` |
| C936 | `in` with null element | `INVALID_VALUE` |
| C937 | `between` with null `from` | `INVALID_VALUE` |
| C938 | `between` with null `to` | `INVALID_VALUE` |
| C939 | `between` with type-mismatched bounds | `INVALID_VALUE` |
| C940 | `arrayContains` type mismatch | `INVALID_VALUE` |
| C941 | `arrayContainsAll` empty array | `INVALID_VALUE` |
| C942 | `arrayContainsAny` type mismatch | `INVALID_VALUE` |
| C943 | `arrayContainsAll` with null element | `INVALID_VALUE` |
| C944 | `notIn` with empty array | `INVALID_VALUE` |
| C945 | `notIn` with type-mismatched elements | `INVALID_VALUE` |
| C946 | `between` missing `from` | `INVALID_VALUE` |
| C947 | `levenshteinLte` missing `text` field | `INVALID_VALUE` |

### 12.4 Column Filter Validity

| ID | Test | Expected error code |
|---|---|---|
| C950 | Column filter type mismatch (decimal > string) — cross-family: numeric vs string rejected | `INVALID_FILTER` |
| C954 | Column filter same numeric family (int > decimal) | no error — int and decimal are compatible |
| C955 | Column filter same temporal family (date > timestamp) | no error — date and timestamp are compatible |
| C951 | Column filter on denied column | `ACCESS_DENIED` |
| C952 | Column filter non-existent refColumn | `UNKNOWN_COLUMN` |
| C953 | Column filter on array column | `INVALID_FILTER` |

### 12.5 Join Validity

| ID | Test | Expected error code |
|---|---|---|
| C960 | Join with no relation defined | `INVALID_JOIN` |
| C961 | Join to table with no role access | `ACCESS_DENIED` |
| C962 | Transitive join with no path (3rd table unrelated to any joined table) | `INVALID_JOIN` |

### 12.6 GroupBy Validity

| ID | Test | Expected error code |
|---|---|---|
| C970 | Column in SELECT not in groupBy | `INVALID_GROUP_BY` |
| C971 | Array column in groupBy | `INVALID_GROUP_BY` |
| C972 | GroupBy `table` references non-joined table | `INVALID_GROUP_BY` |

### 12.7 Having Validity

| ID | Test | Expected error code |
|---|---|---|
| C975 | HAVING on non-existent alias | `INVALID_HAVING` |
| C976 | `table` qualifier in HAVING filter | `INVALID_HAVING` |
| C977 | QueryColumnFilter in HAVING group | `INVALID_HAVING` |
| C978 | EXISTS in HAVING | `INVALID_HAVING` |
| C979 | `contains` operator in HAVING | `INVALID_HAVING` |
| C980 | `levenshteinLte` in HAVING | `INVALID_HAVING` |
| C981 | `arrayContains` in HAVING | `INVALID_HAVING` |
| C982 | Top-level QueryColumnFilter in HAVING (no group wrapper) | `INVALID_HAVING` |
| C983 | Top-level QueryExistsFilter in HAVING (no group wrapper) | `INVALID_HAVING` |

### 12.8 OrderBy Validity

| ID | Test | Expected error code |
|---|---|---|
| C985 | orderBy on non-joined table column | `INVALID_ORDER_BY` |
| C986 | Array column in orderBy | `INVALID_ORDER_BY` |
| C987 | OrderBy `table` references non-joined table | `INVALID_ORDER_BY` |

### 12.9 byIds Validity

| ID | Test | Expected error code |
|---|---|---|
| C990 | Empty byIds array | `INVALID_BY_IDS` |
| C991 | byIds + aggregations | `INVALID_BY_IDS` |
| C992 | byIds scalar on composite PK: orderItems, `byIds: [1, 2]` (scalar values) | `INVALID_BY_IDS` |
| C994 | byIds + groupBy | `INVALID_BY_IDS` |

### 12.10 Limit/Offset Validity

| ID | Test | Expected error code |
|---|---|---|
| C995 | Negative limit | `INVALID_LIMIT` |
| C996 | Offset without limit | `INVALID_LIMIT` |
| C997 | Negative offset | `INVALID_LIMIT` |
| C998 | Fractional limit (e.g., `2.5`) | `INVALID_LIMIT` |

### 12.11 Aggregation Validity

| ID | Test | Expected error code |
|---|---|---|
| C1000 | Duplicate aggregation alias | `INVALID_AGGREGATION` |
| C1001 | Alias collides with column apiName | `INVALID_AGGREGATION` |
| C1002 | Empty columns `[]` without aggregations | `INVALID_AGGREGATION` |
| C1003 | SUM on array column | `INVALID_AGGREGATION` |
| C1004 | Aggregation `table` references non-joined table | `INVALID_AGGREGATION` |
| C1005 | Aggregation column doesn't exist | `UNKNOWN_COLUMN` |

### 12.12 EXISTS Validity

| ID | Test | Expected error code |
|---|---|---|
| C1010 | EXISTS on unrelated table | `INVALID_EXISTS` |
| C1011 | Counted EXISTS with negative count value | `INVALID_EXISTS` |
| C1012 | Counted EXISTS with fractional count value | `INVALID_EXISTS` |
| C1013 | Nested EXISTS invalid inner relation | `INVALID_EXISTS` |

### 12.13 Role Validity

| ID | Test | Expected error code |
|---|---|---|
| C1020 | Unknown role ID | `UNKNOWN_ROLE` |

### 12.14 Multi-Error Collection

| ID | Test | Assertions |
|---|---|---|
| C1030 | Multiple errors collected: `from: 'nonExistentTable'`, column: `'bad'`, filter on `'missing'` | `errors[]` contains multiple entries; all issues reported at once |

---

## 13. Query Result Meta Verification

Detailed assertions on `QueryResultMeta` structure beyond basic checks.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1100 | meta.columns type correctness | orders columns: [id, total, status] | id: `type: 'int'`, total: `type: 'decimal'`, status: `type: 'string'` |
| C1101 | meta.columns nullable correctness | orders columns: [id, productId] | id: `nullable: false`, productId: `nullable: true` |
| C1102 | meta.columns fromTable | orders JOIN products, columns from both | orders columns: `fromTable: 'orders'`; products columns: `fromTable: 'products'` |
| C1103 | meta.columns for aggregations | orders SUM(total) as totalSum, COUNT(*) as cnt | totalSum: `type: 'decimal'`, `fromTable: 'orders'`, `masked: false`; cnt: `type: 'int'` |
| C1104 | AVG always returns decimal | orders AVG(quantity) as avgQty (source: int) | `meta.columns.find(c => c.apiName === 'avgQty').type === 'decimal'` |
| C1105 | meta.tablesUsed for single table | orders | 1 entry: `tableId: 'orders'`, `source: 'original'`, `database: 'pg-main'` |
| C1106 | meta.tablesUsed for join | orders JOIN products | 2 entries |
| C1107 | meta.columns for count mode | orders count | `meta.columns` is empty array |
| C1108 | meta.dialect present (data mode) | orders | `meta.dialect` is one of `'postgres'`, `'clickhouse'`, `'trino'` |
| C1109 | meta.targetDatabase for direct query | orders (admin) | `meta.targetDatabase === 'pg-main'` |
| C1110 | meta.targetDatabase for cross-DB query | events JOIN users (admin) | `meta.targetDatabase` reflects Trino (the executor used) |
| C1111 | meta.dialect in sql-only mode | orders, `executeMode: 'sql-only'` | `meta.dialect` is present (same as data mode) |
| C1112 | meta.dialect in count mode | orders, `executeMode: 'count'` | `meta.dialect` is present |
| C1113 | Aggregation nullable inference | orders, SUM(discount) as discountSum | `meta.columns.find(c => c.apiName === 'discountSum').nullable === true` — nullable source column produces nullable aggregation |

---

## 14. Error Deserialization (HTTP-specific)

These tests verify that errors transmitted over HTTP are properly reconstructed as typed error instances on the client side.

| ID | Test | Assertions |
|---|---|---|
| C1200 | ValidationError via HTTP | server returns 400 with ValidationError body | client throws `ValidationError`; `instanceof ValidationError === true`; `code === 'VALIDATION_FAILED'`; `errors[]` array present with individual issues |
| C1201 | ValidationError preserves fromTable | same as C1200 | `error.fromTable` matches the query's `from` table |
| C1202 | PlannerError via HTTP | server returns 422 with PlannerError body | client throws `PlannerError`; `code` is correct (e.g. `UNREACHABLE_TABLES`) |
| C1203 | ExecutionError via HTTP | server returns 500 | client throws `ExecutionError`; `code` is correct (e.g. `QUERY_FAILED`) |
| C1204 | ConnectionError on network failure | server unreachable | `ConnectionError` with `code: 'NETWORK_ERROR'` |
| C1205 | ConnectionError on timeout | slow server, `timeout: 100ms` in client | `ConnectionError` with `code: 'REQUEST_TIMEOUT'` |
| C1206 | ProviderError via HTTP | server returns 503 with ProviderError body | client throws `ProviderError`; `code: 'METADATA_LOAD_FAILED'` or `'ROLE_LOAD_FAILED'` |

---

## 14b. Planner Errors

These tests exercise `PlannerError` (HTTP 422) scenarios where a valid query cannot be executed due to infrastructure constraints.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1250 | Cross-DB join with Trino disabled | events JOIN users (ch-analytics + pg-main), Trino config: `{ enabled: false }` | `PlannerError` with `code: 'TRINO_DISABLED'`, HTTP 422 |
| C1251 | Cross-DB join, DB missing trinoCatalog | events JOIN users, `ch-analytics` has no `trinoCatalog` configured | `PlannerError` with `code: 'NO_CATALOG'`, HTTP 422 |
| C1252 | Cross-DB tables, no sync, no trino | events JOIN users, Trino disabled, no Debezium sync between DBs | `PlannerError` with `code: 'UNREACHABLE_TABLES'`, HTTP 422 |
| C1253 | Freshness conflict with replica lag | orders (admin), `freshness: 'realtime'`, **pg-main executor removed** (only materialized replica available), orders_replica has `estimatedLag: 'seconds'` | `PlannerError` with `code: 'FRESHNESS_UNMET'`, HTTP 422 — `'realtime'` rejects materialized replicas and no original source is available |
| C1254 | Freshness `'seconds'` accepts `'seconds'` lag | orders (admin), `freshness: 'seconds'` | `meta.strategy === 'materialized'` — replica lag matches required freshness |

---

## 14c. Execution Errors

These tests exercise `ExecutionError` (HTTP 500) scenarios where SQL generation succeeds but execution fails.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1260 | Missing executor | query targets a database with no configured executor | `ExecutionError` with `code: 'EXECUTOR_MISSING'`, HTTP 500 |
| C1261 | Missing cache provider | `byIds` query on cached table, cache provider not configured | `ExecutionError` with `code: 'CACHE_PROVIDER_MISSING'`, HTTP 500 |
| C1262 | Query execution failure | query against a database that returns an error (e.g., table dropped) | `ExecutionError` with `code: 'QUERY_FAILED'`, HTTP 500 |
| C1263 | Query timeout | query against slow database, executor timeout exceeded | `ExecutionError` with `code: 'QUERY_TIMEOUT'`, HTTP 500 |

---

## 14d. Provider Errors

These tests exercise `ProviderError` (HTTP 503) scenarios where metadata or role loading fails.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1270 | Metadata provider failure | metadata provider throws during load | `ProviderError` with `code: 'METADATA_LOAD_FAILED'`, HTTP 503 |
| C1271 | Role provider failure | role provider throws during load | `ProviderError` with `code: 'ROLE_LOAD_FAILED'`, HTTP 503 |

---

## 15. Health Check

| ID | Test | Assertions |
|---|---|---|
| C1300 | Healthy server | `GET /health` → `{ healthy: true, executors: {...}, cacheProviders: {...} }` |
| C1301 | Executor keys present | `executors` contains keys matching configured database IDs |
| C1302 | Each executor has required fields | `healthy: boolean`, `latencyMs: number` |
| C1303 | Unhealthy executor | stop one DB → `healthy: false` at top level; failed executor has `healthy: false` and `error` string |
| C1304 | Cache provider in health check | `GET /health` | `cacheProviders` contains `redis-main` with `healthy: boolean`, `latencyMs: number` |

---

## 15b. Lifecycle

These tests verify runtime lifecycle operations. They require the ability to reconfigure the server at test time.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1310 | Reload metadata makes new table visible | add table to metadata provider, call `reloadMetadata()` | query against new table succeeds |
| C1311 | Reload metadata failure preserves old config | `reloadMetadata()` with broken provider | old queries still work; new load is rejected |
| C1312 | Reload roles updates permissions | add new role via provider, call `reloadRoles()` | query with new role succeeds |
| C1313 | Close prevents further queries | call `close()`, then query | `ExecutionError` with `code: 'EXECUTOR_MISSING'` |

---

## 16. SQL Injection Resistance

These tests verify that user-provided inputs — filter values, column/table names, aggregation aliases, enum-like keywords, and subquery references — cannot inject SQL across **all three dialects** (PostgreSQL, ClickHouse, Trino). Each test sends a malicious value and asserts the query either succeeds safely (value treated as data, not code) or is properly rejected with a validation error.

### 16.1 Identifier & Structural Injection (dialect-agnostic)

Column names, table names, EXISTS references, and **enum-like keyword fields** (ORDER BY direction, aggregation function name, column-filter operator, filter group logic, EXISTS count operator) are validated **before** reaching any dialect's SQL generator. These tests can run via `POST /validate/query` (no database connection needed) — the validation layer rejects them identically regardless of which dialect would have been used.

Enum-like fields (`direction`, `fn`, `operator`, `logic`) are constrained by TypeScript types at compile time, but conforming implementations **must** also validate them at runtime — raw JSON payloads bypass type constraints, and these fields are interpolated directly into SQL (not parameterized). A malicious HTTP client sending `{ direction: "asc; DROP TABLE orders;--" }` would inject SQL if the value is not validated.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1404 | Column name injection (`"` payload) | `POST /validate/query`: orders, `columns: ['id"; DROP TABLE orders; --']` | `UNKNOWN_COLUMN` validation error (rejected before SQL generation) |
| C1418 | Column name injection (`` ` `` payload) | `POST /validate/query`: events, `columns: ['id`; DROP TABLE events; --']` | `UNKNOWN_COLUMN` validation error |
| C1405 | Table name injection | `POST /validate/query`: `from: 'orders; DROP TABLE orders'` | `UNKNOWN_TABLE` validation error |
| C1411 | EXISTS table name injection | `POST /validate/query`: orders with `filters: [{ table: "users; DROP TABLE users" }]` | `UNKNOWN_TABLE` validation error |
| C1421 | Column name on cross-DB table | `POST /validate/query`: events JOIN users, `columns: ['id"; DROP TABLE users; --']` on users side | `UNKNOWN_COLUMN` validation error |
| C1460 | ORDER BY direction injection | `POST /validate/query`: orders, `orderBy: [{ column: 'id', direction: 'asc; DROP TABLE orders;--' }]` | validation error — `direction` must be `'asc'` or `'desc'` (rejected before SQL generation) |
| C1461 | Aggregation function name injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total', fn: 'sum); DROP TABLE orders;--', alias: 'x' }]` | validation error — `fn` must be one of `count`, `sum`, `avg`, `min`, `max` |
| C1462 | Column filter operator injection | `POST /validate/query`: orders, `filters: [{ column: 'id', operator: ') OR 1=1 --', refColumn: 'customerId' }]` | validation error — column-filter `operator` must be one of `=`, `!=`, `>`, `<`, `>=`, `<=` |
| C1463 | Filter group logic injection | `POST /validate/query`: orders, `filters: [{ logic: 'and 1=1);--', conditions: [{ column: 'status', operator: '=', value: 'active' }] }]` | validation error — `logic` must be `'and'` or `'or'` |
| C1464 | EXISTS count operator injection | `POST /validate/query`: orders, `filters: [{ table: 'users', count: { operator: ') UNION SELECT 1;--', value: 1 } }]` | validation error — `count.operator` must be one of `=`, `!=`, `>`, `<`, `>=`, `<=` |
| C1465 | HAVING group logic injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x' }]`, `groupBy: ['status']`, `having: [{ logic: 'or 1=1);--', conditions: [{ column: 'x', operator: '>', value: 0 }] }]` | validation error — HAVING `logic` must be `'and'` or `'or'` |
| C1466 | JOIN table name injection | `POST /validate/query`: orders, `joins: [{ table: 'users; DROP TABLE users' }]` | validation error — `UNKNOWN_TABLE` (join target does not exist) |
| C1467 | ORDER BY column injection | `POST /validate/query`: orders, `orderBy: [{ column: 'id"; DROP TABLE orders;--', direction: 'asc' }]` | validation error — `INVALID_ORDER_BY` (column does not exist) |
| C1468 | GROUP BY column injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x' }]`, `groupBy: [{ column: 'status"; DROP TABLE orders;--' }]` | validation error — `UNKNOWN_COLUMN` (column does not exist) |
| C1469 | Aggregation column injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total"; DROP TABLE orders;--', fn: 'sum', alias: 'x' }]` | validation error — `UNKNOWN_COLUMN` (column does not exist) |
| C1470 | HAVING column (non-alias) injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x' }]`, `groupBy: ['status']`, `having: [{ column: 'x"; DROP TABLE orders;--', operator: '>', value: 0 }]` | validation error — `INVALID_HAVING` (column is not an aggregation alias) |
| C1471 | HAVING operator injection | `POST /validate/query`: orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x' }]`, `groupBy: ['status']`, `having: [{ column: 'x', operator: '> 0); DROP TABLE orders;--', value: 0 }]` | validation error — `INVALID_HAVING` (operator not in allowed set) |
| C1472 | Filter value operator injection | `POST /validate/query`: orders, `filters: [{ column: 'status', operator: '= 1); DROP TABLE orders;--', value: 'active' }]` | validation error — `INVALID_FILTER` (operator not in allowed set) |
| C1473 | Filter column name injection | `POST /validate/query`: orders, `filters: [{ column: 'status"; DROP TABLE orders;--', operator: '=', value: 'active' }]` | validation error — `UNKNOWN_COLUMN` (column does not exist) |

### 16.2 Aggregation Alias Injection (all dialects)

Aggregation aliases are user-provided strings interpolated into SQL as quoted identifiers (`"alias"` for Postgres/Trino, `` `alias` `` for ClickHouse). If the alias contains the quoting character, it could break out of identifier quoting and inject SQL. Conforming implementations must either **reject** aliases containing SQL metacharacters at validation time (`INVALID_AGGREGATION`) or **escape** the quoting character (e.g., `"` → `""` for Postgres/Trino, `` ` `` → ``` `` ``` for ClickHouse).

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1412 | PG alias with double-quote injection | orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x"; DROP TABLE orders;--' }]` | `INVALID_AGGREGATION` validation error **or** alias safely escaped in generated SQL; no SQL injection |
| C1413 | PG alias with backtick injection | orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x`; DROP TABLE orders;--' }]` | same: rejected or escaped; no injection |
| C1414 | PG HAVING referencing injected alias | orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x"; --' }]`, `having: [{ column: 'x"; --', operator: '>', value: 0 }]` | rejected or escaped; no injection in HAVING clause |
| C1415 | PG ORDER BY referencing injected alias | orders, `aggregations: [{ column: 'total', fn: 'sum', alias: 'x"; --' }]`, `orderBy: [{ column: 'x"; --', direction: 'asc' }]` | rejected or escaped; no injection in ORDER BY clause |
| C1419 | CH alias with backtick injection | events, `aggregations: [{ column: 'timestamp', fn: 'count', alias: 'x`; DROP TABLE events;--' }]` | rejected or backtick escaped in `` `alias` `` quoting; no injection |
| C1448 | CH HAVING referencing backtick-injected alias | events, `aggregations: [{ column: 'timestamp', fn: 'count', alias: 'x`; --' }]`, `having: [{ column: 'x`; --', operator: '>', value: 0 }]` | rejected or escaped; no injection in `` `alias` `` HAVING clause |
| C1449 | CH ORDER BY referencing backtick-injected alias | events, `aggregations: [{ column: 'timestamp', fn: 'count', alias: 'x`; --' }]`, `orderBy: [{ column: 'x`; --', direction: 'asc' }]` | rejected or escaped; no injection in `` `alias` `` ORDER BY clause |
| C1422 | Trino alias with double-quote injection | events JOIN users (cross-DB), `aggregations: [{ column: 'id', table: 'users', fn: 'count', alias: 'x"; DROP TABLE users;--' }]` | rejected or double-quote escaped; no injection |
| C1450 | Trino HAVING referencing injected alias | events JOIN users, `aggregations: [{ column: 'id', table: 'users', fn: 'count', alias: 'x"; --' }]`, `having: [{ column: 'x"; --', operator: '>', value: 0 }]` | rejected or escaped; no injection in Trino HAVING clause |
| C1451 | Trino ORDER BY referencing injected alias | events JOIN users, `aggregations: [{ column: 'id', table: 'users', fn: 'count', alias: 'x"; --' }]`, `orderBy: [{ column: 'x"; --', direction: 'asc' }]` | rejected or escaped; no injection in Trino ORDER BY clause |

### 16.3 PostgreSQL Filter Value Injection

These tests target pg-main tables, exercising PostgreSQL's `$N` parameterization, native `ILIKE`, `= ANY($1::type[])`, `@>`, and `&&` operators.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1400 | PG `=` filter injection | orders, `status = "'; DROP TABLE orders; --"` | value parameterized via `$1`; no table dropped |
| C1401 | PG `like` filter injection | users, `email like "%'; DROP TABLE users; --%"` | value parameterized via `$1`; no injection |
| C1402 | PG `contains` injection | users, `email contains "'; DROP TABLE --"` | escapeLike + parameterized via `$1`; no injection |
| C1403 | PG `between` injection | orders, `total between { from: "0; DROP TABLE orders", to: 100 }` | both bounds parameterized via `$1`, `$2`; rejected or treated as literal |
| C1406 | PG `in` filter injection | orders, `status in ["active'; DROP TABLE orders; --"]` | `= ANY($1::text[])` — single array param; no injection |
| C1407 | PG `notIn` filter injection | orders, `status notIn ["active'; DROP TABLE orders; --"]` | `<> ALL($1::text[])` — single array param; no injection |
| C1408 | PG `levenshteinLte` injection | users, `firstName levenshteinLte { text: "'; DROP TABLE users; --", maxDistance: 3 }` | `levenshtein(col, $1) <= $2`; both parameterized; no injection |
| C1409 | PG `arrayContains` injection | products, `labels arrayContains "sale'; DROP TABLE products; --"` | `$1::text = ANY(col)` parameterized; no injection |
| C1431 | PG `icontains` injection | users, `email icontains "'; DROP TABLE users; --"` | `ILIKE $1` (native keyword); no injection |
| C1432 | PG `notBetween` injection | orders, `total notBetween { from: "0; DROP TABLE orders", to: 100 }` | `NOT BETWEEN $1 AND $2`; both parameterized; no injection |
| C1433 | PG `endsWith` injection | users, `email endsWith "'; DROP TABLE users; --"` | `LIKE $1` with `%escaped` pattern; parameterized; no injection |
| C1453 | PG `startsWith` injection | users, `email startsWith "'; DROP TABLE users; --"` | `LIKE $1` with `escaped%` pattern; parameterized; no injection |
| C1434 | PG `arrayContainsAll` injection | products, `labels arrayContainsAll ["sale'; DROP TABLE products; --"]` | `col @> $1::text[]` — PG array containment operator; no injection |
| C1435 | PG `arrayContainsAny` injection | products, `labels arrayContainsAny ["sale'; DROP TABLE products; --"]` | `col && $1::text[]` — PG array overlap operator; no injection |
| C1410 | PG `byIds` injection | users, `byIds: ["'; DROP TABLE users; --"]` | resolved as `= ANY($1::uuid[])` on PK; value parameterized; no injection |

### 16.4 ClickHouse Filter Value Injection

These tests target ch-analytics tables (events), exercising ClickHouse's typed `{pN:Type}` parameterization, `ilike()` function, native `startsWith()`/`endsWith()`, `has()`/`hasAll()`/`hasAny()`, `IN tuple()`, `editDistance()`, and `NOT (col BETWEEN ...)` wrapping.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1416 | CH `=` filter injection | events, `type = "'; DROP TABLE events; --"` | value parameterized via `{pN:String}`; no injection |
| C1423 | CH `in` filter injection | events, `type in ["purchase'; DROP TABLE events; --"]` | expanded `IN tuple({p1:String}, ...)` — each element individually typed; no injection |
| C1424 | CH `contains` filter injection | events, `type contains "'; DROP TABLE events; --"` | escapeLike applied, parameterized via `{pN:String}`; no injection |
| C1425 | CH `between` filter injection | events, `timestamp between { from: "2024-01-01'; DROP TABLE events; --", to: "2024-12-31" }` | both bounds parameterized via `{p1:DateTime}` and `{p2:DateTime}`; no injection |
| C1426 | CH `levenshteinLte` injection | events, `type levenshteinLte { text: "'; DROP TABLE events; --", maxDistance: 5 }` | mapped to `editDistance()` function; text via `{pN:String}`, threshold via `{pN:UInt32}`; no injection |
| C1427 | CH `startsWith` injection | events, `type startsWith "'; DROP TABLE events; --"` | native `startsWith(col, {pN:String})` function (not LIKE); no injection |
| C1436 | CH `endsWith` injection | events, `type endsWith "'; DROP TABLE events; --"` | native `endsWith(col, {pN:String})` function (not LIKE); no injection |
| C1437 | CH `icontains` injection | events, `type icontains "'; DROP TABLE events; --"` | `ilike(col, {pN:String})` function; value escaped and parameterized; no injection |
| C1438 | CH `notBetween` injection | events, `timestamp notBetween { from: "2024-01-01'; DROP TABLE events;--", to: "2024-12-31" }` | `NOT (col BETWEEN {p1:DateTime} AND {p2:DateTime})`; both parameterized; no injection |
| C1439 | CH `arrayContains` injection | events, `tags arrayContains "x'; DROP TABLE events; --"` | `has(col, {p1:String})`; value parameterized; no injection |
| C1440 | CH `arrayContainsAll` injection | events, `tags arrayContainsAll ["x'; DROP TABLE events; --"]` | `hasAll(col, [{p1:String}, ...])`; each element parameterized; no injection |
| C1417 | CH `arrayContainsAny` injection | events, `tags arrayContainsAny ["x'; DROP TABLE events; --"]` | `hasAny(col, [{p1:String}, ...])`; each element parameterized; no injection |
| C1441 | CH `notIn` injection | events, `type notIn ["purchase'; DROP TABLE events; --"]` | `NOT IN tuple({p1:String}, ...)` — each element individually typed; no injection |
| C1446 | CH `byIds` injection | events, `byIds: ["'; DROP TABLE events; --"]` | resolved as `IN tuple({p1:String}, ...)` on PK; each value individually typed; no injection |
| C1454 | CH `like` injection | events, `type like "%'; DROP TABLE events; --%"` | value parameterized via `{pN:String}`; no injection |

### 16.5 Trino Filter Value Injection

These tests use cross-DB joins (events + users/products), forcing Trino as the executor. This exercises Trino's `?` parameterization, `LIKE ? ESCAPE '\\'` pattern operators, `lower()` emulation for case-insensitive ops, `BETWEEN ? AND ?`, `NOT BETWEEN ? AND ?`, `contains()`, `arrays_overlap()`, `array_except()`, and `levenshtein_distance()`.

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1420 | Trino `=` filter injection | events JOIN users (ch-analytics + pg-main), filter `users.email = "'; DROP TABLE users; --"` | Trino `?` parameterization; no injection |
| C1428 | Trino `in` filter injection | events JOIN users, filter `users.email in ["x'; DROP TABLE users; --"]` | expanded `IN (?, ?, ...)` — each element individually parameterized; no injection |
| C1429 | Trino `contains` filter injection | events JOIN users, filter `users.email contains "'; DROP TABLE users; --"` | escapeLike applied, `LIKE ? ESCAPE '\\'` — Trino-specific ESCAPE clause; no injection |
| C1430 | Trino `levenshteinLte` injection | events JOIN users, filter `users.firstName levenshteinLte { text: "'; DROP TABLE users; --", maxDistance: 5 }` | `levenshtein_distance(col, ?) <= ?`; both params via `?`; no injection |
| C1442 | Trino `icontains` injection | events JOIN users, filter `users.email icontains "'; DROP TABLE users; --"` | `lower(col) LIKE lower(?) ESCAPE '\\'`; Trino `lower()` emulation + ESCAPE; no injection |
| C1443 | Trino `arrayContains` injection | events JOIN users and products, filter `products.labels arrayContains "x'; DROP TABLE products; --"` | `contains(col, ?)`; value parameterized; no injection |
| C1444 | Trino `arrayContainsAll` injection | events JOIN users and products, filter `products.labels arrayContainsAll ["x'; DROP TABLE products; --"]` | `cardinality(array_except(ARRAY[?, ...], col)) = 0`; each element parameterized; no injection |
| C1445 | Trino `arrayContainsAny` injection | events JOIN users and products, filter `products.labels arrayContainsAny ["x'; DROP TABLE products; --"]` | `arrays_overlap(col, ARRAY[?, ...])` — each element parameterized; no injection |
| C1452 | Trino `notIn` injection | events JOIN users, filter `users.email notIn ["x'; DROP TABLE users; --"]` | `NOT IN (?, ?, ...)` — each element individually parameterized; no injection |
| C1447 | Trino `byIds` injection | events JOIN users, `byIds: ["'; DROP TABLE users; --"]` on users table (cross-DB) | resolved as `IN (?, ?, ...)` on PK; each value parameterized; no injection |
| C1455 | Trino `like` injection | events JOIN users, filter `users.email like "%'; DROP TABLE users; --%"` | `LIKE ?` parameterized; no injection |
| C1456 | Trino `between` injection | events JOIN users, filter `users.age between { from: "0; DROP TABLE users", to: 100 }` | `BETWEEN ? AND ?`; both parameterized; no injection |
| C1457 | Trino `notBetween` injection | events JOIN users, filter `users.age notBetween { from: "0; DROP TABLE users", to: 100 }` | `NOT BETWEEN ? AND ?`; both parameterized; no injection |
| C1458 | Trino `startsWith` injection | events JOIN users, filter `users.email startsWith "'; DROP TABLE users; --"` | `LIKE ? ESCAPE '\\'`; parameterized; no injection |
| C1459 | Trino `endsWith` injection | events JOIN users, filter `users.email endsWith "'; DROP TABLE users; --"` | `LIKE ? ESCAPE '\\'`; parameterized; no injection |
---

## 17. Validation Endpoints

These tests verify the dedicated validation endpoints that run without DB connections.

### 17.1 Query Validation

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1600 | Valid query passes | POST `/validate/query` with `{ definition: { from: 'orders', columns: ['id'] }, context: { roles: { user: ['admin'] } } }` | response: `{ valid: true }` |
| C1601 | Unknown table rejected | POST `/validate/query` with `from: 'nonExistentTable'` | 400 `ValidationError` with `UNKNOWN_TABLE` |
| C1602 | Unknown column rejected | POST `/validate/query` with invalid column | 400 `ValidationError` with `UNKNOWN_COLUMN` |
| C1603 | Access denied rejected | POST `/validate/query` with restricted column (tenant-user) | 400 `ValidationError` with `ACCESS_DENIED` |
| C1604 | Invalid filter rejected | POST `/validate/query` with `>` on uuid column | 400 `ValidationError` with `INVALID_FILTER` |
| C1605 | Invalid value rejected | POST `/validate/query` with `between` missing `to` | 400 `ValidationError` with `INVALID_VALUE` |
| C1606 | Multiple errors collected | POST `/validate/query` with multiple issues | 400 `ValidationError` with multiple `errors[]` entries |
| C1607 | Unknown role rejected | POST `/validate/query` with `user: ['nonexistent']` | 400 `ValidationError` with `UNKNOWN_ROLE` |
| C1608 | No DB connection used | POST `/validate/query` (server has no executors configured) | still returns `{ valid: true }` for valid query — proves zero I/O |
| C1609 | Same error format as /query | Compare error from `/validate/query` vs `/query` for same invalid input | identical `code`, `errors[]` structure, `fromTable` |

### 17.2 Config Validation

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1620 | Valid config passes | POST `/validate/config` with `{ metadata: {...}, roles: [...] }` | response: `{ valid: true }` |
| C1621 | Invalid apiName format | POST `/validate/config` with table apiName `'Order_Items'` | 400 `ConfigError` with `INVALID_API_NAME` |
| C1622 | Duplicate apiName | POST `/validate/config` with two tables having apiName `'orders'` | 400 `ConfigError` with `DUPLICATE_API_NAME` |
| C1623 | Invalid DB reference | POST `/validate/config` with table referencing non-existent database | 400 `ConfigError` with `INVALID_REFERENCE` |
| C1624 | Invalid relation | POST `/validate/config` with relation referencing non-existent table | 400 `ConfigError` with `INVALID_RELATION` |
| C1625 | Invalid sync reference | POST `/validate/config` with ExternalSync referencing missing table | 400 `ConfigError` with `INVALID_SYNC` |
| C1626 | Invalid cache config | POST `/validate/config` with CacheMeta referencing missing table | 400 `ConfigError` with `INVALID_CACHE` |
| C1627 | Multiple config errors | POST `/validate/config` with multiple issues | 400 `ConfigError` with multiple `errors[]` entries |
| C1628 | Duplicate column apiName | POST `/validate/config` with two columns same apiName in one table | 400 `ConfigError` with `DUPLICATE_API_NAME` |
| C1629 | apiName starting with uppercase | POST `/validate/config` with table apiName `'Orders'` | 400 `ConfigError` with `INVALID_API_NAME` — must match `^[a-z][a-zA-Z0-9]*$` |
| C1630 | apiName with underscore | POST `/validate/config` with table apiName `'order_items'` | 400 `ConfigError` with `INVALID_API_NAME` — underscores not allowed |
| C1631 | Relation source column doesn't exist | POST `/validate/config` with relation `from` referencing non-existent column | 400 `ConfigError` with `INVALID_RELATION` |
| C1632 | Relation target column doesn't exist | POST `/validate/config` with relation `to` referencing non-existent column in target table | 400 `ConfigError` with `INVALID_RELATION` |

---

## 18. Edge Cases

| ID | Test | Definition | Assertions |
|---|---|---|---|
| C1700 | Empty result set | orders, `status = 'nonexistent_status_xyz'` | `kind === 'data'`; `data` is empty array; `meta.columns` still present |
| C1701 | Single row result | orders, `byIds: [1]` | `data.length === 1` |
| C1702 | Large in-list | orders, `status in [50+ values]` | query executes without error |
| C1703 | Nullable column in result | orders columns: [id, discount] | `discount` can be null in returned data |
| C1704 | Boolean column values | orders columns: [id, isPaid] | `isPaid` is `true`, `false`, or `null` — proper boolean (not 0/1) |
| C1705 | Timestamp format | orders columns: [createdAt] | timestamp is ISO 8601 string (or number); consistent format across implementations |
| C1706 | Date format | invoices columns: [dueDate] | date is ISO date string (YYYY-MM-DD) |
| C1707 | Array column in result | products columns: [name, labels] | `labels` is JSON array (e.g. `["sale", "new"]`) or null |
| C1708 | Decimal precision | orders columns: [total] | `total` is a number with decimal precision preserved |
| C1709 | Multiple filters (implicit AND) | orders, 2 top-level filters | both filter conditions applied (AND logic) |
| C1710 | Cache strategy reported | users, `byIds: ['uuid-c1']` (admin) | `meta.strategy === 'cache'` — users table has `redis-main` cache configured |
| C1711 | Materialized replica query | orders (admin), `freshness: 'seconds'` | `meta.strategy === 'materialized'`; `meta.tablesUsed[0].source === 'replica'` — planner routes to Debezium-synced `orders_replica` on ch-analytics (sync lag is `'seconds'`, freshness allows it) |
| C1712 | Cross-DB Trino join | events JOIN users (ch-analytics + pg-main, admin) | `meta.strategy === 'trino-cross-db'` — Trino used to join across databases |
| C1713 | DISTINCT + count mode | orders, `distinct: true, columns: [status], executeMode: 'count'` | `kind === 'count'`; count equals number of distinct statuses (4: active, paid, cancelled, shipped) |
| C1714 | GROUP BY with zero matching rows | orders, `status = 'nonexistent'`, groupBy: [status], SUM(total) | `kind === 'data'`; `data` is empty array; `meta.columns` still present |
| C1715 | Freshness `'realtime'` skips materialized | orders (admin), `freshness: 'realtime'` | `meta.strategy !== 'materialized'` — `'realtime'` always rejects materialized replicas regardless of lag |
| C1716 | Freshness `'hours'` allows stale replica | orders (admin), `freshness: 'hours'` | `meta.strategy === 'materialized'` — replica lag `'seconds'` is fresher than required `'hours'` |

---

## 19. Executor Contract

Individual `DbExecutor` implementations must satisfy the following behavioral guarantees. The `describeExecutorContract(name, factory, config)` suite tests any `DbExecutor` directly — not through the full query pipeline.

| ID | Test | Input | Assertions |
|---|---|---|---|
| C1800 | `ping()` resolves for healthy executor | healthy executor | `ping()` resolves without error |
| C1801 | `execute()` returns `Record[]` for valid SQL | `SELECT 1 AS n` | returns non-empty array of plain objects |
| C1802 | `execute()` throws `ExecutionError` for invalid SQL | `SELECT * FROM __nonexistent_table_xyz__` | `ExecutionError` with `code: 'QUERY_FAILED'` |
| C1803 | `close()` resolves without error | fresh executor | `close()` completes without throwing |
| C1804 | `ping()` throws after close | closed executor | throws `ConnectionError` or `ExecutionError` (stateless executors like Trino REST may not throw — acceptable) |

---

## Implementation Checklist

For implementation developers, verify the following groups pass in order:

1. **Validation Endpoints** (C1600-C1632) — no DB needed, fast feedback *(start here)*
2. **Health Check** (C1300-C1304) — server is running and connected
3. **Execute Modes** (C001-C027) — basic response shapes, sql-only, count
4. **Debug Mode** (C030-C034) — debug logging works
5. **Filtering** (C100-C196 × 3 dialects) — all 30 operators + groups + qualifiers
6. **Joins** (C200-C207 × 3 dialects) — left/inner, multi-table, column selection, collision
7. **Aggregations** (C300-C310 × 3 dialects) — all 5 functions, groupBy interaction, NULLs
8. **GROUP BY & HAVING** (C320-C329 × 3 dialects) — grouping, HAVING conditions, joined column
9. **ORDER BY, LIMIT, OFFSET, DISTINCT** (C400-C407 × 3 dialects) — pagination + sorting
10. **byIds** (C500-C507 × 3 dialects, C505 pg-only) — primary key shortcut, composite PK rejection
11. **EXISTS** (C600-C613 × 3 dialects) — subqueries, all 6 counted operators, self-referencing, nested, join combo
12. **Access Control** (C700-C725) — roles, scopes, intersection, joined table access, multi-scope disjoint/wildcard
13. **Masking** (C800-C816) — all 7 masking functions (number, full, email, phone, name, date, uuid + null pass-through), multi-role, cross-scope
14. **Validation Errors** (C900-C1030) — all 14 rules verified (via /query)
15. **Meta Verification** (C1100-C1113) — response metadata, targetDatabase, dialect per mode, agg nullable
16. **Error Deserialization** (C1200-C1206) — HTTP error transport, all error types
17. **Planner Errors** (C1250-C1254) — Trino disabled, no catalog, unreachable, freshness
18. **Execution Errors** (C1260-C1263) — missing executor/cache, query failure, timeout
19. **Provider Errors** (C1270-C1271) — metadata/role load failure
20. **Lifecycle** (C1310-C1313) — reload metadata/roles, close
21. **SQL Injection** (C1400-C1473) — per-dialect parameterization, identifier validation, alias escaping, enum-keyword validation
22. **Edge Cases** (C1700-C1716) — nulls, types, strategies, freshness, distinct+count, empty groups
23. **Executor Contract** (C1800-C1804) — ping, execute, close, error wrapping for any `DbExecutor` implementation

Total: **420 unique test IDs** × parameterization = ~**661 test executions** (sections 3–9: 113 IDs × 3 dialects + C505 × 1 = 340; other sections: 306 × 1 = 306; section 23: C1800-C1804 × 3 executors = 15; total = 661)

---

## Notes for Implementors

- **Error format:** All errors must be serialized via `toJSON()` format: `{ code, message, errors?: [...], fromTable?, details? }`. The client reconstructs typed error instances from the `code` field.
- **HTTP status mapping:** ValidationError/ConfigError → 400, PlannerError → 422, ExecutionError → 500, ConnectionError/ProviderError → 503.
- **JSON types:** Booleans must be JSON `true`/`false` (not `0`/`1`). Numbers must be JSON numbers. Arrays must be JSON arrays. Timestamps should be ISO 8601 strings.
- **Parameter binding:** All user-provided values must be parameterized (never interpolated into SQL strings). This is verified by SQL injection tests.
- **Column masking:** Masking is a post-query operation. Data is fetched normally, then masked before returning. The `maskingFn` from column metadata determines *how* to mask; the role's `maskedColumns` determines *which* columns to mask.
- **Meta consistency:** `meta.columns` must accurately reflect the actual keys in `data[]` rows. If column names are qualified due to collisions (e.g. `orders.id`), `meta.columns[].apiName` must match the qualified key.
- **Seed data:** Use the exact seed data from the [Fixture](#fixture) section. Assertions on row counts and values depend on this data.
- **Validation endpoints:** `/validate/query` and `/validate/config` must run pure validation logic with zero I/O. They share the same error format as `/query`. Implementation developers should start with validation endpoint tests (section 17) — they require no database and provide fast feedback on metadata handling.
- **Dialect parameterization:** Sections 3–9 must be implemented as parameterized tests that run each test ID three times — once per dialect (pg/ch/trino). See the [Parameterization](#parameterization) header in section 3 for table substitution rules.
