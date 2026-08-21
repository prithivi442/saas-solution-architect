# SaaS Solution Architect

An agent skill that builds a production-grade, multi-tenant SaaS product end to
end — and explains every decision as it goes.

It is written for the case where the person with the product idea is **not an
engineer.** The skill decides every technical question from a complete
engineering standard and asks you only about your business: what the product
does, how it is priced, how sensitive the data is, which regulations apply.

## Install

```bash
npx skills add prithivi442/saas-solution-architect
```

The `skills` CLI detects which coding agent you use — 70+ are supported — and
installs to the right directory automatically.

## Use

Just describe what you want to build:

> "I want to build a SaaS product for scheduling shifts at restaurants. I want
> to charge per location per month."

The skill takes it from there, in ten phases, stopping for your approval at
each one.

You can also point it at an existing product:

> "Audit my SaaS against the production readiness checklist."

## What it covers

A complete standard, not a starter template. Every area a commercial SaaS has
to get right:

| | |
|---|---|
| **Architecture** | Modular monolith with separate request and background entrypoints, service boundaries, when to split and when not to |
| **The platform layer** | The ten modules to build before any feature: config, context, logging, metrics, tracing, data access, outbox, resilience, authorization, health |
| **Multi-tenancy** | Structural tenant scoping, Row-Level Security as a backstop, isolation verified at every layer |
| **Auth** | Four-layer model — authentication, tenant membership, role permission, and plan entitlement as a first-class concern |
| **Billing and money** | Exact-decimal money, append-only ledger, balance as a maintained projection, race-free debits, payment webhook lifecycle, reconciliation |
| **Data correctness** | Transaction boundaries, no I/O inside transactions, conditional writes, idempotency, recovery sweeps |
| **Async** | Transactional outbox, managed queues with dead-letter queues, poison-message handling, idempotent workers, durable scheduling |
| **Resilience** | Timeout, retry with jitter, circuit breaker, bulkhead, kill switches, graceful shutdown, degradation matrix |
| **Observability** | Structured logs, RED and USE metrics, distributed tracing, SLOs with burn-rate alerts, a runbook per alert |
| **Security** | Threat modelling, SSRF and egress control, IDOR, support impersonation, tamper-evident audit logs, SSO and SCIM, per-tenant encryption, supply chain, SOC 2 and GDPR readiness, incident response |
| **Delivery** | Immutable artifacts promoted by digest, blocking CI gates, automated backward-compatible migrations, zero-downtime deploys, infrastructure as code |
| **Testing** | Risk-weighted, with mandatory coverage on money, tenancy and concurrency |

Plus twelve decision trees for *when* to add complexity, a catalogue of
anti-patterns, and a pre-launch readiness checklist.

## Design

The skill is a small router over ~39 reference documents. Only what the current
phase needs is loaded, so the context stays cheap even though the standard
behind it is large.

```
skills/saas-solution-architect/
├── SKILL.md          the router, the operating rule, the non-negotiables
├── playbooks/        one per phase, in plain language
└── references/       the complete engineering standard
```

## The operating rule

> Decide every technical question from the standard. Escalate only business
> questions.

Datastore, isolation level, pagination style, retry policy, pool size — these
are decided from the references and then explained. Pricing tiers, compliance
targets, data residency, budget — these are yours.

## Licence

MIT
