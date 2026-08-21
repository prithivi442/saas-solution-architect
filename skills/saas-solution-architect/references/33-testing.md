# Testing strategy

## 33.1 Test distribution

Testing effort left to accumulate on its own does not land where failure is
most expensive. It lands where the code was most interesting to write.

The distribution to expect, and to correct for:

| Where tests accumulate naturally | Where they are actually needed |
|---|---|
| The subsystem the team is proudest of — usually the most intricate infrastructure | The money path, which is often the least-tested code in the system |
| Identity and tenancy, because the consequences are obvious | Tenant isolation across *every* operation, which needs generating rather than writing |
| Front-end components, which are cheap to test | Concurrency invariants, which need real parallelism to exercise |
| The services with the most contributors | The services nobody has touched in six months |

**Two failure modes are worth naming, because both are common and both are
invisible on a coverage report.** The first: the highest-quality subsystem is
usually also the best-tested one — which is not a coincidence, and means
coverage tracks confidence rather than risk. The second, and far more
dangerous: **the component that computes what customers are charged can end up
with no automated tests at all**, because it is small, it looks arithmetical,
and it was written by whoever understood the billing rules. Weight by failure
cost instead — §33.2.

## 33.2 Reference flow: risk-weighted testing

Coverage percentage is the wrong target. **Weight testing by the cost of the failure.**

```
                    ┌─────────────────────┐
                    │   E2E · a handful   │   critical user journeys only
                    │  signup · pay · use │   slow, brittle, high value
                    ├─────────────────────┤
                    │  Integration · many │   ← the highest-value layer
                    │ real DB · real cache│     for a service-oriented system
                    │  real broker        │
                    ├─────────────────────┤
                    │    Unit · most      │   pure logic, calculations,
                    │  fast · deterministic│   state machines, validation
                    └─────────────────────┘
      +  Contract tests at every boundary
      +  Concurrency tests for every invariant
      +  Failure-injection tests for every dependency
```

Integration tests deserve the largest investment in a system like this, because most of the risk lives in the **interaction** between application code and infrastructure: transaction boundaries, tenant predicates, unique constraints, atomic claims, and idempotency. None of those can be verified with mocks — a mocked database cannot enforce a unique constraint, so the very mechanism you are relying on goes untested.

## 33.3 Mandatory test coverage by risk area

### Money — the highest bar in the system

```
□ Exact-decimal arithmetic: no floating-point drift across many entries
□ Balance projection equals full recomputation, over a large generated ledger
□ Concurrent charges against a limited balance: the balance never goes negative
□ Duplicate charge with the same idempotency key applies exactly once
□ Refund produces a compensating entry; the balance returns exactly
□ Currency mismatch is rejected, never coerced
□ Rounding at a defined boundary with the declared mode
□ Crash mid-transaction leaves no partial financial state
□ Reconciliation detects an injected divergence
□ Auto-recharge triggers once at the threshold, not once per event
```

### Authorization and tenancy

```
□ Every endpoint, unauthenticated → 401
□ Every endpoint, authenticated but wrong tenant → 404 (not 403)
□ Every endpoint, insufficient role → 403
□ Every endpoint, plan lacking the capability → 402
□ A revoked token is rejected
□ A token for tenant A cannot act on tenant B's resources — generated
  systematically across EVERY resource type, not sampled
□ A suspended tenant cannot perform write operations
□ RLS blocks a query that omits its tenant predicate
□ Every per-tenant unique index includes the tenant column
```

The cross-tenant matrix is worth generating rather than writing by hand:

```ts
// One test that covers every endpoint × every resource type.
// This is the highest-value test in a multi-tenant system.
describe.each(ALL_ENDPOINTS)('cross-tenant isolation: %s', (endpoint) => {
  it('returns 404 for a resource owned by another tenant', async () => {
    const a = await seedTenant(), b = await seedTenant();
    const resource = await seedResource(b);                 // owned by B
    const res = await call(endpoint, { as: a, id: resource.id });   // requested by A
    expect(res.status).toBe(404);
  });
});
```

### Webhooks and consumers

```
□ Invalid signature → rejected
□ Valid signature → accepted
□ Replayed event → 200, no duplicate effect
□ Out-of-order events → final state matches the provider
□ Malformed payload → dead-lettered, handler does not crash-loop
□ Handler crash mid-processing → retried, applied exactly once
□ Duplicate queue delivery → exactly one effect
□ Attempt exhaustion → terminal state, alert emitted
```

### Concurrency — the category that sequential tests cannot reach

```ts
// Every limit, transition, and charge gets a test of this shape.
it('never exceeds the quota under concurrency', async () => {
  const tenant = await seedTenant({ quota: 100, used: 99 });

  const results = await Promise.all(
    Array.from({ length: 20 }, () => attemptQuotaConsumingAction(tenant)),
);

  expect(results.filter(r => r.ok)).toHaveLength(1);   // exactly one succeeds
  expect(await usedQuota(tenant)).toBe(100);            // never 101+
});
```

### Failure injection

```
□ Database unavailable → clean failure, no corruption
□ Cache unavailable → security decisions fail closed; caches degrade
□ Broker unavailable → writes still commit; events recovered afterwards
□ Provider timeout → timeout honoured, breaker opens, no hung request
□ Provider 500 → retried; permanent errors not retried
□ Slow provider → does not exhaust the connection pool
```

## 33.4 Reference flow: integration tests against real infrastructure

```ts
// Testcontainers: real PostgreSQL, real Redis, per test run.
// This is what makes constraint- and transaction-level guarantees testable.
let pg: StartedPostgreSqlContainer;

beforeAll(async () => {
  pg = await new PostgreSqlContainer('postgres:16').start();
  process.env.DB_HOST = pg.getHost();
  process.env.DB_PORT = String(pg.getPort());
  await runMigrations();                    // migrations are themselves under test
});

beforeEach(() => truncateAllTables());      // isolation between tests

afterAll(() => pg.stop());
```

**Principle.** *Integration tests run against real infrastructure in containers. A mocked database cannot verify a unique constraint, a transaction rollback, an atomic claim, or a tenant predicate — which is precisely the set of mechanisms your correctness depends on.*

## 33.5 Recommended testing strategy

| Layer | Target | Enforcement |
|---|---|---|
| Money paths | Near-total branch coverage; concurrency and idempotency explicit | CI blocking; a coverage floor on those directories specifically |
| Authorization and tenancy | Generated matrix across all endpoints and resources | CI blocking |
| Webhooks and consumers | Duplicate, out-of-order, malformed, crash-recovery | CI blocking |
| Transactions | Rollback and partial-failure behaviour | Integration, CI blocking |
| Business logic | Unit tests for all branches | Coverage floor |
| API contract | Schema snapshot; breaking-change detection | CI blocking |
| Front end | Component and hook tests; E2E for critical journeys | CI blocking |
| Performance | Load test the hot paths before each major release | Scheduled |
| Failure modes | Dependency-removal drills | Scheduled, in staging |

## 33.6 Testing principles

1. **Tests that do not run in CI do not exist.**
2. **Weight coverage by failure cost, not by line count.**
3. **The money service has the highest bar in the system.**
4. **Integration tests use real infrastructure in containers.**
5. **Cross-tenant isolation is tested by a generated matrix, not by sampling.**
6. **Every invariant has a concurrency test.**
7. **Every consumer has a duplicate-delivery test.**
8. **Every dependency has a failure-injection test.**
9. **Migrations are tested — forward, and by being applied to a realistic dataset.**
10. **A test that fails intermittently is fixed or deleted within the week.** A tolerated flaky test trains the team to ignore red.
11. **New code paths in money, auth, or tenancy require tests to merge** — enforced by CI, not by convention.
