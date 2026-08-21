# Production readiness checklist

Independent of any specific product. Use as a gate before a new SaaS serves paying customers.

## Architecture
```
□ Service boundaries documented, each with owned tables named
□ No table written by more than one deployable
□ Every synchronous inter-service dependency has a timeout and a fallback
□ Every async flow has an outbox, idempotency, a DLQ, and a metric
□ A degradation mode is documented AND TESTED for every dependency
□ Single points of failure enumerated with accepted recovery plans
□ Deployment topology diagram matches reality
```

## Database
```
□ Multi-AZ with automated failover, and failover tested
□ Automated backups with a RESTORE that has been performed successfully
□ RPO and RTO documented and validated by drill
□ Connection budget computed across all consumers; headroom ≥ 30%
□ Connection proxy in place
□ Every tenant-scoped table: tenant column, NOT NULL, leading index
□ RLS enabled with a policy on every tenant-scoped table
□ Every per-tenant unique index includes the tenant column
□ Indexes verified against actual query plans, not assumed
□ Partial indexes on soft-deleted tables and non-terminal states
□ Slow-query logging enabled with something acting on the output
□ Migrations automated, locked, blocking, and backward-compatible
□ Read replica for reporting; routing implemented in the data layer
```

## Transactions and consistency
```
□ Every transaction's protected invariant is documented
□ No external I/O inside any transaction
□ Every state transition is a conditional write
□ Every limit and quota is an atomic reservation
□ Recovery sweeps query the invariant, not a recorded error state
□ Isolation levels chosen deliberately where multi-statement invariants exist
□ Reconciliation jobs exist for every externally-mirrored state
□ Reconciliation divergence is a monitored, alerting metric
```

## API
```
□ Contract published and versioned
□ Cursor pagination on every list endpoint
□ Idempotency-Key honoured on every non-idempotent mutation
□ Correlation ID accepted, generated, propagated, returned
□ Errors carry a stable code, a localised message, and a retryable flag
□ Cross-tenant access returns 404, never 403
□ Rate limiting per tenant and per principal
□ Request size limits enforced
□ GraphQL: depth, complexity, alias, and batch limits; introspection off
□ Validation rejects unknown fields
```

## Authentication and authorization
```
□ Managed identity provider; no home-grown credential storage
□ Short-lived access tokens; refresh rotation
□ Revocation denylist with TTL; fails closed
□ Principal-wide revocation supported
□ Four-layer model implemented
□ Every operation has a DECLARED policy, verified by a CI check
□ Validation identical across all environments
□ Constant-time comparison for every secret
□ Every security decision derives from a verified claim or server state
```

## Multi-tenancy
```
□ Tenant is a required, branded parameter in the data layer
□ RLS enabled as the backstop
□ Global reference data is an explicitly different repository type
□ Cache keys, queue messages, object paths, and search indexes all scoped
□ Cross-tenant test matrix generated across all endpoints and resources
□ Cross-tenant denial metric exists and is monitored
□ No enumerable public identifiers
```

## Caching
```
□ Every key classified: cache / security state / operational state
□ TTL on every key, set at creation
□ Collection types match semantics (sets for membership)
□ Failure mode defined per read: fail closed for security, fail open deliberately
□ maxmemory and eviction policy set explicitly
□ Cache and state on separate instances if eviction policies conflict
□ No secrets, no PII
□ Memory, evictions, and key count monitored
```

## Queues and async
```
□ Durable queues AND persistent messages
□ Bounded prefetch per consumer
□ DLQ per queue, monitored, with alerting on depth > 0
□ Replay tool built and tested
□ Every handler idempotent
□ Bounded retries with backoff, jitter, and a terminal state
□ Queue depth and OLDEST-PENDING-AGE metrics
□ Producers publish to exchanges/topics, not to queue names
□ Message schema versioned
```

## Billing
```
□ Money as NUMERIC or integer minor units, end to end
□ Currency on every amount; no cross-currency arithmetic
□ Append-only ledger; balance as a projection with a snapshot watermark
□ Full-recomputation verification job that must agree with the projection
□ Idempotency key on every charge, derived from its cause
□ Balance checks are write predicates
□ Webhooks signed, deduplicated, persisted, then processed async
□ Reconciliation against the provider, with divergence alerting
□ Failed payment restricts capability; never deletes data
□ Every financial mutation is audit-logged
□ Tests: duplicate, out-of-order, concurrent, insufficient funds, refund, crash
```

## Background jobs
```
□ Business timers in an external durable scheduler
□ Workers are a separate deployable with their own pool and health
□ Atomic claiming for concurrent processors
□ Every job idempotent with a natural key
□ Long jobs record durable progress and resume
□ Maximum runtime with defined behaviour at the limit
□ Job metrics: started, completed, failed, duration, oldest pending
□ Graceful shutdown drains or yields in-flight work
```

## Reliability
```
□ Timeout, retry, breaker, bulkhead on every external dependency
□ Liveness (dependency-free), readiness, and startup endpoints
□ Dependencies verified before accepting traffic
□ Graceful shutdown reachable through the container entrypoint
□ Platform grace period exceeds application drain timeout
□ Unhandled errors flush telemetry and exit non-zero
□ Load shedding before collapse
□ Failure drills performed in staging
```

## Security
```
□ Secrets in a managed store; rotation requires no deploy
□ Platform identity for cloud API access; no long-lived keys
□ Least-privilege IAM per service
□ Non-root containers, pinned bases, lockfile-driven builds
□ Dependency and image scanning in CI, blocking on high severity
□ SBOM generated and retained per build
□ CI actions pinned by digest; OIDC for cloud credentials
□ Inventory of unauthenticated endpoints, reviewed on change
□ Endpoints classified by cost exposure as well as data sensitivity
□ Security headers set explicitly
□ Encryption in transit and at rest, ASSERTED at startup
□ Audit log with longer retention and stricter access
□ Penetration test performed and findings closed
```

## Observability
```
□ Structured JSON logs to stdout; collection is a platform concern
□ Log level INFO in production
□ Correlation ID in every log line, from ambient context
□ RED metrics per endpoint; USE per dependency; saturation gauges
□ Business metrics: money, reconciliation, metered spend, authz denials
□ Distributed tracing across HTTP and queue boundaries
□ One error-tracking system, with release tagging and source maps
□ SLOs with error budgets; alerts on burn rate
□ A runbook per alert
□ Dashboards for: request health, dependency health, queue health, saturation
□ No secrets or PII in logs
```

## Performance
```
□ No per-request cost grows with cumulative history
□ Query plans reviewed for every hot path
□ Relationship resolution batched; loaders scoped per request
□ Every collection, concurrency, retry, and payload bounded
□ Load test performed on the hot paths
□ Event-loop lag (or equivalent) monitored
```

## Infrastructure and CI/CD
```
□ Infrastructure as code for everything in production
□ Build once; promote an immutable digest
□ Blocking CI gate on every PR
□ Migrations then rolling deploy, health-gated
□ Smoke tests after deploy; automatic rollback on regression
□ Rollback is one command and has been rehearsed
□ Environments differ only in configuration
□ Sandbox third-party credentials outside production, enforced at boot
□ Auto-scaling on a measured signal, with minimums and maximums
□ Every deploy traceable: commit, digest, approver, timestamp
```

## Testing
```
□ CI runs all tests and blocks on failure
□ Integration tests against real infrastructure in containers
□ Generated cross-tenant matrix across all endpoints
□ Concurrency test per invariant
□ Duplicate-delivery test per consumer
□ Failure-injection test per dependency
□ Near-total coverage on money paths
□ Migrations tested forward against a realistic dataset
□ No tolerated flaky tests
```

## Disaster recovery
```
□ RPO and RTO documented per data store
□ Restore performed successfully from backup, timed, and recorded
□ Failover tested for database, cache, and broker
□ Full-region loss plan documented (even if the answer is "extended outage")
□ Runbooks for: total outage, data corruption, credential compromise,
  accidental deletion, dependency outage
□ Incident response roles and escalation defined
□ Post-incident review process defined and used
□ Backups encrypted, access-controlled, and retention-tested
```
