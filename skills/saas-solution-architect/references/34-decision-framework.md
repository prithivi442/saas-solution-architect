# Architectural decision framework

Twelve decision trees. The uniform goal: **do not add architectural complexity unless the problem requires it.**

## 39.1 Monolith vs modular monolith vs microservices

```
Team size ≤ 15, or domain boundaries still moving?
  → MODULAR MONOLITH. One deployable, one database, enforced module seams.

Team > 15 AND a domain owns its data cleanly AND it has an independent
scaling curve AND you have CI, IaC, observability, and on-call in place?
  → Extract THAT service, with ITS OWN DATABASE.

Otherwise
  → Stay. Strengthen module seams instead.

Never extract because:
  · a different team works on it
  · the code "feels large"
  · microservices are the industry default
  · a rewrite is more appealing than a refactor
```

## 39.2 PostgreSQL vs NoSQL

```
Is the data relational — entities with relationships and joins?     → PostgreSQL
Do you need multi-record transactions?                              → PostgreSQL
Are queries known and mostly stable?                                → PostgreSQL
Is it schema-variable, append-only, high-write, cheap-retention?     → consider a document store
Is it a key-value cache?                                            → Redis
Is it full-text search?                                             → a search engine (or Postgres FTS to start)
Is it time-series metrics?                                          → a metrics store
Is it an analytical workload over large volumes?                    → a columnar store
Unsure?                                                             → PostgreSQL with jsonb
```

## 39.3 When to introduce Redis

```
Is a specific read MEASURABLY hot, repeated, and staleness-tolerant?  → cache it
Must multiple replicas share ephemeral state?                          → Redis
Do you need a distributed lease?                                       → Redis
Do you need rate-limit counters across replicas?                       → Redis
Do you need pub/sub fan-out for live updates?                           → Redis
Is the query slow because an index is missing?                         → add the index
Is it "probably slow"?                                                 → measure first
Would losing it corrupt business state?                                → PostgreSQL, not Redis
```

## 39.4 When to introduce a queue

```
Must the work survive a process restart?                    → queue
Is the work slow, and is nobody waiting for it?             → queue
Is the work retryable against an unreliable dependency?     → queue
Must the request return before the work completes?          → queue
Is it fast, in-process, and needed for the response?        → do it inline
Is it a UI notification that self-heals?                    → pub/sub is enough
```

## 39.5 When to introduce an outbox

```
Must a state change produce an event, where losing it corrupts state?  → outbox
Does a state change require an external side effect?                    → outbox
Does it move money or provision a resource?                             → outbox, always
Do you need an audit trail of dispatch attempts?                        → outbox
Is it a transient UI notification?                                      → no outbox
Are producer and consumer the same transaction in the same database?    → no outbox
```

## 39.6 Synchronous vs asynchronous

```
Must the user be told the outcome before acting again?          → synchronous
Does the request gate on the answer (authz, validation)?        → synchronous
Does the screen render from it?                                → synchronous
Must a limit PREVENT the action?                               → synchronous
Everything else                                                → asynchronous
                                                                 (+ idempotency,
                                                                    retry, DLQ, metric)
```

## 39.7 When to introduce read replicas

```
Do reporting or analytical queries compete with transactional load?   → replica
Do read-heavy endpoints tolerate seconds of staleness?                → replica
Is the primary CPU- or IO-bound on reads?                             → replica
Is a single query slow?                                               → fix the query
Do you need stronger read-after-write than replication provides?      → primary, per query
```

## 39.8 When to partition a table

```
Is a single table large enough that maintenance is painful?     → partition by time
Are old rows rarely read but retained for compliance?           → partition + archive cold
Does every query filter by tenant, with very large tenants?     → consider tenant partitioning
Is the table merely "big"?                                      → index correctly first
```

## 39.9 When to introduce distributed locks

```
Can it be a conditional write whose predicate is the precondition?  → do that; no lock
Can it be an atomic increment or an upsert?                         → do that; no lock
Is the resource in the database?                                    → SELECT … FOR UPDATE
Is the resource external (a provider, a file)?                      → lease with TTL
Is it scheduled work in a replicated service?                       → lease with TTL
Is the operation naturally idempotent?                              → no lock needed
```

## 39.10 When to introduce event-driven architecture between services

```
Do multiple consumers need the same event?                       → events
Is a synchronous call chain causing cascading failures?          → events
Must the producer complete regardless of consumer health?        → events, via outbox
Is it a single caller needing an immediate answer?               → synchronous call
Would eventual consistency be visible and confusing to a user?   → synchronous, or redesign
```

## 39.11 When to introduce CQRS

```
Do read and write loads conflict, after replicas and indexes?         → separate read model
Do read shapes differ radically from the write model?                → projection
Do reads need aggregations too expensive to compute per request?      → materialised projection
Is a query slow?                                                     → fix the query
Does it "feel cleaner"?                                              → no
```

## 39.12 When to add another service

```
Answer ALL of these YES:
  □ It owns tables no other service writes
  □ It has an independent scaling curve, measured
  □ It has an independent failure domain that isolates something real
  □ Or an independent security posture that justifies separation
  □ Its API contract is stable enough to version
  □ You have CI, IaC, observability, and on-call for one more deployable

Any NO → make it a module.
```
