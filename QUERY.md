← [Back to README](./README.md)

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
  // Join resolution: the joined table must have a relation to the `from` table OR to any
  // already-joined table (transitive joins). For example, {from: 'users', joins: [{table: 'orders'},
  // {table: 'invoices'}]} is valid if invoices→orders has a relation, even though invoices has no
  // direct relation to users. The ON clause references the intermediary (orders), not the from table.
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
// Both columns must have compatible types: same type, or same type family
// (numeric: int/decimal, temporal: date/timestamp). Cross-family rejected.
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

