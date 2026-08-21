# Phase 5 — Billing and money

**Output:** subscription integration, append-only ledger in exact decimals, balance projection, payment webhook lifecycle, reconciliation, entitlement wiring
**Writes code:** yes
**Depends on:** Phase 4 async spine

---

## What this phase is for

Explain it to the user like this:

> This is the phase where mistakes cost money and can't be undone. I'm going to
> do three things. First, buy the complicated part — subscriptions, tax,
> proration, dunning, failed-card retries — from a specialist provider, because
> that's an enormous problem domain and building it would be a mistake. Second,
> keep our own permanent record of every financial movement, so you can always
> explain any customer's balance. Third, build a job that checks our records
> against the provider's every day and alerts if they disagree. That last one is
> the difference between finding a billing bug in a day and finding it in an
> audit.

---

## The rules that cannot bend

Read these to yourself before writing a line, and do not negotiate any of them.

**Never floating point.** Binary floating point cannot represent most decimal
fractions exactly, and the error is cumulative. It surfaces late, as a
reconciliation discrepancy nobody can explain. Use a fixed-precision decimal
type or integer minor units, end to end — database column, application type,
API payload, and every intermediate calculation. A decimal library that is
merely a declared dependency, while the columns are floats, buys nothing.

**Money is an append-only ledger, never a mutable balance field.** Every
financial movement is a new entry. A correction is a compensating entry, never
an edit. This buys three properties a mutable balance cannot:

- **Auditability** — every balance is explainable by the entries that produced it
- **Immutability** — history is never rewritten
- **Reconstructability** — any balance can be recomputed from first principles
  at any point in time

**A balance is a projection, not a source of truth.** And it must be a
*maintained* projection, not a sum over full history — that query gets slower
every day the product succeeds, and it is on the payment path.

**Currency travels with every amount.** A bare number is a future incident.

**The provider is the system of record for subscriptions.** Your database holds
a cache of provider state and treats it as a cache.

**Never trust a webhook payload's contents.** Re-fetch authoritative state from
the provider using the id in the event.

**Failed payment restricts capability; it never deletes data.** A customer who
pays late must get their account back intact.

---

## Step 1 — The provider integration

Buy subscription management. Proration, dunning, tax, currency, invoicing and
payment-method lifecycle are each large domains.

Keep the ledger separate from the biller. The ledger records what happened; the
billing system decides what to charge. Different correctness requirements,
different change cadence, different failure tolerance.

Load `references/18-billing-and-money.md`.

---

## Step 2 — The ledger and the balance projection

- Append-only journal, exact decimals, currency on every entry
- Every entry records **who, when, why, and under what authority**
- Balance maintained as a projection, updated in the same transaction as the
  entry
- Reconstruct-from-entries implemented and tested, because it is the tool you
  will need during an incident

## Step 3 — Race-free, idempotent debits

Two properties, and they are separate:

- **Idempotent.** Every charge has an idempotency key derived from its cause,
  enforced by a unique constraint in the database. Not by a cache, not by an
  application check — a constraint.
- **Race-free.** Every balance check is the *predicate of the write*, not a
  read followed by a write. A read-then-write balance check under concurrency
  produces a negative balance, and two concurrent requests each seeing
  sufficient funds is the normal case, not the rare one.

Load `references/20-concurrency.md` for the conditional-write shape.

## Step 4 — The payment webhook lifecycle

Verify signature and timestamp, deduplicate on the provider's event id,
persist raw, acknowledge, then process asynchronously. Handle out-of-order
delivery: events do not arrive in the order they occurred, so processing must
be tolerant of a later state arriving first.

## Step 5 — Reconciliation

A scheduled job comparing local ledger state against the provider. This is a
**required component**, not a nice-to-have, and its divergence count is a
monitored metric with an alert.

Without it, a divergence is discovered by a customer or an auditor.

## Step 6 — Entitlement wiring

Connect the plan to the capability vocabulary from Phase 3. Entitlement is
data, derived from the plan, cached briefly, and invalidated after commit — so
a plan change takes effect immediately without a deploy.

---

## Alerting

Financial anomalies need their own alerts, separate from technical ones:

- Charges above a threshold
- Any balance going negative
- Reconciliation divergence count above zero
- Unusual per-tenant velocity
- Failed payment rate rising

---

## Gate

This is the most-tested code in the system. All of these are mandatory, not
representative:

- [ ] No floating-point type appears anywhere on the money path — verify by
      inspecting column types, not just application code
- [ ] Currency is present on every amount
- [ ] The ledger is append-only; an attempted update fails
- [ ] Balance projection matches a full recomputation from entries
- [ ] A duplicate webhook produces exactly one ledger entry
- [ ] Out-of-order webhooks converge to the correct state
- [ ] Two concurrent charges against a balance sufficient for only one:
      exactly one succeeds — demonstrate under real concurrency
- [ ] The same charge submitted twice with the same idempotency key produces
      one effect, enforced by a database constraint
- [ ] Insufficient funds is rejected as the predicate of the write
- [ ] A refund produces a compensating entry, not an edit
- [ ] A crash mid-transaction leaves no partial financial state
- [ ] Reconciliation detects a deliberately introduced divergence
- [ ] Failed payment restricts capability and preserves all data
- [ ] A plan change alters entitlement without a deploy
- [ ] Every financial alert fires in a test

---

## References

- `references/18-billing-and-money.md`
- `references/20-concurrency.md` — race-free debits
- `references/06-transactions-and-integrity.md` — atomicity
- `references/12-workflows-and-state-machines.md` — sagas with compensation for
  provisioning flows that span systems
- `references/17-integrations-and-webhooks.md` — the receiver pattern
- `references/15-auth-and-authorization.md` — entitlement layer
- `references/33-testing.md` — mandatory coverage on money
