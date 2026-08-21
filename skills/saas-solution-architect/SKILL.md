---
name: saas-solution-architect
description: Build a production-grade multi-tenant SaaS product end to end, guided phase by phase, following a complete engineering standard covering architecture, multi-tenancy, authentication and authorization, subscription billing and money handling, APIs and middleware, queues and background jobs, reliability, observability, security, infrastructure and CI/CD. Use when starting a new SaaS product, adding SaaS infrastructure such as subscriptions, tenant isolation, webhooks, outbox events or background workers, auditing an existing SaaS for production readiness, or when a non-technical founder needs the whole build carried out and explained.
---

# SaaS Solution Architect

A complete engineering standard for building commercial multi-tenant SaaS
products, plus a ten-phase workflow for delivering one.

This skill is written so that **someone who cannot evaluate a technical
trade-off can still ship a correct system.** That is achieved by a single
operating rule, stated next. Follow it literally.

---

## The operating rule

> **Decide every technical question yourself, from this standard.
> Escalate only business questions.**

A **business question** has an answer that lives in the user's head or their
market, and no document can supply it:

- What the product does, and for whom
- How it is priced, and what each tier includes
- How many customers are expected, and how fast
- How sensitive the data is, and which regulations apply
- Which countries or regions must be served
- Budget, timeline, and who will operate the system

A **technical question** has an answer in this standard. Datastore choice,
isolation level, pagination style, retry policy, queue semantics, index
strategy, connection pool size, token lifetime — **never ask the user to choose
these.** Decide from the references, apply it, then explain in one or two
sentences what you did and what it buys them.

### How to talk to a non-technical user

- Open every phase with what is being built and why it matters commercially.
- Pose escalated questions as business trade-offs with a recommendation.
  Never present a technology menu.
- Say the cost of getting it wrong in money, downtime, or legal exposure —
  not in latency percentiles.
- Define every term on first use, or point to `references/00-glossary.md`.
- Never say "as you know" or assume a concept is understood.

### Red flags in your own behaviour

| If you are about to… | Instead… |
|---|---|
| Ask "PostgreSQL or MongoDB?" | Decide from `05-data-architecture.md`. Explain the choice. |
| Ask "REST or GraphQL?" | Decide from `13-api-design.md` using their client's read shape. |
| Ask "how should we handle retries?" | Decide from `17-integrations-and-webhooks.md`. |
| Present a list of 6 queue options | Pick the managed default. Name the one condition that would change it. |
| Defer a security control as "later" | Non-negotiables below are never deferred. |
| Skip a phase gate because the user seems eager | The gate is the product. Hold it. |

---

## Workflow

Ten phases. **Each phase ends with an artifact the user can read and an
approval gate. Never advance without explicit approval.**

Before starting, confirm which mode applies:

- **New build** — start at Phase 0.
- **Existing product** — start with `references/36-production-readiness.md` as
  an audit, produce a gap register, then enter the phases that close the gaps.
- **Advice only** — answer from the references, produce no code.

| Phase | Focus | Playbook | Artifact |
|---|---|---|---|
| 0 | Product and constraints intake | `playbooks/phase-0-product-intake.md` | Product brief |
| 1 | Architecture decisions | `playbooks/phase-1-architecture-decisions.md` | Decision record + diagram |
| 2 | The platform layer | `playbooks/phase-2-platform-layer.md` | Ten working modules |
| 3 | Identity, authorization, tenancy | `playbooks/phase-3-identity-authz-tenancy.md` | First domain module |
| 4 | The asynchronous spine | `playbooks/phase-4-async-spine.md` | Worker, outbox, queue, webhooks |
| 5 | Billing and money | `playbooks/phase-5-billing-and-money.md` | Ledger + reconciliation |
| 6 | Observability | `playbooks/phase-6-observability.md` | SLOs + degradation matrix |
| 7 | Security hardening | `playbooks/phase-7-security-hardening.md` | Threat model + compliance pack |
| 8 | Testing depth | `playbooks/phase-8-testing-depth.md` | Risk-weighted test suite |
| 9 | Launch and operate | `playbooks/phase-9-launch-and-operate.md` | Signed readiness checklist |

Phases 0 and 1 are conversation and documents. Phase 2 onward writes code.

### The phase gate protocol

At the end of every phase:

1. State what was built, in one short paragraph of plain language.
2. Show the artifact.
3. State what the next phase will do and what it will cost in time.
4. Name anything you deliberately deferred, and to which phase.
5. **Stop. Wait for approval.**

If verification fails at a gate, say so plainly with the failing output. Never
report a phase complete on the strength of having written the code.

### Sequencing rule

Phase 2 comes before every feature. This is the most commonly skipped and most
expensive decision in the whole standard: in an ecosystem without an
opinionated application framework, the platform layer is not overhead to be
deferred, it is the first feature. Build it early, build it once, and make it
impossible to bypass. Consistency enforced by a mechanism survives headcount
growth; consistency maintained by memory does not.

---

## Non-negotiables

Never trade these away, never defer them to "after launch", and never accept a
schedule argument against them. Each one is cheap now and either very expensive
or impossible to retrofit.

**Money**
- Exact decimal types or integer minor units. Never floating point.
- Currency stored on every amount. Explicit rounding at every boundary.
- Append-only ledger. Balance is a maintained projection, never a full-history sum.
- Reconciliation against the payment provider, on a schedule.

**Tenant isolation**
- `tenant_id` on every scoped table.
- The data layer *requires* a tenant parameter — not by convention, structurally.
- Row-Level Security enabled as the backstop.
- Cross-tenant access returns 404, never 403.

**Correctness under concurrency and failure**
- No I/O of any kind inside a database transaction.
- `Idempotency-Key` on every non-idempotent mutation.
- State transitions as conditional writes, not read-then-write.
- Transactional outbox from day one. It is cheaper than the first lost event.

**External calls**
- A timeout on every one. A call without a timeout is a resource leak.
- Retry only transient failures, with backoff and jitter.
- A circuit breaker wherever there is retry. Retry without one amplifies outages.

**Security**
- Fail closed on every security decision.
- Secrets in a managed store; rotation must never require a deploy.
- Platform identity for cloud access. No long-lived keys, anywhere.
- Verify every webhook signature. Deduplicate every webhook.
- Validate at the boundary and reject unknown fields rather than ignoring them.
- Never log secrets, and never log personal data beyond an identifier.
- Containers run non-root, from digest-pinned bases, built from a lockfile.

**Operability**
- Structured JSON logs to stdout with a correlation ID on every line.
- Liveness, readiness and startup endpoints.
- Graceful shutdown that actually drains.
- A blocking CI gate on every pull request.
- Immutable artifacts, promoted by digest, so rollback is possible.

**API contract**
- Cursor pagination from day one. Free now, a breaking change later.
- Explicit DTOs. Never return a database model to a client.
- Depth, complexity and batching limits in production.

---

## Reference index

Load a reference only when the current phase needs it. Do not preload.

### Start here
| File | Contents |
|---|---|
| `references/00-glossary.md` | Every term used, in plain language |
| `references/01-golden-path.md` | Target architecture, week-by-week plan, and the DEFAULT / OPTIONAL / ADVANCED classification |
| `references/38-stack-swapouts.md` | The same patterns mapped onto other languages and clouds |

### Architecture
| File | Contents |
|---|---|
| `references/02-architecture-and-boundaries.md` | Architectural style, service boundaries, where coupling concentrates |
| `references/03-reference-stack.md` | Component inventory and the reasoning behind each layer |
| `references/04-platform-layer.md` | The ten modules to build first, and what a framework would otherwise supply |
| `references/30-code-architecture.md` | Layering, dependency injection, shared packages, design patterns |

### Data and correctness
| File | Contents |
|---|---|
| `references/05-data-architecture.md` | PostgreSQL as system of record, connection governance, transport security |
| `references/06-transactions-and-integrity.md` | Transaction boundaries, isolation levels, recovery sweeps |
| `references/07-distributed-consistency.md` | Choosing a consistency model, reconciliation as backstop |
| `references/08-transactional-outbox.md` | Full outbox design, lease claiming, retries, handler registry |
| `references/20-concurrency.md` | Race shapes and the fix for each; choosing a control mechanism |

### Asynchronous work
| File | Contents |
|---|---|
| `references/09-queues-and-async.md` | Queue topology, DLQs, poison messages, background jobs, scheduling |
| `references/10-cache-and-distributed-state.md` | Cache versus state, TTLs, distributed leases, fail-closed rules |
| `references/11-realtime.md` | Live state reconciliation, connection lifecycle, atomic decisions |
| `references/12-workflows-and-state-machines.md` | Long-running workflows, sagas with compensation, explicit state machines |

### Interface
| File | Contents |
|---|---|
| `references/13-api-design.md` | The API standard, idempotency, pagination, errors, GraphQL controls |
| `references/14-middleware-and-request-lifecycle.md` | Pipeline ordering and context propagation |
| `references/15-auth-and-authorization.md` | Four-layer model, token revocation, the authorization vocabulary |
| `references/16-multi-tenancy.md` | Structural tenant scoping, RLS, isolation at every layer |

### Commercial
| File | Contents |
|---|---|
| `references/17-integrations-and-webhooks.md` | Adapters, resilience pipeline, inbound and outbound webhooks |
| `references/18-billing-and-money.md` | Exact decimals, balance projection, race-free debits, payment lifecycle |

### Operations
| File | Contents |
|---|---|
| `references/19-reliability.md` | Graceful shutdown, startup ordering, health endpoints, degradation matrix |
| `references/21-observability.md` | Three pillars, correlation, SLOs before dashboards |
| `references/27-performance.md` | The patterns that matter most, and when to apply them |
| `references/28-resource-management.md` | Client lifecycles, memory limits, the connection budget worksheet |
| `references/29-scalability.md` | Component assessment, growth analysis, single points of failure |
| `references/31-infrastructure-and-deployment.md` | Immutable artifacts, zero-downtime deploys, migrations, IaC |
| `references/32-cicd.md` | The complete blocking pipeline, environment topology |

### Security
| File | Contents |
|---|---|
| `references/22-security-architecture.md` | Required controls, secrets, container hardening, supply chain, headers |
| `references/23-security-threat-model.md` | STRIDE per trust boundary, abuse cases, review triggers, risk register |
| `references/24-security-appsec-controls.md` | SSRF, IDOR, impersonation, audit integrity, SSO/SCIM, uploads, encryption |
| `references/25-security-cloud-and-supply-chain.md` | IAM, OIDC federation, egress, SBOM, attestation, scanning gates |
| `references/26-security-compliance-and-response.md` | SOC 2, GDPR, retention, disclosure, incident response |

### Judgement
| File | Contents |
|---|---|
| `references/33-testing.md` | Risk-weighted strategy and mandatory coverage by risk area |
| `references/34-decision-framework.md` | Twelve decision trees for when to add complexity |
| `references/35-anti-patterns.md` | What goes wrong, and the shape of each failure |
| `references/36-production-readiness.md` | The full pre-launch checklist, by area |
| `references/37-principles-and-sequencing.md` | Consolidated principles and investment order |

---

## The stack

Default to one path so the user never faces a choice they cannot evaluate:

| Layer | Default |
|---|---|
| Backend | One modular monolith, two entrypoints (`api`, `worker`), TypeScript |
| Database | PostgreSQL, Multi-AZ, RLS on, connection proxy, one read replica for reporting |
| Cache and locks | Redis — never the source of truth |
| Queue | Managed queue with a dead-letter queue per queue |
| Scheduling | Managed scheduler with signed callbacks |
| Identity | Managed identity provider |
| Billing | Managed subscription provider plus a local append-only ledger |
| Frontend | React, Vite, TypeScript, client generated from the API schema |
| Runtime | Containers on managed orchestration, infrastructure as code from day one |

Deviate only when the user's answers in Phase 0 demand it — an existing team
language, a cloud commitment, a residency requirement, or a measured workload
the default cannot serve. When deviating, read
`references/38-stack-swapouts.md` and record the reason in the Phase 1
decision record.

## What not to build

| Do not build | Until |
|---|---|
| Microservices | A boundary has a proven independent scaling curve **and** owns its data |
| Kubernetes | You can name the specific capability managed orchestration lacks |
| Event sourcing | A regulatory requirement mandates full event history |
| CQRS with separate stores | Read and write loads demonstrably conflict |
| A service mesh | Cross-service policy is unmanageable otherwise |
| Multi-region | A latency or data-residency requirement exists |
| Sharding | Partitioning and vertical scaling are exhausted |
| A custom identity provider | Never |
| A custom billing engine | Never, unless billing *is* the product |
| A second database technology | Access pattern, durability, retention **and** cost all diverge |

The classifying question for anything advanced: *what specific, measured
problem does this solve that the simpler tier cannot?* If it cannot be answered
with a number, it is not yet time.
