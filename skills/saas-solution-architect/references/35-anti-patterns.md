# SaaS architecture anti-patterns

For each: the problem, why it is dangerous, and the better approach.

| # | Anti-pattern | Why it is dangerous | Better approach |
|---|---|---|---|
| 1 | **Dual writes** — writing to a database and a queue/API without an atomic record | The second write fails and the systems diverge silently, forever | Transactional outbox: one commit, then publish, then sweep |
| 2 | **Missing transactions** — multi-table writes via `Promise.all` | A partial failure leaves inconsistent state that no error reports | One transaction per consistency boundary |
| 3 | **I/O inside a transaction** | Holds a connection and locks across a network call; a slow third party becomes a database outage | Commit the intent; perform I/O afterwards from a durable record |
| 4 | **Unsafe retries** — retrying non-idempotent operations | Duplicate charges, duplicate provisioning, duplicate notifications | Idempotency key with a unique constraint, before any retry |
| 5 | **Retry without a circuit breaker** | Retries multiply load on a failing dependency; degradation becomes outage | Breaker outside retry, timeout inside |
| 6 | **Missing timeouts** | A hung call holds a connection, worker, and pool slot indefinitely | Explicit timeout on every external call, from the latency budget |
| 7 | **Check-then-act** — read a limit, decide, write | Concurrent callers both pass; the limit is advisory | The precondition is the predicate of the write |
| 8 | **Read-check-write state transitions** | Concurrent transitions both succeed | Conditional `UPDATE … WHERE status = expected` |
| 9 | **Distributed monolith** — services sharing a database | Distributed-systems cost with monolithic coupling; the worst of both | One writer per table; schemas and roles as boundaries |
| 10 | **Shared mutable state via copied code** | Copies diverge; a fix in one never reaches the others | Versioned package in a private registry |
| 11 | **Connection exhaustion** — unbounded pools × replicas | One service starves every other service of database access | Global budget; bounded pools; a connection proxy |
| 12 | **Unbounded queues** | A slow consumer becomes broker memory exhaustion, taking every queue down | `maxLength`, TTL, bounded prefetch, load shedding |
| 13 | **Unbounded prefetch** | One consumer pulls the whole backlog into memory | `prefetch(n)` matched to handler concurrency |
| 14 | **No dead-letter queue** | A rejected message is silently discarded; poison-message protection becomes data loss | DLQ per queue, monitored, with a replay tool |
| 15 | **Cache as source of truth** | A cache eviction or restart destroys business state | Database owns truth; cache is optional at read time |
| 16 | **Cache without TTL** | Memory grows with cumulative rather than concurrent volume | TTL at creation on every non-cache key too |
| 17 | **A list used as a set** | Unbounded growth with duplicates; O(n) removal | Set, or sorted set for self-expiring registries |
| 18 | **Secrets in a cache** | Widens the credential blast radius; makes rotation impossible | Store references; resolve from a secrets manager |
| 19 | **Overusing Redis** | Operational state with no durability, no transactions, no query capability | Use it for cache, counters, leases, and fan-out; nothing else |
| 20 | **Overusing microservices** | Deployment, observability, and on-call cost per service, with no isolation gained | Modular monolith until a boundary is proven |
| 21 | **Missing authorization** — a handler that forgets the check | Every layer passes and data leaks | Declared policy per operation, verified in CI |
| 22 | **Poor tenant isolation** — unscoped primary-key lookups | Cross-tenant reads and writes; an enumerable identifier makes it systematic | Required tenant parameter + RLS + generated cross-tenant tests |
| 23 | **Enumerable public identifiers** | Turns any missing check into bulk enumeration | Sortable opaque external identifiers |
| 24 | **Business logic in controllers** | Untestable without HTTP; duplicated across transports | Controllers orchestrate; services decide |
| 25 | **Business logic in middleware** | Runs invisibly for every route; hard to test and reason about | Middleware handles transport only |
| 26 | **Policy in the database** | Invisible to tests, review, type-checking, and search | Set-based computation in SQL; policy in the application |
| 27 | **Reusing standard error classes for business signals** | A genuine database error is reported as a business rejection | A private error class with a discriminating key |
| 28 | **Branch without an else** in policy logic | Unmapped cases silently permit what they should block | Exhaustive branching with an explicit failure |
| 29 | **Tight integration coupling** — vendor SDK imported everywhere | Vendor change is a codebase-wide migration | One adapter per provider |
| 30 | **Trusting webhook payload contents** | A forged or replayed payload changes state | Verify, deduplicate, then re-fetch authoritative state |
| 31 | **Missing observability** | Incidents are diagnosed by guesswork | Logs, metrics, traces, health, SLOs |
| 32 | **Observability on the business broker** | A broker incident blinds you during the incident | stdout plus platform collection |
| 33 | **Logging as an awaited network call** | Couples request latency to the logging pipeline | Synchronous local write to stdout |
| 34 | **Hard-coded configuration** | Cannot be tuned without a deploy — including during an incident | Environment-driven with validation at boot |
| 35 | **Truthiness checks for required config** | A boolean flag cannot be set to `false`; features cannot be disabled | Check for absence, not falsiness |
| 36 | **Environment-dependent security validation** | Production runs an untested code path | Identical validation everywhere |
| 37 | **Swallowing unhandled errors** | The process continues in unknown state; crashes become corruption | Log, flush, exit non-zero, restart clean |
| 38 | **Accepting traffic before dependencies are ready** | Early requests fail confusingly; errors attributed to the wrong cause | Await dependencies, then assert readiness |
| 39 | **In-process background loops in the API process** | Competes for the event loop and pool; duplicates across replicas | A worker deployable with atomic claiming |
| 40 | **In-process cron in a replicated service** | Fires once per replica | External scheduler, or a distributed lease |
| 41 | **Floating-point money** | Silent precision loss that surfaces as an unexplainable discrepancy | `NUMERIC` or integer minor units |
| 42 | **Recomputing aggregates over full history per request** | Cost grows forever; slowest for your best customer | Snapshot projections with a watermark |
| 43 | **Mutable balance fields** | Destroys the audit trail; races produce wrong balances | Append-only ledger; balance as a projection |
| 44 | **No reconciliation against external systems** | A lost webhook is never detected | Periodic reconciliation with divergence metrics |
| 45 | **Build on the target host** | Non-reproducible; rollback is a rebuild-and-hope | Build once; promote by digest |
| 46 | **No lockfile in the build** | The same commit produces different dependency trees | Copy the lockfile; `npm ci` |
| 47 | **`latest` base images** | Builds change without a code change | Pin by tag, ideally by digest |
| 48 | **Containers as root, wrapper as PID 1** | Escape starts as root; signals never reach the app; graceful shutdown never runs | Non-root user; init shim; run the binary directly |
| 49 | **No CI gate** | Tests, types, and audits are advisory | Blocking checks on every PR |
| 50 | **Deploy authorisation in pipeline YAML** | Changes require commits; mutable identities break silently | Platform environment protection |
| 51 | **Mutable third-party CI action refs** | Arbitrary code runs with access to your secrets | Pin to a commit digest |
| 52 | **Static long-lived cloud credentials** | Manual rotation; broad exposure; no audit trail | Platform identity; OIDC federation |
| 53 | **Unauthenticated endpoints triggering metered calls** | Cost amplification that leaks no data and passes security review | Authenticate, rate-limit, cap spend, meter, alert |
| 54 | **Unbounded `Promise.all` over external calls** | Rate-limit exhaustion and connection storms | Bounded concurrency with `allSettled` |
| 55 | **Liveness probes that check dependencies** | A shared dependency failure restarts the entire fleet | Liveness is dependency-free; readiness is not |
| 56 | **Mocking the database in tests** | The mechanisms you rely on — constraints, transactions, predicates — go untested | Real infrastructure in containers |
| 57 | **Tolerated flaky tests** | Trains the team to ignore red; a real failure hides among them | Fix or delete within the week |
| 58 | **Two systems for one concern** | Divided attention; neither is trusted | Consolidate |
| 59 | **Shallow abstractions that re-export their dependency's types** | A layer with no insulation; the coupling merely moved | Domain vocabulary in, implementation hidden |
| 60 | **Compensating in forward order, or over the full step list** | Undoes the wrong things in the wrong order | Reverse a copy of the executed steps only |
