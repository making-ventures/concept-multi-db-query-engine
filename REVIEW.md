# Review: Problems & Improvements

## Bugs / Inconsistencies

### 1. `having` missing from `QueryDefinition`

`SqlParts` has `having: WhereCondition[]` and test scenario #22 tests HAVING, but `QueryDefinition` had no `having` field.

**Resolution:** Added `having?: QueryFilter[]` to `QueryDefinition`. Reuses existing `QueryFilter` type — semantically same as WHERE but applied post-GROUP BY.

### 2. No join type in `QueryJoin`

`JoinClause` in `SqlParts` has `type: 'inner' | 'left'`, but `QueryJoin` had no way to specify it.

**Resolution:** Added `type?: 'inner' | 'left'` to `QueryJoin`, defaulting to `'left'` (safe for nullable FKs).

### 3. Debug log tag mismatch

Debug example used `[access]`, but `DebugLogEntry.phase` defines it as `'access-control'`.

**Resolution:** Changed example to `[access-control]`.

### 4. Missing `orders → products` relation

Test scenario #2 ("Join within same PG: orders + products") requires a relation, but `ordersRelations` only had `customerId → users` and `tenantId → tenants`.

**Resolution:** Added `productId` column to orders table + `productId → products.id` relation.

### 5. "Matches eng project ecosystem"

Design Decisions table had an internal reference.

**Resolution:** Changed to "Type-safe, wide ecosystem".

---

## Missing Test Fixtures

### 6. No column definitions for 3 tables

`metrics`, `invoices`, and `ordersArchive` were used in test scenarios and roles but had no sample column definitions.

**Resolution:** Added full column definitions for all three tables.

---

## Unaddressed Edge Cases

### 7. `byIds` with composite primary keys

`primaryKey: string[]` can be multi-column, but `byIds?: (string | number)[]` is a flat array.

**Resolution:** Restricted `byIds` to single-column primary keys (documented in the field comment). Composite PK lookups use filters instead.

### 8. `byIds` + filters

If a `byIds` query also has `filters`, should cache results be post-filtered?

**Resolution:** Cache strategy is skipped when `filters` are present alongside `byIds`. Documented in Priority 0 cache section.

### 9. Cache + masking

Cache returns pre-fetched data — is masking still applied?

**Resolution:** Added explicit note: masking is applied on cache hits identically to DB results.

### 10. `count` mode + `columns`

When `executeMode = 'count'`, are `columns` ignored?

**Resolution:** Added clarification: in count mode, `columns`, `orderBy`, `limit`, and `offset` are ignored.

---

## Minor Type Improvements

### 11. `orderBy` inline type

`orderBy` used an inline `{ column: string; direction: 'asc' | 'desc' }[]` while other clauses had named interfaces.

**Resolution:** Extracted to `QueryOrderBy` interface for consistency.

### 12. `WhereCondition.operator` is `string`

`QueryFilter.operator` uses a proper union type, `WhereCondition.operator` in `SqlParts` is just `string`.

**Resolution:** Kept as `string` — it's internal, and dialects may emit operators not in the public union (e.g. `ILIKE`, `ANY`). Added a comment explaining why.

### 13. Return type of `createMultiDb()` not defined

The init example calls `createMultiDb({...})` but the returned object's type wasn't documented.

**Resolution:** Added `MultiDb` interface with `query()` method signature.

---

## Structural

### 14. Objective / Overview overlap

Both sections mentioned strategy selection, access control, name mapping, debug logging, and execution modes.

**Resolution:** Merged into a single "Objective" section. The Overview was redundant — the architecture diagram already covers "how".

---

## Round 2

### 15. Debug log references nonexistent `customers` table

Example showed `[planning] Tables needed: [orders(pg-main), customers(pg-main)]`. No `customers` table exists in metadata.

**Resolution:** Changed to `users(pg-main)`. Also fixed masking reference from `customerEmail` to `total` (which is the column actually masked for tenant-user on orders).

### 16. Test scenario 14b incorrect

Said "customerEmail masked" for tenant-user on orders. But `customerEmail` isn't a column on orders; tenant-user's orders config masks `total`.

**Resolution:** Changed to "total masked".

### 17. Test scenario #4 missing join path

"orders + invoices → trino cross-db" requires a relation between the tables, but none existed.

**Resolution:** Added `orderId` column to invoices table + `invoices → orders` relation. Also added `invoices → tenants` relation.

### 18. `ColumnMapping` carries `type` but needs `maskingFn`

When `masked: true`, post-processing needs to know which masking function to apply. `type` alone can't determine this.

**Resolution:** Added `maskingFn` field to `ColumnMapping`, sourced from `ColumnMeta`.

### 19. `QueryResult` union lacks discriminant tag

`SqlResult | DataResult | CountResult` are structurally distinguishable but lack a proper discriminant.

**Resolution:** Added `kind: 'sql' | 'data' | 'count'` to each result type — idiomatic TypeScript discriminated union.

### 20. `count(*)` not expressible in `QueryAggregation`

`QueryAggregation.column` was `string` (required), but `COUNT(*)` doesn't reference a specific column.

**Resolution:** Changed to `column: string | '*'`. When `'*'`, `table` is ignored.

### 21. `having` column references aggregation aliases

`having?: QueryFilter[]` reuses `QueryFilter` where `column` means apiName. In HAVING context, `column` refers to `QueryAggregation.alias`, not a table column.

**Resolution:** Added inline comment documenting this semantic difference.
