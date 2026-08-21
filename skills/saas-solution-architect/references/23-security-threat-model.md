# Threat modelling

Security controls without a threat model are a collection of habits. The model
is what tells you which controls matter for *this* product, which risks are
being knowingly accepted, and who accepted them.

This document produces one artifact: a threat model that is short enough to be
maintained and specific enough to be actionable.

---

## 1 Why a model rather than a checklist

A checklist answers "did we do the usual things?" A model answers "what would
actually happen if someone tried?" Both are useful; only the second one catches
the vulnerability that is specific to your product's shape.

The most common serious SaaS vulnerability is not exotic. It is **one tenant
reading another tenant's data because a query lost its filter.** A checklist
does not surface that; a data-flow model with trust boundaries does, because the
boundary between tenants is drawn explicitly and then interrogated.

**Principle.** *Model the boundaries, not the technologies. A control list
describes your intentions; a boundary list describes your exposure.*

---

## 2 Step one: inventory and classify

You cannot protect what you have not written down. Produce two tables.

### Assets

| Asset | Where it lives | Who may read | Who may write | Blast radius if leaked |
|---|---|---|---|---|
| Tenant business records | Primary database | Tenant members with permission | Same | One tenant, contractual |
| Credentials and tokens | Identity provider, denylist cache | Nobody | Auth flows | Account takeover |
| Third-party integration credentials | Encrypted column, KMS | The adapter, at call time | Tenant admins | Lateral movement into customer systems |
| Payment and ledger records | Primary database | Tenant admins, finance | Ledger writer only | Financial and regulatory |
| Audit log | Append-only store | Security, tenant admins (own tenant) | System only | Loss of evidence |
| Personal data | Primary database, logs, backups, search index, analytics | Per lawful basis | Per lawful basis | Regulatory, 72-hour clock |
| Secrets and signing keys | Secrets manager | Runtime roles only | Operators | Total compromise |
| Backups | Object storage, separate account | Restore role only | Backup role only | Everything, historically |

### Data classification

Four levels is enough. More than four and nobody remembers which is which.

| Level | Examples | Handling |
|---|---|---|
| **Restricted** | Payment credentials, secrets, health data, government identifiers | Encrypted with a dedicated key; access logged and reviewed; never in logs; never in non-production |
| **Confidential** | Tenant business data, personal data | Tenant-scoped; encrypted at rest; identifier-only in logs |
| **Internal** | Configuration, non-personal telemetry | Access-controlled |
| **Public** | Marketing content, published documentation | — |

**The rule that saves the most trouble:** production data never enters a
non-production environment. Generate or mask instead. The alternative is that
your staging environment, which has weaker controls by design, becomes a
production-grade breach target.

---

## 3 Step two: draw the trust boundaries

A trust boundary is any point where data or a request moves from somewhere less
trusted to somewhere more trusted. Enumerate them; each one is a place where
validation, authentication and authorization must happen.

For a typical SaaS, the boundaries are:

```
   ┌──────────────────────── UNTRUSTED ────────────────────────┐
   │  browser · mobile app · customer's API client · the internet │
   └───────────────────────────────┬───────────────────────────┘
                                   │  B1  edge: TLS, WAF, rate limit
   ┌───────────────────────────────▼───────────────────────────┐
   │  API entrypoint                                            │
   │    B2  authentication boundary                             │
   │    B3  tenant boundary        ← the one that matters most   │
   │    B4  permission boundary                                 │
   │    B5  entitlement boundary                                │
   └───────────────────────────────┬───────────────────────────┘
                                   │
   ┌───────────────────────────────▼───────────────────────────┐
   │  application core                                          │
   └──┬──────────┬──────────┬──────────┬──────────┬────────────┘
      │ B6       │ B7       │ B8       │ B9       │ B10
      ▼          ▼          ▼          ▼          ▼
   database    cache      queue     object     external
                                    storage    providers
                                                  ▲
   ┌──────────────────────────────────────────────┴───────────┐
   │  INBOUND UNTRUSTED, POST-AUTHENTICATION                   │
   │   B11  webhooks from providers    (signed, not trusted)   │
   │   B12  scheduler callbacks        (signed, not trusted)   │
   │   B13  customer-supplied URLs     (SSRF surface)          │
   │   B14  customer-supplied files    (upload surface)        │
   │   B15  support impersonation      (privileged, internal)  │
   │   B16  CI/CD into production      (supply-chain surface)  │
   └──────────────────────────────────────────────────────────┘
```

Boundaries B11 to B16 are the ones most often missed, because each *feels*
internal. A webhook arrives from your payment provider, so it feels trusted —
but the request arrives from the internet and anyone can send one. A scheduler
callback feels internal, but it is an HTTP endpoint. CI holds production
credentials, which makes the build system a production access path.

**Principle.** *Every boundary is a validation point. A request that feels
internal but arrives over the network is external, and "we control the caller"
is a statement about intent, not about reachability.*

---

## 4 Step three: STRIDE at each boundary

Six questions per boundary. Most answers are short; the value is in the ones
that are not.

| | Threat | The question | Primary control |
|---|---|---|---|
| **S** | Spoofing | Can someone claim to be another identity? | Authentication, signature verification, mutual TLS |
| **T** | Tampering | Can data be altered in transit or at rest? | TLS, HMAC over the raw body, integrity constraints, hash-chained audit log |
| **R** | Repudiation | Could someone deny an action they took? | Append-only audit log with actor, authority and source |
| **I** | Information disclosure | Can data leak to someone who should not see it? | Tenant scoping, RLS, field-level authorization, log redaction, 404-not-403 |
| **D** | Denial of service | Can someone exhaust a resource or run up a bill? | Rate limits, quotas, timeouts, bulkheads, complexity limits, spend caps |
| **E** | Elevation of privilege | Can someone gain rights they were not granted? | Four-layer authorization, declared policies verified in CI, impersonation controls |

### Worked example: the tenant boundary (B3)

This is the boundary worth doing thoroughly, because it is where the
characteristic SaaS breach happens.

| Threat | Concrete scenario | Control | Verified by |
|---|---|---|---|
| S | A token from tenant A is replayed against tenant B's subdomain | Tenant membership resolved from the token and the record, never from a header, path or hostname | Cross-tenant matrix test |
| T | A request body carries `tenant_id` and the server trusts it | Tenant never accepted from client input; always derived server-side | Code review rule plus a test asserting a supplied `tenant_id` is ignored |
| R | A tenant admin denies having deleted a record | Audit log with actor, tenant, authority, source IP, request id | Audit query test |
| I | A query omits its tenant predicate | Required branded tenant parameter, RLS backstop | Generated matrix, plus an RLS test with the predicate deliberately removed |
| I | An error message differs for "exists in another tenant" versus "does not exist" | Cross-tenant access returns 404, never 403 | Response-shape test |
| I | A cache key omits the tenant | Tenant mandatory in the cache key helper | Key-format test |
| I | A queue consumer trusts the tenant in the message | Consumer re-resolves and re-authorizes rather than trusting the payload | Handler test with a forged tenant |
| D | One tenant exhausts a shared quota | Per-tenant rate limits and quotas, not only global | Load test per tenant |
| E | A tenant member escalates to another tenant's admin | Roles scoped to tenant membership; no global role assignable by a tenant | Authorization matrix |

Repeat for every boundary. Boundaries B1, B2, B6–B10 are usually brief.
B3, B11, B13, B14, B15 and B16 deserve this level of detail.

---

## 5 Step four: abuse cases

For every feature, write the misuse alongside the use. This is where product
logic flaws are found, and they are invisible to scanners because nothing is
technically broken.

| Feature | Intended use | Abuse case | Control |
|---|---|---|---|
| Invite a teammate by email | Grow the account | Mass invitations as a spam relay carrying attacker text | Rate limit per tenant; no attacker-controlled free text in the email body; verified sender domains |
| Outbound webhook to a customer URL | Notify their systems | Point it at an internal address or the metadata endpoint | Egress allowlist, resolved-IP validation, blocked private ranges |
| CSV export | Reporting | Export the entire dataset before churning | Volume alerts, rate limits, audit entry per export |
| File upload | Attach a document | Upload HTML or SVG that executes against your origin | Separate origin, forced download, content-type by inspection |
| Free trial | Evaluation | Automated signups to consume metered third-party spend | Verification, per-identity limits, spend caps on metered paths |
| Password reset | Recovery | Enumerate which emails have accounts | Identical response and timing regardless of existence |
| API key creation | Automation | Create a key with more scope than the creator holds | Scope cannot exceed the creator's own permissions |
| Support impersonation | Help a customer | Browse customer data without a ticket | Time-boxed, approved, audited separately, visibly indicated |
| Usage-metered feature | Normal use | Drive cost above revenue for that tenant | Per-tenant spend cap and alert |

**Principle.** *A feature is not specified until its abuse case is written. Most
logic vulnerabilities are features working exactly as implemented.*

---

## 6 Step five: the risk register

One table, maintained. Every risk gets an explicit decision and an owner.

```markdown
| ID | Risk | Boundary | Likelihood | Impact | Decision | Control / rationale | Owner | Review |
|----|------|----------|-----------|--------|----------|--------------------|-------|--------|
| R1 | Cross-tenant read via missing predicate | B3 | Medium | Critical | Mitigate | Required tenant param + RLS + generated matrix | | Quarterly |
| R2 | SSRF via customer webhook URL | B13 | High | High | Mitigate | Egress allowlist + IP validation + IMDSv2 | | Quarterly |
| R3 | Insider access to production data | B15 | Low | High | Mitigate | Break-glass approval, session recording, audit | | Quarterly |
| R4 | Supply-chain compromise of a dependency | B16 | Medium | High | Mitigate | Lockfile, SBOM, audit gate, pinned actions | | Monthly |
| R5 | DDoS at the edge | B1 | Medium | Medium | Transfer | Provider protection; accepted residual | | Annually |
```

Four decisions are available, and three of them are legitimate:

- **Mitigate** — build a control
- **Transfer** — insurance, or a provider's responsibility under contract
- **Accept** — document the reason, the owner, and the review date
- **Avoid** — remove the feature

**Accept is a valid engineering decision when it is written down with an
owner.** An undocumented accepted risk is not a decision, it is an oversight
that will be discovered by an auditor or an attacker.

**Principle.** *A threat model listing only mitigated risks is incomplete. The
accepted risks, with their owners and review dates, are the part that makes it
a decision record rather than a wish list.*

---

## 7 When to review the model

Time-based review alone decays. Use triggers.

Review on any of these:

```
□ A new trust boundary appears
    · a new external integration
    · a new unauthenticated endpoint
    · a new file or URL input from customers
    · a new admin or support capability
□ The authorization model changes
    · a new role, a new permission, a new entitlement dimension
□ The tenancy model changes
    · sub-tenants, tenant hierarchies, cross-tenant sharing, guest access
□ Money moves in a new way
    · a new payment flow, payouts, refunds, credits, a new provider
□ Data classification changes
    · a new category of personal data, a regulated data type, a new region
□ A new compliance target is committed to
□ An incident occurs — anywhere in the industry, in a component you use
□ Quarterly, regardless
```

Additionally, require a **written security review before implementation** for
any change that: adds an unauthenticated endpoint, changes an authorization
check, touches the tenancy predicate, moves money, handles a new class of
personal data, adds a customer-supplied URL or file input, or grants a new
internal access path to production.

---

## 8 The artifact

Keep it to a few pages. A thirty-page threat model is written once and never
updated, which makes it worse than a two-page one that is current.

```markdown
# Threat model — <product>
Version · date · owner · last reviewed · next review

## 1 System description
   One paragraph, plus the architecture diagram.

## 2 Assets and data classification
   The two tables from section 2.

## 3 Trust boundaries
   The diagram from section 3, with one line per boundary.

## 4 STRIDE analysis
   The tenant boundary in full. Others as needed, briefly.

## 5 Abuse cases
   Per feature. Grows as the product grows.

## 6 Risk register
   Every risk, with a decision and an owner.

## 7 Accepted risks
   Extracted from the register, so they are impossible to overlook,
   each with the person who accepted it and the date it is revisited.

## 8 Out of scope
   What this model deliberately does not cover, and why.

## 9 Change log
   What changed, when, and which trigger caused the review.
```

---

## Principles

1. **Model boundaries, not technologies.** Exposure lives at boundaries.
2. **Anything arriving over the network is external**, however internal its
   origin feels.
3. **The tenant boundary deserves more analysis than the rest combined.** It is
   where the characteristic failure of this product category happens.
4. **Every feature needs its abuse case** written before it is built.
5. **Accepted risks are written down, owned, and dated.** Silence is not
   acceptance.
6. **Review on triggers, not only on a calendar.**
7. **Keep the model short enough to stay true.** Currency beats completeness.
8. **Production data never leaves production.** Mask or generate for every other
   environment.
