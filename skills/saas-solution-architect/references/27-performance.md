# Performance engineering

## 26.1 The architectural performance decisions that matter

| Decision | Effect | Assessment |
|---|---|---|
| Cursor pagination as the default | Constant-cost page fetch regardless of depth | **Correct and important** |
| Collapsing a hot validation sequence into one database round trip | Removes 3–4 network round trips from the hottest path | **Correct**, with the caveats in `05-data-architecture.md` §4 |
| Caching resolved tenant and plan data | Removes repeated joins from every request | **Correct** |
| Cache-based coordination for latency-sensitive workflows | Sub-millisecond shared state | **Correct for the constraint** |
| Async offloading of email, media, exports, AI, and metering | Keeps request latency independent of slow work | **Correct** |
| A separate compiled service for compute-heavy aggregation | Avoids blocking an event loop | **Correct** |
| Bounded GraphQL response cache | Prevents unbounded memory growth | Correct |
| Atomic counters instead of recomputed aggregates in hot paths | O(1) instead of O(n) | **Correct where applied** |

## 26.2 The performance patterns that matter most

Architectural performance work beats micro-optimisation by orders of magnitude. Six patterns account for most of the available gain:

### 1. Never let a per-request cost grow with cumulative history

This is the single most important scaling rule and the easiest to violate accidentally.

```
✗ Per-request:  SELECT count(*) FROM events WHERE tenant = t AND day = today
                → cost grows with today's volume; slowest for your best tenant

✗ Per-request:  SELECT SUM(credit) - SUM(debit) FROM journal WHERE tenant = t
                → cost grows with ALL history; unbounded

✓ Per-request:  SELECT used FROM counters WHERE tenant = t AND window = w
                → O(1), maintained by atomic increment

✓ Per-request:  snapshot + entries since the snapshot watermark
                → cost proportional to recent activity only
```

**The diagnostic question for any query on a request path:** *what is this query's cost when the tenant is 100× larger?* If the answer is "100× slower," it is a latent outage that will surface first for your most valuable customer.

### 2. Index for the access pattern, leading with the tenant

```sql
-- Tenant-scoped list queries: tenant first, then the sort key
CREATE INDEX records_tenant_created_idx
  ON records (tenant_id, created_at DESC)
  WHERE deleted_at IS NULL;                -- partial: excludes soft-deleted rows

-- Cursor pagination needs the index to match the ORDER BY exactly
-- WHERE tenant_id = ? AND (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC
CREATE INDEX records_tenant_cursor_idx
  ON records (tenant_id, created_at DESC, id DESC)
  WHERE deleted_at IS NULL;

-- Sweeps: partial indexes on non-terminal states stay small forever,
-- because their size tracks CONCURRENT work rather than total history
CREATE INDEX outbox_pending_idx ON outbox (next_attempt_at)
  WHERE state NOT IN ('published', 'terminated');
```

**Two rules that matter with universal soft deletes:** every index on a soft-deleted table should be partial on `deleted_at IS NULL` (otherwise it carries dead rows forever), and **every uniqueness constraint must account for soft deletion** — a plain unique index prevents re-creating a value that was deleted.

### 3. Eliminate N+1 at the data layer

GraphQL makes N+1 the default rather than the exception: a list of 50 items with a nested field produces 50 queries unless something batches them.

```ts
// A per-request DataLoader batches and de-duplicates within one tick.
// Scoped per request so its cache cannot leak across tenants.
const tenantLoader = new DataLoader<Id, Tenant>(async (ids) => {
  const rows = await tenants.findMany({ id: { [Op.in]: [...ids] } });
  const byId = new Map(rows.map(r => [r.id, r]));
  return ids.map(id => byId.get(id) ?? null);
}, { cache: true, maxBatchSize: 100 });
```

**Principle.** *Every GraphQL field that resolves a relationship uses a per-request batching loader. Never construct a loader outside request scope — a cross-request cache in a multi-tenant system is a data-leak mechanism.*

### 4. Move work off the request path

```
On the request path:  authorization · validation · the write the user awaits ·
                      the read that renders the screen
Off it:               notification · metering · denormalisation · third-party
                      sync · media processing · export generation · analytics
```

### 5. Bound everything

| Unbounded thing | Failure |
|---|---|
| Queue prefetch | Consumer memory exhaustion |
| Query results without `LIMIT` | One request loads a table into memory |
| Cache without TTL or `maxmemory` | Eviction of the wrong keys, or write failures |
| Retry attempts | Infinite loops consuming workers |
| Concurrent external calls (`Promise.all` over an unbounded list) | Provider rate-limit exhaustion, connection storms |
| Request body size | Memory exhaustion from a single request |
| GraphQL depth and complexity | Amplification from one small request |
| Batch job runtime | A stuck job blocking a queue indefinitely |

The `Promise.all` case deserves a concrete pattern, because it appears wherever a collection is processed:

```ts
✗ await Promise.all(items.map(i => provider.call(i)));
    // 5,000 items ⇒ 5,000 concurrent calls ⇒ rate-limited, or worse

✓ // Bounded concurrency, with partial-failure visibility
  const limit = pLimit(10);
  const results = await Promise.allSettled(
    items.map(i => limit(() => provider.call(i))),
);
  const failed = results.filter(r => r.status === 'rejected');
  if (failed.length) {
    metrics.counter('batch.partial_failure', failed.length);
    logger.warn('batch completed with failures', { total: items.length,
                                                   failed: failed.length });
  }
```

`allSettled` over `all` is deliberate: `Promise.all` rejects on the first failure and abandons the rest, leaving the batch in an unknown partial state.

### 6. Read the plan, don't guess

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT … ;
-- Look for: Seq Scan on a large table · rows estimate far from actual ·
--           Nested Loop over many rows · Sort spilling to disk ·
--           high shared read (cold cache)
```

Enable `pg_stat_statements` and `auto_explain` for slow queries in production. A slow-query threshold configuration value is only useful if something acts on what it logs.

## 26.3 Performance principles

1. **No per-request cost may grow with cumulative history.**
2. **Index for the access pattern, leading with the tenant column.**
3. **Partial indexes for soft-deleted tables and non-terminal states.**
4. **Batch every relationship resolution; scope loaders per request.**
5. **Bound every collection, concurrency, retry, and payload.**
6. **`allSettled` with a concurrency limit, never bare `Promise.all` over an unbounded list.**
7. **Measure before optimising; read query plans.**
8. **Cache with intent**: a known-hot, repeated, staleness-tolerant read — never as a substitute for an index.
9. **Move everything off the request path that the user is not waiting for.**
10. **Compiled languages for genuinely CPU-bound work**, not for uniformity.
11. **Load-test the paths that matter** — the hot validation path, the list endpoints, the charge path — before customers do.
