← [Back to README](./README.md)

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
| 116 | `between` on boolean | orders WHERE isPaid between { from: true, to: false } | rule 5 — INVALID_FILTER (boolean not orderable) |
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
| 146 | `notIn` on timestamp column | orders WHERE createdAt notIn ['2024-01-01'] | rule 5 — `notIn` rejected on `timestamp` type |
| 150 | `in` with null element | orders WHERE status IN ('active', null) | rule 5 — INVALID_VALUE: null in `in`/`notIn` array rejected (NOT IN + NULL = 0 rows) |
| 151 | QueryColumnFilter in HAVING group | orders GROUP BY status, HAVING group with QueryColumnFilter { column: 'totalSum', refColumn: 'avgTotal' } | rule 8 — HAVING rejects `QueryColumnFilter` (aliases, not table columns) |
| 153 | `contains` operator in HAVING | orders GROUP BY status, HAVING { column: 'totalSum', operator: 'contains', value: '100' } | rule 8 — INVALID_HAVING: pattern operators rejected in HAVING |
| 154 | `levenshteinLte` in HAVING | orders GROUP BY status, HAVING { column: 'totalSum', operator: 'levenshteinLte', value: { text: '100', maxDistance: 1 } } | rule 8 — INVALID_HAVING: function operators rejected in HAVING |
| 165 | Nested EXISTS (invalid relation) | orders EXISTS(invoices WHERE EXISTS(users WHERE role='admin')) | rule 12 — INVALID_EXISTS: inner EXISTS resolves `users` against `invoices` (outer EXISTS), not `orders`; invoices has no relation to users → rejected |
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
| 180 | QueryColumnFilter on array column | events WHERE tags > type (tags is 'string[]') | rule 5 — array columns not allowed in QueryColumnFilter |
| 187 | Null element in `arrayContainsAll` | products WHERE labels arrayContainsAll ['sale', null] | rule 5 — INVALID_VALUE: null elements rejected (same as `in`/`notIn`) |
| 190 | Comparison on boolean | orders WHERE isPaid > true | rule 5 — INVALID_FILTER (comparison operators `>`, `<`, `>=`, `<=` rejected on boolean) |
| 191 | `in` on boolean | orders WHERE isPaid IN (true, false) | rule 5 — INVALID_FILTER (`in`/`notIn` rejected on boolean) |
| 192 | `in` on date | invoices WHERE dueDate IN ('2024-06-01') | rule 5 — INVALID_FILTER (`in`/`notIn` rejected on date type) |
| 195 | `between` value type mismatch | orders WHERE total BETWEEN 'abc' AND 'xyz' (total is decimal) | rule 5 — INVALID_VALUE (`from`/`to` must match column type) |
| 198 | Array column in GROUP BY | orders GROUP BY priorities, COUNT(*) as cnt | rule 7 — INVALID_GROUP_BY (array columns rejected in GROUP BY) |
| 199 | Array column in ORDER BY | orders ORDER BY priorities ASC | rule 9 — INVALID_ORDER_BY (array columns rejected in ORDER BY) |
| 229 | `notIn` on date column | invoices WHERE dueDate notIn ['2024-06-01'] | rule 5 — INVALID_FILTER (`notIn` rejected on `date` type) |
| 230 | `in` on timestamp column | orders WHERE createdAt IN ('2024-01-01T00:00:00Z') | rule 5 — INVALID_FILTER (`in` rejected on `timestamp` type) |
| 231 | `between` on uuid column | orders WHERE id BETWEEN 'uuid1' AND 'uuid2' | rule 5 — INVALID_FILTER (`between` rejected on `uuid` — no meaningful ordering) |
| 232 | `notIn` on boolean column | orders WHERE isPaid NOT IN (true) | rule 5 — INVALID_FILTER (`notIn` rejected on `boolean` type) |
| 234 | `notBetween` on boolean column | orders WHERE isPaid NOT BETWEEN true AND false | rule 5 — INVALID_FILTER (`notBetween` rejected on `boolean` — not orderable) |
| 235 | `notBetween` on uuid column | orders WHERE id NOT BETWEEN 'uuid1' AND 'uuid2' | rule 5 — INVALID_FILTER (`notBetween` rejected on `uuid` — no meaningful ordering) |

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
| 233 | Aggregation alias unmasked | orders (tenant-user): GROUP BY status, SUM(total) as totalSum | totalSum `masked: false` — aggregation aliases never masked even when source column has `maskingFn: 'number'` |

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
| 112 | `icontains` filter | users WHERE email icontains 'EXAMPLE' | PG: `ILIKE '%EXAMPLE%'`, CH: `ilike(t0.\`email\`, '%EXAMPLE%')`, Trino: `lower(t0."email") LIKE lower('%EXAMPLE%')` |
| 113 | `startsWith` filter | users WHERE email startsWith 'admin' | PG: `LIKE 'admin%'`, CH: `startsWith(t0.\`email\`, {p1:String})`, Trino: `LIKE 'admin%'` |
| 114 | `istartsWith` filter | users WHERE email istartsWith 'ADMIN' | PG: `ILIKE 'ADMIN%'`, CH: `ilike(t0.\`email\`, 'ADMIN%')`, Trino: `lower(t0."email") LIKE lower('ADMIN%')` |
| 115 | Column-vs-column filter | orders WHERE total > discount (QueryColumnFilter) | `t0."total_amount" > t0."discount"` — no params |
| 124 | Cross-table column filter | orders JOIN products, orders.total > products.price (QueryColumnFilter) | `t0."total_amount" > t1."price"` — cross-table, no params |
| 125 | `contains` wildcard escaping | users WHERE email contains 'test%user' | `LIKE '%test\%user%'` — `%` in value auto-escaped |
| 126 | Aggregation on joined column | orders JOIN products, SUM(products.price) as totalPrice | `SUM(t1."price")` — aggregation references joined table |
| 127 | 3-table JOIN | orders + products + users (all pg-main) | 2 JOIN clauses in generated SQL |
| 128 | `between` on timestamp | orders WHERE createdAt between { from: '2024-01-01', to: '2024-12-31' } | `t0."created_at" BETWEEN $1 AND $2` |
| 129 | AVG aggregation | orders AVG(total) as avgTotal | `SELECT AVG(t0."total_amount") as "avgTotal"` |
| 155 | HAVING with `between` | orders GROUP BY status, HAVING SUM(total) BETWEEN 100 AND 500 | `HAVING SUM(t0."total_amount") BETWEEN $N AND $N+1` — uses `HavingBetween` IR |
| 156 | byIds + JOIN | orders byIds=[uuid1,uuid2] JOIN products | `SELECT ... FROM orders t0 LEFT JOIN products t1 ON ... WHERE t0."id" = ANY($1)` — cache skipped |
| 133 | `endsWith` filter | users WHERE email endsWith '@example.com' | PG: `LIKE '%@example.com'`, CH: `endsWith(t0.\`email\`, {p1:String})`, Trino: `LIKE '%@example.com'` |
| 134 | `iendsWith` filter | users WHERE email iendsWith '@EXAMPLE.COM' | PG: `ILIKE '%@EXAMPLE.COM'`, CH: `ilike(t0.\`email\`, '%@EXAMPLE.COM')`, Trino: `lower(t0."email") LIKE lower('%@EXAMPLE.COM')` |
| 135 | `notBetween` filter | orders WHERE total NOT BETWEEN 0 AND 10 | PG: `col NOT BETWEEN $1 AND $2`, CH: `NOT (col BETWEEN {p1} AND {p2})`, Trino: `col NOT BETWEEN ? AND ?` |
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
| 188 | `notBetween` in HAVING | orders GROUP BY status, HAVING SUM(total) NOT BETWEEN 0 AND 10 | PG: `"totalSum" NOT BETWEEN $N AND $N+1`, CH: `NOT ("totalSum" BETWEEN {pN} AND {pN+1})`, Trino: `"totalSum" NOT BETWEEN ? AND ?` — uses `HavingBetween` IR with `not: true` |
| 189 | `=` on boolean column | orders WHERE isPaid = true | PG: `t0."is_paid" = $1`, CH: `t0.\`is_paid\` = {p1:Bool}`, Trino: `t0."is_paid" = ?` |
| 193 | `between` on int column | orders WHERE quantity BETWEEN 1 AND 10 | `t0."quantity" BETWEEN $1 AND $2` — int orderable type, same syntax as decimal |
| 194 | `in` on int column | orders WHERE quantity IN (1, 5, 10) | PG: `t0."quantity" = ANY($1::integer[])` — int array cast; CH: `IN tuple(...)`, Trino: `IN (?, ?, ?)` |
| 196 | `_` wildcard escaping in `contains` | users WHERE email contains 'test_user' | `LIKE '%test\_user%'` — `_` in value auto-escaped (single-char wildcard) |
| 197 | `isNull` in HAVING | orders GROUP BY status, HAVING SUM(discount) IS NULL | `HAVING "discountSum" IS NULL` — valid: checks if aggregation returned NULL (all values NULL); no `nullable` check needed on aliases |
| 200 | `isNotNull` on array column | events WHERE tags IS NOT NULL (tags is 'string[]', nullable: true) | `t0."tags" IS NOT NULL` — symmetric to #186 |
| 201 | COUNT on array column | events COUNT(tags) as tagCount | `COUNT(t0."tags")` — only valid aggregation on arrays; counts non-NULL values |
| 202 | Array column in SELECT | products columns: [name, labels] | labels returned as JSON array — `SELECT t0."name", t0."labels"` |
| 203 | `arrayContains` on `int[]` column | orders WHERE priorities arrayContains 1 | PG: `$1::integer = ANY(t0."priorities")`, CH: `has(t0.\`priorities\`, {p1:Int32})`, Trino: `contains(t0."priorities", ?)` |
| 204 | `arrayContainsAll` on `int[]` column | orders WHERE priorities arrayContainsAll [1, 2, 3] | PG: `t0."priorities" @> $1::integer[]`, CH: `hasAll(t0.\`priorities\`, [{p1:Int32}, ...])`, Trino: `cardinality(array_except(ARRAY[?,?,?], t0."priorities")) = 0` |
| 205 | Array filter in AND group | products WHERE (labels arrayContainsAny ['sale'] AND price > 10) | `(t0."labels" && $1::text[] AND t0."price" > $2)` — array filter inside `QueryFilterGroup` |
| 206 | Array filter on joined table | orders JOIN products, filter: { column: 'labels', table: 'products', operator: 'arrayContainsAny', value: ['sale'] } | `t1."labels" && $1::text[]` — array filter using `table` qualifier on joined table column |
| 207 | `arrayContainsAll` single element | products WHERE labels arrayContainsAll ['sale'] | PG: `t0."labels" @> $1::text[]` — single-element array is valid (same syntax, param is `['sale']`) |
| 227 | Nested EXISTS (valid) | orders EXISTS(invoices WHERE EXISTS(tenants)) | `EXISTS (SELECT 1 FROM invoices t1 WHERE t1."order_id" = t0."id" AND EXISTS (SELECT 1 FROM tenants t2 WHERE t2."id" = t1."tenant_id"))` — inner EXISTS resolves tenants against invoices (parent) |

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
| 228 | Reload with invalid config | `reloadMetadata()` with provider returning invalid apiName | `ConfigError: CONFIG_INVALID`, old config preserved — next query still works with previous metadata |

#### `packages/client/tests/client/` — HTTP client behavior

| # | Scenario | Input | Focus |
|---|---|---|---|
| 208 | Successful query via HTTP | POST /query with valid definition | response deserialized to `DataResult`, `kind: 'data'`, correct `meta` |
| 209 | SQL-only mode via HTTP | POST /query with `executeMode: 'sql-only'` | response deserialized to `SqlResult`, `kind: 'sql'`, `sql` + `params` present |
| 210 | Count mode via HTTP | POST /query with `executeMode: 'count'` | response deserialized to `CountResult`, `kind: 'count'`, `count` is number |
| 211 | ValidationError deserialization | POST /query with unknown table → server returns 400 | client throws `ValidationError` with correct `code`, `instanceof` check passes |
| 212 | ExecutionError deserialization | POST /query → server returns 500 with ExecutionError body | client throws `ExecutionError` with `code: 'QUERY_FAILED'`, `sql` + `params` preserved |
| 213 | ConnectionError on network failure | POST /query to unreachable server | client throws `ConnectionError` with `code: 'NETWORK_ERROR'` |
| 214 | Request timeout | POST /query to slow server, timeout: 100ms | client throws `ConnectionError` with `code: 'REQUEST_TIMEOUT'` |
| 215 | Custom headers | POST /query with `headers: { Authorization: 'Bearer tok' }` | request includes custom header (verifiable via MSW interceptor) |
| 216 | Local validation before send | `validateBeforeSend: true`, unknown table | `ValidationError` thrown without network request (no fetch call) |
| 217 | Health check via HTTP | GET /health → server returns HealthCheckResult | client returns typed `HealthCheckResult` with executor/cache status |
| 218 | Custom fetch injection | `fetch: mockFetch` in config | client uses injected fetch; verifiable by asserting on mock |
| 226 | PlannerError deserialization | POST /query → server returns 422 with PlannerError body | client throws `PlannerError` with `code: 'UNREACHABLE_TABLES'`, `instanceof` check passes |
| 239 | Validate query (valid) | POST /validate/query with valid definition | returns `{ valid: true }` |
| 240 | Validate query (invalid) | POST /validate/query with unknown table | throws `ValidationError` with `UNKNOWN_TABLE`, same `errors[]` format as /query |
| 241 | Validate config (valid) | POST /validate/config with valid metadata + roles | returns `{ valid: true }` |
| 242 | Validate config (invalid) | POST /validate/config with duplicate apiName | throws `ConfigError` with `DUPLICATE_API_NAME` |

#### `packages/contract-tests/tests/` — contract tests (wires suites to real executors)

These are the core smoke tests implemented inline. The **full contract test suite** (401 test IDs × 3 dialects for sections 3–9 ≈ 627 test executions) is documented in [`CONTRACT_TESTS.md`](./CONTRACT_TESTS.md) — a standalone spec for implementation developers building compatible servers in any language.

| # | Scenario | Input | Contract assertion |
|---|---|---|---|
| 219 | Simple select | orders columns: [id, status] | both return `kind: 'data'`, same column count, same `meta.columns` structure |
| 220 | Filter + join | orders JOIN products, filter status='active' | both return matching rows with same structure |
| 221 | Aggregation | orders GROUP BY status, SUM(total) | both return same aggregation results |
| 222 | Validation error | unknown table | both throw `ValidationError` with same `code` |
| 223 | Access denied | restricted column (tenant-user) | both throw `ValidationError` with `ACCESS_DENIED` |
| 224 | Count mode | orders (count) | both return `kind: 'count'` with same count |
| 225 | SQL-only mode | orders (sql-only) | both return `kind: 'sql'`; SQL string may differ in formatting but params match |
| 236 | Debug mode | orders debug: true | both return `debugLog` array with entries containing `timestamp`, `phase`, `message` |
| 237 | Masking in meta | orders columns: [id, total] (tenant-user, total masked) | both report `meta.columns[].masked = true` for total, `false` for id |
| 238 | byIds | orders byIds=[1,2] | both return `kind: 'data'` with columns present |

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
  { apiName: 'quantity',     physicalName: 'quantity',        type: 'int',       nullable: false },
  { apiName: 'isPaid',       physicalName: 'is_paid',         type: 'boolean',   nullable: true },
  { apiName: 'priorities',   physicalName: 'priorities',      type: 'int[]',     nullable: true },    // integer[] in Postgres
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
  { apiName: 'age',       physicalName: 'age',         type: 'int',       nullable: true },
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
  { apiName: 'dueDate',   physicalName: 'due_date',    type: 'date',      nullable: true },
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

