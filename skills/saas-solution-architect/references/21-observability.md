# Observability

## 24.1 The reference implementation

| Capability | Implementation |
|---|---|
| Structured logging | Winston, JSON format, console transport, service name attached |
| Log aggregation | Published to the broker, consumed into a document store |
| Error tracking | One system, with severity-based filtering; never two in parallel |
| Correlation identifier | Generated per request; threaded through function parameters |
| Request context | `AsyncLocalStorage`, established on the HTTP path |
| Audit logging | Dedicated audit log level and message type |
| Slow-query threshold | A configuration value exists |
| GraphQL tracing plugin | Inline trace plugin enabled |
| Health endpoint | One service |
| **Metrics** | **Required — commonly missing** |
| **Distributed tracing** | **Required — commonly missing** |
| **Log levels tuned per environment** | Level fixed at `debug` |
| **SLOs / error budgets** | **Required — commonly missing** |

## 24.2 The incident question, answered honestly

> *If a production incident occurs, how effectively can engineers trace the problem across services?*

With logs alone, the achievable workflow is: find an error in Sentry, search logs for the correlation identifier if it was threaded correctly through every hop, and read forward. That works for a **known** error with an intact correlation chain.

It does not answer the questions that dominate real incidents:

| Question | Answerable today | What is needed |
|---|---|---|
| Is the system slow, and where? | No | Latency histograms per endpoint and per dependency |
| Which dependency is degraded? | No | Per-dependency latency and error rate |
| Is the queue backing up? | No | Queue depth and oldest-pending age |
| Is this one tenant or everyone? | Partly, by log grep | Metrics bucketed by tenant tier |
| Where did the request spend its time? | No | Distributed traces with spans |
| Is this worse than an hour ago? | No | Time-series metrics |
| Are we within our error budget? | No | SLOs |
| Which replica is unhealthy? | No | Per-instance metrics and readiness |

**Principle.** *Logs tell you what happened to one request. Metrics tell you what is happening to the system. Traces tell you where the time went. All three are required; none substitutes for another.*

## 24.3 Reference flow: the three pillars, minimally

### Structured logs — to stdout, always

```ts
// One logger; correlation and tenant injected from the ambient context.
// Never passed as parameters; never omitted.
logger.info('subscription.activated', {
  subscriptionId, planId,
  // correlationId, tenantId, userId, service, version, environment
  // are added automatically by a formatter reading AsyncLocalStorage
});
```

```
□ JSON to stdout. Nothing else. Collection is a platform concern.
□ Level driven by configuration; INFO in production, never DEBUG
  (debug logging in production is a cost centre and a PII risk)
□ Every line carries: timestamp, level, service, version, environment,
  correlationId, tenantId, event name
□ Event names are stable identifiers (`subscription.activated`), not prose —
  so they can be counted, aggregated, and alerted on
□ Never log: tokens, passwords, card data, full request bodies,
  provider credentials, personal data beyond an identifier
□ Sample high-volume success logs; never sample errors
□ Logging is synchronous and local. Never an awaited network call.
```

### Metrics — the four that matter most

```ts
// RED for every service surface
metrics.histogram('http.request.duration_ms', ms,
                  { route, method, status_class });
metrics.counter  ('http.request.total',        1, { route, method, status });

// USE for every dependency
metrics.histogram('dependency.duration_ms', ms, { dependency, operation });
metrics.counter  ('dependency.errors',       1, { dependency, error_class });
metrics.gauge    ('dependency.breaker_open', v, { dependency });

// Saturation — the leading indicators
metrics.gauge('db.pool.in_use',        n);
metrics.gauge('db.pool.waiting',       n);   // ← waiting > 0 is the first real warning
metrics.gauge('queue.depth',           n, { queue });
metrics.gauge('queue.oldest_age_ms',   n, { queue });   // ← better than depth alone
metrics.gauge('workflow.count',        n, { state });   // ← entities stuck in a state

// Business signals — the ones that detect what technical metrics miss
metrics.counter('billing.charge.total',        1, { outcome });
metrics.counter('reconciliation.divergence',   1, { kind });
metrics.counter('provider.metered_call',       1, { provider, tenant_tier });
metrics.counter('authz.cross_tenant_denied',   1);
```

**Label cardinality is the discipline that keeps metrics affordable:** bucket tenants by tier or size class, never by raw identifier. A per-tenant label on a metric with 10,000 tenants creates 10,000 time series per metric.

### Traces — one span per boundary

```
Trace: POST /graphql · createSubscription        ─────────────────── 842ms
├─ authenticate                                     12ms
│  └─ cache: revocation check                        1ms
├─ authorize (tenant · role · plan)                 31ms
│  └─ db: 3 queries                                 28ms
├─ payment provider: create subscription           612ms   ← the answer
├─ db: transaction (subscription + outbox)          38ms
└─ publish event                                     9ms
```

Trace context must propagate across **every** boundary: HTTP headers (W3C `traceparent`), queue message headers, and scheduled-callback invocations. A trace that stops at a queue boundary hides the half of the system where the slow work happens.

**On AWS**, the pragmatic path is the OpenTelemetry SDK exporting to AWS Distro for OpenTelemetry, with traces in X-Ray and metrics in CloudWatch or Managed Prometheus. Auto-instrumentation for Express, GraphQL, Postgres, Redis, and the AWS SDK provides most of the value with configuration rather than code.

## 24.4 Reference flow: correlation from the context, not the signature

Threading a correlation identifier through every function signature works only where every call site remembers, and clutters every signature it touches.

```ts
✗ async function chargeTenant(logId: string, tenantId: number, amount: Decimal) { … }

✓ // The formatter reads ambient context, so every log line is correlated
  // without any function needing to know about correlation at all.
  const formatter = winston.format((info) => {
    const c = requestContext.getStore();
    return { ...info, correlationId: c?.correlationId, tenantId: c?.tenantId,
                      userId: c?.userId, service: SERVICE_NAME, version: BUILD_VERSION };
  });
```

The context must be installed at **every** entry point — HTTP, queue consumer, cron, scheduled callback — and the accessor must degrade rather than throw when no context exists (`14-middleware-and-request-lifecycle.md` §3).

## 24.5 Reference flow: SLOs before dashboards

Dashboards proliferate; SLOs focus. Define a small number, from the user's perspective:

```
Availability   99.9% of API requests succeed        (43m error budget / month)
Latency        99% of API requests < 500ms
Freshness      99% of async events processed < 60s
Correctness    100% of reconciliation runs report zero divergence

Alert on BURN RATE, not on instantaneous values:
  fast burn: 2% of the monthly budget in 1 hour   → page
  slow burn: 5% of the monthly budget in 6 hours  → ticket

Everything else is a dashboard, not an alert.
```

**Principle.** *Alert on symptoms the user experiences, at a burn rate that threatens the budget. Alerting on causes produces noise; alerting on instantaneous thresholds produces false pages at 3 a.m.*

## 24.6 Observability standard

```
□ Structured JSON logs to stdout; collection is a platform concern
□ Log level from configuration; INFO in production
□ Correlation ID generated at every entry point, propagated to every hop,
  injected into every log line from ambient context, returned to the client
□ RED metrics per endpoint; USE metrics per dependency; saturation gauges
  for pools, queues, and workflow states
□ Business metrics for money, reconciliation, metered spend, and authz denials
□ Distributed tracing across HTTP and queue boundaries
□ Liveness / readiness / startup endpoints on every deployable
□ Error tracking with grouping, release tagging, and source maps
□ Audit log for every security- and money-relevant action, retained separately
  with a longer retention and stricter access than application logs
□ SLOs with error budgets; alerts on burn rate
□ Never log secrets or personal data beyond an identifier
□ One error-tracking system, not two
□ A runbook per alert: what it means, how to confirm, what to do
```
