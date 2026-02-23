← [Back to README](./README.md)

# Cross-Database Behavior Normalization

## Problem

The library produces different results for identical queries depending on the
target database engine. Three categories of divergence existed in the contract
tests. Two have been fixed; one is a storage-layer constraint that cannot be
resolved at the SQL level.

---

## 1. Counted EXISTS operators (FIXED)

### 1a. `<` / `<=` operators — C611, C613

**Symptom:** `{ table: sampleItems, count: { operator: '<', value: 2 } }` returned
3 rows on PG but 2 on CH/Trino.

**Root cause:** PG uses a correlated subquery
(`WHERE (SELECT COUNT(*) …) < 2`) which naturally includes parent rows with
zero matching children (COUNT = 0). CH/Trino decorrelate the subquery into a
JOIN, so parents with zero children are never produced.

**Fix:** `countedNotIn()` — rewrites to
`parent.id NOT IN (SELECT fk FROM child GROUP BY fk HAVING COUNT(*) >= N)`.
Inverts the operator, avoids correlated subqueries, includes 0-match parents.

### 1b. `>=` / `>` operators — C605, C607, C610

**Symptom:** CH/Trino threw "Cannot decorrelate query, because 'Limit' step is
not supported" because the `countLimit` optimization wrapped the correlated
subquery in `LIMIT N`.

**Fix:** `countedIn()` — rewrites to
`parent.id IN (SELECT fk FROM child GROUP BY fk HAVING COUNT(*) >= N)`.
No correlated subquery, no LIMIT. Parents with zero children are correctly
excluded (they don't appear in the child table at all). Removed the
now-dead `countLimit()` method from both CH and Trino dialects.

**Outcome:** All `variant === 'pg'` branches and try/catch blocks removed
from C605, C607, C610, C611, C613. All engines produce identical results.
Unit tests added for `<`, `<=`, `>=`, `>` in all three dialect test files.

---

## 2. Timestamp `BETWEEN` on ClickHouse (FIXED)

**Test:** C133

**Symptom:** `BETWEEN` on a `DateTime64` column with ISO 8601 string params
(`'2024-01-01T00:00:00Z'`) failed with "cannot be parsed as DateTime…only
19 of 20 bytes was parsed" — CH's `DateTime` type rejects the trailing `Z`.

**Fix:** Added `columnType` to `WhereBetween` IR. ClickHouse dialect now
wraps timestamp params in `parseDateTimeBestEffort({p:String})` which handles
any reasonable datetime format including ISO 8601 with timezone suffixes.
Also added typed params (`chColumnTypeMap`) for non-timestamp column types
in `BETWEEN` (e.g. `{p:Date}` for date columns).

Trino dialect uses `CAST(? AS TIMESTAMP)` for timestamp columns in `BETWEEN`
(not currently triggered since the Trino test variant routes through the CH
executor, but correct for future standalone Trino usage).

**Outcome:** C133 try/catch removed; all engines return 3.

---

## 3. NULL arrays on ClickHouse (DOCUMENT, DO NOT FIX)

**Tests:** C152, C153, C173

**Symptom:** `isNull` on an array column returns 1 on PG, 0 on CH.
`arrayIsEmpty` returns 1 on PG, 2 on CH.

**Root cause:** ClickHouse `Array(T)` columns physically cannot store NULL.
A NULL inserted into an array column is silently converted to `[]` at write
time. No SQL rewrite can recover a value that was never persisted.

**Why not fix:** Rewriting `isNull(arr)` → `length(arr) = 0` on CH would
conflate NULL and empty, violating the principle of least surprise for users
who intentionally distinguish them on PG.

**Actions:**
- Keep `variant === 'pg'` branches in C152, C153, C173.
- Document the caveat in library docs / README under an "Engine Differences"
  section.
- Optionally: emit a validation warning when `isNull`/`isNotNull` targets an
  array column on a CH-backed table.
