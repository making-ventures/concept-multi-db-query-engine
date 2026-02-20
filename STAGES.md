# @mkven/multi-db — Implementation Stages

This document breaks the concept into sequential implementation stages. Each stage produces a working, testable increment. Later stages depend on earlier ones.

---

## Stage 1 — Monorepo Scaffold & Types

**Goal:** Set up the monorepo structure, shared configs, and all TypeScript types — zero runtime code yet.

**Packages touched:** all (scaffold only)

**Tasks:**
1. Init pnpm workspace, root `package.json`, `pnpm-workspace.yaml`
2. Root configs: `tsconfig.json`, `biome.json`, `vitest.config.ts`
3. Create package directories for all 7 packages (validation, core, 3 executors, cache-redis, client)
4. Per-package `package.json` + `tsconfig.json` with correct `references`
5. Define all types in `packages/validation/src/types/`:
   - `metadata.ts` — `DatabaseEngine`, `DatabaseMeta`, `TableMeta`, `ColumnMeta`, `ScalarColumnType`, `ArrayColumnType`, `ColumnType`, `RelationMeta`, `ExternalSync`, `CacheMeta`, `CachedTableMeta`, `MetadataConfig`, `RoleMeta`, `TableRoleAccess`
   - `query.ts` — `QueryDefinition`, `QueryFilter`, `QueryColumnFilter`, `QueryFilterGroup`, `QueryExistsFilter`, `QueryJoin`, `QueryAggregation`, `QueryOrderBy`, `QueryGroupBy`
   - `result.ts` — `QueryResult`, `SqlResult`, `DataResult`, `CountResult`, `QueryResultMeta`, `DebugLogEntry`, `HealthCheckResult`
   - `context.ts` — `ExecutionContext`
6. Define internal IR types in `packages/core/src/`:
   - `SqlParts`, `ColumnRef`, `TableRef`, `WhereNode` (all 8 variants), `HavingNode` (3 variants), `JoinClause`, `OrderByClause`, `AggregationClause`, `ColumnMapping`, `CorrelatedSubquery`
7. Define public executor/cache interfaces in `packages/core/src/`:
   - `DbExecutor`, `CacheProvider` — implemented by executor/cache packages (Stage 11)
8. Export public surface from `packages/validation/src/index.ts`
9. Verify: `pnpm build` succeeds across all packages (types compile)

**Exit criteria:** All interfaces/types compile. `pnpm build` passes. No runtime code yet.

**Test scenarios covered:** none (types only)

---

## Stage 2 — Error Classes

**Goal:** Implement all 7 error classes with `toJSON()` serialization.

**Package:** `@mkven/multi-db-validation`

**Tasks:**
1. Implement `MultiDbError` base class extending `Error`, with `code` and `toJSON()`
2. Implement subclasses:
   - `ConfigError` (`CONFIG_INVALID`, `errors[]` array)
   - `ConnectionError` (`CONNECTION_FAILED | NETWORK_ERROR | REQUEST_TIMEOUT`, discriminated `details`)
   - `ValidationError` (`VALIDATION_FAILED`, `fromTable`, `errors[]` array)
   - `PlannerError` (`UNREACHABLE_TABLES | TRINO_DISABLED | NO_CATALOG | FRESHNESS_UNMET`, `fromTable`, discriminated `details`)
   - `ExecutionError` (`EXECUTOR_MISSING | CACHE_PROVIDER_MISSING | QUERY_FAILED | QUERY_TIMEOUT`, discriminated `details`)
   - `ProviderError` (`METADATA_LOAD_FAILED | ROLE_LOAD_FAILED`, `cause`)
3. `toJSON()` recursive serialization — `cause` chain, `errors[]`, all fields survive `JSON.stringify()`
4. Top-level `Error.message` summaries: `"Config invalid: 3 errors"`, `"Validation failed: 5 errors"`, etc.

**Exit criteria:** All error classes instantiable, `toJSON()` produces correct plain objects, `JSON.stringify()` safe.

**Test scenarios covered:** — (error classes verified by ad hoc unit tests; numbered scenario #132 is the integration test in Stage 12)

---

## Stage 3 — Config Validation

**Goal:** Validate metadata configs — apiName format, uniqueness, references, relations, syncs, caches.

**Package:** `@mkven/multi-db-validation`

**Tasks:**
1. Implement `validateApiName()` — `^[a-z][a-zA-Z0-9]*$`, 1–64 chars, reserved word check
2. Implement `validateConfig(metadata: MetadataConfig): ConfigError | null`
   - `INVALID_API_NAME` — table/column names violating format
   - `DUPLICATE_API_NAME` — table-level global duplicates, column-level per-table duplicates
   - `INVALID_REFERENCE` — tables referencing non-existent databases
   - `INVALID_RELATION` — relations referencing non-existent tables/columns
   - `INVALID_SYNC` — ExternalSync referencing non-existent tables/databases
   - `INVALID_CACHE` — CacheMeta referencing non-existent tables or invalid keyPattern placeholders
3. Collect all errors into a single `ConfigError` (not one-at-a-time)
4. Create shared test fixtures in `packages/validation/tests/fixtures/testConfig.ts`

**Exit criteria:** Config validation catches all structural issues and collects them.

**Test scenarios covered:** #49, #50, #51, #52, #80, #81, #89, #96

---

## Stage 4 — Query Validation (Rules 1–14)

**Goal:** Validate queries against metadata + roles — the full 14-rule set.

**Package:** `@mkven/multi-db-validation`

**Tasks:**
1. Implement `MetadataIndex` (pre-indexed maps for O(1) lookups)
2. Implement `validateQuery(definition, context, metadata, roles): ValidationError | null`
3. Implement all 14 validation rules:
   - Rule 1: Table existence (`UNKNOWN_TABLE`)
   - Rule 2: Column existence (`UNKNOWN_COLUMN`) — including `refColumn` in `QueryColumnFilter`
   - Rule 3: Role permission — table-level (`ACCESS_DENIED`)
   - Rule 4: Column permission — column-level (`ACCESS_DENIED`)
   - Rule 5: Filter validity — operator/type compatibility table, compound value validation (`INVALID_FILTER`, `INVALID_VALUE`), `isNull`/`isNotNull` requires `nullable: true`, array operator restrictions, `table` qualifier validation, `QueryColumnFilter` type/permission checks, `in`/`notIn` null element rejection, `between`/`notBetween` null bounds rejection, element type validation
   - Rule 6: Join validity (`INVALID_JOIN`)
   - Rule 7: Group By validity — ungrouped column check, array column rejection (`INVALID_GROUP_BY`)
   - Rule 8: Having validity — alias existence, `table` rejected, `QueryColumnFilter` rejected, `QueryExistsFilter` rejected, pattern/function/array operators rejected (`INVALID_HAVING`)
   - Rule 9: Order By validity — column/alias existence, array column rejection (`INVALID_ORDER_BY`)
   - Rule 10: ByIds validity — non-empty array, single-column PK, no groupBy/aggregations (`INVALID_BY_IDS`)
   - Rule 11: Limit/Offset validity — non-negative integers, offset requires limit (`INVALID_LIMIT`)
   - Rule 12: Exists filter validity — relation existence, role access, count value checks, nested relation resolution (`INVALID_EXISTS`)
   - Rule 13: Role existence — unknown role IDs (`UNKNOWN_ROLE`)
   - Rule 14: Aggregation validity — alias uniqueness, alias/column collisions, `table` qualifier, empty columns without aggregations, array column restrictions (`INVALID_AGGREGATION`)
4. Recursive validation for filter groups and nested EXISTS
5. All errors collected into single `ValidationError`

**Exit criteria:** All 14 rules enforced. `filterIndex` populated. Recursive nesting validated.

**Test scenarios covered:** #15, #17, #18, #32, #34, #36, #37, #40–43, #46, #47, #65, #78, #82, #86–88, #97, #98, #107, #109, #116–123, #139–141, #143, #145, #146, #150, #151, #153, #154, #157–159, #165, #167–169, #173–180, #187, #190–192, #195, #198, #199, #229–232, #234, #235

---

## Stage 5 — Metadata Registry & Providers

**Goal:** Build the in-memory metadata store with O(1) indexes and the provider abstraction.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Define `MetadataProvider` and `RoleProvider` interfaces
2. Implement `staticMetadata()` and `staticRoles()` helpers
3. Implement `MetadataRegistry`:
   - Load from providers → validate via `validateConfig()` → throw `ConfigError` on failure
   - Build indexes: `Map<apiName, TableMeta>`, `Map<tableId, Map<apiName, ColumnMeta>>`, `Map<roleId, RoleMeta>`, `Map<databaseId, DatabaseMeta>`, `Map<tableId, ExternalSync[]>`, `Map<tableId, CachedTableMeta>`
   - Build database connectivity graph (for planner)
   - Atomic swap on `reload()` — in-flight queries see old config
4. Implement `reloadMetadata()` and `reloadRoles()` — re-call providers, rebuild indexes, preserve old config on failure

**Exit criteria:** Registry loads, indexes, and reloads. Failed reloads preserve previous config. Snapshot isolation for concurrent queries.

**Test scenarios covered:** #54, #55, #62

---

## Stage 6 — Access Control & Masking

**Goal:** Implement role-based column trimming and post-query masking.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Implement scope resolution: UNION within scope, INTERSECTION between scopes
2. Handle edge cases: omitted scope (no restriction), empty array (zero permissions), `tables: '*'`
3. Column trimming — compute effective allowed columns per table
4. Masking resolution — UNION semantics within scope (most permissive), intersection between scopes (most restrictive)
5. Implement masking functions: `email`, `phone`, `name`, `uuid`, `number`, `date`, `full`
6. Default to `full` masking when column has no `maskingFn`
7. Aggregation aliases are never masked

**Exit criteria:** Correct column trimming and masking for all scope combinations.

**Test scenarios covered:** #13, #14, #14b, #14c, #14d, #14f, #16, #38, #95, #104, #106, #233

---

## Stage 7 — Name Resolution

**Goal:** Translate apiNames to physical names, produce `SqlParts` IR and `ColumnMapping[]`.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Implement name resolver: `QueryDefinition` + access-controlled columns → `SqlParts` + `ColumnMapping[]`
2. Table aliasing — `t0`, `t1`, `t2`, ...
3. Column mapping — `apiName → physicalName`, with mask flag and type info
4. Handle column disambiguation — when joined tables share apiNames, qualify result keys with `{tableApiName}.{columnApiName}`
5. Handle `columns: undefined` with aggregations — default to `groupBy` columns only
6. Handle `columns: []` — aggregation-only query
7. Handle `QueryJoin.columns: []` — join for filter/groupBy only
8. Resolve `QueryFilter.table` qualifier — map to correct table alias
9. Resolve `QueryExistsFilter` — correlated subquery with correct parent lookup for nested EXISTS
10. Resolve `QueryColumnFilter` — two `ColumnRef`s, no params
11. Resolve filter operators to `WhereNode` variants (`WhereCondition`, `WhereBetween`, `WhereFunction`, `WhereArrayCondition`, `WhereGroup`, `WhereExists`, `WhereCountedSubquery`)
12. Resolve `having` to `HavingNode` — bare alias strings, not `ColumnRef`
13. Handle `byIds` → produce `WhereCondition` with `in` operator for PK column
14. Handle `count` mode → override to `SELECT COUNT(*)`

**Exit criteria:** All query shapes produce correct `SqlParts` and `ColumnMapping[]`.

**Test scenarios covered:** none directly (resolution is validated through SQL generation tests)

---

## Stage 8 — SQL Generation (Postgres Dialect)

**Goal:** Implement the Postgres `SqlDialect` — first fully working SQL output.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Implement `SqlDialect` interface: `generate(parts: SqlParts): { sql: string; params: unknown[] }`
2. Implement Postgres dialect (~200–300 lines):
   - Identifier quoting: `"column"`
   - Parameter binding: `$1, $2, ...`
   - `in` → `= ANY($1::type[])` with type casts (`text[]`, `integer[]`, `numeric[]`, `uuid[]`)
   - `notIn` → `<> ALL($1::type[])`
   - `like`/`notLike` → `LIKE` / `NOT LIKE` (user-provided pattern)
   - `ilike`/`notIlike` → `ILIKE` / `NOT ILIKE` (native)
   - `startsWith`/`endsWith` → `LIKE 'x%'` / `LIKE '%x'`
   - `istartsWith`/`iendsWith` → `ILIKE 'x%'` / `ILIKE '%x'`
   - `levenshtein(col, $1) <= $2` (requires `fuzzystrmatch`)
   - `BETWEEN $1 AND $2` / `col NOT BETWEEN $1 AND $2`
   - `contains`/`icontains` → `LIKE '%x%'` / `ILIKE '%x%'` with wildcard escaping
   - `notContains`/`notIcontains` → `NOT LIKE` / `NOT ILIKE`
   - Array operators: `= ANY(col)`, `@>`, `&&`, `cardinality()`
   - `IS NULL` / `IS NOT NULL`
   - `SELECT DISTINCT`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, `OFFSET`
   - `EXISTS (SELECT 1 FROM ...)` / `NOT EXISTS`
   - Counted subquery: `(SELECT COUNT(*) FROM ...) >= $N` with optional `LIMIT` optimization
   - `WhereGroup` → `(c1 AND/OR c2)`, `NOT (...)` when `not: true`
   - `WhereColumnCondition` → `t0."col1" > t1."col2"`
   - Aggregation: `COUNT(*)`, `SUM(...)`, `AVG(...)`, `MIN(...)`, `MAX(...)`
   - `INNER JOIN` / `LEFT JOIN` with `ON` condition
3. Wildcard escaping for `%` and `_` in pattern values

**Exit criteria:** All SQL generation tests pass for Postgres dialect.

**Test scenarios covered (Postgres output):** #20–30, #45, #66–75, #77, #83–85, #90–94, #99–102, #108, #110–115, #124–129, #133–138, #142, #144, #147–149, #155–157, #160–164, #166, #181–186, #188–189, #193, #194, #196, #197, #200–207, #227

---

## Stage 9 — SQL Generation (ClickHouse & Trino Dialects)

**Goal:** Add the remaining two dialects.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Implement ClickHouse dialect:
   - Identifier quoting: `` `column` ``
   - Parameter binding: `{p1:Type}` with ClickHouse type mapping (`String`, `Int32`, `Decimal`, `Bool`, `UUID`, `Date`, `DateTime`)
   - `in` → `IN tuple(v1, v2, ...)`; `notIn` → `NOT IN tuple(...)`
   - `ilike(col, pattern)` function syntax
   - `startsWith(col, {p1})` / `endsWith(col, {p2})` function syntax
   - `istartsWith`/`iendsWith` → `ilike(col, 'x%')` / `ilike(col, '%x')`
   - `editDistance(col, {p1:String}) <= {p2:UInt32}`
   - `NOT (col BETWEEN {p1} AND {p2})` for notBetween
   - Boolean: `1/0`
   - Array: `has()`, `hasAll()`, `hasAny()`, `empty()`, `notEmpty()`
2. Implement Trino dialect:
   - Identifier quoting: `"column"` (same as Postgres)
   - Parameter binding: `?` (positional)
   - `in` → `IN (?, ?, ...)` (param expansion); `notIn` → `NOT IN (...)`
   - `lower(col) LIKE lower(pattern)` for case-insensitive
   - `levenshtein_distance(col, ?) <= ?`
   - `col NOT BETWEEN ? AND ?`
   - Array: `contains()`, `array_except()`, `arrays_overlap()`, `cardinality()`
   - Cross-catalog references: `catalog.schema.table`
3. Verify all generator tests pass for all 3 dialects

**Exit criteria:** All SQL generation tests produce correct output for Postgres, ClickHouse, and Trino.

**Test scenarios covered:** same as Stage 8, now verified across all 3 dialects

---

## Stage 10 — Query Planner

**Goal:** Implement strategy selection (P0–P4).

**Package:** `@mkven/multi-db`

**Tasks:**
1. Use the database connectivity graph built in Stage 5 to evaluate reachability
2. Implement P0 — Cache strategy:
   - `byIds` only, no filters, no joins, single-table, single-column PK
   - Column subset check (cache may not have all requested columns)
   - Full hit / partial hit / miss paths
3. Implement P1 — Single database direct:
   - All tables in one database → native dialect
   - Iceberg exception: route through Trino executor with Trino dialect
4. Implement P2 — Materialized replica:
   - Check Debezium syncs for foreign tables
   - Freshness check: `estimatedLag` ≤ query `freshness`
   - Prefer original data; use replicas for foreign tables
   - If multiple candidate databases, prefer the one with most originals
5. Implement P3 — Trino cross-database:
   - Trino enabled, all databases have `trinoCatalog`
   - Select Trino executor and provide catalog-qualified table references to SQL generation (Stage 9)
6. Implement P4 — Error:
   - `PlannerError` with specific code (`UNREACHABLE_TABLES`, `TRINO_DISABLED`, `NO_CATALOG`, `FRESHNESS_UNMET`)
7. Strategy priority: P0 > P1 > P2 > P3 > P4

**Exit criteria:** Planner selects correct strategy for all topology variants.

**Test scenarios covered:** #1–12, #19, #33, #56, #57, #59, #64, #79, #103, #130

---

## Stage 11 — Executor Packages

**Goal:** Implement the three database executor packages and the Redis cache provider.

**Packages:** `executor-postgres`, `executor-clickhouse`, `executor-trino`, `cache-redis`

**Tasks:**
1. **Postgres executor** (`pg` driver):
   - `execute(sql, params)` → `Record<string, unknown>[]`
   - `ping()` → `SELECT 1`
   - `close()` → end pool
   - `timeoutMs` → `statement_timeout`
2. **ClickHouse executor** (`@clickhouse/client`):
   - Named parameter binding
   - `ping()` → `SELECT 1`
   - `timeoutMs` → `max_execution_time`
3. **Trino executor** (`trino-client`):
   - Positional `?` parameter binding
   - `ping()` → `SELECT 1`
   - `timeoutMs` → query timeout
4. **Redis cache provider** (`ioredis`):
   - `getMany(keys)` → `Map<string, Record | null>` via `MGET`
   - `ping()` → `PING`
   - `close()` → `quit()`
5. All factories accept connection config + optional `timeoutMs`

**Exit criteria:** Each package connects, executes, pings, and closes correctly.

**Test scenarios covered:** none directly (executors are integration-tested through e2e)

---

## Stage 12 — Core Pipeline & `createMultiDb()`

**Goal:** Wire all pieces into the full `createMultiDb()` initialization and `query()` pipeline.

**Package:** `@mkven/multi-db`

**Tasks:**
1. Implement `createMultiDb(options)`:
   - Load providers → validate → index → build graph → ping (in order)
   - Throw `ProviderError` / `ConfigError` / `ConnectionError` on failure
   - `validateConnections: false` for lazy ping
2. Implement `multiDb.query()` pipeline:
   - Snapshot current metadata/roles
   - Validate → access control → plan → resolve names → generate SQL → execute (or return SQL / count)
   - Apply masking to results
   - Build `QueryResultMeta` with strategy, dialect, tablesUsed, columns (type inference for aggregations), timing
   - `count` mode: ignore columns, orderBy, limit, offset, distinct, groupBy, aggregations, having
3. Implement `multiDb.healthCheck()` — ping all executors + cache providers, return per-provider status with latency
4. Implement `multiDb.close()` — attempt all providers, collect failures, throw aggregate `ConnectionError`
5. Debug logging — emit `DebugLogEntry` per phase when `debug: true`
6. Executor missing / cache provider missing error paths
7. Query execution failure → `ExecutionError: QUERY_FAILED` with sql + params + dialect + cause
8. Query timeout → `ExecutionError: QUERY_TIMEOUT`

**Exit criteria:** Full query pipeline works end-to-end. Init, reload, health, and close lifecycle complete.

**Test scenarios covered:** #14e, #31, #35, #39, #44, #48, #53, #58, #60, #61, #63, #76, #105, #131, #132, #152, #170, #171, #172, #228

---

## Stage 13 — HTTP Client & Contract Tests

**Goal:** Implement the HTTP client package and shared contract test suite.

**Package:** `@mkven/multi-db-client`

**Tasks:**
1. Implement `createMultiDbClient(config)`:
   - `query()` → POST `/query`, deserialize `QueryResult` by `kind`
   - `healthCheck()` → GET `/health`
   - Error deserialization — reconstruct typed error classes from `toJSON()` body using `code` field
   - Custom headers, injectable `fetch`, timeout via `AbortController`
   - Optional `validateBeforeSend` with local `validateQuery()` call
2. Implement error reconstruction — HTTP status + body → `ValidationError`, `PlannerError`, `ExecutionError`, `ConnectionError`, `ProviderError`
3. Implement `QueryContract` interface
4. Implement `describeQueryContract(name, factory)` — parameterized test suite:
   - Simple select, filter + join, aggregation, validation error, access denied, count mode, SQL-only mode
5. Write in-process factory and HTTP client factory for contract tests

**Exit criteria:** HTTP client sends/receives correctly. Contract tests pass for both in-process and HTTP implementations.

**Test scenarios covered:** #208–218, #219–225, #226

---

## Stage 14 — Final Verification

**Goal:** Run all 235 test scenarios together, verify cross-stage integration, and confirm no regressions.

**Package:** all

**Tasks:**
1. Run full test suite across all packages — `pnpm test` at monorepo root
2. Verify all 240 unique scenarios pass — some appear in multiple packages (e.g. #157 is validated in both validation/query and core/generator)
3. Smoke-test the full pipeline end-to-end: init → query → reload → healthCheck → close
4. Verify `pnpm build` produces clean packages with correct dependency graph
5. Review test coverage gaps — any untested code paths surfaced by coverage reports

**Exit criteria:** All 240 test scenarios pass across all packages. Build succeeds. No regressions.

---

## Stage Summary

| Stage | Package | Focus | Scenarios |
|---|---|---|---|
| 1 | all | Scaffold + types | — |
| 2 | validation | Error classes | — |
| 3 | validation | Config validation | 8 |
| 4 | validation | Query validation (14 rules) | 69 |
| 5 | core | Metadata registry + providers | 3 |
| 6 | core | Access control + masking | 12 |
| 7 | core | Name resolution | — |
| 8 | core | SQL gen — Postgres | 89 |
| 9 | core | SQL gen — ClickHouse + Trino | — (same 89, 2 more dialects) |
| 10 | core | Query planner (P0–P4) | 21 |
| 11 | executor-*, cache-redis | DB executors + Redis cache | — |
| 12 | core | Full pipeline + lifecycle | 20 |
| 13 | client | HTTP client + contract tests | 19 |
| 14 | all | Final verification | — |
| **Total** | | | **240** |

\* Per-stage counts sum to 241 because #157 appears in both Stage 4 (validation) and Stage 8 (SQL generation). Unique total: 240 = 235 integer-numbered scenarios (#1–#235) + 5 sub-variants (#14b, #14c, #14d, #14e, #14f).

**Dependency graph:**

```
Stage 1 (scaffold)
  └─► Stage 2 (errors)
        └─► Stage 3 (config validation)
              └─► Stage 4 (query validation)
                    └─► Stage 5 (metadata registry)
                          ├─► Stage 6 (access control)
                          │     └─► Stage 7 (name resolution)
                          │           └─► Stage 8 (SQL gen: Postgres)
                          │                 └─► Stage 9 (SQL gen: CH + Trino) ─┐
                          ├─► Stage 10 (planner) ──────────────────────────────┤
                          └─► Stage 11 (executors) ────────────────────────────┤
                                                                               ▼
                                                                   Stage 12 (pipeline)
                                                                     ├─► Stage 13 (HTTP client)
                                                                     └─► Stage 14 (final verification)
```

Three branches run in parallel from Stage 5: the SQL generation chain (6→7→8→9), the planner (10), and the executor packages (11). Stage 12 requires all three. This means up to three developers can work simultaneously after Stage 5 is complete.
