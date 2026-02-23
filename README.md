# @mkven/multi-db-query — Concept Document

## Objective

Build a reusable, metadata-driven query engine that lets applications query, filter, join, and aggregate data across Postgres, ClickHouse, Iceberg, and Redis through a single typed API. The engine:

- Accepts queries using **apiNames** (decoupled from physical schema)
- Supports **rich filtering** — 31 operators (comparison, pattern, range, fuzzy, array), column-vs-column comparisons, recursive AND/OR/NOT groups, EXISTS/NOT EXISTS subqueries with optional counted variant and nesting
- Supports **JOINs** (inner/left) resolved from relation metadata, **GROUP BY**, and **aggregations** (count, sum, avg, min, max) with HAVING
- Automatically selects the **optimal execution strategy** — direct DB, cached, materialized replica, or Trino cross-DB federation
- Enforces **scoped access control** — user roles and service roles intersected to determine effective permissions
- Applies **column masking** for sensitive data based on role
- **Validates strictly** — 14 rules covering tables, columns, filters, joins, aggregations, permissions; all errors collected into a single response
- Returns **generated SQL**, **executed results**, or **row counts** depending on the caller's needs
- Provides **structured debug logs** for transparent pipeline tracing
- Supports **hot-reload** of metadata and roles, **health checks** for all providers, and graceful **shutdown**

The monorepo ships a standalone **validation package** (`@mkven/multi-db-validation`) with **zero I/O dependencies** — clients can validate configs and queries locally before sending to the server. The core package (`@mkven/multi-db-query`) depends on it and adds planning, SQL generation, and masking, also with zero I/O deps. An **HTTP client package** (`@mkven/multi-db-client`) provides a typed client and a contract test suite — the same tests run against both in-process and HTTP implementations. Database drivers and cache clients live in separate executor/cache packages.

## Target Databases

| Engine | Role |
|---|---|
| **Postgres** | Primary OLTP |
| **ClickHouse** | Analytics, columnar storage |
| **Iceberg** (via Trino) | Archival / lakehouse |
| **Trino** | Cross-database query federation |
| **Redis** | Cache layer for by-ID lookups |

Tables may live in any database. Some tables are replicated between databases via Debezium (CDC). The planner uses this topology to pick the optimal execution path.

---

## Architecture

```
┌─────────────────────────────────────┐
│       Query Definition (DSL)        │
│  { from, columns, filters, joins }  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  1. VALIDATION                      │
│  - table/column existence           │
│  - role permissions                 │
│  - filter/join/aggregation validity │
│  - exists/ordering/limit checks     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  2. ACCESS CONTROL                  │
│  - role-based column trimming       │
│  - column masking                   │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  3. QUERY PLANNING                  │
│  Strategy selection:                │
│  P0: Cache (redis, by-ID only)      │
│  P1: Single-DB direct               │
│  P2: Materialized replica           │
│  P3: Trino cross-DB                 │
│  P4: Error (unreachable)            │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  4. NAME RESOLUTION                 │
│  apiName → physicalName mapping     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  5. SQL GENERATION                  │
│  Dialect-specific SQL output        │
│  (Postgres / ClickHouse / Trino)    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  6. EXECUTION (optional)            │
│  Run against correct connection     │
│  Return typed results               │
└─────────────────────────────────────┘
```

Every phase emits structured debug log entries (when `debug: true` is set on the query).

---

## Metadata Model

Databases, tables, columns, relations, external syncs (Debezium), cache (Redis), roles & access control, column masking, metadata configuration, and provider interfaces.

→ See [METADATA.md](./METADATA.md)
---

## Module Initialization & Query API

### Initialization (once at startup)

The module is initialized with metadata and role providers, plus optional executor/cache instances:

```ts
import { createMultiDb, staticMetadata, staticRoles } from '@mkven/multi-db-query'
import { createPostgresExecutor } from '@mkven/multi-db-executor-postgres'
import { createClickHouseExecutor } from '@mkven/multi-db-executor-clickhouse'
import { createTrinoExecutor } from '@mkven/multi-db-executor-trino'
import { createRedisCache } from '@mkven/multi-db-cache-redis'
```

```ts
interface CreateMultiDbOptions {
  metadataProvider: MetadataProvider
  roleProvider: RoleProvider
  executors?: Record<string, DbExecutor>
  cacheProviders?: Record<string, CacheProvider>
  validateConnections?: boolean        // default: true — ping all executors/caches at startup
}
```

```ts
// Returns Promise<MultiDb> — async because it loads providers and pings connections
const multiDb = await createMultiDb({
  // Required: metadata and role providers
  metadataProvider: staticMetadata({
    databases: [...],
    tables: [...],
    caches: [...],
    externalSyncs: [...],
    trino: { enabled: true },
  }),
  roleProvider: staticRoles([...]),

  // Optional: executors (only needed for executeMode = 'execute' | 'count')
  // Keys must match DatabaseMeta.id, except 'trino' which is a special key for the federation layer
  executors: {
    'pg-main': createPostgresExecutor({ connectionString: '...', timeoutMs: 30_000 }),
    'pg-tenant': createPostgresExecutor({ connectionString: '...', timeoutMs: 30_000 }),
    'ch-analytics': createClickHouseExecutor({ url: '...', timeoutMs: 60_000 }),
    'trino': createTrinoExecutor({ url: '...', timeoutMs: 120_000 }),  // special key — not a DatabaseMeta.id
  },

  // Optional: cache providers (keys must match CacheMeta.id)
  cacheProviders: {
    'redis-main': createRedisCache({ url: '...' }),
  },
})
```

At init time (`createMultiDb` is async), steps execute in order:
1. **Load providers** — `metadataProvider.load()` and `roleProvider.load()` (throws `ProviderError` on failure)
2. **Validate** — all apiNames checked (format, reserved words, uniqueness) (throws `ConfigError` on failure)
3. **Index** — metadata indexed into in-memory Maps for O(1) lookups:
  - `Map<apiName, TableMeta>` — table by apiName
  - `Map<tableId, Map<apiName, ColumnMeta>>` — column by table + apiName
  - `Map<roleId, RoleMeta>` — role by id
  - `Map<databaseId, DatabaseMeta>` — database by id
  - `Map<tableId, ExternalSync[]>` — syncs per table
  - `Map<tableId, CachedTableMeta>` — cache config per table
4. **Build graph** — database connectivity graph is built (for planner)
5. **Ping** — all executors and cache providers are pinged to verify connectivity (calls `ping()` on each). If any fail, a `ConnectionError` is thrown listing all unreachable providers. Disable with `validateConnections: false` for lazy connection scenarios

**Executor timeout:** Each executor factory accepts an optional `timeoutMs` parameter (milliseconds). The executor implementation enforces it via the driver's native mechanism (e.g. Postgres `statement_timeout`, ClickHouse `max_execution_time`). When exceeded, the executor throws an error that the core wraps as `ExecutionError: QUERY_TIMEOUT` with the configured `timeoutMs` value. Omitting `timeoutMs` means no timeout (the driver's default applies)

### Query Request (per call)

Each query provides the query definition and the execution context:

```ts
// Execute and get data
const result = await multiDb.query({
  definition: {
    from: 'orders',
    columns: ['id', 'total', 'status'],
    filters: [{ column: 'status', operator: '=', value: 'active' }],
    limit: 50,
    offset: 0,
  },
  context: {
    roles: {
      user: ['admin'],
      service: ['orders-service'],
    },
  },
})

// Get SQL only (no execution, no executor needed)
const sqlResult = await multiDb.query({
  definition: {
    from: 'orders',
    columns: ['id', 'total'],
    executeMode: 'sql-only',
    debug: true,
  },
  context: {
    roles: { user: ['viewer'] },
  },
})

// Count rows
const countResult = await multiDb.query({
  definition: {
    from: 'orders',
    filters: [{ column: 'status', operator: '=', value: 'active' }],
    executeMode: 'count',
  },
  context: {
    roles: { user: ['admin'] },
  },
})
```

**What goes where:**

| Provided at init | Provided per query |
|---|---|
| Metadata + role providers | Query definition (from, columns, filters, joins, etc.) |
| Executor instances (DB connections) | Execution context (scoped roles) |
| Cache provider instances | `executeMode`, `debug`, `freshness` |

---

## Query Definition

Query input (`QueryDefinition`), execution context, query result types (`SqlResult`, `DataResult`, `CountResult`), result metadata, and debug log entries.

→ See [QUERY.md](./QUERY.md)
---

## Query Planner Strategy

Given a query touching tables T1, T2, ... Tn:

### Priority 0 — Cache (redis)
- Only for `byIds` queries **without `filters`** (filters skip cache — too ambiguous to post-filter)
- Only for single-table queries **without `joins`** (cache stores single-table data)
- Only for tables with single-column primary keys (composite PKs use filters instead)
- Check if the table has a cache config
- If `CachedTableMeta.columns` is a subset and the query requests columns outside that subset → skip P0 (cache doesn't have the data)
- If cache hit for all IDs → return from cache (trim columns if needed, apply masking identically to DB results)
- If partial hit → return cached + fetch missing from DB, merge

### Priority 1 — Single Database Direct
- ALL required tables exist in ONE database (original data). "Required" includes `from`, all `joins`, and any tables referenced by `QueryExistsFilter` (recursively — EXISTS inside filter groups or nested EXISTS are walked)
- Generate native SQL for that engine
- **Iceberg exception:** Iceberg databases have no standalone executor — they are always queried via the `trino` executor using `trinoCatalog`. A single-Iceberg-table P1 query routes through the Trino executor with Trino dialect (single-catalog, no cross-DB federation)
- This is always preferred when possible (freshest data, no overhead)

### Priority 2 — Materialized Replica
- Some tables are in different databases, BUT debezium replicas exist such that all needed data is available in one database
- Check freshness: if query requires `realtime` → skip this strategy entirely (replicas always have lag); if `freshness` = `seconds` but replica lag is `minutes` → also skip
- Freshness hierarchy (strictest to most relaxed): `realtime` < `seconds` < `minutes` < `hours`. `realtime` is query-only (never an `estimatedLag` value) — it means direct access required. A replica is acceptable when its `estimatedLag` ≤ the query's `freshness` tolerance
- Always prefer the **original** database if the original data is there; use replicas for the "foreign" tables
- If multiple databases could serve via replicas, prefer the one with the most original tables
- Post-resolution, physical table names in `SqlParts` are replaced with the materialized replica names — applies to `from`, `joins`, and `CorrelatedSubquery.from` inside EXISTS subqueries in the WHERE tree

### Priority 3 — Trino Cross-Database
- Trino is enabled and all databases are registered as trino catalogs
- Generate Trino SQL with cross-catalog references
- Post-resolution, catalog qualifiers are set on all `TableRef` nodes — `from`, `joins`, and `CorrelatedSubquery.from` inside EXISTS subqueries in the WHERE tree
- Fallback when no single-DB path exists

### Priority 4 — Error
- Cannot reach all tables via any strategy
- Return clear error: which tables are unreachable, what's missing (trino disabled? no catalog? no sync?)

---

## Validation Rules

1. **Table existence** — only tables defined in metadata can be queried
2. **Column existence** — only columns defined in table metadata can be referenced. When `columns` is `undefined` and `aggregations` is present, the default is `groupBy` columns only (not all allowed columns) — this avoids rule 7 failures from ungrouped columns being added automatically
3. **Role permission** — if a table is not in the role's `tables` list → access denied
4. **Column permission** — if `allowedColumns` is a list and requested column is not in it → denied; if columns not specified in query, return only allowed ones
5. **Filter validity** — filter operators must be valid for the column type (see table below); `isNull`/`isNotNull` additionally require `nullable: true`; malformed compound values (`between`/`notBetween` missing `to`, `between`/`notBetween` with `null` as `from` or `to`, `levenshteinLte` with non-integer or negative `maxDistance`, `in` with empty array) are rejected with `INVALID_VALUE`; `between`/`notBetween` with `null` bounds are rejected for the same reason as `in`/`notIn` NULL — SQL `BETWEEN NULL AND x` always yields false due to 3-valued logic; `in`/`notIn` additionally validate that all array elements match the column type (e.g. passing `['a','b']` on an `int` column → `INVALID_VALUE`); `between`/`notBetween` additionally validate that `from` and `to` values match the column type (e.g. `BETWEEN 'abc' AND 'xyz'` on a `decimal` column → `INVALID_VALUE`); `in`/`notIn` also reject `null` elements — SQL's `NOT IN (1, NULL)` always returns zero rows due to 3-valued logic, which is a major footgun; **array operators** (`arrayContains`, `arrayContainsAll`, `arrayContainsAny`, `arrayIsEmpty`, `arrayIsNotEmpty`) are only valid on array column types (e.g. `'string[]'`); `arrayContains` value must match the element type; `arrayContainsAll`/`arrayContainsAny` value must be a non-empty array with all elements matching the element type (same validation as `in`/`notIn` — includes element type check and null element rejection); conversely, all scalar operators (except `isNull`/`isNotNull`) are rejected on array columns; when `QueryFilter.table` is provided, the table must be the `from` table or one of the joined tables — referencing a non-joined table is rejected; for `QueryColumnFilter`, both columns must exist, the role must allow both, and their types must be compatible (same type, or both orderable) — array columns are not allowed in `QueryColumnFilter`; filter groups and exists filters are validated recursively (all nested conditions checked)
6. **Join validity** — joined tables must have a defined relation in metadata
7. **Group By validity** — if `groupBy` or `aggregations` are present, every column in `columns` that is not an aggregation alias must appear in `groupBy`. Prevents invalid SQL from reaching the database. Array columns are rejected in `groupBy` — ClickHouse does not support `GROUP BY` on array columns, and behavior is inconsistent across dialects. When `QueryGroupBy.table` is provided, the table must be the `from` table or one of the joined tables — same rule as filter `table` (rule 5)
8. **Having validity** — `having` filters must reference aliases defined in `aggregations`; `QueryFilter.table` is rejected inside `having` (HAVING operates on aggregation aliases, not table columns); `QueryColumnFilter` nested inside `having` groups is rejected (HAVING compares aliases, not table columns — column-vs-column comparison is not meaningful); `QueryExistsFilter` nested inside `having` groups is rejected (EXISTS in HAVING is not valid SQL); only comparison, range, and null-check operators are allowed — `=`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `notIn`, `between`, `notBetween`, `isNull`, `isNotNull` (`isNull`/`isNotNull` do not require `nullable: true` in HAVING context — aggregation aliases have no column metadata; checking whether an aggregation returned NULL is always valid); pattern operators (`like`, `ilike`, `contains`, `startsWith`, `endsWith`, and their `not`/`i` variants), `levenshteinLte`, and array operators (`arrayContains`, `arrayContainsAll`, `arrayContainsAny`, `arrayIsEmpty`, `arrayIsNotEmpty`) are rejected — they operate on text values or array columns, not aggregated numbers
9. **Order By validity** — `orderBy` must reference columns from `from` table, joined tables, or aggregation aliases defined in `aggregations`. Array columns are rejected in `orderBy` — arrays have no meaningful total ordering across dialects. When `QueryOrderBy.table` is provided, the table must be the `from` table or one of the joined tables — same rule as filter `table` (rule 5)
10. **ByIds validity** — `byIds` requires a non-empty array and a single-column primary key; cannot combine with `groupBy` or `aggregations`; `byIds` + `joins` is valid — cache (P0) is skipped (cache stores single-table data) but direct DB (P1+) handles it normally (`WHERE pk = ANY($1)` with JOINs)
11. **Limit/Offset validity** — `limit` and `offset` must be non-negative integers when provided; `offset` requires `limit` (offset without limit is rejected)
12. **Exists filter validity** — `QueryExistsFilter.table` must have a defined relation to the `from` table (or a joining table); role must allow access to the related table; when `count` is provided, `value` must be a non-negative integer; `exists` is ignored when `count` is present (the count operator handles directionality); `exists` defaults to `true` when omitted. **Nested EXISTS:** `QueryExistsFilter.filters` may contain another `QueryExistsFilter` (e.g. orders EXISTS invoices WHERE EXISTS lineItems). The inner EXISTS resolves its `table` relation against the **outer** EXISTS's table (not the query's `from`) — the system walks the nesting chain to find the correct parent for relation lookup
13. **Role existence** — all role IDs in `ExecutionContext.roles` must exist in loaded roles; unknown IDs are rejected with `UNKNOWN_ROLE`
14. **Aggregation validity** — aggregation aliases must be unique across all aggregations; aliases must not collide with column apiNames present in the result set (avoids ambiguous output columns); when `QueryAggregation.table` is provided, the table must be the `from` table or one of the joined tables — same rule as filter `table` (rule 5); explicit empty `columns: []` is only valid when `aggregations` is present (aggregation-only query, e.g. `SELECT SUM(total) FROM orders`) — empty `columns: []` without aggregations is rejected; `sum`/`avg`/`min`/`max` on array columns are rejected — only `count` is valid on array columns (counts non-NULL array values, regardless of array contents)

All validation errors are **collected, not thrown one at a time**. The system runs all applicable checks and throws a single `ValidationError` containing every issue found. This lets callers fix all problems at once instead of playing whack-a-mole.

### Filter Operator / Column Type Compatibility

| Operator | string | int | decimal | boolean | uuid | date | timestamp | T[] (array) |
|---|---|---|---|---|---|---|---|---|
| `=` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| `!=` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| `>` `<` `>=` `<=` | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | — |
| `in` `notIn` | ✓ | ✓ | ✓ | — | ✓ | — | — | — |
| `like` `notLike` | ✓ | — | — | — | — | — | — | — |
| `ilike` `notIlike` | ✓ | — | — | — | — | — | — | — |
| `between` | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | — |
| `notBetween` | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | — |
| `contains` `icontains` | ✓ | — | — | — | — | — | — | — |
| `notContains` `notIcontains` | ✓ | — | — | — | — | — | — | — |
| `startsWith` `istartsWith` | ✓ | — | — | — | — | — | — | — |
| `endsWith` `iendsWith` | ✓ | — | — | — | — | — | — | — |
| `isNull` `isNotNull` | ✓* | ✓* | ✓* | ✓* | ✓* | ✓* | ✓* | ✓* |
| `levenshteinLte` | ✓ | — | — | — | — | — | — | — |
| `arrayContains` | — | — | — | — | — | — | — | ✓ |
| `arrayContainsAll` | — | — | — | — | — | — | — | ✓ |
| `arrayContainsAny` | — | — | — | — | — | — | — | ✓ |
| `arrayIsEmpty` | — | — | — | — | — | — | — | ✓ |
| `arrayIsNotEmpty` | — | — | — | — | — | — | — | ✓ |

`QueryColumnFilter` is not listed — it supports `=`, `!=`, `>`, `<`, `>=`, `<=` with the same type restrictions as those operators. Both columns must have compatible types. Array columns are not allowed in `QueryColumnFilter`.

\* `isNull` / `isNotNull` are valid on any type (including arrays) but only on columns with `nullable: true`.

Comparison operators (`>`, `<`, `>=`, `<=`) are rejected on `uuid` and `boolean` — UUIDs have no meaningful ordering, booleans should use `=`/`!=`. `in`/`notIn` are rejected on `date`/`timestamp` — use range comparisons instead. `between`/`notBetween` follow the same type rules as `>=`/`<=` (orderable types only — excludes `uuid` and `boolean`); `notBetween` negates the range check — Postgres/Trino: `col NOT BETWEEN`, ClickHouse: `NOT (col BETWEEN ...)` (see [SQL Dialect Differences](./SQL_DIALECTS.md)). `like`/`ilike`/`contains`/`icontains`/`notContains`/`notIcontains`/`startsWith`/`istartsWith`/`endsWith`/`iendsWith` are string-only. `contains`/`icontains` map to `LIKE '%x%'` / `ILIKE '%x%'`; `notContains`/`notIcontains` map to `NOT LIKE '%x%'` / `NOT ILIKE '%x%'` — wildcard characters (`%` and `_`) in the value are escaped automatically. `startsWith`/`istartsWith` map to `LIKE 'x%'` / `ILIKE 'x%'`; `endsWith`/`iendsWith` map to `LIKE '%x'` / `ILIKE '%x'`. `levenshteinLte` is string-only — it matches rows where the Levenshtein edit distance between the column value and the target text is ≤ `maxDistance`. No index support in any dialect — always a full scan. PostgreSQL requires the `fuzzystrmatch` extension (`CREATE EXTENSION fuzzystrmatch`).

**Array operators** are valid only on array column types (`'string[]'`, `'int[]'`, etc.). All scalar operators (except `isNull`/`isNotNull`) are rejected on array columns. `arrayContains` checks if the array column contains a single element (value must match element type). `arrayContainsAll` checks if the array contains ALL given elements (value is a non-empty array). `arrayContainsAny` checks if the array contains ANY of the given elements (overlap check, value is a non-empty array). `arrayIsEmpty` / `arrayIsNotEmpty` check if the array has zero / non-zero elements — no value needed. Note: `arrayIsEmpty` is distinct from `isNull` — an empty array `[]` is not NULL. Array operators on a NULL column value follow SQL 3-valued logic — the result is NULL (treated as false in WHERE). `arrayIsEmpty` on a NULL column returns NULL, not true — use `isNull` to check for NULL arrays specifically.

`QueryFilter.table` allows filtering on joined table columns directly in the top-level `filters[]` array, as an alternative to placing filters in `QueryJoin.filters`. Both approaches produce the same SQL (filter goes in WHERE, not ON). When `table` is omitted, the column is resolved against the `from` table. If the `from` table and a joined table share an apiName, the unqualified name resolves to the `from` table — use `table` to disambiguate.

---

## Error Handling

All errors are thrown as typed exceptions (never returned in the result). Error types:

```ts
class MultiDbError extends Error {
  code: string                        // machine-readable error code

  toJSON(): Record<string, unknown>   // safe serialization for logging / API responses
  // Returns { code, message, ...details }. Recursively serializes `cause` and `errors[]`.
  // Error objects don't serialize with JSON.stringify by default — this ensures all
  // diagnostic info survives logging pipelines, structured loggers, and HTTP transports.
}

class ConfigError extends MultiDbError {
  code: 'CONFIG_INVALID'              // always this code — individual issues are in `errors`
  errors: {
    code: 'INVALID_API_NAME' | 'DUPLICATE_API_NAME' | 'INVALID_REFERENCE' | 'INVALID_RELATION' | 'INVALID_SYNC' | 'INVALID_CACHE'
    message: string
    details: { entity?: string; field?: string; expected?: string; actual?: string; database?: string; cacheId?: string }
                                      // database: for INVALID_SYNC / INVALID_REFERENCE — which database is involved
                                      // cacheId: for INVALID_CACHE — which cache config failed
  }[]
}

class ConnectionError extends MultiDbError {
  code: 'CONNECTION_FAILED' | 'NETWORK_ERROR' | 'REQUEST_TIMEOUT'
  details: {
    unreachable: {
      id: string                      // provider id (DatabaseMeta.id or CacheMeta.id)
      type: 'executor' | 'cache'
      engine?: 'postgres' | 'clickhouse' | 'trino' | 'redis'  // which engine failed — helps troubleshooting
      cause?: Error
    }[]                               // for CONNECTION_FAILED (init-time, multi-provider)
  } | {
    url?: string                      // for NETWORK_ERROR / REQUEST_TIMEOUT (HTTP client)
    timeoutMs?: number                // for REQUEST_TIMEOUT
  }
}

class ValidationError extends MultiDbError {
  code: 'VALIDATION_FAILED'           // always this code — individual issues are in `errors`
  fromTable: string                   // the query's `from` table — correlates error to query
  errors: {
    code: 'UNKNOWN_TABLE' | 'UNKNOWN_COLUMN' | 'UNKNOWN_ROLE' | 'ACCESS_DENIED'
         | 'INVALID_FILTER' | 'INVALID_VALUE' | 'INVALID_JOIN' | 'INVALID_GROUP_BY'
         | 'INVALID_HAVING' | 'INVALID_ORDER_BY' | 'INVALID_BY_IDS' | 'INVALID_LIMIT'
         | 'INVALID_EXISTS' | 'INVALID_AGGREGATION'
    message: string
    details: {
      expected?: string; actual?: string
      table?: string; column?: string; role?: string; alias?: string; operator?: string
      refColumn?: string; refTable?: string    // for QueryColumnFilter errors — the right-side column/table
      filterIndex?: number                     // zero-based index into filters[] — identifies which filter failed
    }
  }[]
}

class PlannerError extends MultiDbError {
  code: 'UNREACHABLE_TABLES' | 'TRINO_DISABLED' | 'NO_CATALOG' | 'FRESHNESS_UNMET'
  fromTable: string                   // the query's `from` table — correlates error to query
  details:
    | { code: 'UNREACHABLE_TABLES'; tables: string[] }
    | { code: 'TRINO_DISABLED' }       // no additional detail needed
    | { code: 'NO_CATALOG'; databases: string[] }
    | { code: 'FRESHNESS_UNMET'; requiredFreshness: string; availableLag: string }
}

class ExecutionError extends MultiDbError {
  code: 'EXECUTOR_MISSING' | 'CACHE_PROVIDER_MISSING' | 'QUERY_FAILED' | 'QUERY_TIMEOUT'
  details:
    | { code: 'EXECUTOR_MISSING'; database: string }        // DatabaseMeta.id
    | { code: 'CACHE_PROVIDER_MISSING'; cacheId: string }   // CacheMeta.id
    | { code: 'QUERY_FAILED'; database: string; dialect: 'postgres' | 'clickhouse' | 'trino'; sql: string; params: unknown[]; cause?: Error }
    | { code: 'QUERY_TIMEOUT'; database: string; dialect: 'postgres' | 'clickhouse' | 'trino'; sql: string; timeoutMs: number }
                                      // QUERY_FAILED: sql + params + dialect for debugging; cause = original DB error (ES2022 Error.cause)
                                      // QUERY_TIMEOUT: when executor-level timeout is exceeded; timeoutMs = configured limit
}

class ProviderError extends MultiDbError {
  code: 'METADATA_LOAD_FAILED' | 'ROLE_LOAD_FAILED'
  details: { provider: 'metadata' | 'role' }
  cause?: Error                       // original error from provider (uses ES2022 Error.cause)
}
```

The top-level `Error.message` for multi-error types summarizes the count: e.g. `"Config invalid: 3 errors"` for `ConfigError`, `"Validation failed: 5 errors"` for `ValidationError`. Individual `errors[].message` provides per-issue detail.

`MultiDbError.toJSON()` returns a plain object safe for `JSON.stringify()`, structured loggers, and HTTP error responses. It recursively serializes `cause` (if present) and `errors[]` arrays, so all diagnostic data survives transport. Standard `Error` objects lose all custom fields during `JSON.stringify()` — `toJSON()` ensures `code`, `details`, `message`, and `cause` are always included.

`ConfigError` is thrown at init time and during `reloadMetadata()` — it collects **all** config issues (invalid apiNames, duplicate names, broken references, broken relations, broken sync references, invalid cache configs) into a single error with an `errors[]` array, same philosophy as `ValidationError`. Each error's `details` includes `database` / `cacheId` when relevant, so the caller knows exactly which database or cache config is broken. `ConnectionError` is thrown at init time when executor/cache pings fail — conceptually distinct from config validation (config is correct, infrastructure is unreachable). Each unreachable entry carries `engine` (postgres/clickhouse/trino/redis) and `cause?: Error` to preserve the original ping failure (stack trace, message). `ValidationError` is thrown per query — it collects **all** validation issues into a single error with an `errors[]` array, so callers can see every problem at once. It carries `fromTable` to correlate the error to the specific query. Individual errors include `filterIndex` (zero-based) to identify which filter caused the issue, and `refColumn` / `refTable` for `QueryColumnFilter` errors. `INVALID_VALUE` is used for structurally malformed compound values (missing `to` in `between`, negative `maxDistance` in `levenshteinLte`, empty array in `in`) — distinct from `INVALID_FILTER` which covers type/operator mismatches. `PlannerError` is thrown when no execution strategy can satisfy the query — `details` is a discriminated union keyed by `code`, so each variant carries only its relevant fields. It also carries `fromTable` to correlate to the query. `ExecutionError` is thrown during SQL execution or cache access — `details` is a discriminated union keyed by `code` (`QUERY_FAILED` includes `sql` + `params` + `dialect` + `cause`, `QUERY_TIMEOUT` includes `sql` + `timeoutMs` + `dialect`, `EXECUTOR_MISSING` includes `database`, `CACHE_PROVIDER_MISSING` includes `cacheId`). The `dialect` field in `QUERY_FAILED` and `QUERY_TIMEOUT` tells the caller which engine failed — essential when debugging Trino cross-catalog queries vs direct Postgres/ClickHouse. `ProviderError` is thrown when `MetadataProvider.load()` or `RoleProvider.load()` fails — at init time or during `reloadMetadata()` / `reloadRoles()`. Both `ExecutionError` and `ProviderError` use ES2022 `Error.cause` to chain the original error instead of a custom field.

---

## apiName Rules

All `apiName` values (tables and columns) must follow these rules:

| Rule | Constraint |
|---|---|
| Format | `^[a-z][a-zA-Z0-9]*$` (camelCase) |
| Length | 1–64 characters |
| Reserved words | Cannot be: `from`, `select`, `where`, `having`, `limit`, `offset`, `order`, `group`, `join`, `distinct`, `exists`, `null`, `true`, `false`, `and`, `or`, `not`, `in`, `like`, `as`, `on`, `by`, `asc`, `desc`, `count`, `sum`, `avg`, `min`, `max` |
| Uniqueness | Table apiNames must be globally unique; column apiNames must be unique within their table |

camelCase is natural in TypeScript and visually distinct from the snake_case used in physical databases — making it obvious which layer you're working in.

The system validates all apiNames at metadata registration time and rejects invalid names with descriptive errors.

---

## Name Mapping

Every table and column has:
- `apiName` — what the user provides in queries and receives in results
- `physicalName` — what's actually in the database

The system:
1. Accepts queries using `apiName`
2. Resolves to `physicalName` for SQL generation
3. Returns results mapped back to `apiName`

This decouples the API contract from database schema evolution.

---

## SQL Dialect Differences

Per-dialect SQL generation: identifier quoting, parameter binding, `in`/`notIn`, ILIKE, BETWEEN, array operators, counted subqueries, type mappings, and the `SqlParts` intermediate representation.

→ See [SQL_DIALECTS.md](./SQL_DIALECTS.md)
---

## Debug Logging — Example Flow

```
[validation]  Resolving table 'orders' → found in metadata (database: pg-main)
[validation]  Column 'id' → valid (type: uuid)
[validation]  Column 'total' → valid (type: decimal)
[validation]  Columns not explicitly listed — deferring full set to access control
[access-control] Role 'tenant-user': allowed columns [id, total, status, createdAt]
[access-control] Trimming result set to allowed columns
[access-control] Masking column 'total' for roles [tenant-user]
[planning]    Tables needed: [orders(pg-main)]
[planning]    All tables in 'pg-main' → strategy: DIRECT
[name-resolution] orders.total → public.orders.total_amount
[name-resolution] orders.tenantId → public.orders.tenant_id
[sql-generation]  Dialect: postgres
[sql-generation]  SELECT t0."id", t0."total_amount", ... FROM "public"."orders" t0 WHERE t0."tenant_id" = $1
[execution]   Executing on pg-main → 42 rows, 8ms
[execution]   Mapping results: total_amount → total (via ColumnMapping)
```

For cross-database scenario:
```
[planning]    Tables needed: [orders(pg-main), events(ch-analytics)]
[planning]    Not in same database. Checking materializations...
[planning]    Found: orders → ch-analytics (debezium, lag: seconds)
[planning]    Query freshness: 'minutes' → materialized replica acceptable
[planning]    Strategy: MATERIALIZED via ch-analytics
[planning]      orders → default.orders_replica (materialized)
[planning]      events → default.events (original)
[sql-generation]  Dialect: clickhouse
```

---

## Recommended Test Configuration

Test databases, tables, columns, relations, roles, and the full test scenario catalog (~240 scenarios across validation, init, access, planner, generator, cache, e2e, client, and contract test suites).

→ See [TESTS.md](./TESTS.md)
---

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Package name | `@mkven/multi-db-query` | Org-scoped, reusable |
| Language | TypeScript | Type-safe, wide ecosystem |
| Query format | Typed object literals | Type-safe, IDE support, no parser |
| Metadata source | Provider-based (always). Static helpers `staticMetadata()` / `staticRoles()` for simple cases | Uniform API — `reload` always works; no static/dynamic split |
| Execution mode | SQL-only (`SqlResult`), execution (`DataResult`), or count (`CountResult`) | Distinct return types per mode |
| Freshness | Prefer original, allow specifying lag tolerance | Correctness by default, performance opt-in |
| Access control | Scoped roles: UNION within scope, INTERSECTION between scopes | User accumulates perms, service restricts them |
| SQL generation | Hand-rolled via `SqlParts` IR (internal, physical names only), no external packages | Full control over 3 divergent dialects, zero deps |
| Pagination | Offset-based (`limit` + `offset`) | Simple, sufficient for most use cases |
| Name mapping | apiName ↔ physicalName on tables and columns | Decouples API from schema |
| Cache | Redis synced by Debezium, no TTL | Always fresh, fast by-ID reads |
| Debug logging | Structured entries per pipeline phase, opt-in via `debug: true` | Zero overhead when not debugging |
| Validation | Strict — only metadata-defined entities | Fail fast with clear errors |
| Join results | Flat denormalized rows (no nesting) | `limit` applies to DB rows; with one-to-many joins, `limit: 10` may yield fewer than 10 parent entities. Nesting would require a two-query approach (fetch parent IDs first, then children), breaking the single-query model. Callers can group results using `meta.columns[].fromTable` |
| Filter logic | Recursive `QueryFilterGroup` with `and`/`or` and optional `not` | Top-level filters array is implicit AND; use `QueryFilterGroup` for OR, nested combinations, or negation (`not: true` → `NOT (...)`). Mirrors to `WhereGroup` in `SqlParts` IR. Simple queries stay simple, complex logic is opt-in |
| Exists filters | `QueryExistsFilter` with correlated subquery, optional `count` | Leverages existing relation metadata for `EXISTS`/`NOT EXISTS`. No implicit JOIN — keeps result set clean. Access control applied to the related table. Mirrors to `WhereExists` / `WhereCountedSubquery` in `SqlParts` IR (both share `CorrelatedSubquery`). Optional `count` changes from boolean EXISTS to a counted comparison (`(SELECT COUNT(*) ...) >= N`). `count: { operator: '>=', value: 1 }` is equivalent to plain `exists: true` — prefer the simpler form. For `>=` / `>`, system may add `LIMIT N` inside the subquery to short-circuit counting |
| Filter table qualifier | Optional `table?` on `QueryFilter` | Allows filtering on joined-table columns at top level without using `QueryJoin.filters`. Resolves ambiguity when `from` and joined tables share a column apiName |
| Imports | Absolute paths, no `../` | Clean, refactor-friendly |
| Column disambiguation | Qualified apiNames in result rows: `orders.id`, `products.id` when join produces collisions | Flat rows use `Record<string, unknown>` — keys must be unique. When the `from` table and a joined table share a column apiName (e.g. both have `id`), the result row qualifies colliding keys with `{tableApiName}.{columnApiName}`. Non-colliding columns keep their bare apiName. `meta.columns[].apiName` reflects the actual key used in the result row (qualified if colliding). SQL generation aliases columns via `AS "{tableAlias}__{physicalName}"` internally (e.g. `AS "t0__total_amount"`), then the pipeline's `remapRows()` step uses `ColumnMapping[]` to translate those internal aliases to the final apiName keys (`total`, or `orders.id` / `products.id` when colliding) |
| Executor timeout | Per-executor `timeoutMs` in factory config, no global default | Each DB has different performance profiles — Postgres: 30s, ClickHouse: 60s, Trino: 120s. Driver-level enforcement (`statement_timeout`, `max_execution_time`) is more reliable than `Promise.race` |
| Concurrent safety | `query()` uses snapshot of metadata/roles; `reload*()` atomically swaps references | No locking needed — immutable config per query, atomic reference swap for reloads |
| `close()` error handling | Attempt all providers, collect failures, throw aggregate error | Partial close would leak connections — always try all, report all failures |
| Array columns | `ScalarColumnType` + `ArrayColumnType` union, 5 array operators, element type derived by stripping `[]` | All three backends (Postgres, ClickHouse, Trino/Iceberg) support arrays natively. Element type validation reuses `ScalarColumnType`. Array columns excluded from `QueryColumnFilter`, `sum`/`avg`/`min`/`max` aggregations, `groupBy`, and `orderBy` (dialect-inconsistent behavior) |
| HTTP client | Thin typed wrapper over `fetch`; contract tests shared with in-process engine | Same test suite verifies both `MultiDb` (direct) and `MultiDbClient` (HTTP) — catches serialization drift, error mapping mismatches, and behavioral divergence. Native `fetch` = zero HTTP deps, works in Node 18+, Bun, Deno, browsers |
| Contract testing | Parameterized test suite with factory function per implementation | Write once, run against any `QueryContract` — avoids duplicating integration tests for each transport |

---

## Monorepo Packages

| Package | Purpose | Dependencies |
|---|---|---|
| `@mkven/multi-db-validation` | Types, error classes, config validation, query validation (rules 1–14), apiName validation | **zero** I/O deps |
| `@mkven/multi-db-query` | Core: metadata registry, planner, SQL generators, name resolution, masking, debug logger | `@mkven/multi-db-validation` |
| `@mkven/multi-db-executor-postgres` | Postgres connection + execution | `pg` |
| `@mkven/multi-db-executor-clickhouse` | ClickHouse connection + execution | `@clickhouse/client` |
| `@mkven/multi-db-executor-trino` | Trino connection + execution | `trino-client` |
| `@mkven/multi-db-cache-redis` | Redis cache provider (Debezium-synced) | `ioredis` |
| `@mkven/multi-db-client` | Typed HTTP client + contract test suite | `@mkven/multi-db-validation` (types, errors; uses native `fetch`) |

All error classes (including runtime ones like `ExecutionError`, `PlannerError`, `ConnectionError`) live in the validation package so that client code can use `instanceof` checks and access typed error fields without depending on the core package. The validation package is a type+error+validation-only package — it contains no I/O, no planner, no SQL generators.

`@mkven/multi-db-validation` is the **client-side** package — it contains all types, error classes, and validation logic. Clients can validate configs and queries locally before sending to the server, failing fast without pulling in planner/SQL generators. The core package depends on it and re-exports its types.

```ts
import { validateQuery, validateConfig, validateApiName } from '@mkven/multi-db-validation'
import type { QueryDefinition, MetadataConfig, ExecutionContext } from '@mkven/multi-db-validation'

// At config time — validate metadata before shipping to server
const configErrors = validateConfig(metadata)       // returns ConfigError | null

// At query time — validate query before sending
const index = new MetadataIndex(metadata, roles)    // pre-index once
const queryErrors = validateQuery(definition, context, index, roles)  // returns ValidationError | null
```

Both `validateConfig()` and `validateQuery()` return `null` on success, or the corresponding error object (not thrown) so the caller can decide how to handle it. The core package still throws these errors internally.

`validateQuery()` always requires a pre-indexed `MetadataIndex` (exported from the package). Callers pre-index metadata once via `new MetadataIndex(config, roles)` and reuse it across calls — no hidden per-call indexing.

```ts
// Pre-indexed metadata for hot-path validation
interface MetadataIndex {
  tablesByApiName: Map<string, TableMeta>
  tablesById: Map<string, TableMeta>       // tableId → TableMeta
  columnsByTable: Map<string, Map<string, ColumnMeta>>  // tableId → columnApiName → ColumnMeta
  databasesById: Map<string, DatabaseMeta> // databaseId → DatabaseMeta
  rolesById: Map<string, RoleMeta>
}
```

Core has **zero I/O dependencies** — usable for SQL-only mode without any DB drivers. Each executor is a thin adapter that consumers install only if needed.

---

## HTTP Client & Contract Testing

HTTP API contract (endpoints, error status codes), `MultiDbClient` configuration, error deserialization, local validation, and the contract test suite (`QueryContract` interface).

→ See [HTTP_CLIENT.md](./HTTP_CLIENT.md)
---

## Project Structure

```
@mkven/multi-db/                     # monorepo root
├── package.json                     # workspace config
├── pnpm-workspace.yaml
├── tsconfig.json                    # shared tsconfig
├── biome.json
├── vitest.config.ts
├── README.md
│
├── packages/
│   ├── validation/                  # @mkven/multi-db-validation
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── index.ts             # public API: validateQuery, validateConfig, validateApiName + re-exports
│   │   │   ├── errors.ts            # MultiDbError, ConfigError, ConnectionError, ValidationError, PlannerError, ExecutionError, ProviderError
│   │   │   ├── metadataIndex.ts     # MetadataIndex — pre-indexed metadata for O(1) lookups
│   │   │   ├── types/
│   │   │   │   ├── metadata.ts      # DatabaseMeta, TableMeta, ColumnMeta, RelationMeta, ExternalSync, CacheMeta, RoleMeta
│   │   │   │   ├── query.ts         # QueryDefinition, QueryFilter, QueryJoin, QueryAggregation, etc.
│   │   │   │   ├── result.ts        # QueryResult, QueryResultMeta, DebugLogEntry
│   │   │   │   └── context.ts       # ExecutionContext
│   │   │   └── validation/
│   │   │       ├── configValidator.ts   # validateConfig — apiName format, uniqueness, references, relations, syncs, caches
│   │   │       ├── queryValidator.ts    # validateQuery — rules 1–14 against metadata + roles
│   │   │       └── rules.ts             # per-rule validation logic (table, column, filter, join, etc.)
│   │   └── tests/
│   │       ├── fixtures/
│   │       │   └── testConfig.ts     # shared test config (metadata, roles, tables)
│   │       ├── errors.test.ts       # error class construction + toJSON serialization (34 tests)
│   │       ├── config/              # scenarios 49–52, 80, 81, 89, 96
│   │       └── query/               # scenarios 15, 17, 18, 32, 34, 36, 37, 40–43, 46, 47, 65, 78, 82, 86–88, 97, 98, 107, 109, 116–123, 139–141, 143, 145, 146, 150, 151, 153, 154, 157–159, 165, 167–169, 173–180, 187, 190–192, 195, 198, 199, 229–232, 234, 235
│   │
│   ├── core/                        # @mkven/multi-db-query
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts             # public API (re-exports types + errors from validation package)
│   │       ├── pipeline.ts          # createMultiDb, query pipeline orchestration, result building
│   │       ├── types/
│   │       │   ├── ir.ts            # SqlParts, WhereNode, ColumnMapping, CorrelatedSubquery (dialect-agnostic IR)
│   │       │   ├── interfaces.ts    # DbExecutor, CacheProvider interfaces
│   │       │   └── providers.ts     # MetadataProvider, RoleProvider interfaces
│   │       ├── metadata/
│   │       │   ├── registry.ts      # MetadataRegistry — stores and indexes all metadata
│   │       │   └── providers.ts     # static helpers (staticMetadata, staticRoles)
│   │       ├── access/
│   │       │   └── access.ts        # role-based column trimming
│   │       ├── masking/
│   │       │   └── masking.ts       # post-query masking (email, phone, name, etc.)
│   │       ├── planner/
│   │       │   ├── planner.ts       # QueryPlanner — strategy selection
│   │       │   ├── strategies.ts    # direct, materialized, trino, cache
│   │       │   └── graph.ts         # database connectivity / sync graph
│   │       ├── dialects/
│   │       │   ├── dialect.ts       # SqlDialect interface
│   │       │   ├── postgres.ts
│   │       │   ├── clickhouse.ts
│   │       │   └── trino.ts
│   │       ├── resolution/
│   │       │   └── resolver.ts      # apiName → physicalName, produces SqlParts + ColumnMapping[]
│   │       ├── generator/
│   │       │   ├── generator.ts     # SQL generation from resolved plan
│   │       │   └── fragments.ts     # reusable SQL building blocks
│   │       └── debug/
│   │           └── logger.ts        # DebugLogger — collects entries per query
│   │   └── tests/
│   │       ├── init/                # scenarios 53, 54, 55, 63
│   │       ├── access/              # scenarios 13, 14, 14b–14f, 16, 38, 95, 104, 106, 233
│   │       ├── planner/             # scenarios 1–12, 19, 33, 56, 57, 59, 64, 79, 103, 130
│   │       ├── generator/           # scenarios 20–30, 45, 66–75, 77, 83–85, 90–94, 99–102, 108, 110–115, 124–129, 133–138, 142, 144, 147–149, 155–157, 160–164, 166, 181–186, 188–189, 193, 194, 196, 197, 200–207, 227
│   │       └── e2e/                 # scenarios 14e, 31, 35, 39, 44, 48, 58, 60–62, 76, 105, 131, 132, 152, 170–172, 228
│   │
│   ├── client/                      # @mkven/multi-db-client
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts             # public API: createMultiDbClient, MultiDbClient, MultiDbClientConfig
│   │       ├── client.ts            # HTTP client implementation (fetch-based)
│   │       ├── errors.ts            # error deserialization — toJSON() → typed error class reconstruction
│   │       └── contract/
│   │           └── queryContract.ts  # describeQueryContract — parameterized contract test suite
│   │   └── tests/
│   │       ├── client/              # scenarios 208–218, 226
│   │       └── contract/            # scenarios 219–225, 236–238
│   │
│   ├── executor-postgres/           # @mkven/multi-db-executor-postgres
│   │   ├── package.json
│   │   └── src/
│   │       └── index.ts
│   │
│   ├── executor-clickhouse/         # @mkven/multi-db-executor-clickhouse
│   │   ├── package.json
│   │   └── src/
│   │       └── index.ts
│   │
│   ├── executor-trino/              # @mkven/multi-db-executor-trino
│   │   ├── package.json
│   │   └── src/
│   │       └── index.ts
│   │
│   └── cache-redis/                 # @mkven/multi-db-cache-redis
│       ├── package.json
│       └── src/
│           └── index.ts
```

---

## Open Items for Further Discussion

- [ ] Builder pattern API on top of object literals?
- [ ] Write operations (INSERT/UPDATE/DELETE) — future scope?
- [ ] Connection pool configuration and lifecycle
- [ ] Trino session properties for optimization
- [ ] Schema migration strategy for metadata storage
- [ ] How to handle schema drift between original and replicated tables
- [ ] RLS (row-level security) — deferred, may add later
- [ ] Cursor-based pagination as alternative to offset?
- [ ] Custom masking functions beyond predefined set?
- [ ] Nested/grouped results for one-to-many joins? (requires two-query approach: fetch parent IDs with limit, then fetch all children — significant complexity)
- [ ] OpenTelemetry integration — spans per pipeline phase, traces + metrics
- [ ] Concurrent reload ordering — snapshot isolation is chosen (see Design Decisions), but what happens when two `reloadMetadata()` calls overlap? Does the second reload see the first's changes? Should reloads be serialized internally?
- [ ] Automatic tenant filtering via `ExecutionContext.tenantId`? — auto-inject `WHERE tenant_id = ?` without callers adding explicit filter; needs metadata annotation for which column is the tenant column per table
- [ ] Window functions (ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD) — useful for analytics; ClickHouse + Trino syntax differs from Postgres
- [ ] Batch queries — accept multiple `QueryDefinition` objects in a single call, return results in order; useful for dashboards
- [ ] Computed/virtual columns — derived expressions (e.g. `total * quantity`) defined in metadata, projected as regular columns
