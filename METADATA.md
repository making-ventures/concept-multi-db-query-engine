← [Back to README](./README.md)

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
  id: string                          // logical table id: 'orders', 'orders-archive' — free-form (may contain hyphens), unlike apiName which follows camelCase rules
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
  references: { table: string; column: string }  // table by id, column by apiName
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
| `name` | Show first + last char | `John Smith` → `J********h` |
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
import { staticMetadata, staticRoles } from '@mkven/multi-db-query'

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

