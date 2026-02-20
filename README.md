# @mkven/multi-db — Concept Document

## Objective

Build a reusable, metadata-driven query engine that lets applications query data across Postgres, ClickHouse, Iceberg, and Redis through a single typed API. The engine:

- Accepts queries using **apiNames** (decoupled from physical schema)
- Automatically selects the **optimal execution strategy** — direct DB, cached, materialized replica, or Trino cross-DB federation
- Enforces **scoped access control** — user roles and service roles intersected to determine effective permissions
- Applies **column masking** for sensitive data based on role
- Returns either **generated SQL** or **executed results** depending on the caller's needs
- Provides **structured debug logs** for transparent pipeline tracing

The package (`@mkven/multi-db`) is built as a standalone, reusable library with zero I/O dependencies in the core.

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
│  - filter operator validity         │
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

### Databases

```ts
type DatabaseEngine = 'postgres' | 'clickhouse' | 'iceberg'

interface DatabaseMeta {
  id: string                          // logical id: 'pg-main', 'ch-analytics'
  engine: DatabaseEngine
  trinoCatalog?: string               // if accessible via trino, its catalog name
}
```

### Tables

```ts
interface TableMeta {
  id: string                          // logical table id: 'orders'
  apiName: string                     // exposed to API consumers: 'orders'
  database: string                    // references DatabaseMeta.id
  physicalName: string                // actual table name in DB: 'public.orders'
  columns: ColumnMeta[]
  primaryKey: string[]                // using apiName references
  relations: RelationMeta[]
}
```

### Columns

```ts
interface ColumnMeta {
  apiName: string                     // 'customerEmail'
  physicalName: string                // 'customer_email'
  type: string                        // logical type: 'string', 'int', 'date', 'uuid', 'decimal', 'timestamp', etc.
  nullable: boolean
  maskingFn?: 'email' | 'phone' | 'name' | 'uuid' | 'number' | 'date' | 'full'
                                      // how to mask this column when masking is applied by role
}
```

### Relations

Relations are defined on the table that holds the FK column. When resolving joins, the system performs **bidirectional lookup** — if a query says `from: 'orders', joins: [{ table: 'invoices' }]`, both `ordersRelations` and `invoicesRelations` are checked to find a connection.

```ts
interface RelationMeta {
  column: string                      // apiName of FK column in this table
  references: { table: string; column: string }  // apiName references
  type: 'many-to-one' | 'one-to-many' | 'one-to-one'
}
```

### External Sync (Debezium CDC)

```ts
interface ExternalSync {
  sourceTable: string                 // TableMeta.id of the original
  targetDatabase: string              // DatabaseMeta.id where materialized
  targetPhysicalName: string          // physical name in target DB
  method: 'debezium'
  estimatedLag: 'realtime' | 'seconds' | 'minutes' | 'hours'
}
```

### Cache

```ts
interface CacheMeta {
  id: string
  engine: 'redis'                     // extensible later
  tables: CachedTableMeta[]
}

interface CachedTableMeta {
  tableId: string                     // references TableMeta.id
  keyPattern: string                  // e.g. 'users:{id}' — placeholders reference PK column apiNames
  columns?: string[]                  // subset cached (apiNames); undefined = all
}
```

### Roles & Access Control

Access control is defined **per role**, not per table. Roles themselves are scope-agnostic — scoping is determined at query time via `ExecutionContext`. The effective permissions are computed as:

1. **Within a scope** — UNION (most permissive). If one role allows columns A,B and another allows C,D, the effective set is A,B,C,D. This is natural: a user with multiple user-roles accumulates permissions. **Masking follows the same UNION rule** — if *any* role in the scope provides unmasked access to a column, the column is unmasked. A more privileged role overrides masking from a less privileged one.
2. **Between scopes** — INTERSECTION (most restrictive). The effective permissions are the intersection of all scope unions. Masking from any scope is preserved — if one scope unmasks a column but another masks it, the column stays masked.

This ensures that an admin user making a request through a service with restricted access only sees data the service is permitted to handle.

**Example:** Admin user (user scope: all tables, all columns) requests via `orders-service` (service scope: only `orders` + `products` tables). Effective access = only `orders` + `products`.

**Example 2:** User has roles `viewer` (orders access) + `analyst` (events access) in user scope, and `orders-service` in service scope. User scope union = orders + events. Service scope = orders + products. Intersection = orders only.

```ts
interface RoleMeta {
  id: string                          // 'admin', 'viewer', 'orders-service'
  tables: TableRoleAccess[] | '*'     // '*' = all tables, all columns, no masking
}

interface TableRoleAccess {
  tableId: string
  allowedColumns: string[] | '*'      // apiNames, or '*' for all
  maskedColumns?: string[]            // apiNames of columns to mask
}
```

### Column Masking

Masking replaces column values in results with obfuscated variants. Masking is a **post-query** operation — data is fetched normally from the database, then masked before returning to the caller.

Which masking function to apply is defined on the **column metadata** via the optional `maskingFn` field. The role just declares *which* columns to mask — the column itself knows *how* to be masked.

Predefined masking functions:

| maskingFn | Masking Behavior | Example |
|---|---|---|
| `email` | Show first char + domain hint | `john@example.com` → `j***@***.com` |
| `phone` | Show country code + last 3 digits | `+1234567890` → `+1***890` |
| `name` | Show first + last char | `John Smith` → `J*********h` |
| `uuid` | Show first 4 chars | `a1b2c3d4-...` → `a1b2****` |
| `number` | Replace with `0` | `12345` → `0` |
| `date` | Truncate to year | `2025-03-15` → `2025-01-01` |
| `full` | Replace entirely | `anything` → `***` |

If a column is declared as masked by a role but has no `maskingFn` in metadata, the `full` masking function is applied as a safe default.

Masked columns are still returned in results (not hidden), but their values are obfuscated.

### Full Configuration

```ts
interface MultiDbConfig {
  databases: DatabaseMeta[]
  tables: TableMeta[]
  caches: CacheMeta[]
  externalSyncs: ExternalSync[]
  roles: RoleMeta[]
  trino?: { enabled: boolean }
}
```

Metadata source is **abstracted** — hardcoded for now, in future loaded from a database or external service and cached.

### Metadata & Role Providers

The config can be provided statically or loaded dynamically via providers:

```ts
interface MetadataProvider {
  load(): Promise<Omit<MultiDbConfig, 'roles'>>
}

interface RoleProvider {
  load(): Promise<RoleMeta[]>
}
```

Providers are optional — if omitted, the system uses the static `config` object. Both providers support cache invalidation:

```ts
interface MultiDb {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>

  reloadMetadata(): Promise<void>     // re-calls MetadataProvider.load(), rebuilds indexes
  reloadRoles(): Promise<void>        // re-calls RoleProvider.load(), rebuilds role map

  healthCheck(): Promise<HealthCheckResult>  // checks all executors + cache providers
}

interface HealthCheckResult {
  healthy: boolean                    // true only if ALL checks pass
  executors: Record<string, { healthy: boolean; latencyMs: number; error?: string }>
  cacheProviders: Record<string, { healthy: boolean; latencyMs: number; error?: string }>
}
```

External systems signal staleness by calling `reloadMetadata()` or `reloadRoles()`. This is explicit — no polling, no TTL. Typical triggers: admin UI saves new config, CI/CD deploys schema changes, webhook from config service.

```ts
const multiDb = createMultiDb({
  // Static config OR providers (not both)
  configProvider: myMetadataProvider,
  roleProvider: myRoleProvider,

  executors: { ... },
  cacheProviders: { ... },
})

// Later, when external system signals config change:
await multiDb.reloadMetadata()
```

### Module Return Type

```ts
interface MultiDb {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>
}
```

### Executor & Cache Provider Interfaces

Each executor and cache provider must implement a minimal interface:

```ts
// Implemented by each executor package (postgres, clickhouse, trino)
interface DbExecutor {
  execute(sql: string, params: unknown[]): Promise<Record<string, unknown>[]>
  ping(): Promise<void>              // lightweight connectivity check (e.g. SELECT 1)
  close(): Promise<void>
}

// Implemented by each cache provider package (redis)
interface CacheProvider {
  getMany(keys: string[]): Promise<Map<string, Record<string, unknown> | null>>
  ping(): Promise<void>              // lightweight connectivity check (e.g. Redis PING)
  close(): Promise<void>
}
```

---

## Module Initialization & Query API

### Initialization (once at startup)

The module is initialized with the full metadata configuration and optional executor/cache providers:

```ts
import { createMultiDb } from '@mkven/multi-db'
import { createPostgresExecutor } from '@mkven/multi-db-executor-postgres'
import { createClickHouseExecutor } from '@mkven/multi-db-executor-clickhouse'
import { createTrinoExecutor } from '@mkven/multi-db-executor-trino'
import { createRedisCache } from '@mkven/multi-db-cache-redis'

// Returns MultiDb instance
const multiDb = createMultiDb({
  // Required: metadata configuration
  config: {
    databases: [...],
    tables: [...],
    caches: [...],
    externalSyncs: [...],
    roles: [...],
    trino: { enabled: true },
  },

  // Optional: executors (only needed for executeMode = 'execute' | 'count')
  // Keys must match DatabaseMeta.id, except 'trino' which is a special key for the federation layer
  executors: {
    'pg-main': createPostgresExecutor({ connectionString: '...' }),
    'ch-analytics': createClickHouseExecutor({ url: '...' }),
    'trino': createTrinoExecutor({ url: '...' }),     // special key — not a DatabaseMeta.id
  },

  // Optional: cache providers (keys must match CacheMeta.id)
  cacheProviders: {
    'redis-main': createRedisCache({ url: '...' }),
  },
})
```

At init time:
- All apiNames are validated (format, reserved words, uniqueness)
- Metadata is indexed into in-memory Maps for O(1) lookups:
  - `Map<apiName, TableMeta>` — table by apiName
  - `Map<tableId, Map<apiName, ColumnMeta>>` — column by table + apiName
  - `Map<roleId, RoleMeta>` — role by id
  - `Map<databaseId, DatabaseMeta>` — database by id
  - `Map<tableId, ExternalSync[]>` — syncs per table
  - `Map<tableId, CachedTableMeta>` — cache config per table
- Database connectivity graph is built (for planner)
- All executors and cache providers are **pinged** to verify connectivity (calls `ping()` on each). If any fail, a `ConfigError` with code `CONNECTION_FAILED` is thrown. Disable with `validateConnections: false` for lazy connection scenarios
- Configuration errors are thrown immediately

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
| Metadata config (databases, tables, roles, syncs, caches) | Query definition (from, columns, filters, joins, etc.) |
| Executor instances (DB connections) | Execution context (scoped roles) |
| Cache provider instances | `executeMode`, `debug`, `freshness` |

---

## Query Definition

### Query Input

```ts
interface QueryDefinition {
  from: string                        // table apiName
  columns?: string[]                  // apiNames; undefined = all allowed for role
  distinct?: boolean                  // SELECT DISTINCT (default: false)
  filters?: QueryFilter[]
  joins?: QueryJoin[]
  groupBy?: QueryGroupBy[]            // columns to group by
  aggregations?: QueryAggregation[]   // aggregate functions
  having?: QueryFilter[]              // filters on aggregated values (applied after GROUP BY)
                                      // column references aggregation aliases, not table columns
  limit?: number
  offset?: number
  orderBy?: QueryOrderBy[]
  freshness?: 'realtime' | 'seconds' | 'minutes' | 'hours'  // acceptable lag
  byIds?: (string | number)[]        // shortcut: fetch by single-column primary key(s)
  executeMode?: 'sql-only' | 'execute' | 'count'  // default: 'execute'
  debug?: boolean                     // include debugLog in result (default: false)
}

interface QueryAggregation {
  column: string | '*'                // apiName of column, or '*' for count(*)
  table?: string                      // apiName of table; omit for `from` table (ignored when column is '*')
  fn: 'count' | 'sum' | 'avg' | 'min' | 'max'
  alias: string                       // result column name
}

interface QueryOrderBy {
  column: string                      // apiName
  table?: string                      // apiName of table; omit for `from` table
  direction: 'asc' | 'desc'
}

interface QueryGroupBy {
  column: string                      // apiName
  table?: string                      // apiName of table; omit for `from` table
}

interface QueryJoin {
  table: string                       // related table apiName
  type?: 'inner' | 'left'            // default: 'left' (safe for nullable FKs)
  columns?: string[]                  // columns to select from joined table
  filters?: QueryFilter[]            // filters on joined table
}

interface QueryFilter {
  column: string                      // apiName
  operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'in' | 'not_in' | 'like' | 'not_like' | 'is_null' | 'is_not_null'
  value?: unknown                     // scalar for most operators; array for 'in'/'not_in'; omit for 'is_null'/'is_not_null'
}
```

### Execution Context

```ts
interface ExecutionContext {
  roles: {                            // scoped role lists
    user?: string[]                   // user-level roles (union within scope)
    service?: string[]                // service-level roles (union within scope)
  }                                   // between scopes: intersection
}
```

Roles within a scope are unioned (accumulated permissions). The final effective permissions are the intersection of all scope unions. If a scope is omitted, it imposes no restriction (treated as "all access" for that scope).

### Query Result

Two distinct return types depending on `executeMode`:

```ts
// When executeMode = 'sql-only'
interface SqlResult {
  kind: 'sql'                        // discriminant for union
  sql: string                        // generated SQL
  params: unknown[]                  // bound parameters
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// When executeMode = 'execute' (default)
interface DataResult<T = unknown> {
  kind: 'data'                       // discriminant for union
  data: T[]                          // actual query results (masked if applicable)
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// When executeMode = 'count'
interface CountResult {
  kind: 'count'                      // discriminant for union
  count: number                      // total matching rows
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// Discriminated union
type QueryResult<T = unknown> = SqlResult | DataResult<T> | CountResult

interface QueryResultMeta {
  strategy: 'direct' | 'cache' | 'materialized' | 'trino-cross-db'
  targetDatabase: string             // which DB was queried; for cache strategy: the CacheMeta.id (e.g. 'redis-main')
  dialect?: 'postgres' | 'clickhouse' | 'trino'  // omitted for cache-only hits; iceberg is always queried via trino
  tablesUsed: {
    tableId: string
    source: 'original' | 'materialized' | 'cache'
    database: string
    physicalName: string
  }[]
  columns: {
    apiName: string
    type: string
    nullable: boolean
    fromTable: string                // table apiName this column came from
    masked: boolean                  // whether this column was masked
  }[]
  timing: {
    planningMs: number
    generationMs: number
    executionMs?: number             // only in DataResult / CountResult
  }
}

interface DebugLogEntry {
  timestamp: number
  phase: 'validation' | 'access-control' | 'planning' | 'name-resolution' | 'sql-generation' | 'cache' | 'execution'
  message: string
  details?: unknown                  // structured data for inspection
}
```

When you request execution (`executeMode = 'execute'`), you get data back — no SQL. When you request SQL only (`executeMode = 'sql-only'`), you get SQL + params — no execution, no data. When you request count (`executeMode = 'count'`), you get just the row count — `columns`, `orderBy`, `limit`, and `offset` are ignored. All modes include metadata. Debug log is included only when `debug: true`.

In `sql-only` mode, masking cannot be applied (no data to mask). However, `meta.columns[].masked` still reports masking intent so the caller can apply masking themselves after execution.

---

## Query Planner Strategy

Given a query touching tables T1, T2, ... Tn:

### Priority 0 — Cache (redis)
- Only for `byIds` queries **without `filters`** (filters skip cache — too ambiguous to post-filter)
- Only for single-table queries **without `joins`** (cache stores single-table data)
- Only for tables with single-column primary keys (composite PKs use filters instead)
- Check if the table has a cache config
- If cache hit for all IDs → return from cache (trim columns if needed, apply masking identically to DB results)
- If partial hit → return cached + fetch missing from DB, merge

### Priority 1 — Single Database Direct
- ALL required tables exist in ONE database (original data)
- Generate native SQL for that engine
- This is always preferred when possible (freshest data, no overhead)

### Priority 2 — Materialized Replica
- Some tables are in different databases, BUT debezium replicas exist such that all needed data is available in one database
- Check freshness: if query requires `realtime` but replica lag is `minutes` → skip this strategy
- Always prefer the **original** database if the original data is there; use replicas for the "foreign" tables
- If multiple databases could serve via replicas, prefer the one with the most original tables

### Priority 3 — Trino Cross-Database
- Trino is enabled and all databases are registered as trino catalogs
- Generate Trino SQL with cross-catalog references
- Fallback when no single-DB path exists

### Priority 4 — Error
- Cannot reach all tables via any strategy
- Return clear error: which tables are unreachable, what's missing (trino disabled? no catalog? no sync?)

---

## Validation Rules

1. **Table existence** — only tables defined in metadata can be queried
2. **Column existence** — only columns defined in table metadata can be referenced
3. **Role permission** — if a table is not in the role's `tables` list → access denied
4. **Column permission** — if `allowedColumns` is a list and requested column is not in it → denied; if columns not specified in query, return only allowed ones
5. **Filter validity** — filter operators must be valid for the column type
6. **Join validity** — joined tables must have a defined relation in metadata
7. **Group By validity** — if `groupBy` or `aggregations` are present, every column in `columns` that is not an aggregation alias must appear in `groupBy`. Prevents invalid SQL from reaching the database
8. **Having validity** — `having` filters must reference aliases defined in `aggregations`
9. **Order By validity** — `orderBy` must reference columns from `from` table or joined tables
10. **ByIds validity** — `byIds` requires a single-column primary key; cannot combine with `groupBy` or `aggregations`
11. **Limit/Offset validity** — `limit` and `offset` must be non-negative integers when provided

All validation errors are **collected, not thrown one at a time**. The system runs all applicable checks and throws a single `ValidationError` containing every issue found. This lets callers fix all problems at once instead of playing whack-a-mole.

---

## Error Handling

All errors are thrown as typed exceptions (never returned in the result). Error types:

```ts
class MultiDbError extends Error {
  code: string                        // machine-readable error code
}

class ConfigError extends MultiDbError {
  code: 'INVALID_API_NAME' | 'DUPLICATE_API_NAME' | 'INVALID_REFERENCE' | 'INVALID_RELATION' | 'CONNECTION_FAILED'
  details: { entity?: string; field?: string; expected?: string; actual?: string; unreachable?: string[] }
}

class ValidationError extends MultiDbError {
  code: 'VALIDATION_FAILED'           // always this code — individual issues are in `errors`
  errors: {
    code: 'UNKNOWN_TABLE' | 'UNKNOWN_COLUMN' | 'ACCESS_DENIED' | 'INVALID_FILTER' | 'INVALID_JOIN' | 'INVALID_GROUP_BY' | 'INVALID_HAVING' | 'INVALID_ORDER_BY' | 'INVALID_BY_IDS' | 'INVALID_LIMIT'
    message: string
    details: { expected?: string; actual?: string; table?: string; column?: string; role?: string }
  }[]
}

class PlannerError extends MultiDbError {
  code: 'UNREACHABLE_TABLES' | 'TRINO_DISABLED' | 'NO_CATALOG' | 'FRESHNESS_UNMET'
  details: { unreachableTables?: string[]; missingCatalogs?: string[]; requiredFreshness?: string; availableLag?: string }
}

class ExecutionError extends MultiDbError {
  code: 'EXECUTOR_MISSING' | 'CACHE_PROVIDER_MISSING' | 'QUERY_FAILED'
  details: { database?: string; sql?: string; params?: unknown[]; originalError?: Error }
}

class ProviderError extends MultiDbError {
  code: 'METADATA_LOAD_FAILED' | 'ROLE_LOAD_FAILED'
  details: { provider: 'metadata' | 'role'; originalError?: Error }
}
```

`ConfigError` is thrown at init time (invalid apiNames, duplicate names, broken DB/table/relation references). `ValidationError` is thrown per query — it collects **all** validation issues into a single error with an `errors[]` array, so callers can see every problem at once. `PlannerError` is thrown when no execution strategy can satisfy the query. `ExecutionError` is thrown during SQL execution or cache access — for `QUERY_FAILED`, the `sql` and `params` that caused the failure are included for debugging. `ProviderError` is thrown when `MetadataProvider.load()` or `RoleProvider.load()` fails (wraps the original error).

---

## apiName Rules

All `apiName` values (tables and columns) must follow these rules:

| Rule | Constraint |
|---|---|
| Format | `^[a-z][a-zA-Z0-9]*$` (camelCase) |
| Length | 1–64 characters |
| Reserved words | Cannot be: `from`, `select`, `where`, `limit`, `offset`, `order`, `group`, `join`, `null`, `true`, `false`, `and`, `or`, `not`, `in`, `like`, `as`, `on`, `by` |
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

| Feature | Postgres | ClickHouse | Trino |
|---|---|---|---|
| Identifier quoting | `"column"` | `` `column` `` | `"column"` |
| Parameter binding | `$1, $2` | `{p1:Type}` | `?` |
| Arrays | `= ANY($1)` | `arrayJoin` | `UNNEST` |
| Date functions | `date_trunc(...)` | `toStartOfDay(...)` | `date_trunc(...)` |
| LIMIT/OFFSET | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` |
| Boolean | `true/false` | `1/0` | `true/false` |

Each engine gets a `SqlDialect` implementation.

### SQL Generation Architecture

SQL generation uses an intermediate `SqlParts` representation that is **internal** and dialect-agnostic. It operates entirely in physical names — no apiNames. The pipeline is:

```
QueryDefinition → (planner + access control) → name resolution → SqlParts → SqlDialect → { sql, params }
                                                     ↓
                                              ColumnMapping[] (for result mapping)
```

Name resolution produces two outputs:
1. `SqlParts` — purely physical names, used for SQL generation
2. `ColumnMapping[]` — the mapping table used to rename result columns back to apiNames

```ts
// Built during name resolution, used after execution to map results back
interface ColumnMapping {
  physicalName: string                // 'total_amount'
  apiName: string                     // 'total'
  tableAlias: string                  // 't0'
  masked: boolean                     // apply masking after fetch
  type: string                        // logical column type
  maskingFn?: 'email' | 'phone' | 'name' | 'uuid' | 'number' | 'date' | 'full'
                                      // which masking function to apply (from ColumnMeta)
}
```

`SqlParts` is strictly internal — no apiNames, no masking concerns. All column references use `ColumnRef` so each dialect controls quoting:

```ts
// A reference to a column in a specific table — dialect handles quoting
interface ColumnRef {
  tableAlias: string                  // 't0'
  columnName: string                  // 'created_at' — physical name, unquoted
}

interface SqlParts {
  select: ColumnRef[]                 // columns to select
  distinct?: boolean                  // SELECT DISTINCT
  from: TableRef
  joins: JoinClause[]
  where: WhereCondition[]
  groupBy: ColumnRef[]
  having: WhereCondition[]            // filters on aggregated values
  aggregations: AggregationClause[]
  orderBy: OrderByClause[]
  limit?: number
  offset?: number
}

interface OrderByClause {
  column: ColumnRef
  direction: 'asc' | 'desc'
}

interface TableRef {
  physicalName: string                // 'public.orders'
  alias: string                       // 't0'
  catalog?: string                    // for trino: 'pg_main'
}

interface JoinClause {
  type: 'inner' | 'left'
  table: TableRef
  leftColumn: ColumnRef               // e.g. { tableAlias: 't0', columnName: 'customer_id' }
  rightColumn: ColumnRef              // e.g. { tableAlias: 't1', columnName: 'id' }
}

interface WhereCondition {
  column: ColumnRef
  operator: string                    // '=', 'ILIKE', 'ANY', etc. — string (not union) because dialects may emit operators beyond the public QueryFilter set
  paramIndex?: number                 // for parameterized values
  literal?: string                    // for IS NULL, IS NOT NULL
}

interface AggregationClause {
  fn: 'count' | 'sum' | 'avg' | 'min' | 'max'
  column: ColumnRef | '*'             // '*' for count(*)
  alias: string                       // result column name
}
```

Each `SqlDialect` takes a `SqlParts` and produces `{ sql: string, params: unknown[] }`. The dialect resolves each `ColumnRef` with its own quoting rules:
- Postgres: `t0."created_at"`
- ClickHouse: `` t0.`created_at` ``
- Trino: `t0."created_at"`

No external SQL generation packages are used — the query shape is predictable (SELECT with optional WHERE/JOIN/GROUP BY/HAVING/ORDER BY/LIMIT/OFFSET) and each dialect is ~200–300 lines.

---

## Debug Logging — Example Flow

```
[validation]  Resolving table 'orders' → found in metadata (database: pg-main)
[validation]  Column 'id' → valid (type: uuid)
[validation]  Column 'total' → valid (type: decimal)
[validation]  Column 'internalNote' → DENIED for role 'tenant-user' (not in allowedColumns)
[access-control] Trimming columns to allowed set: [id, total, status, createdAt]
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
[sql-gen]     Dialect: clickhouse
```

---

## Recommended Test Configuration

### Databases

| ID | Engine | Trino Catalog | Purpose |
|---|---|---|---|
| `pg-main` | postgres | `pg_main` | Primary OLTP |
| `pg-tenant` | postgres | `pg_tenant` | Tenant-specific data |
| `ch-analytics` | clickhouse | `ch_analytics` | Analytics / events |
| `iceberg-archive` | iceberg | `iceberg_archive` | Historical archives |

### Tables

| Table ID | API Name | Database | Physical Name |
|---|---|---|---|
| `users` | users | pg-main | public.users |
| `orders` | orders | pg-main | public.orders |
| `products` | products | pg-main | public.products |
| `tenants` | tenants | pg-tenant | public.tenants |
| `invoices` | invoices | pg-tenant | public.invoices |
| `events` | events | ch-analytics | default.events |
| `metrics` | metrics | ch-analytics | default.metrics |
| `orders-archive` | ordersArchive | iceberg-archive | warehouse.orders_archive |

### External Syncs (Debezium)

| Source | Target DB | Target Physical Name | Lag |
|---|---|---|---|
| orders | ch-analytics | default.orders_replica | seconds |
| orders | iceberg-archive | warehouse.orders_current | minutes |
| tenants | pg-main | replicas.tenants | seconds |

### Cache (Redis, synced by Debezium — no TTL)

| Table | Key Pattern |
|---|---|
| users | `users:{id}` |
| products | `products:{id}` |

### Roles

| Role | Scope Usage | Access |
|---|---|---|
| `admin` | user | All tables, all columns |
| `tenant-user` | user | orders + users + products, subset columns, total + email masked |
| `regional-manager` | user | orders + users + products (all cols, phone+email masked) |
| `analytics-reader` | user | events + metrics + ordersArchive only |
| `no-access` | user | No tables (edge case) |
| `orders-service` | service | orders + products + users (limited cols) |
| `full-service` | service | All tables, all columns |

Roles have no `scope` field — the same role can be used in any scope via `ExecutionContext`.

### Test Scenarios

| # | Scenario | Tables | Expected Strategy |
|---|---|---|---|
| 1 | Single PG table | orders | direct → pg-main |
| 2 | Join within same PG | orders + products | direct → pg-main |
| 3 | Cross-PG, debezium available | orders + tenants | materialized → pg-main (tenants replicated) |
| 4 | Cross-PG, no debezium | orders + invoices | trino cross-db |
| 5 | PG + CH, debezium available | orders + events | materialized → ch-analytics (orders replicated) |
| 6 | PG + CH, no debezium | products + events | trino cross-db |
| 7 | PG + Iceberg | orders + ordersArchive | trino or materialized |
| 8 | By-ID with cache hit | users byIds=[1,2,3] | cache → redis |
| 9 | By-ID cache miss | orders byIds=[1,2] | direct → pg-main |
| 10 | By-ID partial cache | users byIds=[1,2,3] (1,2 cached) | cache + direct merge |
| 11 | Freshness=realtime, has materialized | orders + events (realtime) | skip materialized (lag=seconds), trino |
| 12 | Freshness=hours, has materialized | orders + events (hours ok) | materialized → ch-analytics |
| 13 | Admin role | any table | all columns visible |
| 14 | Tenant-user role | orders | subset columns only |
| 14b | Column masking | orders (tenant-user) | total masked |
| 14c | Multi-role within scope | orders (tenant-user + regional-manager) | union within user scope (all order columns) |
| 14d | Cross-scope restriction | orders (admin user + orders-service) | restricted to orders-service tables |
| 14e | Count mode | orders (count) | returns count only |
| 14f | Omitted scope | orders (only user: ['admin'], no service) | no service restriction |
| 15 | No-access role | any table | denied |
| 16 | Column trimming on byIds | users byIds + limited columns in role | only intersected columns |
| 17 | Invalid table name | nonexistent | validation error |
| 18 | Invalid column name | orders.nonexistent | validation error |
| 19 | Trino disabled | cross-db query | error with explanation |
| 20 | Aggregation query | orders GROUP BY status, SUM(total) | correct SQL per dialect |
| 21 | Aggregation + join | orders + products GROUP BY category | correct cross-table aggregation |
| 22 | HAVING clause | orders GROUP BY status HAVING SUM(total) > 100 | correct HAVING per dialect |
| 23 | DISTINCT query | orders DISTINCT status | correct SELECT DISTINCT per dialect |
| 24 | Cross-table ORDER BY | orders + products ORDER BY products.category | correct qualified ORDER BY |

### Sample Column Definitions (orders table)

```ts
const ordersColumns: ColumnMeta[] = [
  { apiName: 'id',           physicalName: 'id',              type: 'uuid',      nullable: false },
  { apiName: 'tenantId',     physicalName: 'tenant_id',       type: 'uuid',      nullable: false },
  { apiName: 'customerId',   physicalName: 'customer_id',     type: 'uuid',      nullable: false },
  { apiName: 'productId',    physicalName: 'product_id',      type: 'uuid',      nullable: true },
  { apiName: 'regionId',     physicalName: 'region_id',       type: 'string',    nullable: false },
  { apiName: 'total',        physicalName: 'total_amount',    type: 'decimal',   nullable: false, maskingFn: 'number' },
  { apiName: 'status',       physicalName: 'order_status',    type: 'string',    nullable: false },
  { apiName: 'internalNote', physicalName: 'internal_note',   type: 'string',    nullable: true,  maskingFn: 'full' },
  { apiName: 'createdAt',    physicalName: 'created_at',      type: 'timestamp', nullable: false, maskingFn: 'date' },
]
```

### Sample Column Definitions (users table)

```ts
const usersColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',         type: 'uuid',      nullable: false },
  { apiName: 'email',     physicalName: 'email',      type: 'string',    nullable: false, maskingFn: 'email' },
  { apiName: 'phone',     physicalName: 'phone',      type: 'string',    nullable: true,  maskingFn: 'phone' },
  { apiName: 'firstName', physicalName: 'first_name',  type: 'string',    nullable: false, maskingFn: 'name' },
  { apiName: 'lastName',  physicalName: 'last_name',   type: 'string',    nullable: false, maskingFn: 'name' },
  { apiName: 'role',      physicalName: 'role',        type: 'string',    nullable: false },
  { apiName: 'tenantId',  physicalName: 'tenant_id',   type: 'uuid',      nullable: false },
  { apiName: 'createdAt', physicalName: 'created_at',  type: 'timestamp', nullable: false },
]
```

### Sample Column Definitions (products table)

```ts
const productsColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',          type: 'uuid',    nullable: false },
  { apiName: 'name',      physicalName: 'name',        type: 'string',  nullable: false },
  { apiName: 'category',  physicalName: 'category',    type: 'string',  nullable: false },
  { apiName: 'price',     physicalName: 'price',       type: 'decimal', nullable: false, maskingFn: 'number' },
  { apiName: 'tenantId',  physicalName: 'tenant_id',   type: 'uuid',    nullable: false },
]
```

### Sample Column Definitions (events table — ClickHouse)

```ts
const eventsColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',          type: 'uuid',      nullable: false },
  { apiName: 'type',      physicalName: 'event_type',  type: 'string',    nullable: false },
  { apiName: 'userId',    physicalName: 'user_id',     type: 'uuid',      nullable: false },
  { apiName: 'payload',   physicalName: 'payload',     type: 'string',    nullable: true,  maskingFn: 'full' },
  { apiName: 'timestamp', physicalName: 'event_ts',    type: 'timestamp', nullable: false },
]
```

### Sample Column Definitions (tenants table)

```ts
const tenantsColumns: ColumnMeta[] = [
  { apiName: 'id',       physicalName: 'id',        type: 'uuid',   nullable: false },
  { apiName: 'name',     physicalName: 'name',      type: 'string', nullable: false },
  { apiName: 'plan',     physicalName: 'plan',      type: 'string', nullable: false },
  { apiName: 'apiKey',   physicalName: 'api_key',   type: 'string', nullable: false, maskingFn: 'full' },
]
```

### Sample Column Definitions (invoices table)

```ts
const invoicesColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',          type: 'uuid',      nullable: false },
  { apiName: 'tenantId',  physicalName: 'tenant_id',   type: 'uuid',      nullable: false },
  { apiName: 'orderId',   physicalName: 'order_id',    type: 'uuid',      nullable: true },
  { apiName: 'amount',    physicalName: 'amount',      type: 'decimal',   nullable: false, maskingFn: 'number' },
  { apiName: 'status',    physicalName: 'status',      type: 'string',    nullable: false },
  { apiName: 'issuedAt',  physicalName: 'issued_at',   type: 'timestamp', nullable: false },
  { apiName: 'paidAt',    physicalName: 'paid_at',     type: 'timestamp', nullable: true },
]
```

### Sample Column Definitions (metrics table — ClickHouse)

```ts
const metricsColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',          type: 'uuid',      nullable: false },
  { apiName: 'name',      physicalName: 'metric_name', type: 'string',    nullable: false },
  { apiName: 'value',     physicalName: 'value',       type: 'decimal',   nullable: false },
  { apiName: 'tags',      physicalName: 'tags',        type: 'string',    nullable: true },
  { apiName: 'timestamp', physicalName: 'ts',          type: 'timestamp', nullable: false },
]
```

### Sample Column Definitions (ordersArchive table — Iceberg)

```ts
const ordersArchiveColumns: ColumnMeta[] = [
  { apiName: 'id',         physicalName: 'id',           type: 'uuid',      nullable: false },
  { apiName: 'tenantId',   physicalName: 'tenant_id',    type: 'uuid',      nullable: false },
  { apiName: 'customerId', physicalName: 'customer_id',  type: 'uuid',      nullable: false },
  { apiName: 'total',      physicalName: 'total_amount', type: 'decimal',   nullable: false },
  { apiName: 'status',     physicalName: 'order_status', type: 'string',    nullable: false },
  { apiName: 'createdAt',  physicalName: 'created_at',   type: 'timestamp', nullable: false },
  { apiName: 'archivedAt', physicalName: 'archived_at',  type: 'timestamp', nullable: false },
]
```

### Sample Relations

```ts
const ordersRelations: RelationMeta[] = [
  { column: 'customerId', references: { table: 'users', column: 'id' }, type: 'many-to-one' },
  { column: 'productId',  references: { table: 'products', column: 'id' }, type: 'many-to-one' },
  { column: 'tenantId',   references: { table: 'tenants', column: 'id' }, type: 'many-to-one' },
]

const invoicesRelations: RelationMeta[] = [
  { column: 'tenantId', references: { table: 'tenants', column: 'id' }, type: 'many-to-one' },
  { column: 'orderId',  references: { table: 'orders', column: 'id' }, type: 'many-to-one' },
]

const usersRelations: RelationMeta[] = [
  { column: 'tenantId', references: { table: 'tenants', column: 'id' }, type: 'many-to-one' },
]

const productsRelations: RelationMeta[] = [
  { column: 'tenantId', references: { table: 'tenants', column: 'id' }, type: 'many-to-one' },
]

const eventsRelations: RelationMeta[] = [
  { column: 'userId', references: { table: 'users', column: 'id' }, type: 'many-to-one' },
]
```

### Sample Role Configurations

```ts
const roles: RoleMeta[] = [
  {
    id: 'admin',
    tables: '*',
  },
  {
    id: 'tenant-user',
    tables: [
      { tableId: 'orders',   allowedColumns: ['id', 'total', 'status', 'createdAt'], maskedColumns: ['total'] },
      { tableId: 'users',    allowedColumns: ['id', 'firstName', 'lastName', 'email'], maskedColumns: ['email'] },
      { tableId: 'products', allowedColumns: ['id', 'name', 'category', 'price'] },
    ],
  },
  {
    id: 'regional-manager',
    tables: [
      { tableId: 'orders',   allowedColumns: '*' },
      { tableId: 'users',    allowedColumns: '*', maskedColumns: ['phone', 'email'] },
      { tableId: 'products', allowedColumns: '*' },
    ],
  },
  {
    id: 'analytics-reader',
    tables: [
      { tableId: 'events',         allowedColumns: '*', maskedColumns: ['payload'] },
      { tableId: 'metrics',        allowedColumns: '*' },
      { tableId: 'orders-archive', allowedColumns: '*' },
    ],
  },
  {
    id: 'no-access',
    tables: [],
  },
  {
    id: 'orders-service',
    tables: [
      { tableId: 'orders',   allowedColumns: '*' },
      { tableId: 'products', allowedColumns: '*' },
      { tableId: 'users',    allowedColumns: ['id', 'firstName', 'lastName'] },
    ],
  },
  {
    id: 'full-service',
    tables: '*',
  },
]
```

---

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Package name | `@mkven/multi-db` | Org-scoped, reusable |
| Language | TypeScript | Type-safe, wide ecosystem |
| Query format | Typed object literals | Type-safe, IDE support, no parser |
| Metadata source | Abstracted (hardcoded now, DB/service later) | Flexibility without premature complexity |
| Execution mode | SQL-only (`SqlResult`) or execution (`DataResult`) | Distinct return types per mode |
| Freshness | Prefer original, allow specifying lag tolerance | Correctness by default, performance opt-in |
| Access control | Scoped roles: UNION within scope, INTERSECTION between scopes | User accumulates perms, service restricts them |
| SQL generation | Hand-rolled via `SqlParts` IR (internal, physical names only), no external packages | Full control over 3 divergent dialects, zero deps |
| Pagination | Offset-based (`limit` + `offset`) | Simple, sufficient for most use cases |
| Observability | OpenTelemetry | Industry standard, traces + metrics |
| Name mapping | apiName ↔ physicalName on tables and columns | Decouples API from schema |
| Cache | Redis synced by Debezium, no TTL | Always fresh, fast by-ID reads |
| Debug logging | Structured entries per pipeline phase, opt-in via `debug: true` | Zero overhead when not debugging |
| Validation | Strict — only metadata-defined entities | Fail fast with clear errors |
| Join results | Flat denormalized rows (no nesting) | `limit` applies to DB rows; with one-to-many joins, `limit: 10` may yield fewer than 10 parent entities. Nesting would require a two-query approach (fetch parent IDs first, then children), breaking the single-query model. Callers can group results using `meta.columns[].fromTable` |
| Imports | Absolute paths, no `../` | Clean, refactor-friendly |

---

## Monorepo Packages

| Package | Purpose | Dependencies |
|---|---|---|
| `@mkven/multi-db` | Core: types, metadata registry, validation, planner, SQL generators | **zero** I/O deps |
| `@mkven/multi-db-executor-postgres` | Postgres connection + execution | `pg` |
| `@mkven/multi-db-executor-clickhouse` | ClickHouse connection + execution | `@clickhouse/client` |
| `@mkven/multi-db-executor-trino` | Trino connection + execution | `trino-client` |
| `@mkven/multi-db-cache-redis` | Redis cache provider (Debezium-synced) | `ioredis` |

Core has **zero I/O dependencies** — usable for SQL-only mode without any DB drivers. Each executor is a thin adapter that consumers install only if needed.

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
│   ├── core/                        # @mkven/multi-db
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts             # public API
│   │       ├── types/
│   │       │   ├── metadata.ts      # DatabaseMeta, TableMeta, ColumnMeta, etc.
│   │       │   ├── query.ts         # QueryDefinition, QueryFilter, QueryJoin, QueryAggregation
│   │       │   ├── result.ts        # QueryResult, QueryResultMeta, DebugLogEntry
│   │       │   └── context.ts       # ExecutionContext
│   │       ├── metadata/
│   │       │   ├── registry.ts      # MetadataRegistry — stores and indexes all metadata
│   │       │   └── loader.ts        # abstracted loader interface
│   │       ├── validation/
│   │       │   ├── validator.ts     # query validation against metadata + role
│   │       │   └── errors.ts        # typed validation errors
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
│
└── tests/
    ├── fixtures/
    │   └── testConfig.ts            # the full test config described above
    ├── validation/
    ├── access/
    ├── planner/
    ├── generator/
    ├── cache/
    └── e2e/
```

---

## Open Items for Further Discussion

- [ ] Builder pattern API on top of object literals?
- [ ] Subquery / nested filter support?
- [ ] Write operations (INSERT/UPDATE/DELETE) — future scope?
- [ ] Connection pool configuration and lifecycle
- [ ] Trino session properties for optimization
- [ ] Schema migration strategy for metadata storage
- [ ] How to handle schema drift between original and replicated tables
- [ ] RLS (row-level security) — deferred, may add later
- [ ] Cursor-based pagination as alternative to offset?
- [ ] Custom masking functions beyond predefined set?
- [ ] Nested/grouped results for one-to-many joins? (requires two-query approach: fetch parent IDs with limit, then fetch all children — significant complexity)
