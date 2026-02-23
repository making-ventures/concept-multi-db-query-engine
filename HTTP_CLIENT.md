← [Back to README](./README.md)

## HTTP Client & Contract Testing

### HTTP API Contract

The system defines a minimal HTTP API contract that any server wrapping `@mkven/multi-db-query` must implement:

| Endpoint | Method | Request Body | Response Body |
|---|---|---|---|
| `/query` | POST | `{ definition, context }` | `QueryResult` (JSON, discriminated by `kind`) |
| `/health` | GET | — | `HealthCheckResult` |
| `/validate/query` | POST | `{ definition, context }` | `{ valid: true }` |
| `/validate/config` | POST | `{ metadata, roles }` | `{ valid: true }` |

`/validate/query` runs all 14 validation rules (table/column existence, role permissions, filter/join/aggregation validity, etc.) **without executing** — no DB connections, no executors needed. Returns `{ valid: true }` on success, or throws `ValidationError` (400) with the same `errors[]` array as `/query`. This lets contract tests verify all validation behavior without a running database.

`/validate/config` validates metadata + role configuration (apiName format, uniqueness, references, relations, syncs, caches). Returns `{ valid: true }` on success, or throws `ConfigError` (400). Useful for CI pipelines and config validation tooling.

Error responses use HTTP status codes with the error's `toJSON()` body:

| Error Type | HTTP Status |
|---|---|
| `ValidationError` | 400 |
| `ConfigError` | 400 |
| `PlannerError` | 422 |
| `ExecutionError` | 500 |
| `ConnectionError` | 503 |
| `ProviderError` | 503 |

The contract is intentionally minimal — no auth, no routing framework, no middleware. The server wraps `multiDb.query()` and `multiDb.healthCheck()` and maps errors to status codes. Any HTTP framework (Express, Fastify, Hono, etc.) can implement it.

### MultiDbClient

```ts
import { createMultiDbClient } from '@mkven/multi-db-client'

const client = createMultiDbClient({ baseUrl: 'http://localhost:3000' })

const result = await client.query({
  definition: { from: 'orders', columns: ['id', 'total', 'status'] },
  context: { roles: { user: ['admin'] } }
})
// result is QueryResult<T> — same type as multiDb.query()
```

```ts
interface MultiDbClient {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>

  healthCheck(): Promise<HealthCheckResult>

  validateQuery(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<{ valid: true }>          // throws ValidationError on failure

  validateConfig(input: {
    metadata: MetadataConfig
    roles: RoleMeta[]
  }): Promise<{ valid: true }>            // throws ConfigError on failure
}

interface MultiDbClientConfig {
  baseUrl: string                       // server URL (no trailing slash)
  headers?: Record<string, string>      // e.g. { Authorization: 'Bearer ...' }
  fetch?: typeof globalThis.fetch       // injectable for testing (MSW, custom interceptors)
  timeout?: number                      // request timeout in ms (default: 30_000)
  validateBeforeSend?: boolean          // pre-validate query locally (default: false)
  metadata?: MetadataConfig             // required when validateBeforeSend is true
  roles?: RoleMeta[]                    // required when validateBeforeSend is true
}
```

Key behaviors:
- **Error deserialization** — server returns `toJSON()` body; client reconstructs typed error classes (`ValidationError`, `ExecutionError`, etc.) using the `code` field. Callers use `instanceof` checks as usual — no transport-awareness needed
- **Optional local validation** — when `validateBeforeSend: true`, calls `validateQuery()` from the validation package before sending. Fails fast with `ValidationError` without a network round trip. Requires metadata + roles to be provided in config
- **Custom fetch** — injectable `fetch` function for testing. Default: `globalThis.fetch`. Enables MSW-based tests without a running server
- **Timeout** — wraps the underlying `fetch` with `AbortController`; throws `ConnectionError` with code `REQUEST_TIMEOUT` on expiry
- **No retry logic** — the client is intentionally simple. Callers handle retries at a higher level (middleware, wrapper, etc.)

### Contract Testing

Both `MultiDb` (in-process) and `MultiDbClient` (HTTP) implement the same query surface. The `@mkven/multi-db-contract` package exports **contract test suites** that run against both:

```ts
// Shared contract — both implementations satisfy this
interface QueryContract {
  query<T = unknown>(input: {
    definition: QueryDefinition
    context: ExecutionContext
  }): Promise<QueryResult<T>>
}
```

Contract tests are parameterized by a factory function:

```ts
// packages/contract/src/queryContract.ts
export function describeQueryContract(name: string, factory: () => Promise<QueryContract>) {
  describe(`QueryContract: ${name}`, () => {
    let engine: QueryContract

    beforeAll(async () => { engine = await factory() })

    it('simple select', async () => {
      const result = await engine.query({
        definition: { from: 'orders', columns: ['id', 'status'] },
        context: { roles: { user: ['admin'] } }
      })
      expect(result.kind).toBe('data')
      expect(result.meta.columns).toHaveLength(2)
    })

    it('validation error', async () => {
      await expect(engine.query({
        definition: { from: 'nonexistent' },
        context: { roles: { user: ['admin'] } }
      })).rejects.toThrow(ValidationError)
    })

    // ... all shared scenarios
  })
}
```

Each implementation provides a factory:

```ts
// Direct (in-process)
describeQueryContract('direct', async () => {
  const multiDb = await createMultiDb({ ... })
  return multiDb
})

// HTTP client
describeQueryContract('http-client', async () => {
  const client = createMultiDbClient({ baseUrl: 'http://localhost:3000' })
  return client
})
```

This ensures both implementations behave identically — same results, same errors, same metadata structure. Catches serialization drift (e.g. `Date` vs ISO string), error mapping mismatches, and behavioral divergence before they reach production.

The private `@mkven/multi-db-contract-tests` package wires these suites to the real executors for integration testing.

