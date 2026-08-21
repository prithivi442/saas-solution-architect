# Glossary

Every term this standard uses, in plain language. When a non-technical user asks
what something means, answer from here — and prefer the "why you care" column
over the definition.

---

## The shape of the system

**SaaS** — Software as a Service. Customers pay a recurring fee to use software
you host and operate, rather than buying a copy to run themselves.

**Tenant** — one customer organisation. If a restaurant chain signs up, the
chain is the tenant and its staff are users within it. Almost every rule in
this standard about isolation is about keeping one tenant's data away from
another's.

**Multi-tenancy** — one running system serving many tenants. Cheaper than a
separate deployment per customer, which is why it is the default — and the
reason isolation has to be engineered rather than assumed.

**Monolith** — the whole application as one deployable unit.

**Modular monolith** — one deployable unit, internally divided into modules with
enforced boundaries. The recommended default: the simplicity of one deployment
with the internal structure to split later if genuinely needed.

**Microservices** — many independently deployed services. Buys independent
scaling and failure isolation; costs a distributed system, which is a large and
permanent operational tax.

**Entrypoint** — a way of starting the same codebase for a different job. This
standard uses two: `api` serves customer requests, `worker` does background
work. Same code, different process, separate scaling.

**Deployable** — a thing you ship and run as its own process.

**Replica** — one running copy of a deployable. Several replicas share the load
and survive one dying.

**Stateless** — a service holding nothing important in its own memory, so any
replica can serve any request and losing one loses nothing.

**Platform layer** — the shared internal foundation every feature is built on:
configuration, logging, database access, security checks. This standard insists
it is built before any feature.

---

## Data

**PostgreSQL** — a relational database. The default here for anything that
matters, because it enforces relationships and offers real transactions.

**Relational** — data organised as tables that reference each other.

**Transaction** — a group of database changes that either all happen or none
do. What stops a crash halfway through leaving half-finished state.

**Isolation level** — how much two simultaneous transactions can see of each
other's work. Choosing wrongly is a source of bugs that only appear under load.

**Migration** — a versioned, repeatable change to the database structure.

**Backward-compatible migration** — a schema change the *previous* version of
the application still works against. Required, because it is what makes rolling
back safe.

**Index** — a lookup structure making certain queries fast.

**Row-Level Security (RLS)** — the database itself refusing to return rows that
don't belong to the current tenant. Used here as a safety net beneath the
application's own filtering, so a forgotten filter returns nothing instead of
someone else's data.

**Read replica** — a copy of the database that can be read but not written.
Used to keep heavy reporting queries away from live traffic.

**Connection pool** — a reusable set of database connections. Databases accept
a limited number, so pool size multiplied by replica count is a real ceiling.

**Sharding** — splitting data across multiple databases. Powerful, complex, and
close to a last resort.

**Partitioning** — splitting one large table into pieces, usually by date, so
maintenance stays manageable.

**Projection** — a stored, maintained answer to a question that would otherwise
be expensive to compute. A customer's balance is a projection of their ledger.

**Append-only** — records are added, never changed or deleted. How financial
history is kept trustworthy.

**Ledger** — the permanent record of every financial movement. The source of
truth for money.

**Exact decimal** — a number type that stores `0.10` as exactly `0.10`.
Required for money, because the usual number types don't.

**Floating point** — the usual computer number type. Cannot represent most
decimal fractions exactly. Banned from money in this standard.

**DTO** — Data Transfer Object. A deliberately shaped object returned to a
client, instead of handing over a raw database row. Stops internal structure
leaking into your public contract.

---

## Requests and interfaces

**API** — the interface other software uses to talk to your product.

**REST** — an API style organised around resources and standard HTTP verbs.

**GraphQL** — an API style where the client specifies exactly which fields it
wants. Useful when one screen needs data from many places; brings its own
required safety controls.

**Endpoint** — a single callable address in your API.

**Middleware** — code that runs on every request before the handler: logging,
authentication, validation. Order matters.

**Handler** — the code that actually does the work for one endpoint.

**Idempotency** — doing something twice has the same effect as doing it once.
Essential, because networks retry and you cannot charge a customer twice.

**Idempotency-Key** — a value the client sends so the server can recognise a
retry of the same request and not repeat the effect.

**Cursor pagination** — returning results in pages using a pointer to the last
item, rather than "page 5". Stable when data changes underneath, and free to
adopt now but a breaking change to add later.

**Correlation ID** — a unique value attached to one request and carried through
every log line and downstream call, so one customer's problem can be traced
through the whole system.

**Rate limiting** — capping how often someone can call something.

**Backpressure** — a system slowing down its intake rather than accepting more
work than it can handle and collapsing.

---

## Security and access

**Authentication** — establishing who someone is.

**Authorization** — deciding what they may do. A different question, and this
standard keeps it in four separate layers.

**Entitlement** — whether the customer's *plan* includes a capability. Kept
separate from permissions so pricing can change without a software release.

**Permission** — whether this person's *role* allows an action.

**Token** — a signed credential proving an identity, sent with each request.

**Refresh rotation** — issuing a new long-lived credential each time it's used,
so a stolen one becomes useless quickly.

**Revocation denylist** — a list of tokens to reject before they expire, so
logout and account suspension take effect immediately.

**Fail closed** — when a security check cannot be completed, deny. The
alternative, failing open, turns an outage into a breach.

**Least privilege** — every component gets only the access it needs.

**Secrets manager** — a dedicated service holding passwords and keys, so they
can be rotated without redeploying.

**Envelope encryption** — encrypting data with a key that is itself encrypted
by a master key. Enables per-tenant keys, and therefore real per-tenant
deletion.

**Crypto-shredding** — destroying a tenant's encryption key so their data
becomes permanently unreadable. A practical way to prove deletion.

**IDOR** — Insecure Direct Object Reference. Changing an identifier in a
request and getting back someone else's record. One of the most common real
SaaS vulnerabilities.

**SSRF** — Server-Side Request Forgery. Tricking your server into making
requests on the attacker's behalf, often to internal addresses they cannot
reach. A live risk wherever customers supply URLs, such as outbound webhooks.

**STRIDE** — a checklist for finding threats: spoofing, tampering,
repudiation, information disclosure, denial of service, elevation of privilege.

**Threat model** — a written analysis of how the system could be attacked and
what is being done about each path.

**SBOM** — Software Bill of Materials. A list of every third-party component in
a build, so you can answer "are we affected?" when a vulnerability is announced.

**Supply chain attack** — compromising you through a dependency or build tool
rather than attacking you directly.

**SSO / SAML / OIDC** — letting customers sign in with their own company
identity system. Usually required to sell to larger companies.

**SCIM** — automatic user provisioning and, more importantly, automatic
*de*-provisioning when someone leaves the customer's company.

**MFA** — multi-factor authentication. A second proof beyond a password.

**Audit log** — a tamper-evident record of who did what, when. Evidence, so it
must be append-only.

**Impersonation** — support staff acting as a customer to help them. Useful and
dangerous; needs its own audit trail and limits.

**CSP** — Content Security Policy. A browser instruction restricting what a
page may load, limiting the damage of injected scripts.

**CSRF** — Cross-Site Request Forgery. Tricking a logged-in user's browser into
performing an action they didn't intend.

**Penetration test** — a paid expert attacking your system with permission.

**SOC 2 / ISO 27001** — security certifications enterprise buyers ask for.

**GDPR** — European data protection law. Grants individuals rights over their
data and sets a 72-hour breach notification clock.

**DSR** — Data Subject Request. Someone exercising a right: see my data, export
it, correct it, delete it. The system has to be able to actually do these.

**Data residency** — a requirement that data physically stays in a country or
region.

---

## Asynchronous work

**Synchronous** — the caller waits for the answer.

**Asynchronous** — the work is accepted now and done shortly after.

**Queue** — a durable list of work waiting to be done. Survives restarts, which
is the point.

**Dead-letter queue (DLQ)** — where messages go after repeatedly failing, so
they can be inspected instead of blocking everything behind them.

**Poison message** — a message that fails every time. Must be removed, or it
blocks the queue forever.

**At-least-once delivery** — the guarantee real queues actually give: a message
may arrive more than once. Why every handler must be idempotent.

**Transactional outbox** — writing an event into the same database transaction
as the change that caused it, then delivering it separately. The reliable way to
never lose an event and never send one for a change that rolled back.

**Sweeper** — a periodic job that finds work left stranded by a crash and
recovers it.

**Lease** — a time-limited claim on a piece of work, so two workers don't do
the same job and a crashed worker's claim expires.

**Saga** — a multi-step operation across systems that cannot share a single
transaction, where each step knows how to undo itself.

**Compensation** — the undo step of a saga.

**Reconciliation** — periodically comparing your records against another
system's and alerting on disagreement. Required for money.

**Eventual consistency** — two systems agreeing shortly, not instantly.
Acceptable in many places, but visible to users if applied carelessly.

**State machine** — an explicit list of the states something can be in and the
allowed moves between them, so illegal transitions become impossible rather
than merely unlikely.

**Webhook** — an HTTP call another system makes to you when something happens.
Untrusted input, so it must be signature-verified and deduplicated.

---

## Reliability and operations

**Cache** — a fast copy of data that is expensive to fetch. Never the source of
truth.

**TTL** — time to live. How long a cached item stays valid. Cached data without
one is a leak.

**Redis** — the usual cache and shared-counter store. Fast, but its contents
must always be reconstructible.

**Timeout** — the maximum time to wait for something. A call without one can
hold a resource forever.

**Retry with backoff and jitter** — trying again after increasing, randomised
delays. The randomisation stops every replica retrying in unison.

**Circuit breaker** — after enough failures, stop calling a broken dependency
for a while. Without one, retries turn someone else's outage into yours.

**Bulkhead** — separate resource pools per dependency, so one slow thing cannot
consume all capacity.

**Kill switch** — a configuration flag disabling a feature or integration
without a deploy.

**Graceful shutdown** — finishing in-flight work before exiting. Required for
deploys with no dropped requests.

**Health endpoints** — liveness (restart me if this fails), readiness (send me
traffic), startup (I'm still booting). Three separate questions.

**Race condition** — two things happening at once producing a wrong result.
Common at real volume, not rare.

**Conditional write** — a change whose precondition is part of the write
itself, so two simultaneous attempts cannot both succeed.

**Optimistic locking** — detecting that someone else changed a record since you
read it, and rejecting your write.

**Observability** — being able to answer what is wrong and where, without
guessing.

**Structured logging** — logs as machine-readable records rather than prose, so
they can be searched and aggregated.

**Metrics** — numbers over time. Request rates, error rates, durations.

**RED** — Rate, Errors, Duration. What to measure per endpoint.

**USE** — Utilisation, Saturation, Errors. What to measure per resource.

**Cardinality** — how many distinct values a metric label can take. High
cardinality, such as a customer ID, will break a metrics system.

**Distributed tracing** — following one request across every service and queue
hop it touches.

**SLO** — Service Level Objective. A target for reliability, such as 99.9% of
requests succeeding.

**Error budget** — the amount of failure the SLO permits. Spending it is
allowed; spending it too fast is the alert.

**Burn rate** — how fast the error budget is being consumed. The right thing to
alert on.

**Runbook** — instructions for what to do when a specific alert fires.

**Degradation matrix** — a written table of what happens to the product when
each dependency fails.

**RPO / RTO** — how much data you can afford to lose, and how long you can
afford to be down. Set these in business terms before designing backups.

---

## Delivery

**CI** — Continuous Integration. Automated checks on every change. A test that
doesn't run automatically is documentation.

**CD** — Continuous Delivery or Deployment. Automated release.

**Quality gate** — checks that must pass before a change can merge. Blocking,
or it is decoration.

**Immutable artifact** — a built package that is never modified after building.
You cannot roll back to an image you cannot rebuild identically.

**Digest** — a cryptographic fingerprint of an artifact. Promoting by digest
guarantees the thing you tested is the thing you shipped.

**Pinning** — locking a dependency or base image to an exact version or digest,
so builds are reproducible and a third party cannot change what you run.

**Lockfile** — a file recording the exact resolved version of every dependency.
Only useful if the build actually installs from it.

**Rolling deploy** — replacing replicas gradually so there is no downtime.

**Rollback** — returning to the previous version. Must be practised, not
assumed.

**Expand and contract** — a safe schema change in three releases: add the new
thing, migrate to it, then remove the old thing.

**Smoke test** — a quick check after deploy that the basics work, with
automatic rollback if not.

**Infrastructure as code (IaC)** — your servers, networks and databases defined
in files under version control, rather than configured by hand in a console.

**Environment** — an isolated copy of the system: development, staging,
production.

**Feature flag** — a switch that turns a feature on or off without deploying,
separating release from deploy.

**Testcontainers** — running real databases and queues in containers during
tests, so tests exercise real constraints instead of mocks.
