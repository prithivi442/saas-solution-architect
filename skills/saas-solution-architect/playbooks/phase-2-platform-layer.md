# Phase 2 — The platform layer

**Output:** ten working modules under `platform/`, a repository, a blocking CI gate
**Writes code:** yes
**Depends on:** Phase 1 decision record

---

## What this phase is for

This is the phase people skip, and skipping it is the single most expensive
decision in this standard.

Explain it to the user like this:

> Before I build any feature, I'm going to spend the first stretch building the
> foundation the features sit on — ten small modules that handle
> configuration, logging, database access, security checks and so on. It will
> look like nothing is happening from the outside. What it buys you is that
> every feature built afterwards is automatically correct in the ways that
> matter: it can't leak one customer's data to another, it can't lose an event,
> it can't run without its security check. If we build features first and the
> foundation later, every one of those guarantees becomes something a person
> has to remember on every change, forever.

The engineering statement of the same thing: consistency enforced by a
mechanism scales with headcount. Consistency maintained by memory does not.

---

## Build order

Repository and pipeline first, so nothing merges unverified.

### Step 1 — Repository and CI gate

- One repository, two entrypoints from one codebase: `api` and `worker`
- Module boundaries enforced by a lint rule, verified in CI — not by directory
  convention alone
- A CI gate that blocks merge on: typecheck, lint, unit tests, integration
  tests, dependency audit, secret scan
- Infrastructure-as-code skeleton
- Immutable artifact pipeline: build once, promote by digest

Load `references/32-cicd.md` and `references/31-infrastructure-and-deployment.md`.

**A test that does not run automatically is documentation.** Get the gate
working before writing anything it will guard.

### Step 2 — The ten modules

```
platform/
  config/        typed, validated at boot, fails fast; absence is not falsity
  context/       correlation id, tenant id, user id, locale — installed at
                 EVERY entrypoint: http, consumer, cron, scheduler callback
  logging/       structured JSON to stdout; context injected automatically
  metrics/       RED, USE and saturation helpers; low-cardinality labels enforced
  tracing/       propagation across HTTP and queue boundaries
  db/            pool, transaction helper, tenant-scoped repository base with a
                 REQUIRED tenant parameter, global repository for reference data
  outbox/        write(tx, event), publisher, sweeper that queries the invariant
  resilience/    timeout, retry, circuit breaker, bulkhead — composable
  authz/         authenticate, tenant, permission, entitlement — declared per
                 operation and verified by a CI check
  health/        liveness, readiness, startup
```

Build them in that order. Each one depends only on those above it.

Load `references/04-platform-layer.md` for the reference implementation of
each, and the twenty-six capabilities a mature application framework would
otherwise have supplied.

---

## The three that carry most of the risk

If time pressure forces a triage conversation, these three are the ones that
must not be reduced. Each removes an entire class of defect that is otherwise
invisible until it is a breach or a corrupted balance.

### 1. Declarative transaction demarcation

A transaction helper that puts the transaction in ambient context so
repositories pick it up automatically. No call site threads a transaction
object, so no call site can forget to.

Without it: someone eventually writes two statements that should have been one
transaction, and the bug appears only under a crash at exactly the wrong
moment.

### 2. Structural tenant filtering

A required, branded tenant parameter in the data layer, plus Row-Level Security
in the database as the backstop.

Without it: a forgotten tenant predicate returns **another customer's data.**
With it: a forgotten predicate returns zero rows — a visible, testable bug
instead of a data breach.

This is the difference between isolation as a mechanism and isolation as a
convention, and conventions degrade with every new call site.

### 3. Declared, verified authorization

One wrapper composing authentication, tenant resolution, permission,
entitlement and the tenant-scoped transaction context — plus a CI test
asserting that every operation in the API has a declared policy.

Without it: a new endpoint ships with no authorization check and nothing in a
normal review surfaces it, because the absence of a call is invisible.

The handler must be *unable* to bypass it: give the handler a context type that
only exists on the other side of the wrapper.

---

## Rules that apply while building

- **Configuration:** absence and `false` are different questions. A required
  check written as a truthiness test rejects `false`, `0` and `""` as missing,
  which means a flag intended to *disable* a feature cannot be set and the
  feature is permanently on. Security-relevant flags default to the safe value
  and are explicitly enabled, never explicitly disabled.
- **Context:** install at every entrypoint, not just HTTP. A background worker
  without correlation context is a worker whose failures cannot be traced to a
  cause.
- **Metrics:** enforce low-cardinality labels at the helper. A tenant id or a
  user id in a metric label is an outage in a metrics backend.
- **Health:** three endpoints, not one. Liveness restarts, readiness routes,
  startup gates. Collapsing them causes the platform to kill a container that
  is merely still starting.
- **Environment supplies values, never behaviour.** One code path, one
  configuration schema. A conditional on environment inside security code is
  how staging-only validation reaches production.

---

## Gate

Verification is the deliverable here, not the code. Run each and show output.

- [ ] `api` and `worker` both boot from one codebase
- [ ] Boot fails loudly on a missing required configuration value — demonstrate it
- [ ] A boolean config flag can be set to `false` and is honoured — demonstrate it
- [ ] Every log line carries a correlation id, from ambient context
- [ ] Liveness, readiness and startup endpoints respond correctly
- [ ] A repository call without a tenant parameter **does not compile**
- [ ] Row-Level Security is enabled and a cross-tenant read returns zero rows —
      demonstrate with a test
- [ ] An operation without a declared authorization policy **fails CI** —
      demonstrate by adding one
- [ ] The transaction helper joins an existing transaction rather than nesting
- [ ] An outbox write and its state change commit atomically — demonstrate with
      a crash-injection test
- [ ] Module boundary violations fail the lint rule in CI
- [ ] The CI gate blocks a merge on a failing test — demonstrate it
- [ ] An artifact is built once and promoted by digest

Do not report this phase complete because the modules exist. Report it complete
when the demonstrations above have run and you can show the output.

---

## References

- `references/04-platform-layer.md` — the ten modules and the parity checklist
- `references/05-data-architecture.md` — pool and connection governance
- `references/06-transactions-and-integrity.md` — transaction boundaries
- `references/08-transactional-outbox.md` — outbox design
- `references/16-multi-tenancy.md` — structural scoping and RLS
- `references/15-auth-and-authorization.md` — the four layers
- `references/17-integrations-and-webhooks.md` — the resilience pipeline
- `references/21-observability.md` — logs, metrics, tracing
- `references/19-reliability.md` — health endpoints, startup ordering
- `references/30-code-architecture.md` — layering and dependency injection
- `references/32-cicd.md` — the blocking pipeline
- `references/31-infrastructure-and-deployment.md` — artifacts and IaC
