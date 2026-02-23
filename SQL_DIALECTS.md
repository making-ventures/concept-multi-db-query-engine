← [Back to README](./README.md)

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
| BETWEEN | `col BETWEEN $1 AND $2` | `col BETWEEN {p1:Type} AND {p2:Type}` | `col BETWEEN ? AND ?` |
| NOT BETWEEN | `col NOT BETWEEN $1 AND $2` | `NOT (col BETWEEN {p1:Type} AND {p2:Type})` | `col NOT BETWEEN ? AND ?` |
| Boolean | `true/false` | `1/0` | `true/false` |
| Counted subquery (`=`/`!=`) | `(SELECT COUNT(*) FROM ... WHERE ...) = $N` | `(SELECT COUNT(*) FROM ... WHERE ...) = {pN:UInt64}` | `(SELECT COUNT(*) FROM ... WHERE ...) = ?` |
| Counted subquery (`>=`/`>`) | `(SELECT COUNT(*) FROM (SELECT 1 ... LIMIT N)) >= $N` | `col IN (SELECT fk ... GROUP BY fk HAVING COUNT(*) >= {pN:UInt64})` | `col IN (SELECT fk ... GROUP BY fk HAVING COUNT(*) >= ?)` |
| Counted subquery (`<`/`<=`) | `(SELECT COUNT(*) FROM ... WHERE ...) < $N` | `col NOT IN (SELECT fk ... GROUP BY fk HAVING COUNT(*) >= {pN:UInt64})` | `col NOT IN (SELECT fk ... GROUP BY fk HAVING COUNT(*) >= ?)` |
| Array column type | `text[]`, `integer[]`, etc. | `Array(String)`, `Array(Int32)`, etc. | `array(varchar)`, `array(integer)`, etc. |
| `arrayContains` | `$1 = ANY(col)` | `has(col, {p1})` | `contains(col, ?)` |
| `arrayContainsAll` | `col @> $1::type[]` | `hasAll(col, [{p1}, ...])` | `cardinality(array_except(ARRAY[?,...], col)) = 0` |
| `arrayContainsAny` | `col && $1::type[]` | `hasAny(col, [{p1}, ...])` | `arrays_overlap(col, ARRAY[?,...])` |
| `arrayIsEmpty` | `cardinality(col) = 0` | `empty(col)` | `cardinality(col) = 0` |
| `arrayIsNotEmpty` | `cardinality(col) > 0` | `notEmpty(col)` | `cardinality(col) > 0` |

**Performance note — counted subqueries:** For `>=` / `>` comparisons, each dialect optimizes to avoid scanning all rows. **Postgres** wraps in a `LIMIT`-ed inner query: `(SELECT COUNT(*) FROM (SELECT 1 FROM ... LIMIT N)) >= N`, short-circuiting once the threshold is reached. **ClickHouse** and **Trino** decorrelate into semi-joins: `col IN (SELECT fk GROUP BY fk HAVING COUNT(*) >= N)`, avoiding correlated sub-selects entirely. For `<` / `<=`, CH/Trino invert with `NOT IN` and a flipped `HAVING` operator — this correctly includes parent rows with zero matching children (which a correlated `COUNT(*)` comparison would also include). For `=` / `!=` the exact count is needed; all dialects use standard correlated `(SELECT COUNT(*) FROM ...) = N`.

**Postgres `in`/`notIn` type mapping:** The `= ANY($1::type[])` syntax requires an explicit array type cast. The system maps `ColumnType` → SQL type:

| ColumnType | Postgres SQL Type | Array Cast |
|---|---|---|
| `string` | `text` | `$1::text[]` |
| `int` | `integer` | `$1::integer[]` |
| `decimal` | `numeric` | `$1::numeric[]` |
| `uuid` | `uuid` | `$1::uuid[]` |

Only types that support `in`/`notIn` are listed (`boolean`, `date`, `timestamp` are rejected by rule 5). ClickHouse and Trino expand values inline and don't need type casts.

**ClickHouse typed parameters:** Every ClickHouse parameter requires a type annotation (`{p1:String}`, `{p2:Int32}`, etc.). When `columnType` is available (from `WhereCondition.columnType` or `WhereArrayCondition.elementType`), the dialect maps `ColumnType` → ClickHouse type:

| ColumnType | ClickHouse Type | Example |
|---|---|---|
| `string` | `String` | `{p1:String}` |
| `int` | `Int32` | `{p1:Int32}` |
| `decimal` | `Decimal` | `{p1:Decimal}` |
| `boolean` | `Bool` | `{p1:Bool}` |
| `uuid` | `UUID` | `{p1:UUID}` |
| `date` | `Date` | `{p1:Date}` |
| `timestamp` | `DateTime` | `{p1:DateTime}` |

Array element types use the same mapping — e.g. `has(col, {p1:String})` for `arrayContains` on `'string[]'`. `in`/`notIn` expand inline as `IN tuple(...)` with each value typed individually. When no `columnType` is available, the dialect infers the type from the JS runtime value (string → `String`, integer → `Int32`, float → `Float64`, boolean → `Bool`). Some contexts use hardcoded types: `UInt64` for counted subquery comparison values, `UInt32` for levenshtein distance thresholds. The `Type` placeholder in the dialect table above denotes a type resolved at generation time.

**Postgres array operator type mapping:** `arrayContainsAll` (`@>`) and `arrayContainsAny` (`&&`) use the same type casts as `in`/`notIn` — e.g. `col @> $1::text[]`. `arrayContains` (`= ANY`) uses the scalar type cast — e.g. `$1::text = ANY(col)`. ClickHouse and Trino use function-based syntax and don't need casts.

Each engine gets a `SqlDialect` implementation.

### SQL Generation Architecture

SQL generation uses an intermediate `SqlParts` representation that is **internal** and dialect-agnostic. It operates entirely in physical names — no apiNames. The pipeline is:

```
QueryDefinition → (planner + access control) → name resolution → ResolveResult → SqlDialect.generate(parts, params) → { sql, params }
                                                                       ↓
                                        { parts: SqlParts, params: unknown[], columnMappings: ColumnMapping[], mode }
```

Name resolution produces a `ResolveResult` containing:
1. `SqlParts` — purely physical names, used for SQL generation
2. `params: unknown[]` — ordered parameter values matching `SqlParts` param indexes
3. `ColumnMapping[]` — the mapping table used to rename result columns back to apiNames
4. `mode` — `'data'` for normal queries, `'count'` when `countMode` was requested

```ts
// Built during name resolution, used after execution to map results back
interface ColumnMapping {
  physicalName: string                // 'total_amount'; for aggregations: same as alias (e.g. 'totalSum')
  apiName: string                     // 'total'; for aggregations: the alias (e.g. 'totalSum')
  tableAlias: string                  // 't0'; for aggregations: the source column's table alias (or from-table alias for count(*))
  tableApiName: string                // table's apiName — needed for meta.columns[].fromTable and collision-qualified keys
  masked: boolean                     // apply masking after fetch (always false for aggregation aliases)
  nullable: boolean                   // mirrors ColumnMeta.nullable — needed for meta.columns[].nullable
  type: ColumnType                    // logical column type; for aggregations: inferred from fn (count → 'int', avg → always 'decimal', sum/min/max → source column type)
  maskingFn?: 'email' | 'phone' | 'name' | 'uuid' | 'number' | 'date' | 'full'
                                      // which masking function to apply — sourced from effective access resolution
                                      // (defaults to 'full' when column is masked but ColumnMeta has no maskingFn)
}
```

The resolver's full output is captured in `ResolveResult`:

```ts
interface ResolveResult {
  parts: SqlParts                     // physical-name SQL parts — fed to dialect
  params: unknown[]                   // ordered parameter values matching SqlParts param indexes
  columnMappings: ColumnMapping[]     // apiName ↔ physicalName mapping for result rows
  mode: 'data' | 'count'             // 'data' for normal queries; 'count' when countMode is requested
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
  countMode?: boolean                 // when true, dialect emits SELECT COUNT(*) instead of regular SELECT
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
  paramIndexes?: number[]             // contains: single index; containsAll/containsAny: single index pointing to an array value (dialects expand per-element as needed); omit for isEmpty/isNotEmpty
  elementType: string                 // logical element type (e.g. 'string', 'int') — each dialect maps to SQL type (Postgres array casts, ClickHouse typed params, Trino inline expansion)
}

// HAVING tree — same as WhereNode but excludes EXISTS, WhereColumnCondition, WhereFunction, and WhereArrayCondition
// Only comparison + range + null operators are allowed (no pattern/function/array operators on aliases)
type HavingNode = WhereCondition | HavingBetween | HavingGroup

// Range condition on an aggregation alias — uses bare string, not ColumnRef
interface HavingBetween {
  alias: string                       // aggregation alias (e.g. 'totalSum')
  not?: boolean                       // when true, negates the range; per-dialect form varies (see dialect table above)
  fromParamIndex: number
  toParamIndex: number
}

interface HavingGroup {
  logic: 'and' | 'or'
  not?: boolean                       // when true, emits NOT (...)
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

// Subquery table aliases use a separate prefix `s` and continue the shared alias counter.
// If the outer query uses t0 and t1 (a join), the correlated subquery's table gets s2.
// This avoids alias collisions when the same physical table appears in both contexts.

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
  paramIndex?: number                 // index into SqlParts.params; omitted for operator-only conditions (isNull, isNotNull)
  columnType?: string                 // logical column type (e.g. 'string', 'int', 'uuid') — needed for type-specific SQL (ClickHouse typed params, Postgres IN/NOT IN array casts)
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
  not?: boolean                       // when true, negates the range — used by 'notBetween' operator; per-dialect form varies (see dialect table above)
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
  generate(parts: SqlParts, params: unknown[]): { sql: string; params: unknown[] }
}
```

Each `SqlDialect` takes a `SqlParts` and the resolver's collected parameter array, and produces `{ sql: string, params: unknown[] }`. The dialect reads values from the incoming `params` array by index (using `paramIndex`, `fromParamIndex`, etc. from `SqlParts` nodes) and emits dialect-specific placeholders (`$1`, `{p1:Type}`, `?`). The dialect resolves each `ColumnRef` with its own quoting rules:
- Postgres: `t0."created_at"`
- ClickHouse: `` t0.`created_at` ``
- Trino: `t0."created_at"`

No external SQL generation packages are used — the query shape is predictable (SELECT with optional WHERE/JOIN/GROUP BY/HAVING/ORDER BY/LIMIT/OFFSET) and each dialect is ~200–300 lines.

