# Scalability

## 28.1 Component-by-component assessment

| Component | Horizontal | Stateful constraint | Bottleneck |
|---|---|---|---|
| API services | Yes — stateless | None | Database connections |
| Realtime service | Yes | Connections are node-local | Connection count per node; pub/sub fan-out |
| Queue consumers | Yes | None | Database; provider rate limits |
| Ledger service | Yes | None | Aggregate query cost |
| **PostgreSQL** | **No — single writable primary** | All state | **The system-wide ceiling** |
| Redis | Vertically; cluster for horizontal | Shared state | Memory; single-threaded command throughput |
| RabbitMQ | Cluster | Queue-local ordering | Broker memory under backlog |
| MongoDB | Replica set / shard | — | Write throughput |
| Background pollers | Only with atomic claiming | Duplicate work without it | Poll rate; database load |
| Web client | CDN | — | — |

## 28.2 Growth analysis

*Conceptual, ordered by which constraint binds first. No invented benchmarks.*

### 10× — the constraints that bind first

| # | Constraint | Why it binds | Intervention |
|---|---|---|---|
| 1 | **Database connections** | Replica count multiplies pool size against a fixed `max_connections` | Connection proxy; smaller per-replica pools |
| 2 | **Aggregate queries whose cost grows with history** | Balance and counter queries on request paths slow as tenants accumulate data | Snapshot projections; maintained counters |
| 3 | **Read contention from analytics on the write primary** | Reporting competes with transactional work for buffers and locks | Read replica; move reporting off the primary |
| 4 | **Queue consumer throughput** | Unbounded prefetch and per-message database round trips | Bounded prefetch; batch writes; more consumer replicas |
| 5 | **Missing indexes on new access patterns** | Table growth turns tolerable scans into timeouts | Query-plan review; `pg_stat_statements` |
| 6 | **In-process background loops** | Each replica repeats the same work | Extract to a worker deployable with atomic claiming |
| 7 | **Cache memory** | Unbounded keys accumulate | TTLs; correct collection types; `maxmemory` policy |

**Nothing architectural must change at 10×.** These are all configuration, indexing, and projection work — the reward for having chosen a relational primary and a stateless service tier.

### 100× — where structure must change

| # | Constraint | Intervention |
|---|---|---|
| 1 | **Write throughput on the single primary** | Vertical scaling first (it goes far). Then move the highest-volume append-only tables — events, journal, logs — to their own database or a purpose-built store. |
| 2 | **Table sizes on the largest tables** | Declarative partitioning by time and/or tenant; archival of cold partitions to object storage. |
| 3 | **Read volume** | Multiple read replicas with routing at the data layer, and explicit staleness tolerance per query. |
| 4 | **Shared-database coupling becomes the limiting factor on change** | This is where data ownership stops being a design preference and becomes a delivery constraint. Extract the highest-volume domains with their own databases. |
| 5 | **Broker throughput** | Partition queues by tenant or shard; consider a partitioned log for high-volume streams. |
| 6 | **Cache capacity and throughput** | Cluster; separate instances by usage class so eviction policy fits each. |
| 7 | **Realtime connection count** | More nodes; a dedicated fan-out tier; consider a managed WebSocket service. |
| 8 | **Cost** | At this scale, per-request cost visibility becomes an engineering requirement rather than a finance report. |

### 1000× — a different architecture

| Constraint | Intervention |
|---|---|
| Single primary is impossible | **Shard by tenant.** Tenant → shard mapping in a routing layer; cross-shard queries become forbidden by design rather than discouraged. |
| Cross-tenant queries | A separate analytical store fed by CDC; no cross-shard transactional queries. |
| Regional latency and data residency | Multi-region with tenant pinning; regional data stores. |
| Operational load | Full automation: infrastructure as code, self-healing, automated capacity management. Manual operations are not viable at this scale. |
| Cost | Per-tenant unit economics measured continuously and fed back into pricing. |

**The honest observation:** the architecture as it stands scales cleanly to 10× with configuration and projection work. It reaches 100× with genuine but well-understood investment. Beyond that, the shared database is the binding constraint, and removing it requires the data-ownership work described in `02-architecture-and-boundaries.md` — which is precisely why that work is worth starting long before it is forced.

## 28.3 Single points of failure

| SPOF | Blast radius | Mitigation |
|---|---|---|
| PostgreSQL primary | Total outage | Multi-AZ with automated failover; **tested** restore; documented RPO/RTO |
| Redis instance | Live coordination and caching lost | Replication with automatic failover; the application must tolerate a cold cache |
| RabbitMQ | Async processing halts | Cluster with quorum queues; the outbox makes the commit sufficient meanwhile |
| Federation router | Entire API unavailable | Multiple replicas behind the load balancer; health-gated |
| Identity provider | No new logins | Managed service with an SLA; existing tokens remain valid until expiry |
| Deployment host | Cannot deploy; may affect running services | Multiple hosts; a registry-based artifact pipeline |
| A single shared secret | Broad credential exposure | Per-service secrets with rotation |

**The most valuable exercise here is not the table but the drill:** remove each dependency in a staging environment and observe what actually happens. The gap between the documented mitigation and the observed behaviour is where the next incident lives.

## 28.4 Scalability principles

1. **Stateless services scale horizontally; make everything stateless that can be.**
2. **The database is the ceiling.** Every scaling conversation begins there.
3. **No per-request cost may grow with cumulative data.**
4. **Decouple replica count from database connections with a proxy.**
5. **Reads go to replicas; writes go to the primary; the routing lives in the data layer.**
6. **Partition before you shard; shard only when partitioning is exhausted.**
7. **Data ownership is a prerequisite for scaling beyond one database** — start it early.
8. **Bound everything.** Unbounded growth is the mechanism of every scaling failure.
9. **Measure per-tenant cost** before it becomes a margin problem.
10. **Scale operations with the system**: infrastructure as code and automation are scalability features.
