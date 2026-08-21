# Reliability engineering

## 22.1 The mechanisms, and what each protects against

| Mechanism | Requirement | Protects against |
|---|---|---|
| Broker auto-reconnect with connection lifecycle logging | **Required** | Transient broker unavailability |
| Cache auto-reconnect with capped backoff | **Required** | Transient cache unavailability |
| Publish retry with exponential delay | **Required** | Transient publish failure |
| Outbox with recovery sweep | **Required** | Lost events after commit |
| Retry with exponential backoff and jitter | **Yes** (outbox) | Transient dependency failure; thundering herd |
| Transient-error classification on retry | **Required** | Wasted retries; load amplification |
| Timeout policy on key external calls | **Yes** (partial) | Hung requests holding resources |
| Circuit breaker | Implemented, not wired | Retry storms |
| Bulkhead | Implemented, not wired | Cross-dependency resource starvation |
| Rate limiting (distributed) | **Required** | Provider quota exhaustion; abuse |
| Kill switch per integration | **Required** | Runaway integration; vendor incident |
| Graceful shutdown with signal handling | **Yes** (Node services) | Dropped in-flight requests during deploy |
| Attempt caps with terminal states | **Required** | Infinite retry loops |
| Health endpoint | One service | Orchestrator routing decisions |
| Dead-letter queues in the broker topology | Handled out-of-band | Silent message loss |
| Fallback / graceful degradation | Implemented, not wired | Total failure on partial outage |

## 22.2 Reference flow: graceful shutdown that actually runs

The graceful-shutdown implementation in the Node services is thorough — signal handlers, HTTP server drain, ordered closure of database, cache, and broker connections, and a forced-exit timer as a backstop. The pattern is correct and worth copying:

```ts
private async shutdown(): Promise<void> {
  this.server?.close(async () => {           // stop accepting; drain in-flight
    await database.close();
    await cache.disconnect();
    await broker.close();
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10_000); // backstop: never hang a deploy
}
process.on('SIGTERM', () => this.shutdown());
process.on('SIGINT',  () => this.shutdown());
```

**The property that determines whether it ever runs is the container entrypoint.** A signal delivered to a shell or a package-manager wrapper as PID 1 may not reach the application, and PID 1 in a container does not reap zombies by default.

```dockerfile
✗ CMD ["npm", "start"]
    # npm is PID 1; SIGTERM handling depends on the wrapper forwarding it,
    # and PID 1 performs no zombie reaping

✓ # Run the application directly as PID 1, under an init that forwards signals
  # and reaps children.
  ENTRYPOINT ["/usr/bin/dumb-init", "--"]
  CMD ["node", "dist/server.js"]
  #  …or `docker run --init`, or Kubernetes shareProcessNamespace,
  #     or ECS `initProcessEnabled: true`
```

Also required: **the orchestrator's termination grace period must exceed the application's drain timeout.** A 10-second drain under a 5-second grace period is a 5-second drain followed by `SIGKILL`.

**Principle.** *Graceful shutdown is only as good as the process supervision beneath it. Run the application as PID 1 under an init shim, and align this standard's grace period with the application's drain timeout.*

## 22.3 Reference flow: process-level error handling

```ts
✗ process.on('uncaughtException',  (err) => { logger.error(err); });
  process.on('unhandledRejection', (err) => { logger.error(err); });
     // The process continues after an unknown failure. Its internal state is
     // unknown: a half-completed transaction, a leaked connection, a corrupted
     // cache entry. Subsequent behaviour is undefined and undebuggable.

✓ process.on('uncaughtException', (err, origin) => {
    logger.fatal('uncaught exception — exiting', { err, origin });
    // Best effort: flush telemetry, then exit. Let the orchestrator restart
    // a clean process. A short-lived crash loop is visible and diagnosable;
    // a zombie process serving wrong answers is neither.
    void Sentry.flush(2_000).finally(() => process.exit(1));
  });

  process.on('unhandledRejection', (reason) => {
    logger.fatal('unhandled rejection — exiting', { reason });
    void Sentry.flush(2_000).finally(() => process.exit(1));
  });
```

The objection to this is "restarting on every error causes an outage." The response is that the alternative — continuing in an unknown state — causes a *silent* outage, and the real fix is to handle expected errors where they occur so that an uncaught one genuinely is a bug. **A process that swallows unknown failures converts crashes into corruption.**

**Principle.** *An unhandled exception means an unknown failure and unknown state. Log it, flush telemetry, exit non-zero, and let the system restart a clean process. Restart is a recovery mechanism; continuation is not.*

## 22.4 Reference flow: startup ordering

```ts
✗ public async start() {
      this.connectDependencies();     // not awaited
      this.registerConsumers();
      await this.listen();            // accepting traffic before dependencies are ready
  }
     // The first requests arrive before the database connection is established.
     // Health checks pass. Errors are attributed to the wrong cause.

✓ public async start() {
      await this.connectDependencies();     // fail fast, before serving anything
      await this.runStartupAssertions();    // TLS verified, migrations current, config valid
      this.registerConsumers();
      await this.listen();                  // only now is traffic accepted
      this.markReady();                     // readiness flips only after all of the above
  }
```

**Principle.** *A process does not accept traffic until every dependency it needs is verified. Readiness is a state the application asserts after its dependencies are proven, not a side effect of the port being open.*

## 22.5 Reference flow: health endpoints for orchestration

An orchestrator can only make good routing decisions if the application tells it the truth. Three distinct endpoints, three distinct meanings:

```ts
// LIVENESS — "is this process wedged?" Must be cheap and dependency-free.
// A liveness check that touches the database will restart every replica
// during a database incident, converting a degradation into an outage.
app.get('/health/live', (_req, res) => res.status(200).json({ status: 'alive' }));

// READINESS — "should this replica receive traffic right now?"
// Checks dependencies. Returning 503 removes it from the load balancer
// without killing it, so it can recover.
app.get('/health/ready', async (_req, res) => {
  const checks = await Promise.allSettled([
    withTimeout(db.query('SELECT 1'),   500, 'database'),
    withTimeout(cache.ping(),           300, 'cache'),
    withTimeout(broker.checkChannel(),  300, 'broker'),
  ]);
  const failed = checks.filter(c => c.status === 'rejected');
  res.status(failed.length ? 503 : 200).json({
    status: failed.length ? 'degraded' : 'ready',
    checks: describe(checks),
  });
});

// STARTUP — "has initialisation finished?" Lets a slow-starting process
// have a generous startup budget without weakening its liveness timeout.
app.get('/health/startup', (_req, res) =>
  res.status(ready ? 200 : 503).json({ ready }));
```

Three rules govern these:

1. **Liveness never checks dependencies.** Otherwise a shared dependency failure restarts the entire fleet simultaneously.
2. **Every readiness check has a timeout.** A readiness probe that hangs is worse than one that fails.
3. **Readiness reflects the ability to serve, including queue-consumer health** in a worker deployable. A worker whose consumer has silently stopped should report not-ready.

**Principle.** *Every deployable exposes liveness, readiness, and startup endpoints. Liveness is dependency-free; readiness is dependency-aware with timeouts; startup separates initialisation from steady-state health.*

## 22.6 SaaS reliability engineering standard

```
For every external call:
  □ Explicit timeout, derived from the caller's latency budget
  □ Transient-vs-permanent error classification
  □ Bounded retry with exponential backoff and jitter (transient only)
  □ Circuit breaker
  □ Bulkhead limiting concurrency
  □ A defined degradation behaviour when it is unavailable

For every process:
  □ Dependencies verified before accepting traffic
  □ Liveness / readiness / startup endpoints
  □ Graceful shutdown, reachable through the container entrypoint
  □ Unhandled errors exit non-zero after flushing telemetry
  □ Bounded memory: no unbounded cache, buffer, or in-flight collection
  □ Resource limits declared; the system can enforce them

For every queue and job:
  □ Bounded prefetch
  □ Dead-letter queue with monitoring
  □ Idempotent handler
  □ Bounded retries with a terminal state
  □ Oldest-pending-age metric

For every stateful store:
  □ Automated backups with a TESTED restore procedure
  □ Documented RPO and RTO
  □ Connection limits understood and enforced
  □ Failover behaviour tested, not assumed

For the system:
  □ Documented, rehearsed degradation modes per dependency
  □ No single point of failure without a documented, accepted recovery plan
  □ Load shedding before collapse: reject early rather than queue unboundedly
  □ Regular failure drills — dependency removal in a staging environment
```

## 22.7 Degradation matrix — the artifact to produce for your own system

For each dependency, decide the behaviour **in advance**, write it down, and test it:

| Dependency down | Product must still | Product may lose | Mechanism |
|---|---|---|---|
| Primary database | Nothing — fail cleanly with a clear error | Everything | Fast failure, honest status page |
| Cache | Serve all reads and writes, slower | Latency only | Cache reads must be optional at the call site |
| Broker | Accept writes; commit intent | Async processing, until recovery | Outbox makes the commit sufficient |
| Payment provider | Serve the product; queue billing events | New subscriptions | Queue + reconcile on recovery |
| Email provider | Everything except delivery | Timely notifications | Outbox retains and retries |
| Object storage | Everything except upload and download | Media operations | Clear, specific error surfaced to the user |
| Realtime tier | Everything, via polling fallback | Live updates | The client must be able to poll |
| A specific integration | Everything else | That integration | Per-integration kill switch |

**The right-hand column is the design work.** A degradation mode you have not built is a total outage you have not predicted.
