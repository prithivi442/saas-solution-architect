# Phase 3 — Identity, authorization and tenancy

**Output:** working sign-up and sign-in, the four-layer authorization model, tenant isolation proven by test, the first domain module
**Writes code:** yes
**Depends on:** Phase 2 platform layer

---

## What this phase is for

Explain it to the user like this:

> Now I'll build who can log in, which company they belong to, what they're
> allowed to do, and what their plan has paid for. Those are four separate
> questions and I'm going to keep them separate — because the fourth one, "has
> this customer paid for this feature?", is what lets you change your pricing
> later without a software release.

---

## Step 1 — Identity, bought not built

Integrate a managed identity provider. Do not build credential handling.

You are outsourcing: password hashing, multi-factor enrolment, reset-token
lifecycle, lockout, breach-password checks. Every one of these is a
well-understood way to be breached, and none of them differentiates the
product.

Build only:
- Short-lived access tokens
- Refresh rotation
- A revocation denylist with a TTL matching the access-token lifetime, which
  **fails closed** — if the denylist is unreachable, deny

Load `references/15-auth-and-authorization.md`.

**Never build a custom identity provider.** There is no revisit condition on
that one.

---

## Step 2 — The four layers

Four questions, answered in order, on every request:

| Layer | Question | Failure mode if merged into another layer |
|---|---|---|
| 1. Authentication | Who is this? | — |
| 2. Tenant membership | Which customer do they belong to, and are they still a member? | A removed employee retains access |
| 3. Role permission | Is this person allowed to do this? | Every user is effectively an admin |
| 4. Plan entitlement | Has this customer paid for this capability? | Pricing changes require a deploy |

Layer 4 is the one most systems omit and the most valuable to keep separate.
Model permissions and entitlements **as data over one shared capability
vocabulary**, so a new plan is a row, not a release.

Then wire the CI check from Phase 2: every operation declares a policy, or the
build fails.

---

## Step 3 — Tenancy, structurally

Load `references/16-multi-tenancy.md`.

- `tenant_id` on every scoped table, not null, indexed as the leading column of
  the composite indexes that matter
- The repository base **requires** a tenant parameter, branded so an ordinary
  string will not satisfy the type
- Row-Level Security enabled on every scoped table, with the policy reading the
  tenant from a session variable set inside the transaction
- Reference data lives in a separate global repository, so "no tenant" is an
  explicit choice rather than an omission
- **Cross-tenant access returns 404, never 403.** A 403 confirms the resource
  exists, which is an enumeration oracle.

### Isolation must hold at every layer

Not only the database. Verify each:

| Layer | The isolation question |
|---|---|
| Database | Does every query filter by tenant, and does RLS catch it if not? |
| Cache | Is the tenant in every cache key? |
| Queue | Does every message carry a tenant, and does the consumer re-verify it rather than trusting it? |
| Object storage | Is the tenant in the key prefix, and is access scoped? |
| Logs and metrics | Is the tenant present for attribution, without being a metric label? |
| Search indexes | Is the tenant a mandatory filter, applied server-side? |
| Background jobs | Does a job restore tenant context rather than running unscoped? |
| Exports and reports | Is the tenant applied before aggregation, not after? |

---

## Step 4 — The first domain module

Build one real feature end to end, through the platform layer. This proves the
foundation and gives every later module a pattern to copy.

Pick the smallest feature that has a create, a read, a list and a state change.
It must:
- Go through the authorization wrapper
- Use the tenant-scoped repository
- Return explicit DTOs, never a database model
- Use cursor pagination on the list
- Accept `Idempotency-Key` on the create
- Emit an outbox event on the state change

Load `references/13-api-design.md` and
`references/14-middleware-and-request-lifecycle.md`.

---

## Gate

- [ ] Sign-up, sign-in, refresh and sign-out all work
- [ ] A revoked token is rejected — demonstrate it
- [ ] The denylist being unreachable results in denial, not admission —
      demonstrate it
- [ ] All four authorization layers are enforced, separately
- [ ] A permission change takes effect without a deploy
- [ ] A plan entitlement change takes effect without a deploy
- [ ] The generated cross-tenant test matrix passes: for every operation, a
      user from tenant A cannot reach tenant B's resource
- [ ] Cross-tenant access returns 404, not 403
- [ ] RLS blocks a query that omits the tenant predicate — demonstrate it
- [ ] Tenant appears in cache keys, queue messages and storage prefixes
- [ ] The first domain module works and follows every rule above
- [ ] Adding an operation without a policy fails CI

---

## References

- `references/15-auth-and-authorization.md`
- `references/16-multi-tenancy.md`
- `references/13-api-design.md`
- `references/14-middleware-and-request-lifecycle.md`
- `references/24-security-appsec-controls.md` — object-level authorization
  within a tenant, which the cross-tenant matrix does not cover
- `references/33-testing.md` — the cross-tenant matrix generator
