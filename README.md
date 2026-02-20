# @mkven/multi-db — Concept Document

## Objective

Build a reusable, metadata-driven query engine that lets applications query data across Postgres, ClickHouse, Iceberg, and Redis through a single typed API. The engine:

- Accepts queries using **apiNames** (decoupled from physical schema)
- Automatically selects the **optimal execution strategy** — direct DB, cached, materialized replica, or Trino cross-DB federation
- Enforces **scoped access control** — user roles and service roles intersected to determine effective permissions
- Applies **column masking** for sensitive data based on role
- Returns either **generated SQL** or **executed results** depending on the caller's needs
- Provides **structured debug logs** for transparent pipeline tracing

The package (`@mkven/multi-db`) is designed for the eng project but built as a standalone, reusable library with zero I/O dependencies in the core.

## Overview

A metadata-driven query planner and executor that abstracts over heterogeneous database backends. Converts a declarative query definition into optimized database requests with:

- Automatic strategy selection (direct, cached, materialized replica, Trino cross-db)
- Role-based access control (column visibility + column masking per role)
- Name mapping between API-facing names and physical database names
- Structured debug logging for full pipeline transparency
- Support for SQL-only or SQL+execution modes

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
  id: string                          // logical id: 'main-pg', 'analytics-ch'
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
}
```

### Relations

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
  keyPattern: string                  // e.g. 'users:{id}' or 'users:{id}:{tenantId}'
  columns?: string[]                  // subset cached (apiNames); undefined = all
}
```

### Roles & Access Control

Access control is defined **per role**, not per table. Roles are organized into **scopes** (e.g. `user`, `service`). The effective permissions are computed as:

1. **Within a scope** — INTERSECTION (most restrictive). If one role allows all columns and another allows a subset, only the subset is allowed. If roles allow different subsets, only the overlap is allowed.
2. **Between scopes** — INTERSECTION. The effective permissions are the intersection of all scopes.

This ensures that an admin user making a request through a service with restricted access only sees data the service is permitted to handle.

**Example:** Admin user (user scope: all tables, all columns) requests via `orders-service` (service scope: only `orders` + `products` tables). Effective access = only `orders` + `products`.

```ts
type RoleScope = 'user' | 'service'     // extensible

interface RoleMeta {
  id: string                          // 'admin', 'manager', 'orders-service'
  scope: RoleScope                    // which scope this role belongs to
  tables: TableRoleAccess[]
}

interface TableRoleAccess {
  tableId: string
  allowedColumns: string[] | '*'      // apiNames, or '*' for all
  maskedColumns?: string[]            // apiNames of columns to mask (e.g. show '***' or partial)
}
```

### Column Masking

Masking replaces column values in results with obfuscated variants. Masking is a **post-query** operation — data is fetched normally from the database, then masked before returning to the caller.

Masking functions are **predefined** per column type:

| Column Type | Masking Function | Example |
|---|---|---|
| `email` | Show first char + domain hint | `john@example.com` → `j***@***.com` |
| `phone` | Show country code + last 3 digits | `+1234567890` → `+1***890` |
| `string` | Show first + last char | `John Smith` → `J*********h` |
| `uuid` | Show first 4 chars | `a1b2c3d4-...` → `a1b2****` |
| `decimal` / `int` | Replace with `0` | `12345` → `0` |
| `date` / `timestamp` | Truncate to year | `2025-03-15` → `2025-01-01` |

The masking function is selected automatically based on the column's `type` in metadata. Masked columns are still returned in results (not hidden), but their values are obfuscated.

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

---

## Query Definition

### Query Input

```ts
interface QueryDefinition {
  from: string                        // table apiName
  columns?: string[]                  // apiNames; undefined = all allowed for role
  filters?: QueryFilter[]
  joins?: QueryJoin[]
  groupBy?: string[]                  // apiNames to group by
  aggregations?: QueryAggregation[]   // aggregate functions
  limit?: number
  offset?: number
  orderBy?: { column: string; direction: 'asc' | 'desc' }[]
  freshness?: 'realtime' | 'seconds' | 'minutes' | 'hours'  // acceptable lag
  byIds?: (string | number)[]        // shortcut: fetch by primary key(s)
  executeMode?: 'sql-only' | 'execute' | 'count'  // default: 'execute'
  debug?: boolean                     // include debugLog in result (default: false)
}

interface QueryAggregation {
  column: string                      // apiName of column to aggregate
  fn: 'count' | 'sum' | 'avg' | 'min' | 'max'
  alias: string                       // result column name
}

interface QueryJoin {
  table: string                       // related table apiName
  columns?: string[]                  // columns to select from joined table
  filters?: QueryFilter[]            // filters on joined table
}

interface QueryFilter {
  column: string                      // apiName
  operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'in' | 'like' | 'is_null' | 'is_not_null'
  value?: unknown
}
```

### Execution Context

```ts
interface ExecutionContext {
  roles: string[]                     // list of RoleMeta.id — scoped resolution applies
}
```

Roles from different scopes are intersected. Roles within the same scope are also intersected (most restrictive wins within a scope).

### Query Result

Two distinct return types depending on `executeMode`:

```ts
// When executeMode = 'sql-only'
interface SqlResult {
  sql: string                        // generated SQL
  params: unknown[]                  // bound parameters
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// When executeMode = 'execute' (default)
interface DataResult<T = unknown> {
  data: T[]                          // actual query results (masked if applicable)
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// When executeMode = 'count'
interface CountResult {
  count: number                      // total matching rows
  meta: QueryResultMeta
  debugLog?: DebugLogEntry[]          // present only if debug: true
}

// Discriminated union
type QueryResult<T = unknown> = SqlResult | DataResult<T> | CountResult

interface QueryResultMeta {
  strategy: 'direct' | 'cache' | 'materialized' | 'trino-cross-db'
  targetDatabase: string             // which DB was queried
  dialect: DatabaseEngine | 'trino'
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
    fromTable: string                // which table this column came from
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

When you request execution (`executeMode = 'execute'`), you get data back — no SQL. When you request SQL only (`executeMode = 'sql-only'`), you get SQL + params — no execution, no data. When you request count (`executeMode = 'count'`), you get just the row count. All include metadata. Debug log is included only when `debug: true`.

---

## Query Planner Strategy

Given a query touching tables T1, T2, ... Tn:

### Priority 0 — Cache (redis)
- Only for `byIds` queries
- Check if the table has a cache config
- If cache hit for all IDs → return from cache (trim columns if needed)
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

All validation errors are descriptive and include what was expected vs what was provided.

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
  type: string                        // column type (for masking function selection)
}
```

`SqlParts` is strictly internal — no apiNames, no masking concerns:

```ts
interface SqlParts {
  select: string[]                    // e.g. ['t0."id"', 't0."total_amount"', 't1."name"']
  from: TableRef
  joins: JoinClause[]
  where: WhereCondition[]
  groupBy: string[]                   // e.g. ['t0."order_status"']
  having: WhereCondition[]            // filters on aggregated values
  aggregations: AggregationClause[]
  orderBy: string[]                   // e.g. ['t0."created_at" DESC']
  limit?: number
  offset?: number
}

interface TableRef {
  physicalName: string                // 'public.orders'
  alias: string                       // 't0'
  catalog?: string                    // for trino: 'pg_main'
}

interface JoinClause {
  type: 'inner' | 'left'
  table: TableRef
  on: string                          // 't0."customer_id" = t1."id"'
}

interface WhereCondition {
  expression: string                  // 't0."order_status"'
  operator: string                    // '='
  paramIndex?: number                 // for parameterized values
  literal?: string                    // for IS NULL, IS NOT NULL
}

interface AggregationClause {
  fn: 'count' | 'sum' | 'avg' | 'min' | 'max'
  expression: string                  // 't0."total_amount"'
  alias: string                       // result column name (apiName for the aggregation)
}
```

Each `SqlDialect` takes a `SqlParts` and produces `{ sql: string, params: unknown[] }` with correct identifier quoting, parameter binding syntax, and engine-specific functions. No external SQL generation packages are used — the query shape is predictable (SELECT with optional WHERE/JOIN/GROUP BY/HAVING/ORDER BY/LIMIT/OFFSET) and each dialect is ~200–300 lines.

---

## Debug Logging — Example Flow

```
[validation]  Resolving table 'orders' → found in metadata (database: pg-main)
[validation]  Column 'id' → valid (type: uuid)
[validation]  Column 'total' → valid (type: decimal)
[validation]  Column 'internalNote' → DENIED for role 'tenant-user' (not in allowedColumns)
[access]      Trimming columns to allowed set: [id, total, status, createdAt]
[access]      Masking column 'customerEmail' for roles [tenant-user]
[planning]    Tables needed: [orders(pg-main), customers(pg-main)]
[planning]    All tables in 'pg-main' → strategy: DIRECT
[name-res]    orders.total → public.orders.total_amount
[name-res]    orders.tenantId → public.orders.tenant_id
[sql-gen]     Dialect: postgres
[sql-gen]     SELECT "id", "total_amount" AS "total", ... FROM "public"."orders" WHERE "tenant_id" = $1
[execution]   Executing on pg-main → 42 rows, 8ms
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

| Role | Scope | Access |
|---|---|---|
| `admin` | user | All tables, all columns |
| `tenant-user` | user | orders + customers, subset columns |
| `regional-manager` | user | orders (all cols) |
| `analytics-reader` | user | events + metrics + ordersArchive only |
| `no-access` | user | No tables (edge case) |
| `orders-service` | service | orders + products only |
| `full-service` | service | All tables, all columns |

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
| 14b | Column masking | orders (tenant-user) | customerEmail masked |
| 14c | Multi-role union | orders (tenant-user + regional-manager) | intersection within user scope |
| 14d | Cross-scope restriction | orders (admin user + orders-service) | restricted to orders-service tables |
| 14e | Count mode | orders (count) | returns count only |
| 15 | No-access role | any table | denied |
| 16 | Column trimming on byIds | users byIds + limited columns in role | only intersected columns |
| 17 | Invalid table name | nonexistent | validation error |
| 18 | Invalid column name | orders.nonexistent | validation error |
| 19 | Trino disabled | cross-db query | error with explanation |
| 20 | Aggregation query | orders GROUP BY status, SUM(total) | correct SQL per dialect |
| 21 | Aggregation + join | orders + products GROUP BY category | correct cross-table aggregation |
| 22 | HAVING clause | orders GROUP BY status HAVING SUM(total) > 100 | correct HAVING per dialect |

### Sample Column Definitions (orders table)

```ts
const ordersColumns: ColumnMeta[] = [
  { apiName: 'id',           physicalName: 'id',              type: 'uuid',      nullable: false },
  { apiName: 'tenantId',     physicalName: 'tenant_id',       type: 'uuid',      nullable: false },
  { apiName: 'customerId',   physicalName: 'customer_id',     type: 'uuid',      nullable: false },
  { apiName: 'regionId',     physicalName: 'region_id',       type: 'string',    nullable: false },
  { apiName: 'total',        physicalName: 'total_amount',    type: 'decimal',   nullable: false },
  { apiName: 'status',       physicalName: 'order_status',    type: 'string',    nullable: false },
  { apiName: 'internalNote', physicalName: 'internal_note',   type: 'string',    nullable: true  },
  { apiName: 'createdAt',    physicalName: 'created_at',      type: 'timestamp', nullable: false },
]
```

---

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Package name | `@mkven/multi-db` | Org-scoped, reusable |
| Language | TypeScript | Matches eng project ecosystem |
| Query format | Typed object literals | Type-safe, IDE support, no parser |
| Metadata source | Abstracted (hardcoded now, DB/service later) | Flexibility without premature complexity |
| Execution mode | SQL-only (`SqlResult`) or execution (`DataResult`) | Distinct return types per mode |
| Freshness | Prefer original, allow specifying lag tolerance | Correctness by default, performance opt-in |
| Access control | Scoped roles: intersection between scopes, intersection within scope | Admin user via restricted service = restricted |
| SQL generation | Hand-rolled via `SqlParts` IR (internal, physical names only), no external packages | Full control over 3 divergent dialects, zero deps |
| Pagination | Offset-based (`limit` + `offset`) | Simple, sufficient for most use cases |
| Observability | OpenTelemetry | Industry standard, traces + metrics |
| Name mapping | apiName ↔ physicalName on tables and columns | Decouples API from schema |
| Cache | Redis synced by Debezium, no TTL | Always fresh, fast by-ID reads |
| Debug logging | Structured entries per pipeline phase, opt-in via `debug: true` | Zero overhead when not debugging |
| Validation | Strict — only metadata-defined entities | Fail fast with clear errors |
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
│   │       ├── planner/
│   │       │   ├── planner.ts       # QueryPlanner — strategy selection
│   │       │   ├── strategies.ts    # direct, materialized, trino, cache
│   │       │   └── graph.ts         # database connectivity / sync graph
│   │       ├── dialects/
│   │       │   ├── dialect.ts       # SqlDialect interface
│   │       │   ├── postgres.ts
│   │       │   ├── clickhouse.ts
│   │       │   └── trino.ts
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
