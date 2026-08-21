# The platform layer

An opinionated application framework hands a team several dozen capabilities as
defaults. An ecosystem without one hands them nothing. This document is the
list of those capabilities, converted into concrete prescriptions, plus
reference implementations for the three that carry most of the risk.

Almost every prescription elsewhere in this standard is a row in the table
below. That is why this document is the one to read first.

---

## 1 The ten modules

```
platform/
  config/        typed, validated at boot, fails fast; absence ≠ false
  context/       AsyncLocalStorage: correlationId · tenantId · userId · locale
                 installed at EVERY entrypoint (http · consumer · cron · scheduler)
  logging/       structured JSON to stdout; context injected automatically
  metrics/       RED · USE · saturation helpers; low-cardinality labels enforced
  tracing/       OpenTelemetry; propagation across HTTP and queue boundaries
  db/            connection + pool; transaction helper; TenantRepository base with
                 a REQUIRED branded tenant parameter; GlobalRepository for reference data
  outbox/        write(tx, event) · publisher · sweeper (queries the invariant)
  resilience/    timeout · retry(transient) · breaker · bulkhead, composable
  authz/         authenticate · tenant · permission · entitlement; declared per
                 operation and verified by a CI check
  health/        /health/live · /health/ready · /health/startup
```

Build them in that order; each depends only on those above it.

Make them the **only sanctioned way** to do the corresponding thing. A platform
module that can be bypassed is a suggestion.

---

## 2 The capability gap, in full

Twenty-six capabilities a mature application framework provides by default, and
what to build in place of each.

| # | Framework capability | What it provides by default | Without a framework | What to build |
|---|---|---|---|---|
| 1 | **Application context / dependency injection** | Central object graph, lifecycle scopes, trivially substitutable collaborators | Manual construction at every call site | Constructor injection with production defaults everywhere; a lightweight container only when wiring becomes the bottleneck |
| 2 | **Declarative transactions** | Transaction demarcation, propagation and rollback rules; no transaction object threaded through call chains | Hand-threaded transaction objects; forgetting one is silent | A `withTransaction()` helper that puts the transaction in ambient context so repositories pick it up automatically — §3.1 |
| 3 | **Method-level security** | Authorization declared on the method and enforced by the framework; a missing declaration is visible | Guard calls inside handler bodies; omission is invisible | Wrapper composition or schema directives, plus a CI check that every operation has a declared policy — §3.3 |
| 4 | **Query specifications** | Type-safe, composable queries; the repository interface speaks domain vocabulary | Repository interfaces re-export ORM types | Domain-shaped query objects; the ORM stays behind the repository |
| 5 | **Automatic tenant filtering** | **Framework-level tenant predicates applied to every query** | Per-call-site tenant predicates | Required branded tenant parameter plus database row-level security — §3.2 |
| 6 | **Health and management endpoints** | Liveness, readiness, metrics, info — for free | Nothing | The `health` module |
| 7 | **Metrics facade** | Vendor-neutral metrics; runtime, pool and HTTP metrics auto-instrumented | Nothing | `metrics`: RED, USE, saturation, with enforced low-cardinality labels |
| 8 | **Trace propagation** | Automatic propagation across HTTP, messaging and scheduling | Nothing | OpenTelemetry SDK with auto-instrumentation, wired in `tracing` |
| 9 | **Typed, validated configuration** | Typed, hierarchical, profile-aware configuration; startup fails on invalid values | Environment-variable string soup | Typed config module with schema validation and absence-is-not-falsity checks — §4 |
| 10 | **Profiles** | Environment-specific configuration with one code path | Ad-hoc environment conditionals, sometimes inside security code | One configuration schema; the environment supplies values only, never behaviour |
| 11 | **Declarative scheduling with distributed locking** | Scheduled methods that run once across replicas | In-process cron firing once per replica | External durable scheduler, or a distributed lease |
| 12 | **Declarative retry** | Retry with backoff and exception classification | Hand-rolled loops | The composable resilience pipeline |
| 13 | **Resilience primitives** | Breaker, bulkhead, rate limiter, time limiter, with metrics | Nothing standard | The resilience pipeline, with breaker and bulkhead actually wired |
| 14 | **Centralised exception mapping** | One place mapping exceptions to responses | Per-service error handlers | One shared error handler in the platform layer |
| 15 | **Declarative validation** | Validation on DTOs, enforced at the boundary | Explicit validator calls that can be omitted | Schema validation in the wrapper, so a handler cannot receive unvalidated input |
| 16 | **Security filter chain** | An ordered, declarative, auditable security pipeline | Hand-assembled middleware per entrypoint | One shared bootstrap defining the pipeline once |
| 17 | **Session management** | Externalised session state | Stateless tokens (a legitimate alternative) | A revocation denylist that fails closed |
| 18 | **Versioned migrations** | Migrations with checksums, baselines and an out-of-order policy | A CLI run by hand | Automated, locked, blocking migration step; a startup schema-version assertion |
| 19 | **Test slices** | Fast, focused integration slices against real infrastructure | Everything mocked, or everything end-to-end | A container-based integration harness in the platform layer |
| 20 | **Reproducible dependency resolution** | Managed, compatible version sets | Version ranges; a lockfile only if the build uses it | Copy the lockfile; install from it; a shared dependency-version package |
| 21 | **Graceful shutdown** | In-flight request draining out of the box | Hand-rolled, and dependent on the container entrypoint | Shutdown sequence plus a correct process-one signal handler |
| 22 | **Bounded async execution** | Bounded thread pools for background work | Unbounded concurrent promises | Bounded concurrency helpers; a worker entrypoint |
| 23 | **Enforced module boundaries** | Module boundaries **verified by a test** | Directory conventions | A dependency-graph lint rule enforced in CI |
| 24 | **Declarative caching** | Caching with a pluggable provider | Manual cache calls scattered through services | A cache decorator with a mandatory TTL parameter |
| 25 | **Curated starters** | Compatible dependency sets | Assemble and verify compatibility yourself | An internal starter package encapsulating the platform layer |
| 26 | **In-process domain events** | Events with transaction-bound listeners | Direct method calls, or premature use of the broker | A typed in-process event bus, with the outbox for anything crossing a process boundary |

---

## 3 The three highest-value items to build first

Of the twenty-six, three account for most of the risk.

### 3.1 Declarative transaction demarcation

```ts
// platform/db/transaction.ts
const txStore = new AsyncLocalStorage<Transaction>();

export async function withTransaction<T>(fn: () => Promise<T>): Promise<T> {
  const existing = txStore.getStore();
  if (existing) return fn();                    // propagation: join the caller's transaction
  return sequelize.transaction((t) => txStore.run(t, fn));
}

// The repository base picks the transaction up from ambient context.
// No call site ever threads a transaction object, so none can forget to.
protected get tx(): Transaction | undefined { return txStore.getStore(); }

// Usage — the transaction boundary is visible and the plumbing is invisible:
await withTransaction(async () => {
  await orders.create(tenantId, order);         // both enlist automatically
  await outbox.write({ type: 'order.created', payload });
});
```

Declarative transactions in twelve lines. It removes an entire class of "forgot
to pass the transaction" divergence — a defect that is invisible in review and
only manifests under a crash at exactly the wrong moment.

### 3.2 Framework-level tenant filtering

```ts
// platform/db/tenantContext.ts
export async function withTenant<T>(tenantId: TenantId, fn: () => Promise<T>): Promise<T> {
  return withTransaction(async () => {
    // RLS reads this; every query in the transaction is filtered by PostgreSQL
    await sequelize.query('SELECT set_config($1, $2, true)', {
      bind: ['app.tenant_id', String(tenantId)],
      transaction: txStore.getStore(),
    });
    return tenantStore.run(tenantId, fn);
  });
}
```

With this in place, a forgotten tenant predicate returns zero rows instead of
another tenant's data — the failure mode becomes a visible, testable bug rather
than a breach.

### 3.3 Declared, verified authorization

```ts
// platform/authz/declare.ts
export function authorized<A, R>(
  policy: Policy,
  handler: (args: A, ctx: AuthedContext) => Promise<R>,
) {
  POLICY_REGISTRY.set(policy.operation, policy);      // registered for CI verification
  return async (_p: unknown, args: A, raw: RawContext): Promise<R> => {
    const user   = await authenticate(raw);
    const tenant = await resolveTenant(user, raw);
    if (policy.permission) await requirePermission(tenant, policy.permission);
    if (policy.plan)       await requireEntitlement(tenant, policy.plan);
    return withTenant(tenant.id, () => handler(args, { user, tenant }));
  };
}

// CI test: the schema's operations and the policy registry must match exactly.
// A new operation without a declared policy fails the build.
it('every operation declares an authorization policy', () => {
  expect(schemaOperations().sort()).toEqual([...POLICY_REGISTRY.keys()].sort());
});
```

Note what this composes: authentication, tenant resolution, permission,
entitlement **and** the tenant-scoped transaction context — in one wrapper a
handler cannot bypass, because the handler's context type only exists on the
other side of it.

**Principle.** *Make the correct path the only path that type-checks. A
guarantee enforced by a type is a guarantee; a guarantee enforced by a code
review comment is a hope.*

---

## 4 Configuration: absence is not falsity

A required-value check implemented as a truthiness test rejects `false`, `0` and
`""` as "missing" — which means a flag intended to *disable* a feature cannot be
set, and the feature is permanently on. Security-relevant flags are the usual
casualty.

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

**Principle.** *Boolean configuration must distinguish absence from `false`.
Security-relevant flags default to the safe value and are explicitly enabled,
never explicitly disabled.*

---

## 5 What this ecosystem does better, and should keep

The comparison is not one-directional. These are real advantages and the
platform layer should not sacrifice them.

| Advantage | Why it matters |
|---|---|
| **One language across client and server** | Shared types, shared validation schemas, one hiring profile, no serialisation mismatch |
| **Generated types from the API schema** | A build-enforced client contract, genuinely harder to achieve across a language split |
| **Fast startup** | Containers start in hundreds of milliseconds; rolling deploys and scale-out are quick |
| **Low memory footprint per instance** | Cheaper horizontal scaling for I/O-bound work |
| **Excellent I/O concurrency** | Most SaaS work is I/O-bound, which is exactly the event loop's strength |
| **Structural typing and discriminated unions** | State machines and result types are more naturally expressed than with nominal typing |
| **No framework magic** | The cost is building the platform; the benefit is that behaviour is traceable to code you can read |

**The conclusion is not "use a different ecosystem."** It is: **the capabilities
a framework supplies as defaults must be built explicitly, and they must be
built first.** Build these twenty-six once, in one module, make them the only
sanctioned way to do the corresponding thing, and the principal disadvantage
disappears while every advantage above is retained.

---

## Principles

1. **The platform layer is the first feature**, not overhead to be deferred.
2. **Make it impossible to bypass.** A bypassable module is a suggestion.
3. **Make the correct path the only path that type-checks.**
4. **Install context at every entrypoint**, not only the HTTP one.
5. **Absence is not falsity.** Configuration must distinguish them.
6. **The environment supplies values, never behaviour.** One code path.
7. **Verify guarantees in CI.** A declared policy registry with no CI check
   verifying completeness is documentation.
8. **Enforce module boundaries with a test**, not a directory convention.
