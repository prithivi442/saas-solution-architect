# Stack swap-outs

The default stack is TypeScript, PostgreSQL, Redis, a managed queue, containers
on managed orchestration, and a React front end. Every pattern in this standard
is independent of that choice — what changes is how much you build versus
inherit.

Read this only when the product brief gives a reason to deviate: an existing
team language, a cloud commitment, a residency requirement, or a measured
workload the default cannot serve. Preference and novelty are not reasons.

---

## 1 What the language choice actually changes

Nothing about the architecture. It changes which of the platform layer's ten
modules you write yourself and which arrive with the framework.

| Platform module | TypeScript | Python / Django | Go | Java / Spring | .NET | Ruby / Rails |
|---|---|---|---|---|---|---|
| Config, validated at boot | build | partly inherited | build | **inherited** | **inherited** | partly inherited |
| Ambient context | build (`AsyncLocalStorage`) | build (`contextvars`) | build (`context.Context`, explicit) | **inherited** | **inherited** (`AsyncLocal`) | build (`ActiveSupport::CurrentAttributes`) |
| Structured logging | build | partly inherited | build | **inherited** | **inherited** | partly inherited |
| Metrics | build | build | build | **inherited** (Micrometer) | **inherited** | build |
| Tracing | build (OTel SDK) | build (OTel SDK) | build (OTel SDK) | **inherited** (OTel starter) | **inherited** | build |
| Declarative transactions | build (12 lines) | **inherited** (`atomic`) | build (explicit `tx`) | **inherited** (`@Transactional`) | **inherited** | **inherited** |
| Tenant-scoped repositories | build | build (managers) | build | **inherited** (`@TenantId`) + RLS | build | build (`default_scope`, carefully) |
| Outbox | build | build | build | build | build | build |
| Resilience pipeline | build | build | build | **inherited** (Resilience4j) | **inherited** (Polly) | build |
| Declared authorization | build + CI check | build + CI check | build + CI check | **inherited** (`@PreAuthorize`) + CI check | **inherited** | build + CI check |
| Health endpoints | build | build | build | **inherited** (Actuator) | **inherited** | build |

**The inversion worth understanding:** a framework-rich ecosystem hands you the
platform layer and you spend your time on the domain. A framework-light
ecosystem hands you nothing and you spend two weeks building the platform layer
first. Neither is wrong. What is wrong is choosing the framework-light option
and then *skipping* the two weeks — which is the single most common way a SaaS
codebase becomes unmaintainable.

**The outbox is the one row nobody inherits.** In every ecosystem, it is
yours to build.

---

## 2 Per-language notes

### Python / Django

A good choice when the team knows it, or when the product has a data-science or
machine-learning component.

| Concern | Guidance |
|---|---|
| Transactions | `transaction.atomic()` gives you declarative demarcation; use it, do not hand-roll |
| Tenancy | A custom model manager requiring tenant, plus RLS. Avoid implicit thread-local tenancy — it breaks under async and in Celery workers |
| Workers | Celery, or the managed queue with a thin consumer. Prefer the managed queue: Celery's broker becomes another thing to operate |
| Scheduling | Never Celery Beat in a replicated deployment without a lock. Prefer the managed scheduler |
| Money | `Decimal` and `DecimalField` — never `FloatField` |
| API | Django REST Framework with schema generation; cursor pagination is built in and must be used from day one |
| Migrations | Strong. Enforce backward-compatibility review; Django will happily generate a destructive migration |
| Concurrency | `select_for_update()`, and `F()` expressions for conditional writes |
| Watch out for | The ORM's lazy loading producing N+1 queries under serialisation; `.only()`/`select_related` deliberately |

### Go

Best as a second language for a measured CPU-bound path — money computation,
usage aggregation, media processing — not usually as the whole product.

| Concern | Guidance |
|---|---|
| Context | `context.Context` threaded explicitly. More verbose, and it cannot be forgotten silently |
| Transactions | Explicit `*sql.Tx`. Wrap in a helper so the boundary is one call |
| Tenancy | Required tenant parameter is natural here; RLS still the backstop |
| Money | `shopspring/decimal` or integer minor units. Go's lack of operator overloading makes float misuse more visible, which helps |
| Resilience | Build it; the ecosystem has libraries but no standard |
| Watch out for | Accepting a second language means a second full complement of tests, migrations, observability and deployment tooling. A polyglot service with half the platform support costs more than a monoglot one |

### Java / Spring, or Kotlin

The framework-rich end. You inherit most of the platform layer.

| Concern | Guidance |
|---|---|
| What you inherit | Dependency injection, `@Transactional`, method security, Actuator health and metrics, Micrometer, tracing starters, validated configuration, Flyway migrations, graceful shutdown, test slices |
| Tenancy | Hibernate tenant filters plus RLS. Verify the filter applies to native queries too — it often does not |
| What you still build | The outbox, the entitlement layer, reconciliation, and the CI check asserting every operation declares a policy |
| Watch out for | Slower startup affects rolling deploys and scale-out responsiveness; size the readiness probe accordingly. Also: inheriting `@Transactional` does not mean inheriting *correct* transaction boundaries — I/O inside a transaction is just as wrong here |

### .NET

Similar inheritance profile to Spring. Polly for resilience, built-in health
checks, `AsyncLocal` for context, EF Core migrations. Same caveat: you still
build the outbox, entitlement and reconciliation.

### Ruby / Rails

Fast to build in, strong migrations, and `default_scope` is a tempting tenancy
mechanism that leaks — it is bypassed by `unscoped` and by joins more often than
teams expect. Use an explicit tenant-required query object plus RLS.

---

## 3 Cloud swap-outs

The architecture is identical. Only the service names change.

| Capability | AWS | Google Cloud | Azure |
|---|---|---|---|
| Container orchestration | ECS on Fargate | Cloud Run | Container Apps |
| Managed PostgreSQL | RDS / Aurora | Cloud SQL / AlloyDB | Database for PostgreSQL |
| Connection pooling proxy | RDS Proxy | built into Cloud SQL connectors | PgBouncer sidecar |
| Cache | ElastiCache | Memorystore | Azure Cache for Redis |
| Queue + DLQ | SQS | Pub/Sub with dead-letter topic | Service Bus |
| Object storage | S3 | Cloud Storage | Blob Storage |
| Secrets | Secrets Manager / Parameter Store | Secret Manager | Key Vault |
| Key management | KMS | Cloud KMS | Key Vault Managed HSM |
| Identity provider | Cognito | Identity Platform | Entra External ID |
| Durable scheduler | EventBridge Scheduler | Cloud Scheduler | Logic Apps / Functions timer |
| Workload identity | IAM roles for tasks | Workload Identity | Managed Identity |
| Audit trail | CloudTrail | Cloud Audit Logs | Activity Log |
| Threat detection | GuardDuty | Security Command Center | Defender for Cloud |
| Container registry | ECR | Artifact Registry | ACR |
| Edge / WAF | CloudFront + WAF | Cloud Armor | Front Door + WAF |
| CI federation | OIDC → IAM role | Workload Identity Federation | OIDC → Managed Identity |

**Invariant across all three, and non-negotiable in each:** no long-lived
credentials, federated identity from CI, separate accounts or projects per
environment, control-plane audit logs delivered somewhere production cannot
delete, and encryption asserted at startup rather than assumed.

### Choosing a cloud

Almost always: whichever one the team already knows, or where the credits or
commitment are. The differences that matter to a new SaaS are small compared to
the cost of operating a cloud nobody on the team has used.

Genuine tie-breakers:

- **Residency in a specific country** — check the region list first, and check
  it for *every* service you need, including logging and error tracking
- **An existing enterprise agreement** — the discount is usually larger than any
  architectural difference
- **A specific managed service you actually need** — name it, and confirm the
  alternatives genuinely cannot serve

---

## 4 Front-end swap-outs

| Choice | When |
|---|---|
| **React + Vite** (default) | Largest hiring pool, best generated-client tooling |
| Vue, Svelte | Team preference; equivalent outcomes |
| Next.js or similar meta-framework | You need server rendering for SEO or first-paint. Accept a second runtime to operate |
| Server-rendered templates | The product is form-driven and low-interactivity. Genuinely simpler; do not rule it out reflexively |
| Mobile native or cross-platform | The product is mobile-first. Then the API contract matters more, not less: mobile clients cannot be force-upgraded, so versioning by evolution becomes mandatory |

**The one requirement that does not vary:** the client's API types are
**generated from the API schema** and the build fails when they diverge. It is
the only cross-boundary contract that fails a build rather than a runtime, and
it is worth choosing a stack that supports it.

---

## 5 What never swaps out

Whatever the language, cloud or front end:

- Exact-decimal money and an append-only ledger
- Structural tenant scoping plus a database-level backstop
- A transactional outbox
- Idempotency keys on non-idempotent mutations
- Timeout, retry with jitter, and a circuit breaker on every external call
- Fail closed on every security decision
- No long-lived cloud credentials
- Structured logs with a correlation id, plus metrics and traces
- Health endpoints, graceful shutdown
- A blocking CI gate and immutable artifacts promoted by digest
- Signature-verified, deduplicated webhooks
- Cursor pagination and explicit DTOs

If a stack makes any of these materially harder, that is an argument against the
stack — not against the requirement.
