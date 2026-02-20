# @mkven/multi-db — Concept Document

## Objective

Build a reusable, metadata-driven query engine that lets applications query, filter, join, and aggregate data across Postgres, ClickHouse, Iceberg, and Redis through a single typed API. The engine:

- Accepts queries using **apiNames** (decoupled from physical schema)
- Supports **rich filtering** — 31 operators (comparison, pattern, range, fuzzy, array), column-vs-column comparisons, recursive AND/OR groups, EXISTS/NOT EXISTS subqueries with optional counted variant
- Supports **JOINs** (inner/left) resolved from relation metadata, **GROUP BY**, and **aggregations** (count, sum, avg, min, max) with HAVING
- Automatically selects the **optimal execution strategy** — direct DB, cached, materialized replica, or Trino cross-DB federation
- Enforces **scoped access control** — user roles and service roles intersected to determine effective permissions
- Applies **column masking** for sensitive data based on role
- **Validates strictly** — 14 rules covering tables, columns, filters, joins, aggregations, permissions; all errors collected into a single response
- Returns **generated SQL**, **executed results**, or **row counts** depending on the caller's needs
- Provides **structured debug logs** for transparent pipeline tracing
- Supports **hot-reload** of metadata and roles, **health checks** for all providers, and graceful **shutdown**

The monorepo ships a standalone **validation package** (`@mkven/multi-db-validation`) with **zero I/O dependencies** — clients can validate configs and queries locally before sending to the server. The core package (`@mkven/multi-db`) depends on it and adds planning, SQL generation, and masking, also with zero I/O deps. Database drivers and cache clients live in separate executor/cache packages.

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
type ScalarColumnType = 'string' | 'int' | 'decimal' | 'boolean' | 'uuid' | 'date' | 'timestamp'
type ArrayColumnType = 'string[]' | 'int[]' | 'decimal[]' | 'boolean[]' | 'uuid[]' | 'date[]' | 'timestamp[]'
type ColumnType = ScalarColumnType | ArrayColumnType

interface ColumnMeta {
  apiName: string                     // 'customerEmail'
  physicalName: string                // 'customer_email'
  type: ColumnType                    // logical type; array types end with '[]' (e.g. 'string[]'); element type derived by stripping '[]'
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
  estimatedLag: 'seconds' | 'minutes' | 'hours'
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

### Metadata Configuration

```ts
interface MetadataConfig {
  databases: DatabaseMeta[]
  tables: TableMeta[]
  caches: CacheMeta[]
  externalSyncs: ExternalSync[]
  trino?: { enabled: boolean }
}
```

### Metadata & Role Providers

Metadata and roles are always loaded via providers. This keeps the API uniform — `reloadMetadata()` and `reloadRoles()` always work the same way regardless of source.

```ts
interface MetadataProvider {
  load(): Promise<MetadataConfig>
}

interface RoleProvider {
  load(): Promise<RoleMeta[]>
}
```

For static configs (tests, simple apps), use the built-in helpers:

```ts
import { staticMetadata, staticRoles } from '@mkven/multi-db'

const metadataProvider = staticMetadata({ databases: [...], tables: [...], ... })
const roleProvider = staticRoles([...])
```

The MultiDb instance exposes reload and lifecycle methods:

```ts
interface MultiDb {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>

  reloadMetadata(): Promise<void>     // re-calls MetadataProvider.load(), rebuilds indexes
  reloadRoles(): Promise<void>        // re-calls RoleProvider.load(), rebuilds role map

  healthCheck(): Promise<HealthCheckResult>  // checks all executors + cache providers

  close(): Promise<void>              // calls close() on all executors + cache providers
                                      // Always attempts ALL providers even if some throw — collects errors
                                      // into a single ConnectionError listing all failed close() calls.
                                      // Subsequent query() calls throw ExecutionError (EXECUTOR_MISSING)
}

// Safe for concurrent use — query() captures a snapshot of the current metadata/roles
// at call start. reloadMetadata() / reloadRoles() atomically replaces the internal
// reference; in-flight queries see the config that was active when they started.

interface HealthCheckResult {
  healthy: boolean                    // true only if ALL checks pass
  executors: Record<string, { healthy: boolean; latencyMs: number; error?: string }>
  cacheProviders: Record<string, { healthy: boolean; latencyMs: number; error?: string }>
}
```

External systems signal staleness by calling `reloadMetadata()` or `reloadRoles()`. This is explicit — no polling, no TTL. Typical triggers: admin UI saves new config, CI/CD deploys schema changes, webhook from config service.

```ts
const multiDb = await createMultiDb({
  metadataProvider: staticMetadata({ databases: [...], tables: [...], ... }),
  roleProvider: staticRoles([...]),

  executors: { ... },
  cacheProviders: { ... },
})

// Later, when external system signals config change:
await multiDb.reloadMetadata()
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

The module is initialized with metadata and role providers, plus optional executor/cache instances:

```ts
import { createMultiDb, staticMetadata, staticRoles } from '@mkven/multi-db'
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

### Query Input

```ts
interface QueryDefinition {
  from: string                        // table apiName
  columns?: string[]                  // apiNames; undefined = all allowed for role (but see aggregation-only note below); empty [] = aggregation-only query (no regular columns, only aggregation results); rejected if empty AND no aggregations
  distinct?: boolean                  // SELECT DISTINCT (default: false)
  filters?: (QueryFilter | QueryColumnFilter | QueryFilterGroup | QueryExistsFilter)[]  // implicit AND at top level; use QueryFilterGroup for OR / nested logic
  joins?: QueryJoin[]
  groupBy?: QueryGroupBy[]            // columns to group by
  aggregations?: QueryAggregation[]   // aggregate functions
  having?: (QueryFilter | QueryFilterGroup)[]   // filters on aggregated values (applied after GROUP BY)
                                      // column references aggregation aliases, not table columns
  limit?: number
  offset?: number
  orderBy?: QueryOrderBy[]
  freshness?: 'realtime' | 'seconds' | 'minutes' | 'hours'  // acceptable lag; omit = any lag acceptable (planner still prefers original via P1 > P2)
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
  column: string                      // apiName (or aggregation alias when used with groupBy/aggregations)
  table?: string                      // apiName of table; omit for `from` table (omit when referencing an alias)
  direction: 'asc' | 'desc'
}

interface QueryGroupBy {
  column: string                      // apiName
  table?: string                      // apiName of table; omit for `from` table
}

interface QueryJoin {
  table: string                       // related table apiName
  type?: 'inner' | 'left'            // default: 'left' (safe for nullable FKs)
  columns?: string[]                  // columns to select from joined table; undefined = all allowed for role; [] = no columns (join used for filter/groupBy only)
  filters?: (QueryFilter | QueryColumnFilter | QueryFilterGroup | QueryExistsFilter)[]  // filters on joined table
  // Within join filters, omitting `table` on a QueryFilter resolves the column against the **joined**
  // table (not the `from` table). This is the natural default — you're declaring filters *for* this join.
  // Specifying `table` explicitly overrides this and allows referencing any table in the query (from or any join).
  // NOTE: Join filters are placed in WHERE, not ON. The ON clause only contains the join condition
  // (FK = PK from relation metadata). For LEFT JOINs this means join-scoped filters effectively
  // convert to INNER JOIN semantics — rows where the joined table doesn't match the filter are
  // excluded. This is intentional: if you filter on a joined table, you want matching rows only.
}

interface QueryFilter {
  column: string                      // apiName (or aggregation alias when used in `having`)
  table?: string                      // apiName of table; omit for `from` table. Allows filtering on joined table columns at top level
                                      // Exception: inside QueryJoin.filters, omitting `table` resolves against the joined table (not `from`)
  operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'in' | 'notIn' | 'like' | 'notLike' | 'ilike' | 'notIlike'
           | 'isNull' | 'isNotNull' | 'between' | 'notBetween'
           | 'contains' | 'icontains' | 'notContains' | 'notIcontains'
           | 'startsWith' | 'istartsWith' | 'endsWith' | 'iendsWith'
           | 'levenshteinLte'
           | 'arrayContains' | 'arrayContainsAll' | 'arrayContainsAny'
           | 'arrayIsEmpty' | 'arrayIsNotEmpty'
  value?: unknown                     // scalar for most operators; array for 'in'/'notIn'; omit for 'isNull'/'isNotNull'/'arrayIsEmpty'/'arrayIsNotEmpty'
                                      // for 'between'/'notBetween': { from: unknown, to: unknown } (inclusive range)
                                      // for 'levenshteinLte': { text: string, maxDistance: number }
                                      // for 'contains'/'icontains'/'notContains'/'notIcontains'/'startsWith'/'istartsWith'/'endsWith'/'iendsWith': plain string (no wildcards — added internally)
                                      // for 'arrayContains': single element matching the array's element type
                                      // for 'arrayContainsAll'/'arrayContainsAny': non-empty array of elements matching element type
}

// Column-vs-column comparison — right side is another column, not a literal value
interface QueryColumnFilter {
  column: string                      // left column apiName
  table?: string                      // left table apiName; omit for `from` table
  operator: '=' | '!=' | '>' | '<' | '>=' | '<='  // only comparison operators (no pattern/null/function ops)
  refColumn: string                   // right column apiName
  refTable?: string                   // right table apiName; omit for `from` table
}

interface QueryFilterGroup {
  logic: 'and' | 'or'
  not?: boolean                       // default: false — when true, negates the entire group: NOT (c1 AND/OR c2 ...)
  conditions: (QueryFilter | QueryColumnFilter | QueryFilterGroup | QueryExistsFilter)[]  // recursive — supports arbitrary nesting
}

interface QueryExistsFilter {
  exists?: boolean                    // true = EXISTS (default), false = NOT EXISTS
                                      // when `count` is present, `exists` is ignored — the count operator
                                      // handles both directions (e.g. < 3 for "fewer than 3")
  table: string                       // related table apiName (relation resolved via metadata, same as joins)
  filters?: (QueryFilter | QueryColumnFilter | QueryFilterGroup | QueryExistsFilter)[]  // optional conditions on the related table
  count?: {                           // optional: require specific count of matching related rows
    operator: '=' | '!=' | '>' | '<' | '>=' | '<='  // comparison applied to the subquery count
    value: number                     // non-negative integer
  }                                   // when present, changes SQL from EXISTS to a counted correlated subquery:
                                      //   (SELECT COUNT(*) FROM related WHERE ...) >= N
                                      // count: { operator: '>=', value: 1 } is semantically identical to
                                      // plain `exists: true` — prefer the simpler form for clarity
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

Roles within a scope are unioned (accumulated permissions). The final effective permissions are the intersection of all scope unions. If a scope is omitted, it imposes no restriction (treated as "all access" for that scope). An empty array (`user: []`) is different: it means zero roles → zero permissions → all tables denied.

### Query Result

Three distinct return types depending on `executeMode`:

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
    apiName: string                  // for aggregations: the alias (e.g. 'totalSum')
    type: ColumnType                  // for aggregations: inferred from fn (count → 'int', avg → always 'decimal', sum/min/max → source column type)
    nullable: boolean
    fromTable: string                // table apiName; for aggregations: the source column's table (or `from` table for count(*))
    masked: boolean                  // whether this column was masked (always false for aggregation aliases)
  }[]                                // in count mode: empty array (no columns are selected)
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

When you request execution (`executeMode = 'execute'`), you get data back — no SQL. When you request SQL only (`executeMode = 'sql-only'`), you get SQL + params — no execution, no data. When you request count (`executeMode = 'count'`), you get just the row count — `columns`, `orderBy`, `limit`, `offset`, `distinct`, `groupBy`, `aggregations`, and `having` are all ignored (always emits `SELECT COUNT(*) FROM ...`, never grouped counts); `filters` and `joins` remain active (they affect which rows are counted); `meta.columns` is an empty array since no columns are selected. `byIds` + `count` is valid — it counts how many of the provided IDs actually exist (`SELECT COUNT(*) FROM ... WHERE id = ANY($1)`). All modes include metadata. Debug log is included only when `debug: true`.

In `sql-only` mode, masking cannot be applied (no data to mask). However, `meta.columns[].masked` still reports masking intent so the caller can apply masking themselves after execution.

**Masking and aggregations:** aggregation aliases are never masked. Masking applies to raw column values, not aggregated results. If `total` has `maskingFn: 'number'` and you query `SUM(total) as totalSum`, `totalSum` is returned unmasked — the aggregate collapses rows, so row-level masking is not meaningful.

**NULL handling in aggregations:** SQL aggregation functions (`SUM`, `AVG`, `MIN`, `MAX`) ignore NULL values — only non-NULL rows contribute to the result. `COUNT(*)` counts all rows including NULLs; `COUNT(column)` ignores NULLs. If all values are NULL, `SUM`/`AVG`/`MIN`/`MAX` return NULL (the result column is `nullable: true` when the source column is nullable). `AVG` always returns `'decimal'` type regardless of the source column type — `AVG(int_column)` produces a fractional result in all SQL dialects.

**DISTINCT and GROUP BY:** when both `distinct: true` and `groupBy` are present, `DISTINCT` is redundant — `GROUP BY` already deduplicates grouped columns. The system does not reject this combination (valid SQL), but the `DISTINCT` keyword has no effect on the result. Callers should prefer one or the other.

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
- ALL required tables exist in ONE database (original data)
- Generate native SQL for that engine
- **Iceberg exception:** Iceberg databases have no standalone executor — they are always queried via the `trino` executor using `trinoCatalog`. A single-Iceberg-table P1 query routes through the Trino executor with Trino dialect (single-catalog, no cross-DB federation)
- This is always preferred when possible (freshest data, no overhead)

### Priority 2 — Materialized Replica
- Some tables are in different databases, BUT debezium replicas exist such that all needed data is available in one database
- Check freshness: if query requires `realtime` → skip this strategy entirely (replicas always have lag); if `freshness` = `seconds` but replica lag is `minutes` → also skip
- Freshness hierarchy (strictest to most relaxed): `realtime` < `seconds` < `minutes` < `hours`. `realtime` is query-only (never an `estimatedLag` value) — it means direct access required. A replica is acceptable when its `estimatedLag` ≤ the query's `freshness` tolerance
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
2. **Column existence** — only columns defined in table metadata can be referenced. When `columns` is `undefined` and `aggregations` is present, the default is `groupBy` columns only (not all allowed columns) — this avoids rule 7 failures from ungrouped columns being added automatically
3. **Role permission** — if a table is not in the role's `tables` list → access denied
4. **Column permission** — if `allowedColumns` is a list and requested column is not in it → denied; if columns not specified in query, return only allowed ones
5. **Filter validity** — filter operators must be valid for the column type (see table below); `isNull`/`isNotNull` additionally require `nullable: true`; malformed compound values (`between`/`notBetween` missing `to`, `between`/`notBetween` with `null` as `from` or `to`, `levenshteinLte` with non-integer or negative `maxDistance`, `in` with empty array) are rejected with `INVALID_VALUE`; `between`/`notBetween` with `null` bounds are rejected for the same reason as `in`/`notIn` NULL — SQL `BETWEEN NULL AND x` always yields false due to 3-valued logic; `in`/`notIn` additionally validate that all array elements match the column type (e.g. passing `['a','b']` on an `int` column → `INVALID_VALUE`); `in`/`notIn` also reject `null` elements — SQL's `NOT IN (1, NULL)` always returns zero rows due to 3-valued logic, which is a major footgun; **array operators** (`arrayContains`, `arrayContainsAll`, `arrayContainsAny`, `arrayIsEmpty`, `arrayIsNotEmpty`) are only valid on array column types (e.g. `'string[]'`); `arrayContains` value must match the element type; `arrayContainsAll`/`arrayContainsAny` value must be a non-empty array with all elements matching the element type (same validation as `in`/`notIn` — includes element type check and null element rejection); conversely, all scalar operators (except `isNull`/`isNotNull`) are rejected on array columns; when `QueryFilter.table` is provided, the table must be the `from` table or one of the joined tables — referencing a non-joined table is rejected; for `QueryColumnFilter`, both columns must exist, the role must allow both, and their types must be compatible (same type, or both orderable) — array columns are not allowed in `QueryColumnFilter`; filter groups and exists filters are validated recursively (all nested conditions checked)
6. **Join validity** — joined tables must have a defined relation in metadata
7. **Group By validity** — if `groupBy` or `aggregations` are present, every column in `columns` that is not an aggregation alias must appear in `groupBy`. Prevents invalid SQL from reaching the database. When `QueryGroupBy.table` is provided, the table must be the `from` table or one of the joined tables — same rule as filter `table` (rule 5)
8. **Having validity** — `having` filters must reference aliases defined in `aggregations`; `QueryFilter.table` is rejected inside `having` (HAVING operates on aggregation aliases, not table columns); `QueryColumnFilter` nested inside `having` groups is rejected (HAVING compares aliases, not table columns — column-vs-column comparison is not meaningful); `QueryExistsFilter` nested inside `having` groups is rejected (EXISTS in HAVING is not valid SQL); only comparison, range, and null-check operators are allowed — `=`, `!=`, `>`, `<`, `>=`, `<=`, `in`, `notIn`, `between`, `notBetween`, `isNull`, `isNotNull`; pattern operators (`like`, `ilike`, `contains`, `startsWith`, `endsWith`, and their `not`/`i` variants), `levenshteinLte`, and array operators (`arrayContains`, `arrayContainsAll`, `arrayContainsAny`, `arrayIsEmpty`, `arrayIsNotEmpty`) are rejected — they operate on text values or array columns, not aggregated numbers
9. **Order By validity** — `orderBy` must reference columns from `from` table, joined tables, or aggregation aliases defined in `aggregations`. When `QueryOrderBy.table` is provided, the table must be the `from` table or one of the joined tables — same rule as filter `table` (rule 5)
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

Comparison operators (`>`, `<`, `>=`, `<=`) are rejected on `uuid` and `boolean` — UUIDs have no meaningful ordering, booleans should use `=`/`!=`. `in`/`notIn` are rejected on `date`/`timestamp` — use range comparisons instead. `between`/`notBetween` follow the same type rules as `>=`/`<=` (orderable types only — excludes `uuid` and `boolean`); `notBetween` emits `NOT (col BETWEEN $1 AND $2)`. `like`/`ilike`/`contains`/`icontains`/`notContains`/`notIcontains`/`startsWith`/`istartsWith`/`endsWith`/`iendsWith` are string-only. `contains`/`icontains` map to `LIKE '%x%'` / `ILIKE '%x%'`; `notContains`/`notIcontains` map to `NOT LIKE '%x%'` / `NOT ILIKE '%x%'` — wildcard characters in the value are escaped automatically. `startsWith`/`istartsWith` map to `LIKE 'x%'` / `ILIKE 'x%'`; `endsWith`/`iendsWith` map to `LIKE '%x'` / `ILIKE '%x'`. `levenshteinLte` is string-only — it matches rows where the Levenshtein edit distance between the column value and the target text is ≤ `maxDistance`. No index support in any dialect — always a full scan. PostgreSQL requires the `fuzzystrmatch` extension (`CREATE EXTENSION fuzzystrmatch`).

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
  code: 'CONNECTION_FAILED'
  details: {
    unreachable: {
      id: string                      // provider id (DatabaseMeta.id or CacheMeta.id)
      type: 'executor' | 'cache'
      engine?: 'postgres' | 'clickhouse' | 'trino' | 'redis'  // which engine failed — helps troubleshooting
      cause?: Error
    }[]
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

| Feature | Postgres | ClickHouse | Trino |
|---|---|---|---|
| Identifier quoting | `"column"` | `` `column` `` | `"column"` |
| Parameter binding | `$1, $2` | `{p1:Type}` | `?` |
| `in` | `= ANY($1::type[])` | `IN tuple(v1, v2, ...)` | `IN (?, ?, ...)` (param expansion) |
| `notIn` | `<> ALL($1::type[])` | `NOT IN tuple(v1, v2, ...)` | `NOT IN (?, ?, ...)` (param expansion) |
| Date functions | `date_trunc(...)` | `toStartOfDay(...)` | `date_trunc(...)` |
| LIMIT/OFFSET | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` |
| Case-insensitive LIKE | `ILIKE` | `ilike(col, pattern)` | `lower(col) LIKE lower(pattern)` |
| `startsWith` / `endsWith` | `LIKE 'x%'` / `LIKE '%x'` | `startsWith(col, {p1})` / `endsWith(col, {p2})` | `LIKE 'x%'` / `LIKE '%x'` |
| `istartsWith` / `iendsWith` | `ILIKE 'x%'` / `ILIKE '%x'` | `ilike(col, 'x%')` / `ilike(col, '%x')` | `lower(col) LIKE lower('x%')` / `lower(col) LIKE lower('%x')` |
| Levenshtein distance | `levenshtein(col, $1) <= $2` | `editDistance(col, {p1:String}) <= {p2:UInt32}` | `levenshtein_distance(col, ?) <= ?` |
| BETWEEN | `col BETWEEN $1 AND $2` | `col BETWEEN {p1} AND {p2}` | `col BETWEEN ? AND ?` |
| NOT BETWEEN | `col NOT BETWEEN $1 AND $2` | `NOT (col BETWEEN {p1} AND {p2})` | `col NOT BETWEEN ? AND ?` |
| Boolean | `true/false` | `1/0` | `true/false` |
| Counted subquery | `(SELECT COUNT(*) FROM ... WHERE ...) >= $N` | `(SELECT COUNT(*) FROM ... WHERE ...) >= {pN:UInt64}` | `(SELECT COUNT(*) FROM ... WHERE ...) >= ?` |
| Array column type | `text[]`, `integer[]`, etc. | `Array(String)`, `Array(Int32)`, etc. | `array(varchar)`, `array(integer)`, etc. |
| `arrayContains` | `$1 = ANY(col)` | `has(col, {p1})` | `contains(col, ?)` |
| `arrayContainsAll` | `col @> $1::type[]` | `hasAll(col, [{p1}, ...])` | `cardinality(array_except(ARRAY[?,...], col)) = 0` |
| `arrayContainsAny` | `col && $1::type[]` | `hasAny(col, [{p1}, ...])` | `arrays_overlap(col, ARRAY[?,...])` |
| `arrayIsEmpty` | `cardinality(col) = 0` | `empty(col)` | `cardinality(col) = 0` |
| `arrayIsNotEmpty` | `cardinality(col) > 0` | `notEmpty(col)` | `cardinality(col) > 0` |

**Performance note — counted subqueries:** `COUNT(*)` scans all matching rows even when only a threshold is needed. For `>= N` / `> N` comparisons, the system can optimize by adding `LIMIT N` inside the subquery: `(SELECT COUNT(*) FROM (SELECT 1 FROM ... WHERE ... LIMIT N)) >= N`. This short-circuits counting once the threshold is reached. Not applied for `=` / `!=` / `<` / `<=` (which need the exact count). The optimization is per-dialect and transparent to the caller.

**Postgres `in`/`notIn` type mapping:** The `= ANY($1::type[])` syntax requires an explicit array type cast. The system maps `ColumnType` → SQL type:

| ColumnType | Postgres SQL Type | Array Cast |
|---|---|---|
| `string` | `text` | `$1::text[]` |
| `int` | `integer` | `$1::integer[]` |
| `decimal` | `numeric` | `$1::numeric[]` |
| `uuid` | `uuid` | `$1::uuid[]` |

Only types that support `in`/`notIn` are listed (`boolean`, `date`, `timestamp` are rejected by rule 5). ClickHouse and Trino expand values inline and don't need type casts.

**Postgres array operator type mapping:** `arrayContainsAll` (`@>`) and `arrayContainsAny` (`&&`) use the same type casts as `in`/`notIn` — e.g. `col @> $1::text[]`. `arrayContains` (`= ANY`) uses the scalar type cast — e.g. `$1::text = ANY(col)`. ClickHouse and Trino use function-based syntax and don't need casts.

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
  physicalName: string                // 'total_amount'; for aggregations: same as alias (e.g. 'totalSum')
  apiName: string                     // 'total'; for aggregations: the alias (e.g. 'totalSum')
  tableAlias: string                  // 't0'; for aggregations: the source column's table alias (or from-table alias for count(*))
  masked: boolean                     // apply masking after fetch (always false for aggregation aliases)
  type: ColumnType                    // logical column type; for aggregations: inferred from fn (count → 'int', avg → always 'decimal', sum/min/max → source column type)
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
  where?: WhereNode                   // recursive AND/OR tree (omit if no WHERE clause)
  groupBy: ColumnRef[]
  having?: HavingNode                 // recursive AND/OR tree for HAVING (omit if no HAVING clause); excludes EXISTS
  aggregations: AggregationClause[]
  orderBy: OrderByClause[]
  limit?: number
  offset?: number
}

// Recursive WHERE tree — mirrors QueryFilterGroup at the physical level
type WhereNode = WhereCondition | WhereColumnCondition | WhereBetween | WhereFunction | WhereArrayCondition | WhereGroup | WhereExists | WhereCountedSubquery

// Function-based condition — for operators that wrap the column in a function (e.g. levenshteinLte)
interface WhereFunction {
  fn: string                          // dialect resolves to actual function name (e.g. 'levenshtein', 'editDistance', 'levenshtein_distance')
  column: ColumnRef
  fnParamIndex: number                // param index for the function argument (the target text)
  operator: string                    // comparison operator applied to the function result (e.g. '<=')
  compareParamIndex: number           // param index for the comparison value (the max distance)
}

// Array filter condition — for arrayContains, arrayContainsAll, arrayContainsAny, arrayIsEmpty, arrayIsNotEmpty
interface WhereArrayCondition {
  column: ColumnRef
  operator: 'contains' | 'containsAll' | 'containsAny' | 'isEmpty' | 'isNotEmpty'
  paramIndexes?: number[]             // contains: single index; containsAll/containsAny: one index per element; omit for isEmpty/isNotEmpty
  elementType: string                 // SQL element type for Postgres casting (e.g. 'text', 'integer'); ClickHouse/Trino don't use it
}

// HAVING tree — same as WhereNode but excludes EXISTS, WhereColumnCondition, WhereFunction, and WhereArrayCondition
// Only comparison + range + null operators are allowed (no pattern/function/array operators on aliases)
type HavingNode = WhereCondition | HavingBetween | HavingGroup

// Range condition on an aggregation alias — uses bare string, not ColumnRef
interface HavingBetween {
  alias: string                       // aggregation alias (e.g. 'totalSum')
  not?: boolean                       // when true, emits NOT (alias BETWEEN ... AND ...)
  fromParamIndex: number
  toParamIndex: number
}

interface HavingGroup {
  logic: 'and' | 'or'
  not?: boolean
  conditions: HavingNode[]
}

interface WhereGroup {
  logic: 'and' | 'or'
  not?: boolean                       // when true, emits NOT (...)
  conditions: WhereNode[]
}

// Shared shape for correlated subqueries — used by both WhereExists and WhereCountedSubquery
interface CorrelatedSubquery {
  from: TableRef
  join: { leftColumn: ColumnRef; rightColumn: ColumnRef }  // correlated condition (outer.fk = inner.pk)
  where?: WhereNode                   // additional conditions inside the subquery
}

interface WhereExists {
  exists: boolean                     // true = EXISTS, false = NOT EXISTS
  subquery: CorrelatedSubquery
}

// Counted variant — replaces EXISTS with a counted correlated subquery
// Used when QueryExistsFilter.count is provided
interface WhereCountedSubquery {
  subquery: CorrelatedSubquery
  operator: string                    // '>=', '>', '=', '!=', '<', '<='
  countParamIndex: number             // param index for the count value
  // Emits: (SELECT COUNT(*) FROM <from> WHERE <join> AND <where>) <operator> $N
  // For >= / > operators, system may add LIMIT inside subquery to short-circuit counting
}

interface OrderByClause {
  column: ColumnRef | string           // ColumnRef for table columns; bare string for aggregation aliases (e.g. 'totalSum')
  direction: 'asc' | 'desc'
}

interface TableRef {
  physicalName: string                // 'public.orders'
  alias: string                       // 't0'
  catalog?: string                    // for trino: 'pg_main'
}

// QueryJoin.filters are placed in WHERE, not ON. The ON clause only contains the join condition
// (leftColumn = rightColumn). For LEFT JOINs this means join-scoped filters effectively convert
// to INNER JOIN semantics — rows where the joined table doesn't match the filter are excluded.
// This is intentional: if you filter on a joined table, you want matching rows only.
interface JoinClause {
  type: 'inner' | 'left'
  table: TableRef
  leftColumn: ColumnRef               // e.g. { tableAlias: 't0', columnName: 'customer_id' }
  rightColumn: ColumnRef              // e.g. { tableAlias: 't1', columnName: 'id' }
}

interface WhereCondition {
  column: ColumnRef | string          // ColumnRef for table columns; bare string for aggregation aliases in HAVING (e.g. 'totalSum')
  operator: string                    // '=', 'ILIKE', 'ANY', etc. — string (not union) because dialects may emit operators beyond the public QueryFilter set
  paramIndex?: number                 // for parameterized values (mutually exclusive with `literal`)
  literal?: string                    // for IS NULL, IS NOT NULL (mutually exclusive with `paramIndex`)
}

// Column-vs-column condition — no parameters, both sides are column references
interface WhereColumnCondition {
  leftColumn: ColumnRef
  operator: string                    // '=', '!=', '>', '<', '>=', '<='
  rightColumn: ColumnRef
}

// Range condition — for 'between' / 'notBetween' operators
interface WhereBetween {
  column: ColumnRef
  not?: boolean                       // when true, emits NOT (col BETWEEN ... AND ...) — used by 'notBetween' operator
  fromParamIndex: number              // param index for lower bound
  toParamIndex: number                // param index for upper bound
}

interface AggregationClause {
  fn: 'count' | 'sum' | 'avg' | 'min' | 'max'
  column: ColumnRef | '*'             // '*' for count(*)
  alias: string                       // result column name
}
```

```ts
interface SqlDialect {
  generate(parts: SqlParts): { sql: string; params: unknown[] }
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

### Databases

| ID | Engine | Trino Catalog | Purpose |
|---|---|---|---|
| `pg-main` | postgres | `pg_main` | Primary OLTP |
| `pg-tenant` | postgres | `pg_tenant` | Tenant-specific data |
| `ch-analytics` | clickhouse | `ch_analytics` | Analytics / events |
| `iceberg-archive` | iceberg | `iceberg_archive` | Historical archives |

### Tables

| Table ID | API Name | Database | Physical Name | Primary Key |
|---|---|---|---|---|
| `users` | users | pg-main | public.users | `id` |
| `orders` | orders | pg-main | public.orders | `id` |
| `products` | products | pg-main | public.products | `id` |
| `tenants` | tenants | pg-tenant | public.tenants | `id` |
| `invoices` | invoices | pg-tenant | public.invoices | `id` |
| `events` | events | ch-analytics | default.events | `id` |
| `metrics` | metrics | ch-analytics | default.metrics | `id` |
| `orders-archive` | ordersArchive | iceberg-archive | warehouse.orders_archive | `id` |

### External Syncs (Debezium)

| Source | Target DB | Target Physical Name | Lag |
|---|---|---|---|
| orders | ch-analytics | default.orders_replica | seconds |
| orders | iceberg-archive | warehouse.orders_current | minutes |
| tenants | pg-main | replicas.tenants | seconds |

### Cache (Redis, synced by Debezium — no TTL)

| Table | Key Pattern | Columns |
|---|---|---|
| users | `users:{id}` | all (undefined) |
| products | `products:{id}` | `id`, `name`, `category` (subset — price excluded) |

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

Tests are split between packages. Validation package tests run without DB connections; core package tests need executors/providers.

#### `packages/validation/tests/config/` — config validation (ConfigError)

| # | Scenario | Input | Error |
|---|---|---|---|
| 49 | Invalid apiName format | table apiName 'Order_Items' (has underscore) | ConfigError: CONFIG_INVALID (INVALID_API_NAME) |
| 50 | Duplicate apiName | two tables with apiName 'orders' | ConfigError: CONFIG_INVALID (DUPLICATE_API_NAME) |
| 51 | Invalid DB reference | table references non-existent database 'pg-other' | ConfigError: CONFIG_INVALID (INVALID_REFERENCE) |
| 52 | Invalid relation | relation references non-existent table 'invoiceLines' | ConfigError: CONFIG_INVALID (INVALID_RELATION) |
| 80 | Multiple config errors | invalid apiName + duplicate + broken reference | ConfigError: CONFIG_INVALID, errors[] contains all 3 |
| 81 | Invalid sync reference | ExternalSync references non-existent table/database | ConfigError: CONFIG_INVALID (INVALID_SYNC) |
| 89 | Invalid cache config | CacheMeta references non-existent table or invalid keyPattern placeholder | ConfigError: CONFIG_INVALID (INVALID_CACHE) |
| 96 | Duplicate column apiName | two columns in same table with apiName 'status' | ConfigError: CONFIG_INVALID (DUPLICATE_API_NAME) |

#### `packages/validation/tests/query/` — query validation against metadata + roles (ValidationError)

| # | Scenario | Input | Rule |
|---|---|---|---|
| 15 | No-access role | any table | rule 3 — ACCESS_DENIED (table) |
| 17 | Invalid table name | nonexistent | rule 1 — UNKNOWN_TABLE |
| 18 | Invalid column name | orders.nonexistent | rule 2 — UNKNOWN_COLUMN |
| 32 | Invalid join (no relation) | orders + metrics | rule 6 — INVALID_JOIN |
| 34 | Multiple validation errors | from: 'nonexistent', column: 'bad', filter on 'missing' | multi-error collection |
| 36 | Invalid limit/offset | orders limit: -1 | rule 11 — INVALID_LIMIT |
| 37 | byIds + aggregations | users byIds + GROUP BY | rule 10 — INVALID_BY_IDS |
| 40 | Invalid GROUP BY | orders columns: [status, total], groupBy: [status] | rule 7 — INVALID_GROUP_BY |
| 41 | Invalid HAVING | orders having on non-existent alias | rule 8 — INVALID_HAVING |
| 42 | Invalid ORDER BY | orders orderBy: products.category (not joined) | rule 9 — INVALID_ORDER_BY |
| 43 | Invalid EXISTS filter | orders EXISTS metrics (no relation) | rule 12 — INVALID_EXISTS |
| 157 | `exists: false` + `count` (allowed) | orders EXISTS(false) invoices count: { operator: '>=', value: 3 } | `exists` ignored when `count` present — no error (SQL generation tested in generator #157) |
| 158 | `count.value` negative | orders EXISTS invoices count: { operator: '>=', value: -1 } | rule 12 — INVALID_EXISTS (value must be ≥ 0) |
| 159 | `count.value` non-integer | orders EXISTS invoices count: { operator: '>=', value: 2.5 } | rule 12 — INVALID_EXISTS (value must be integer) |
| 46 | Invalid filter operator | orders WHERE id > 'some-uuid' (uuid type) | rule 5 — INVALID_FILTER |
| 47 | Access denied on column | orders columns: [internalNote] (tenant-user) | rule 4 — ACCESS_DENIED (column) |
| 65 | Empty byIds | orders byIds=[] | rule 10 — INVALID_BY_IDS |
| 78 | Empty columns array (no aggregations) | orders columns: [] | rule 14 — INVALID_AGGREGATION |
| 82 | Unknown role ID | context roles: { user: ['nonexistent'] } | rule 13 — UNKNOWN_ROLE |
| 86 | EXISTS inside HAVING | orders HAVING group with EXISTS invoices | rule 8 — INVALID_HAVING (EXISTS not valid in HAVING) |
| 87 | Duplicate aggregation alias | orders SUM(total) as x, COUNT(*) as x | rule 14 — INVALID_AGGREGATION |
| 88 | Alias collides with column apiName | orders columns: [status], SUM(total) as status | rule 14 — INVALID_AGGREGATION |
| 97 | Offset without limit | orders offset: 10 (no limit) | rule 11 — INVALID_LIMIT |
| 98 | Filter on joined non-existent column | orders JOIN products, filter: products.nonexistent = 'x' | rule 2 — UNKNOWN_COLUMN |
| 107 | `isNull` on non-nullable | orders WHERE id IS NULL (id: nullable=false) | rule 5 — INVALID_FILTER |
| 109 | `levenshteinLte` on non-string | orders WHERE total levenshteinLte { text: '100', maxDistance: 1 } | rule 5 — INVALID_FILTER (decimal column) |
| 116 | `between` on boolean | orders WHERE status between { from: true, to: false } | rule 5 — INVALID_FILTER (boolean not orderable) |
| 117 | `contains` on non-string | orders WHERE total contains '100' | rule 5 — INVALID_FILTER (decimal column) |
| 118 | Column filter type mismatch | orders WHERE total(decimal) > status(string) | rule 5 — INVALID_FILTER (incompatible types) |
| 119 | Column filter on denied column | orders WHERE internalNote > status (tenant-user) | rule 4 — ACCESS_DENIED (column in filter) |
| 120 | `between` malformed value | orders WHERE total between { from: 100 } (missing `to`) | rule 5 — INVALID_VALUE (malformed compound value) |
| 121 | `levenshteinLte` negative maxDistance | users WHERE lastName levenshteinLte { text: 'x', maxDistance: -1 } | rule 5 — INVALID_VALUE (maxDistance must be non-negative integer) |
| 122 | `in` with empty array | orders WHERE status in [] | rule 5 — INVALID_VALUE (empty array produces invalid SQL) |
| 123 | Column filter non-existent refColumn | orders WHERE total > nonexistent (QueryColumnFilter) | rule 2 — UNKNOWN_COLUMN (refColumn side) |
| 139 | Filter with `table` referencing non-joined table | users query, filter `{ column: 'status', table: 'orders', operator: '=' }` without joining orders | rule 5 — table 'orders' is not `from` and not in `joins` |
| 140 | `in` with mismatched element types | orders WHERE status IN (1, 2) but status is 'string' | rule 5 — INVALID_VALUE: array elements must match column type |
| 141 | `table` in having filter rejected | orders GROUP BY status, HAVING { column: 'totalSum', table: 'orders' } | rule 8 — `table` not allowed in `having` filters |
| 143 | byIds with composite PK | orders byIds=[1,2] but table has composite PK [tenantId, id] | rule 10 — byIds requires single-column primary key |
| 145 | `notBetween` malformed value | orders WHERE total notBetween { from: 100 } (missing `to`) | rule 5 — INVALID_VALUE (malformed compound value, same as between) |
| 146 | `notIn` on date column | orders WHERE createdAt notIn ['2024-01-01'] | rule 5 — `notIn` rejected on `timestamp` type |
| 150 | `in` with null element | orders WHERE status IN ('active', null) | rule 5 — INVALID_VALUE: null in `in`/`notIn` array rejected (NOT IN + NULL = 0 rows) |
| 151 | QueryColumnFilter in HAVING group | orders GROUP BY status, HAVING group with QueryColumnFilter { column: 'totalSum', refColumn: 'avgTotal' } | rule 8 — HAVING rejects `QueryColumnFilter` (aliases, not table columns) |
| 153 | `contains` operator in HAVING | orders GROUP BY status, HAVING { column: 'totalSum', operator: 'contains', value: '100' } | rule 8 — INVALID_HAVING: pattern operators rejected in HAVING |
| 154 | `levenshteinLte` in HAVING | orders GROUP BY status, HAVING { column: 'totalSum', operator: 'levenshteinLte', value: { text: '100', maxDistance: 1 } } | rule 8 — INVALID_HAVING: function operators rejected in HAVING |
| 165 | Nested EXISTS | orders EXISTS(invoices WHERE EXISTS(users WHERE role='admin')) | rule 12 — inner EXISTS resolves `users` relation against `invoices` (outer EXISTS table), not `orders` |
| 167 | `levenshteinLte` fractional maxDistance | users WHERE lastName levenshteinLte { text: 'x', maxDistance: 1.5 } | rule 5 — INVALID_VALUE (maxDistance must be non-negative integer) |
| 168 | `between` with null `from` | orders WHERE total between { from: null, to: 100 } | rule 5 — INVALID_VALUE (`from`/`to` must not be null — SQL BETWEEN NULL always false) |
| 169 | `notBetween` with null `to` | orders WHERE total notBetween { from: 0, to: null } | rule 5 — INVALID_VALUE (same rationale as `between` null rejection) |
| 173 | `arrayContains` on scalar column | orders WHERE status arrayContains 'active' (status is 'string', not array) | rule 5 — INVALID_FILTER (array operators only valid on array column types) |
| 174 | Scalar operator on array column | events WHERE tags = 'urgent' (tags is 'string[]') | rule 5 — INVALID_FILTER (scalar operators except isNull/isNotNull rejected on array columns) |
| 175 | `arrayContains` type mismatch | events WHERE tags arrayContains 123 (tags is 'string[]', value is number) | rule 5 — INVALID_VALUE (element type must match: expected string) |
| 176 | `arrayContainsAll` empty array | products WHERE labels arrayContainsAll [] | rule 5 — INVALID_VALUE (empty array — same as `in` empty rejection) |
| 177 | `arrayContainsAny` type mismatch | products WHERE labels arrayContainsAny [1, 2] (labels is 'string[]') | rule 5 — INVALID_VALUE (all elements must match element type) |
| 178 | `arrayContains` in HAVING | orders GROUP BY status, HAVING { column: 'totalSum', operator: 'arrayContains' } | rule 8 — INVALID_HAVING: array operators rejected in HAVING |
| 179 | SUM on array column | events SUM(tags) as tagSum | rule 14 — INVALID_AGGREGATION: sum/avg/min/max rejected on array columns |
| 180 | QueryColumnFilter on array column | events WHERE tags > status (tags is 'string[]') | rule 5 — array columns not allowed in QueryColumnFilter |
| 187 | Null element in `arrayContainsAll` | products WHERE labels arrayContainsAll ['sale', null] | rule 5 — INVALID_VALUE: null elements rejected (same as `in`/`notIn`) |

#### `packages/core/tests/init/` — init-time errors (ConnectionError, ProviderError)

| # | Scenario | Input | Error |
|---|---|---|---|
| 53 | Connection failed | executor ping fails at init | ConnectionError: CONNECTION_FAILED |
| 54 | Metadata provider fails | MetadataProvider.load() throws | ProviderError: METADATA_LOAD_FAILED |
| 55 | Role provider fails | RoleProvider.load() throws | ProviderError: ROLE_LOAD_FAILED |
| 63 | Lazy connections | validateConnections: false | init succeeds, healthCheck detects issues |

#### `packages/core/tests/access/` — role-based column trimming, masking, scope logic

| # | Scenario | Input | Focus |
|---|---|---|---|
| 13 | Admin role | any table | all columns visible |
| 14 | Tenant-user role | orders | subset columns only |
| 14b | Column masking | orders (tenant-user) | total masked |
| 14c | Multi-role within scope | orders (tenant-user + regional-manager) | union within user scope (all order columns) |
| 14d | Cross-scope restriction | orders (admin user + orders-service) | restricted to orders-service tables |
| 14f | Omitted scope | orders (only user: ['admin'], no service) | no service restriction |
| 16 | Column trimming on byIds | users byIds + limited columns in role | only intersected columns |
| 38 | Columns omitted | orders (no columns specified, tenant-user) | returns only role-allowed columns |
| 95 | Empty scope intersection | events (analytics-reader user + orders-service) | service scope excludes events → ACCESS_DENIED |
| 104 | Empty roles array | orders (user: []) | zero roles → zero permissions → ACCESS_DENIED |
| 106 | Cross-scope masking | orders (user: [regional-manager], service: [svc-role w/ maskedColumns: ['total']]) | user scope unmasks total, service scope masks it → stays masked (scope intersection) |

#### `packages/core/tests/planner/` — strategy selection (P0–P4)

| # | Scenario | Tables | Strategy |
|---|---|---|---|
| 1 | Single PG table | orders | direct → pg-main |
| 2 | Join within same PG | orders + products | direct → pg-main |
| 3 | Cross-PG, debezium available | orders + tenants | materialized → pg-main (tenants replicated) |
| 4 | Cross-PG, no debezium | orders + invoices | trino cross-db |
| 5 | PG + CH, debezium available | orders + events | materialized → ch-analytics (orders replicated) |
| 6 | PG + CH, no debezium | users + events | trino cross-db |
| 7 | PG + Iceberg | orders + ordersArchive | trino or materialized |
| 8 | By-ID with cache hit | users byIds=[1,2,3] | cache → redis |
| 9 | By-ID cache miss | orders byIds=[1,2] | direct → pg-main |
| 10 | By-ID partial cache | users byIds=[1,2,3] (1,2 cached) | cache + direct merge |
| 11 | Freshness=realtime | orders + events (realtime) | skip materialized (lag=seconds), trino |
| 12 | Freshness=hours | orders + events (hours ok) | materialized → ch-analytics |
| 19 | Trino disabled | cross-db query | PlannerError: TRINO_DISABLED |
| 33 | byIds + filters (cache skip) | users byIds=[1,2] + filter role='admin' | direct → pg-main (cache skipped) |
| 56 | No Trino catalog | trino enabled, database missing trinoCatalog | PlannerError: NO_CATALOG |
| 57 | Freshness unmet | realtime required, all paths have lag | PlannerError: FRESHNESS_UNMET |
| 59 | Unreachable tables | metrics + tenants (no replica, no trino catalog) | PlannerError: UNREACHABLE_TABLES |
| 64 | Multi-table join (3 tables) | orders + products + users (all pg-main) | direct → pg-main, 2 JOINs |
| 79 | Single Iceberg table query | ordersArchive | direct via trino executor (Trino dialect, single catalog) |
| 103 | Cache column subset → P0 skip | products byIds=[1,2], columns: [id, name, price] but cache only has [id, name, category] | skip cache → direct DB |
| 130 | byIds + columns subset, cache hit | users byIds=[1,2], columns: [id, email] (all in cache) | cache → redis (requested columns are subset of cached columns) |

#### `packages/core/tests/generator/` — SQL generation per dialect

| # | Scenario | Input | SQL Feature |
|---|---|---|---|
| 20 | Aggregation query | orders GROUP BY status, SUM(total) | GROUP BY + SUM |
| 21 | Aggregation + join | orders + products GROUP BY category | cross-table GROUP BY |
| 22 | HAVING clause | orders GROUP BY status HAVING SUM(total) > 100 | HAVING with aggregate |
| 23 | DISTINCT query | orders DISTINCT status | SELECT DISTINCT |
| 24 | Cross-table ORDER BY | orders + products ORDER BY products.category | qualified ORDER BY |
| 25 | EXISTS filter | orders WHERE EXISTS invoices(status='paid') | EXISTS subquery |
| 26 | NOT EXISTS filter | users WHERE NOT EXISTS orders | NOT EXISTS subquery |
| 27 | Nested EXISTS + filter group | orders WHERE (status='active' OR EXISTS invoices) | EXISTS inside OR group |
| 160 | Counted EXISTS | orders WHERE EXISTS invoices(count: { operator: '>=', value: 3 }) | `(SELECT COUNT(*) FROM invoices WHERE ...) >= $N` — `WhereCountedSubquery` IR |
| 161 | Counted EXISTS with filters | orders WHERE EXISTS invoices(status='paid', count: { operator: '=', value: 2 }) | `(SELECT COUNT(*) FROM invoices WHERE ... AND status='paid') = $N` |
| 162 | `exists` omitted (defaults true) | orders { table: 'invoices' } (no explicit `exists`) | EXISTS subquery — `exists` defaults to `true` |
| 157 | `exists: false` + `count` (counted subquery SQL) | orders EXISTS(false) invoices count: { operator: '>=', value: 3 } | `(SELECT COUNT(*) FROM invoices WHERE ...) >= $N` — `exists` ignored, `WhereCountedSubquery` emitted |
| 166 | COUNT(column) NULL-skipping | invoices COUNT(orderId) as orderCount (some orderId are NULL) | `SELECT COUNT(t0."order_id") ...` — counts non-NULL rows only; meta.columns[].type = 'int' |
| 28 | OR filter group | orders WHERE (status='active' OR total > 100) | OR clause |
| 29 | Negated filter group | orders WHERE NOT (status='cancelled' AND total = 0) | NOT (...) |
| 30 | ILIKE filter | users WHERE email ILIKE '%@example%' | dialect-specific ILIKE |
| 45 | isNull filter | orders WHERE productId IS NULL | IS NULL |
| 66 | `in` filter | orders WHERE status IN ('active','shipped') | IN array param binding |
| 67 | `notIn` filter | orders WHERE status NOT IN ('cancelled') | NOT IN |
| 68 | `isNotNull` filter | orders WHERE productId IS NOT NULL | IS NOT NULL |
| 69 | Join-scoped filter | orders JOIN products, products.category = 'electronics' | filter on QueryJoin.filters |
| 70 | Deeply nested WHERE | orders WHERE (status='active' OR (total > 100 AND ...)) | 3-level nested AND/OR |
| 71 | Mixed top-level filters | orders WHERE status + group + exists combined | filter + group + exists |
| 72 | Multiple HAVING conditions | orders GROUP BY status HAVING SUM > 100 AND COUNT > 5 | two aggregate HAVING conditions |
| 73 | HAVING with OR group | orders HAVING (SUM(total) > 1000 OR AVG(total) > 200) | OR inside HAVING |
| 74 | `like` filter | orders WHERE status LIKE 'act%' | case-sensitive LIKE |
| 75 | `notLike` filter | orders WHERE status NOT LIKE '%cancel%' | NOT LIKE |
| 77 | Order by aggregation alias | GROUP BY status, SUM(total) as totalSum, ORDER BY totalSum | ORDER BY aggregate alias |
| 83 | Aggregation-only query (`columns: []`) | orders columns: [], SUM(total) | `SELECT SUM(total) FROM orders` |
| 84 | `columns: undefined` + aggregations | orders columns: undefined, GROUP BY status, SUM(total) | groupBy columns only (not all) |
| 85 | Join with `columns: []` | orders JOIN products (columns: []), GROUP BY products.category | join for groupBy only, no product columns in SELECT |
| 90 | Basic happy-path | orders WHERE tenantId = 'abc' | simple SELECT + WHERE, no joins/aggregations |
| 91 | LEFT JOIN | orders LEFT JOIN products | LEFT JOIN clause, null-safe behavior |
| 92 | Trino cross-catalog SQL | orders (pg-main) + events (ch-analytics) via Trino | `pg_main.public.orders` cross-catalog references |
| 93 | COUNT(*) aggregation | orders COUNT(*) as totalOrders | `SELECT COUNT(*) as "totalOrders" FROM ...` |
| 94 | Dialect: parameter binding | orders WHERE status = 'active' | PG: `$1`, CH: `{p1:String}`, Trino: `?` |
| 99 | `!=` filter | orders WHERE status != 'cancelled' | `!=` / `<>` per dialect |
| 100 | `notIlike` filter | users WHERE email NOT ILIKE '%@test%' | dialect-specific NOT ILIKE |
| 101 | `>=` / `<=` filters | orders WHERE total >= 100 AND total <= 500 | range comparison |
| 102 | MIN/MAX aggregations | orders MIN(createdAt) as earliest, MAX(createdAt) as latest | MIN/MAX preserve source type (timestamp) |
| 108 | `levenshteinLte` filter | users WHERE lastName levenshteinLte { text: 'smith', maxDistance: 2 } | PG: `levenshtein(col,$1)<=$2`, CH: `editDistance(col,{p1:String})<={p2:UInt32}`, Trino: `levenshtein_distance(col,?)<= ?` |
| 110 | `between` filter | orders WHERE total BETWEEN 100 AND 500 | `col BETWEEN $1 AND $2` per dialect |
| 111 | `contains` filter | users WHERE email contains 'example' | `LIKE '%example%'` (value auto-escaped) |
| 112 | `icontains` filter | users WHERE email icontains 'EXAMPLE' | dialect-specific case-insensitive `LIKE '%example%'` |
| 113 | `startsWith` filter | users WHERE email startsWith 'admin' | `LIKE 'admin%'` |
| 114 | `istartsWith` filter | users WHERE email istartsWith 'ADMIN' | dialect-specific case-insensitive `LIKE 'ADMIN%'` |
| 115 | Column-vs-column filter | orders WHERE total > discount (QueryColumnFilter) | `t0."total_amount" > t0."discount"` — no params |
| 124 | Cross-table column filter | orders JOIN products, orders.total > products.price (QueryColumnFilter) | `t0."total_amount" > t1."price"` — cross-table, no params |
| 125 | `contains` wildcard escaping | users WHERE email contains 'test%user' | `LIKE '%test\%user%'` — `%` in value auto-escaped |
| 126 | Aggregation on joined column | orders JOIN products, SUM(products.price) as totalPrice | `SUM(t1."price")` — aggregation references joined table |
| 127 | 3-table JOIN | orders + products + users (all pg-main) | 2 JOIN clauses in generated SQL |
| 128 | `between` on timestamp | orders WHERE createdAt between { from: '2024-01-01', to: '2024-12-31' } | `t0."created_at" BETWEEN $1 AND $2` |
| 129 | AVG aggregation | orders AVG(total) as avgTotal | `SELECT AVG(t0."total_amount") as "avgTotal"` |
| 155 | HAVING with `between` | orders GROUP BY status, HAVING SUM(total) BETWEEN 100 AND 500 | `HAVING SUM(t0."total_amount") BETWEEN $N AND $N+1` — uses `HavingBetween` IR |
| 156 | byIds + JOIN | orders byIds=[uuid1,uuid2] JOIN products | `SELECT ... FROM orders t0 LEFT JOIN products t1 ON ... WHERE t0."id" = ANY($1)` — cache skipped |
| 133 | `endsWith` filter | users WHERE email endsWith '@example.com' | `LIKE '%@example.com'` |
| 134 | `iendsWith` filter | users WHERE email iendsWith '@EXAMPLE.COM' | dialect-specific case-insensitive `LIKE '%@EXAMPLE.COM'` |
| 135 | `notBetween` filter | orders WHERE total NOT BETWEEN 0 AND 10 | `NOT (col BETWEEN $1 AND $2)` per dialect |
| 136 | `notContains` filter | users WHERE email notContains 'spam' | `NOT LIKE '%spam%'` |
| 137 | `notIcontains` filter | users WHERE email notIcontains 'SPAM' | dialect-specific case-insensitive `NOT LIKE '%SPAM%'` |
| 138 | Top-level filter on joined column | orders JOIN products, top-level filter: { column: 'category', table: 'products', operator: '=', value: 'electronics' } | `t1."category" = $1` in WHERE (same as QueryJoin.filters) |
| 142 | INNER JOIN | orders INNER JOIN products | `INNER JOIN` clause (vs default LEFT) |
| 144 | NOT in HAVING group | orders GROUP BY status, HAVING NOT (SUM(total) > 100 OR COUNT(*) > 5) | `NOT (HAVING_cond1 OR HAVING_cond2)` — negated HAVING group |
| 147 | Multi-join with per-table filters | orders JOIN products (category='electronics') JOIN users (role='admin') | 2 JOINs + 2 WHERE conditions from joined tables |
| 148 | Filter with `table` = `from` table | orders, filter: { column: 'status', table: 'orders', operator: '=', value: 'active' } | `t0."order_status" = $1` — explicit from-table reference, same as omitting `table` |
| 149 | `endsWith` wildcard escaping | users WHERE email endsWith '100%off' | `LIKE '%100\%off'` — `%` in value auto-escaped |
| 163 | `distinct` + `groupBy` | orders DISTINCT, GROUP BY status, SUM(total) as totalSum | both accepted (valid SQL) — `DISTINCT` has no effect when `GROUP BY` is present |
| 164 | SUM on all-NULL column | orders SUM(discount) as totalDiscount (all discount values are NULL) | result: `totalDiscount = null`, `meta.columns[].nullable = true` (source column is nullable) |
| 181 | `arrayContains` filter | events WHERE tags arrayContains 'urgent' | PG: `$1::text = ANY(t0."tags")`, CH: `has(t0.\`tags\`, {p1})`, Trino: `contains(t0."tags", ?)` |
| 182 | `arrayContainsAll` filter | products WHERE labels arrayContainsAll ['sale', 'new'] | PG: `t0."labels" @> $1::text[]`, CH: `hasAll(t0.\`labels\`, [{p1}, {p2}])`, Trino: `cardinality(array_except(ARRAY[?,?], t0."labels")) = 0` |
| 183 | `arrayContainsAny` filter | products WHERE labels arrayContainsAny ['sale', 'clearance'] | PG: `t0."labels" && $1::text[]`, CH: `hasAny(...)`, Trino: `arrays_overlap(t0."labels", ARRAY[?,?])` |
| 184 | `arrayIsEmpty` filter | events WHERE tags arrayIsEmpty | PG: `cardinality(t0."tags") = 0`, CH: `empty(t0.\`tags\`)`, Trino: `cardinality(t0."tags") = 0` |
| 185 | `arrayIsNotEmpty` filter | products WHERE labels arrayIsNotEmpty | PG: `cardinality(t0."labels") > 0`, CH: `notEmpty(...)`, Trino: `cardinality(...) > 0` |
| 186 | `isNull` on array column | events WHERE tags IS NULL (tags is 'string[]', nullable: true) | `t0."tags" IS NULL` — `isNull`/`isNotNull` valid on array columns (same syntax as scalars) |
| 188 | `notBetween` in HAVING | orders GROUP BY status, HAVING SUM(total) NOT BETWEEN 0 AND 10 | `NOT ("totalSum" BETWEEN $N AND $N+1)` — uses `HavingBetween` IR with `not: true` |

#### `packages/core/tests/cache/` — cache strategy + masking on cached data

| # | Scenario | Input | Focus |
|---|---|---|---|
| 35 | Masking on cached results | users byIds=[1,2] (tenant-user) | cache → redis, email still masked |

#### `packages/core/tests/e2e/` — full pipeline integration

| # | Scenario | Input | Focus |
|---|---|---|---|
| 14e | Count mode | orders (count) | returns count only |
| 31 | SQL-only mode | orders (sql-only) | returns SqlResult with sql + params |
| 39 | Debug mode | orders (debug: true) | result includes debugLog entries |
| 152 | byIds + count mode | users byIds=[1,2,3] (count) | `SELECT COUNT(*) FROM ... WHERE id = ANY($1)` — counts existing IDs |
| 44 | Executor missing | events (no ch-analytics executor) | ExecutionError: EXECUTOR_MISSING |
| 48 | Cache provider missing | users byIds=[1] (no redis provider) | ExecutionError: CACHE_PROVIDER_MISSING |
| 58 | Query execution fails | orders (executor throws at runtime) | ExecutionError: QUERY_FAILED (includes sql + params + dialect) |
| 131 | Query timeout | orders (executor exceeds timeoutMs) | ExecutionError: QUERY_TIMEOUT (includes sql + timeoutMs + dialect) |
| 132 | Error toJSON() serialization | any error type | toJSON() returns plain object, cause chain preserved, JSON.stringify() safe |
| 60 | Health check | all executors + cache providers | healthCheck() returns per-provider status |
| 61 | Hot-reload metadata | reloadMetadata() with new table added | next query sees new table |
| 62 | Reload failure | reloadRoles() with failing provider | ProviderError, old config preserved |
| 76 | Count + groupBy ignored | orders GROUP BY status, SUM(total) (count mode) | groupBy/aggregations/having ignored, returns scalar count |
| 105 | close() lifecycle | call multiDb.close() | all executors + cache providers closed; subsequent queries throw `ExecutionError` (`EXECUTOR_MISSING`) — the executor is no longer available |
| 170 | close() partial failure | 2 executors, 1 cache; executor pg-tenant close() throws | all 3 close() calls attempted; thrown error lists pg-tenant failure; subsequent queries throw EXECUTOR_MISSING |
| 171 | Reload during in-flight query | `reloadMetadata()` called while `query()` is executing | in-flight query uses old config (snapshot); next query sees new config |
| 172 | Executor timeout enforcement | orders query with pg-main timeoutMs: 100, slow query | `ExecutionError: QUERY_TIMEOUT` with `timeoutMs: 100` |

### Sample Column Definitions (orders table)

```ts
const ordersColumns: ColumnMeta[] = [
  { apiName: 'id',           physicalName: 'id',              type: 'uuid',      nullable: false },
  { apiName: 'tenantId',     physicalName: 'tenant_id',       type: 'uuid',      nullable: false },
  { apiName: 'customerId',   physicalName: 'customer_id',     type: 'uuid',      nullable: false },
  { apiName: 'productId',    physicalName: 'product_id',      type: 'uuid',      nullable: true },
  { apiName: 'regionId',     physicalName: 'region_id',       type: 'string',    nullable: false },
  { apiName: 'total',        physicalName: 'total_amount',    type: 'decimal',   nullable: false, maskingFn: 'number' },
  { apiName: 'discount',     physicalName: 'discount',        type: 'decimal',   nullable: true },
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
  { apiName: 'labels',    physicalName: 'labels',      type: 'string[]', nullable: true },   // text[] in Postgres
  { apiName: 'tenantId',  physicalName: 'tenant_id',   type: 'uuid',    nullable: false },
]
```

### Sample Column Definitions (events table — ClickHouse)

```ts
const eventsColumns: ColumnMeta[] = [
  { apiName: 'id',        physicalName: 'id',          type: 'uuid',      nullable: false },
  { apiName: 'type',      physicalName: 'event_type',  type: 'string',    nullable: false },
  { apiName: 'userId',    physicalName: 'user_id',     type: 'uuid',      nullable: false },
  { apiName: 'orderId',   physicalName: 'order_id',    type: 'uuid',      nullable: true },
  { apiName: 'payload',   physicalName: 'payload',     type: 'string',    nullable: true,  maskingFn: 'full' },
  { apiName: 'tags',      physicalName: 'tags',        type: 'string[]',  nullable: true },   // Array(String) in ClickHouse
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
  { apiName: 'tags',      physicalName: 'tags',        type: 'string[]',  nullable: true },  // Array(String) in ClickHouse
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
  { column: 'userId',  references: { table: 'users', column: 'id' }, type: 'many-to-one' },
  { column: 'orderId', references: { table: 'orders', column: 'id' }, type: 'many-to-one' },
]

const ordersArchiveRelations: RelationMeta[] = [
  { column: 'id',         references: { table: 'orders', column: 'id' }, type: 'one-to-one' },
  { column: 'customerId', references: { table: 'users', column: 'id' }, type: 'many-to-one' },
  { column: 'tenantId',   references: { table: 'tenants', column: 'id' }, type: 'many-to-one' },
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
| Column disambiguation | Qualified apiNames in result rows: `orders.id`, `products.id` when join produces collisions | Flat rows use `Record<string, unknown>` — keys must be unique. When the `from` table and a joined table share a column apiName (e.g. both have `id`), the result row qualifies colliding keys with `{tableApiName}.{columnApiName}`. Non-colliding columns keep their bare apiName. `meta.columns[].apiName` reflects the actual key used in the result row (qualified if colliding). SQL generation aliases columns via `AS "orders__id"` / `AS "products__id"` internally, then ColumnMapping translates to the qualified apiName |
| Executor timeout | Per-executor `timeoutMs` in factory config, no global default | Each DB has different performance profiles — Postgres: 30s, ClickHouse: 60s, Trino: 120s. Driver-level enforcement (`statement_timeout`, `max_execution_time`) is more reliable than `Promise.race` |
| Concurrent safety | `query()` uses snapshot of metadata/roles; `reload*()` atomically swaps references | No locking needed — immutable config per query, atomic reference swap for reloads |
| `close()` error handling | Attempt all providers, collect failures, throw aggregate error | Partial close would leak connections — always try all, report all failures |
| Array columns | `ScalarColumnType` + `ArrayColumnType` union, 5 array operators, element type derived by stripping `[]` | All three backends (Postgres, ClickHouse, Trino/Iceberg) support arrays natively. Element type validation reuses `ScalarColumnType`. Array columns excluded from `QueryColumnFilter` and `sum`/`avg`/`min`/`max` aggregations |

---

## Monorepo Packages

| Package | Purpose | Dependencies |
|---|---|---|
| `@mkven/multi-db-validation` | Types, error classes, config validation, query validation (rules 1–14), apiName validation | **zero** I/O deps |
| `@mkven/multi-db` | Core: metadata registry, planner, SQL generators, name resolution, masking, debug logger | `@mkven/multi-db-validation` |
| `@mkven/multi-db-executor-postgres` | Postgres connection + execution | `pg` |
| `@mkven/multi-db-executor-clickhouse` | ClickHouse connection + execution | `@clickhouse/client` |
| `@mkven/multi-db-executor-trino` | Trino connection + execution | `trino-client` |
| `@mkven/multi-db-cache-redis` | Redis cache provider (Debezium-synced) | `ioredis` |

All error classes (including runtime ones like `ExecutionError`, `PlannerError`, `ConnectionError`) live in the validation package so that client code can use `instanceof` checks and access typed error fields without depending on the core package. The validation package is a type+error+validation-only package — it contains no I/O, no planner, no SQL generators.

`@mkven/multi-db-validation` is the **client-side** package — it contains all types, error classes, and validation logic. Clients can validate configs and queries locally before sending to the server, failing fast without pulling in planner/SQL generators. The core package depends on it and re-exports its types.

```ts
import { validateQuery, validateConfig, validateApiName } from '@mkven/multi-db-validation'
import type { QueryDefinition, MetadataConfig, ExecutionContext } from '@mkven/multi-db-validation'

// At config time — validate metadata before shipping to server
const configErrors = validateConfig(metadata)       // returns ConfigError | null

// At query time — validate query before sending
const queryErrors = validateQuery(definition, context, metadata, roles)  // returns ValidationError | null
```

Both `validateConfig()` and `validateQuery()` return `null` on success, or the corresponding error object (not thrown) so the caller can decide how to handle it. The core package still throws these errors internally.

`validateQuery()` builds lightweight internal indexes (table-by-apiName, column-by-apiName maps) on each call. For hot paths, callers can pre-index metadata once and pass the indexed form — the function accepts both raw `MetadataConfig` and a pre-indexed `MetadataIndex` (exported from the package) to avoid repeated indexing.

```ts
// Pre-indexed metadata for hot-path validation
interface MetadataIndex {
  tablesByApiName: Map<string, TableMeta>
  columnsByTable: Map<string, Map<string, ColumnMeta>>  // tableApiName → columnApiName → ColumnMeta
  rolesByName: Map<string, RoleMeta>
}
```

Core has **zero I/O dependencies** — usable for SQL-only mode without any DB drivers. Each executor is a thin adapter that consumers install only if needed.

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
│   │       ├── config/              # scenarios 49–52, 80, 81, 89, 96
│   │       └── query/               # scenarios 15, 17, 18, 32, 34, 36, 37, 40–43, 46, 47, 65, 78, 82, 86–88, 97, 98, 107, 109, 116–123, 139–141, 143, 145, 146, 150, 151, 153, 154, 157–159, 165, 167–169, 173–180, 187
│   │
│   ├── core/                        # @mkven/multi-db
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts             # public API (re-exports types + errors from validation package)
│   │       ├── metadata/
│   │       │   ├── registry.ts      # MetadataRegistry — stores and indexes all metadata
│   │       │   └── providers.ts     # MetadataProvider / RoleProvider interfaces + static helpers
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
│   │       ├── fixtures/
│   │       │   └── testConfig.ts     # shared test config (reuses validation fixtures + adds executors)
│   │       ├── init/                # scenarios 53, 54, 55, 63
│   │       ├── access/              # scenarios 13, 14, 14b–14f, 16, 38, 95, 104, 106
│   │       ├── planner/             # scenarios 1–12, 19, 33, 56, 57, 59, 64, 79, 103, 130
│   │       ├── generator/           # scenarios 20–30, 45, 66–77, 83–85, 90–94, 99–102, 108, 110–115, 124–129, 133–138, 142, 144, 147–149, 155–157, 160–164, 166, 181–186, 188
│   │       ├── cache/               # scenario 35
│   │       └── e2e/                 # scenarios 14e, 31, 39, 44, 48, 58, 60–62, 76, 105, 131, 132, 152, 170–172
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
