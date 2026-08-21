# API architecture

## 14.1 The reference implementation

| Concern | Implementation |
|---|---|
| Primary API | Federated GraphQL — one graph, many subgraphs |
| Secondary API | REST for webhooks, uploads, scheduler callbacks, admin |
| Realtime | GraphQL subscriptions over WebSocket |
| Internal calls | Via the federation router; some direct HTTP between services |
| Versioning | Field-level evolution; `V2` suffixes on some operations |
| DTOs | Explicit DTO layer, separate from ORM models |
| Validation | Joi schemas invoked per operation |
| Pagination | **Cursor-based (Relay-style connections)** with a shared pagination package |
| Filtering / sorting | Per-operation arguments |
| Error responses | GraphQL extensions carrying `code`, `argumentName`, and a localised message |
| Localisation | Errors localised server-side from `Accept-Language` |
| Correlation ID | Generated per request |
| Rate limiting | On specific paths; not a global API concern |
| `Idempotency-Key` support | **Required — commonly missing** |
| Response caching directives | Cache-control plugin present with a 1-second default |

## 14.2 The choices that matter, and why

**Cursor pagination as the default — the right default, without qualification.** Offset pagination has two failure modes that appear exactly when a product succeeds: `OFFSET 50000` forces the database to scan and discard 50,000 rows, and concurrent inserts cause rows to be skipped or repeated across pages. Cursor pagination has neither problem. A shared pagination package, rather than each endpoint improvising, is what makes it applied consistently.

**Principle.** *Cursor pagination is the default for every list endpoint from day one. Retrofitting it is a breaking API change; adopting it initially costs nothing.*

**Explicit DTOs — required.** A separate DTO layer prevents ORM models from becoming the API contract. Without it, adding a database column silently adds an API field, and renaming one silently breaks clients.

**Principle.** *The API contract is defined by types you write deliberately, never by the shape of a database row.*

**Server-side localised errors — required.** Returning a stable machine-readable `code` alongside a localised human message lets clients branch on the code and display the message. Both audiences are served by one response.

**Principle.** *Every error carries a stable machine code, a human-readable message, and a correlation ID. Clients branch on the code, humans read the message, support engineers search the correlation ID.*

## 14.3 Reference: the SaaS API standard

### Request contract

| Element | Requirement |
|---|---|
| **Authentication** | `Authorization: Bearer <token>` — uniformly required, identically validated in every environment |
| **Tenant context** | An explicit tenant header or a tenant claim inside the token — never inferred from the request body |
| **Correlation** | Accept an inbound `X-Request-Id`; generate one when absent; return it on every response; propagate it to every downstream call and log line |
| **Idempotency** | Accept `Idempotency-Key` on **every** non-idempotent mutation (see 14.4) |
| **Content type** | `application/json`, except deliberate multipart upload endpoints |
| **Localisation** | `Accept-Language`, applied to all human-readable strings |
| **Schema version** | Explicit on asynchronous message payloads; field-level evolution for the synchronous graph |

### Error contract

```jsonc
{
  "error": {
    "code": "QUOTA_EXCEEDED",              // stable, documented, never localised
    "message": "Monthly limit reached.",   // localised, safe to display
    "field": "recipients",                 // present for validation errors
    "requestId": "req_2f8a…",              // always present; matches the log line
    "retryable": false,                    // tells the client whether to retry
    "retryAfterMs": null                   // present when retryable and known
  }
}
```

`retryable` is the field most often omitted and the most useful: without it, every client hard-codes its own guess about which of your errors are worth retrying, and the guesses are wrong in both directions.

### Standard status and code mapping

| Situation | HTTP | GraphQL extension code | Retryable |
|---|---|---|---|
| Malformed input | 400 | `BAD_USER_INPUT` | No |
| Missing or invalid credentials | 401 | `UNAUTHENTICATED` | No |
| Authenticated, not permitted | 403 | `FORBIDDEN` | No |
| Plan does not include the capability | 402 / 403 | `PAYMENT_REQUIRED` / `PLAN_UPGRADE_REQUIRED` | No |
| Not found, or not visible to this tenant | 404 | `NOT_FOUND` | No |
| Concurrent modification conflict | 409 | `CONFLICT` | Yes, after reload |
| Quota or rate limit exceeded | 429 | `RATE_LIMITED` | Yes, after `Retry-After` |
| Internal failure | 500 | `INTERNAL` | Yes, with backoff |
| Dependency unavailable | 503 | `DEPENDENCY_UNAVAILABLE` | Yes, with backoff |

**Note on 404 vs 403 for cross-tenant access:** return **404**, not 403. A 403 confirms the resource exists, which leaks information across the tenant boundary. Absence and inaccessibility must be indistinguishable to a caller.

## 14.4 Reference flow: idempotency on mutations

Any mutation that creates a resource, moves money, or triggers a costed external effect needs client-supplied idempotency. Without it, a client retry after a network timeout produces a duplicate — and a network timeout is the single most common failure a client experiences.

```
Client                                Server
  │  POST /orders                        │
  │  Idempotency-Key: 7f3c-…             │
  │─────────────────────────────────────▶│
  │                                      │  BEGIN
  │                                      │    INSERT idempotency_keys (key, endpoint,
  │                                      │      request_hash, tenant_id, status='in_progress')
  │                                      │      ── UNIQUE(key, endpoint, tenant_id)
  │                                      │    ↳ conflict?
  │                                      │        status='completed'   → return stored response
  │                                      │        status='in_progress' → 409, retry later
  │                                      │        request_hash differs → 422, key reuse
  │                                      │    … perform the mutation …
  │                                      │    UPDATE idempotency_keys
  │                                      │      SET status='completed', response=…
  │                                      │  COMMIT
  │◀─────────────────────────────────────│
```

Four details that make this correct rather than approximately correct:

1. **The key row is inserted in the same transaction as the effect.** Otherwise a crash between them leaves a key claiming an effect that never happened.
2. **The request body is hashed and compared.** The same key with a different body is a client error (`422`), not a cache hit — this catches key-reuse bugs that would otherwise silently return the wrong response.
3. **The stored response is replayed verbatim**, including the status code, so a retry is indistinguishable from the original.
4. **Keys are scoped by tenant and endpoint** and expire (24–48 hours is typical). An unscoped key namespace is a cross-tenant information channel.

**Principle.** *Idempotency is a property of the API contract, not an implementation detail. Publish which endpoints honour `Idempotency-Key`, and honour it on every endpoint whose repetition would cost a customer money.*

## 14.5 GraphQL-specific requirements

GraphQL removes some problems and introduces others. Four controls are mandatory in production and each addresses a specific attack:

| Control | Without it |
|---|---|
| **Query depth limit** | A recursive nested query is an amplification attack from a single request |
| **Query complexity budget** | A syntactically small query can request an unbounded result set |
| **Introspection disabled in production**, driven by a boolean that genuinely accepts `false` | The full schema — including admin operations and internal field names — is published to anyone |
| **Persisted queries / operation allow-list** | Any client can execute any operation the schema permits, including combinations never intended |
| **Per-field authorization** | A nested field can return another tenant's data even when the root field was authorized |
| **Batching and alias limits** | 1,000 aliased copies of one field in one request bypasses per-request rate limiting entirely |

A configuration note worth generalising: a boolean feature flag must be readable as `false`. A required-value check implemented as a truthiness test (`if (!value) fail`) rejects `false`, `0`, and `""` as "missing" — which means a flag intended to *disable* a feature cannot be set, and the feature is permanently on.

```ts
// Reference: presence and value are separate questions
function required(name: string): string {
  const v = process.env[name];
  if (v === undefined || v === '') {          // absence, not falsiness
    logger.fatal(`Missing required configuration: ${name}`);
    process.exit(1);
  }
  return v;
}
const bool = (name: string, dflt: boolean) => {
  const v = process.env[name];
  return v === undefined ? dflt : v === 'true';   // 'false' is a valid, honoured value
};

export const introspectionEnabled = bool('GRAPHQL_INTROSPECTION', false);  // off by default
```

**Principle.** *Boolean configuration must distinguish absence from `false`. Security-relevant flags default to the safe value and are explicitly enabled, never explicitly disabled.*

## 14.6 API principles

1. **One primary API style.** Add a second only for what the first genuinely cannot do — webhooks, binary streaming, and scheduler callbacks are legitimate; convenience is not.
2. **Cursor pagination everywhere, from day one.**
3. **Explicit DTOs.** Never return an ORM model.
4. **`Idempotency-Key` on every non-idempotent mutation.**
5. **Every response carries a correlation ID.**
6. **Errors carry a stable code, a localised message, and a retryable flag.**
7. **Cross-tenant access returns 404**, never 403.
8. **Validate at the boundary**, once, with a schema — and reject unknown fields rather than ignoring them.
9. **Rate-limit by tenant *and* by principal**, not by IP alone. NAT makes IP limits punish innocent tenants and shared credentials make them useless.
10. **Version by evolution** (add fields, deprecate with a timeline, remove on a published schedule) rather than by URL proliferation. Suffixing operations with `V2` works but accumulates: track deprecations with a sunset date and a metric counting remaining callers.
11. **Depth, complexity, and batching limits are production requirements**, not optimisations.
12. **Publish the contract.** A schema that is only discoverable by introspection is not a contract.
