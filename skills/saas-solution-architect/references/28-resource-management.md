# Resource management

## 27.1 Resource handling

| Resource | Pooled | Reused | Bounded | Released | Notes |
|---|---|---|---|---|---|
| Database connections | Yes | Yes | Yes | On shutdown | Bound hard-coded in some services |
| Cache connections | Yes | Yes (singleton) | Yes | On shutdown | Auto-reconnect with capped backoff |
| Broker connections | Yes | Yes (per queue) | Per queue | On shutdown | Connection manager handles reconnect |
| Broker channels | Yes | Yes | — | On shutdown | Prefetch not bounded |
| Cloud SDK clients | Partial | Partial | — | Process exit | Constructed per call in some paths |
| HTTP agents | Default | Default | Default | — | Keep-alive not configured explicitly |
| Worker goroutines | **Yes** (pool library) | Yes | Yes | — | Correct pattern in the Go service |
| File handles (media processing) | N/A | N/A | — | Stream-based | |
| Memory | — | — | Not declared | — | No explicit heap limits found |
| WebSocket connections | Per process | Yes | — | On disconnect | |

## 27.2 Reference flow: one client per process, configured explicitly

```ts
// clients.ts — constructed once at module load, shared for the process lifetime
import https from 'node:https';
import { NodeHttpHandler } from '@smithy/node-http-handler';

// One keep-alive agent, shared: connection reuse across all AWS calls
const agent = new https.Agent({
  keepAlive: true,
  keepAliveMsecs: 30_000,
  maxSockets: 50,             // bounded: cannot exhaust file descriptors
  maxFreeSockets: 10,
  timeout: 60_000,
});

const handler = new NodeHttpHandler({
  httpsAgent: agent,
  connectionTimeout: 1_000,   // fail fast on an unreachable endpoint
  requestTimeout:    5_000,   // bound every request
});

export const s3  = new S3Client ({ region, requestHandler: handler, maxAttempts: 3 });
export const kms = new KMSClient({ region, requestHandler: handler, maxAttempts: 3 });
export const ses = new SESClient({ region, requestHandler: handler, maxAttempts: 3 });
```

**Principle.** *Every client that owns a connection pool is a process-level singleton, created at startup, with explicit timeouts and bounded sockets. A client created inside a request handler discards its pool on every call.*

## 27.3 Reference flow: declaring and enforcing memory limits

```ts
// Node's default heap limit is derived from available memory, not from the
// container limit — so a container limit of 512MB with a default heap can
// be OOM-killed by the system before V8 ever runs a full GC.
// Set the heap explicitly, below the container limit.
//   node --max-old-space-size=400 dist/server.js     (for a 512MB container)
```

```yaml
# Declare limits so the system can schedule and enforce them
resources:
  requests: { memory: "256Mi", cpu: "250m" }
  limits:   { memory: "512Mi", cpu: "1000m" }
```

And instrument the outcome:

```ts
setInterval(() => {
  const m = process.memoryUsage();
  metrics.gauge('process.heap_used_bytes',  m.heapUsed);
  metrics.gauge('process.heap_total_bytes', m.heapTotal);
  metrics.gauge('process.rss_bytes',        m.rss);
  metrics.gauge('process.external_bytes',   m.external);   // buffers, native memory
  metrics.gauge('event_loop.lag_ms',        measureLoopLag());
}, 15_000);
```

**Event-loop lag is the most valuable single Node metric.** Rising lag means the loop is blocked by synchronous work — the failure mode most likely to make a Node service appear "randomly slow" while CPU and memory look normal.

## 27.4 The connection budget worksheet

Produce this table for your own system and keep it current:

```
Component              Replicas   Per-replica   Total
──────────────────────────────────────────────────────
API service                  4    pool 10          40
Worker service               2    pool 10          20
Realtime service             2    pool  5          10
Ledger service               2    pool 10          20
Scheduled jobs               1    pool  5           5
Migrations (transient)       1          2           2
Ops tooling (transient)      —          5           5
──────────────────────────────────────────────────────
Peak total                                        102
Database max_connections                          200
Headroom                                          49%      ← keep above ~30%
```

Then: **halve the per-replica pools and put a proxy in front**, so replica count and database connections are decoupled entirely.

## 27.5 Resource-management principles

1. **Every pool is bounded, environment-configurable, and documented in one budget.**
2. **Clients that own connections are process-level singletons.**
3. **Keep-alive with bounded sockets on every HTTP agent.**
4. **Declare memory and CPU limits; set the runtime heap below the container limit.**
5. **Bound concurrency on every fan-out.**
6. **Release on shutdown, in dependency order, with a forced-exit backstop.**
7. **Instrument saturation**: pool in-use and waiting, socket counts, heap, event-loop lag.
8. **A pool `waiting` count above zero is the earliest warning of contention** — alert on it.
9. **Streams, not buffers**, for anything whose size scales with input.
10. **Decouple replica count from database connections with a proxy.**
