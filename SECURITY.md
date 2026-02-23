# SQL Injection Prevention — Three-Layer Defense

The query engine prevents SQL injection through three independent defense layers.  Each layer is sufficient on its own; together they provide defense-in-depth.

---

## Layer 1 — Metadata Validation (before SQL generation)

Every **identifier** (table name, column name, join target) and **enum-like keyword** (operator, direction, aggregation function, filter group logic) is validated against registered metadata or a hard-coded whitelist **before** any SQL is generated.

| Input | Check |
|---|---|
| `from`, `joins[].table`, `filters[].table` | Must exist in `MetadataIndex` → `UNKNOWN_TABLE` |
| `columns[]`, `filters[].column`, `orderBy[].column`, `groupBy[].column`, `aggregations[].column` | Must exist on resolved table → `UNKNOWN_COLUMN` |
| `filters[].operator` | Must be one of 31 known operators → `INVALID_FILTER` |
| `orderBy[].direction` | Must be `'asc'` or `'desc'` → `INVALID_ORDER_BY` |
| `aggregations[].fn` | Must be `count`, `sum`, `avg`, `min`, `max` → `INVALID_AGGREGATION` |
| `having[].operator`, `filters[].logic`, `having[].logic` | Validated against allowed sets → `INVALID_HAVING` / `INVALID_FILTER` |
| `filters[].count.operator` (EXISTS subquery) | Must be one of `=`, `!=`, `>`, `<`, `>=`, `<=` → `INVALID_FILTER` |

This layer is **always** applied and runs with **zero I/O** — no database connection is needed.  Injected identifiers like `'id"; DROP TABLE orders; --'` fail with `UNKNOWN_COLUMN` because they don't match any registered table/column name.

TypeScript types enforce valid values at compile time, but the validation layer **also** checks at runtime — raw JSON payloads (e.g. from HTTP clients) bypass TypeScript constraints.

### Fragment-level helpers

Even after validation, identifiers and function names pass through additional runtime guards:

| Helper | Purpose |
|---|---|
| `safeAggFn(fn)` | Whitelist check — returns `'COUNT'` if `fn` is not in `{count, sum, avg, min, max}` |
| `safeWhereFn(fn)` | Whitelist check — throws if `fn` is not in `{levenshtein, levenshtein_distance, editdistance}` |

---

## Layer 2 — Parameterized Queries (values never touch SQL text)

User-provided **values** (filter values, `byIds`, `between` bounds, `levenshteinLte` text/threshold, array elements) are **never** interpolated into the SQL string.  Each dialect uses its native parameterization mechanism:

| Dialect | Mechanism | Example |
|---|---|---|
| **PostgreSQL** | Positional parameters `$1`, `$2`, … | `WHERE "status" = $1` with `params: ['active']` |
| **ClickHouse** | Typed parameters `{p1:String}`, `{p2:Int32}`, … | `WHERE "type" = {p1:String}` with `params: { p1: 'active' }` |
| **Trino** | Inline escaping via `escapeTrinoValue()` | `WHERE "type" = 'active'` — `'` doubled to `''` |

PostgreSQL and ClickHouse drivers handle parameterization natively — the SQL string and parameter values are sent separately to the database, making injection **structurally impossible** regardless of value content.

Trino does not support server-side parameterization through `trino-client`, so values are escaped inline by `escapeTrinoValue()`:

```typescript
function escapeTrinoValue(value: unknown): string {
  if (value === null || value === undefined) return 'NULL'
  if (typeof value === 'number') return String(value)
  if (typeof value === 'boolean') return value ? 'TRUE' : 'FALSE'
  if (typeof value === 'string') return `'${value.replace(/'/g, "''")}'`
  if (Array.isArray(value)) {
    return `ARRAY[${value.map((v) => escapeTrinoValue(v)).join(', ')}]`
  }
  throw new Error(`Unsupported Trino parameter type: ${typeof value}`)
}
```

Trino's default SQL mode does **not** use C-style backslash escapes — single-quote doubling (`'` → `''`) is the only required escaping.

---

## Layer 3 — Identifier Escaping (defense-in-depth for quoted identifiers)

Even though identifiers are validated against metadata (Layer 1), the SQL generator **also** escapes quoting characters inside identifiers — this is defense-in-depth in case a future code path bypasses validation.

| Helper | Dialect | Escaping |
|---|---|---|
| `escapeIdentDQ(value)` | PostgreSQL, Trino | `"` → `""` (double-quote doubling) |
| `escapeIdentBT(value)` | ClickHouse | `` ` `` → ` `` `` ` (backtick doubling) |

Aggregation aliases are user-provided strings that pass through these escaping functions before interpolation into SQL.

### LIKE pattern escaping

Pattern-matching operators (`contains`, `icontains`, `like`, `startsWith`, `endsWith` and their negations) call `escapeLike()` on the value before parameterization:

```typescript
function escapeLike(value: string): string {
  return value.replace(/[%_\\]/g, '\\$&')
}
```

This prevents LIKE wildcard characters (`%`, `_`) and backslash in user input from being interpreted as pattern metacharacters.

---

## Comparison with `samples-generation/escape.ts`

The `samples-generation` project uses a different approach — it escapes **literals** before interpolation because it generates INSERT/INSERT-SELECT statements with inline values:

| samples-generation | multi-db-query | Why different |
|---|---|---|
| `pg-format.ident()` / `pg-format.literal()` | `escapeIdentDQ()` + `$N` parameterization | multi-db uses native driver parameterization — no literal escaping needed |
| `escapeClickHouseLiteral()` — `\` → `\\`, `'` → `\'` | `{pN:Type}` parameterization | CH driver sends params separately — backslash escaping not needed |
| `escapeTrinoLiteral()` — `'` → `''` | `escapeTrinoValue()` — `'` → `''` | Same escaping logic; Trino doesn't use backslash escapes |

**Key difference:** `samples-generation` needs literal escaping because it generates INSERT statements with inline data.  `multi-db-query` uses parameterized queries for PG/CH, making the driver responsible for escaping.  Only Trino requires manual escaping (quote doubling), which matches what `samples-generation` does.

---

## Contract Test Coverage

The injection contract test suite (`describeInjectionContract`) verifies all three layers with 60+ test cases across 6 categories:

| Section | Category | Tests | Defense layer |
|---|---|---|---|
| **16.1** | Identifier & structural injection | 20 tests | Layer 1 — `expectValidationError` with specific code |
| **16.2** | Aggregation alias injection | 10 tests | Layer 3 + Layer 1 — `expectInjectionSafe` |
| **16.3** | PostgreSQL filter value injection | 15 tests | Layer 2 — `expectInjectionSafe` |
| **16.4** | ClickHouse filter value injection | 15 tests | Layer 2 — `expectInjectionSafe` |
| **16.5** | Trino filter value injection | 15 tests | Layer 2 — `expectInjectionSafe` |
| **16.6** | Advanced vectors (all dialects) | 20 tests | Layers 2 + 3 — `expectInjectionSafe` |

### Test assertion philosophy

The `expectInjectionSafe(engine, definition, expected?)` helper takes an explicit `expected` parameter that declares the intended outcome:

| `expected` value | Assertion | Typical use |
|---|---|---|
| `'escaped'` (default) | Query **must succeed** — the value was parameterized/escaped and treated as literal data | Filter value injection (16.3–16.6), alias injection (16.2) |
| `'rejected'` | Query **must throw** `ValidationError` or `ExecutionError` — the payload was caught by validation or the DB itself rejected it (e.g. type mismatch) | Tests where the injection payload also triggers a structural or type error |

Making the expected outcome explicit ensures that **behavior regressions are caught immediately**.  If a test that should succeed starts throwing (or vice versa), the test fails — rather than silently accepting either outcome.

Where a **specific** validation code is expected (e.g. identifier injection must always be caught by metadata validation), the stricter `expectValidationError(engine, definition, code)` is used instead.

### Attack vectors tested

| Vector | Payload example | Tested in |
|---|---|---|
| Single-quote breakout | `'; DROP TABLE orders; --` | 16.3, 16.4, 16.5 |
| Double-quote identifier breakout | `id"; DROP TABLE orders; --` | 16.1, 16.2 |
| Backtick identifier breakout | `` id`; DROP TABLE events; -- `` | 16.1, 16.2 |
| Stacked queries via `;` | `orders; DROP TABLE orders` | 16.1 |
| UNION injection | `) UNION SELECT 1;--` | 16.1 |
| OR 1=1 tautology | `) OR 1=1 --` | 16.1 |
| Comment-based bypass | `x' /**/; DROP TABLE orders; --` | 16.6 |
| Backslash-quote bypass | `\'; DROP TABLE orders; --` | 16.6 |
| Null byte truncation | `\0'; DROP TABLE orders; --` | 16.6 |
| Unicode apostrophe (U+02BC) | `ʼ; DROP TABLE orders; --` | 16.6 |
| Nested/odd quote count | `'''; DROP TABLE orders; --` | 16.6 |
| Newline payload splitting | `x'\n; DROP TABLE orders\n--` | 16.6 |
| Triple double-quote bomb | `"""; DROP TABLE orders;--` | 16.6 |
| Triple backtick bomb | ` ``` ; DROP TABLE events;-- ` | 16.6 |
