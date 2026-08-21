# Phase 8 — Testing depth

**Output:** a risk-weighted test suite, a cross-tenant matrix, concurrency tests on every invariant, failure injection, a load test of the hot paths
**Writes code:** yes
**Depends on:** Phases 2–7

---

## What this phase is for

Explain it to the user like this:

> Tests have been going in the whole way through. What this phase adds is depth
> in the three places where a bug is not an inconvenience but a business
> problem: money, one customer seeing another customer's data, and two things
> happening at the same time. Those three are worth more testing than
> everything else combined, and I'm going to weight the effort accordingly
> rather than chasing a coverage percentage.

---

## The principle: weight by failure cost, not by coverage

A uniform coverage target spends the same effort on a settings page as on a
debit. Weight instead by what it costs when the code is wrong.

| Risk area | Cost of a defect | Coverage expectation |
|---|---|---|
| Money and ledger | Unrecoverable, auditable, legally exposed | Near-total, including every failure ordering |
| Tenant isolation | Data breach, contractual and regulatory | Generated matrix over every operation |
| Concurrency invariants | Silent corruption, discovered late | A test per invariant, under real concurrency |
| Authorization | Privilege escalation | Every operation, every layer |
| Outbox and queue handlers | Lost or duplicated work | Idempotency and crash-recovery per handler |
| State machines | Illegal transitions reaching production | Every transition, plus every rejected one |
| Everything else | Recoverable | Proportionate |

Load `references/33-testing.md`.

---

## Step 1 — Integration tests against real infrastructure

Run against a containerised real database, real cache, real queue.

Mocks cannot verify a unique constraint, a foreign key, a Row-Level Security
policy, an isolation-level behaviour, or a transaction rollback. Those are
exactly the mechanisms the correctness of this system rests on, so they must be
exercised for real.

## Step 2 — The cross-tenant matrix

Generate it rather than writing it by hand: for every operation in the API, a
principal from tenant A attempting to reach a resource in tenant B. Generation
matters because a hand-written matrix silently fails to cover the operation
added last week.

Assert 404, not 403.

## Step 3 — Object-level authorization within a tenant

The separate matrix from Phase 7: user A against user B's resources inside the
same tenant. Different bug class, different test.

## Step 4 — Concurrency tests on every invariant

For each invariant the system claims to hold, write a test that attacks it with
real parallelism:

- Two concurrent debits against a balance sufficient for one
- Two publishers claiming the same outbox row
- Two workers claiming the same job
- Duplicate webhook delivery
- Concurrent state transitions on the same entity
- Concurrent writes to the same unique constraint

A concurrency bug that is not tested is a concurrency bug that ships. These
races are not rare in production; they are the normal case at any real volume.

## Step 5 — Failure injection

- Kill the process mid-transaction; assert no partial state
- Kill a worker holding a lease; assert the sweeper recovers it
- Make a dependency time out; assert the timeout fires and the breaker opens
- Make the cache unreachable; assert security decisions fail closed and
  non-security paths degrade rather than fail
- Deliver a webhook twice, and out of order
- Fill a queue; assert backpressure rather than collapse

## Step 6 — Load test the hot paths

Only the hot paths, and only after there is something to measure. Establish:
where saturation begins, which resource saturates first, whether autoscaling
responds, and whether the connection budget from Phase 1 held.

---

## Gate

- [ ] Integration tests run against containerised real infrastructure in CI
- [ ] The cross-tenant matrix is generated, covers every operation, and passes
- [ ] The within-tenant object-level matrix passes
- [ ] Money paths have near-total coverage including every failure ordering
- [ ] Every claimed invariant has a concurrency test that genuinely runs in
      parallel
- [ ] Every outbox and queue handler has an idempotency test and a
      crash-recovery test
- [ ] Every state machine transition is tested, including illegal ones
- [ ] Failure injection covers every item in Step 5
- [ ] A load test exists for each hot path, with the saturation point recorded
- [ ] The whole suite runs in CI as a blocking gate
- [ ] Test run time is acceptable enough that people will not skip it

---

## References

- `references/33-testing.md`
- `references/20-concurrency.md` — the race shapes worth testing
- `references/16-multi-tenancy.md` — isolation at every layer
- `references/18-billing-and-money.md` — the money failure orderings
- `references/19-reliability.md` — failure injection targets
- `references/29-scalability.md` — what the load test should establish
