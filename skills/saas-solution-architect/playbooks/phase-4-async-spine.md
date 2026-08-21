# Phase 4 — The asynchronous spine

**Output:** worker deployable, transactional outbox, queue with dead-letter queue, first integration adapter, webhook receiver
**Writes code:** yes
**Depends on:** Phase 3

---

## What this phase is for

Explain it to the user like this:

> So far, everything happens while the customer waits. Now I'll build the part
> that does work in the background — sending emails, calling other companies'
> systems, processing things that take time. The hard requirement is that none
> of it can be lost. If your server restarts halfway through charging someone
> or provisioning their account, the work has to resume, and it must not happen
> twice.

Two guarantees are being bought here, and both are cheap now and very expensive
later:

- **Nothing is lost.** A state change and its event commit together or not at
  all.
- **Nothing happens twice.** Every handler is safe to run again, because
  at-least-once is the only delivery guarantee available.

---

## Step 1 — The worker deployable

Background work moves out of the request process into its own deployable, from
the same codebase.

- Its own scaling policy and its own connection pool
- No background loops in the `api` process — an API replica running a poller
  multiplies that poller by the replica count
- Bounded concurrency, always. Unbounded promises are an outage waiting for a
  traffic spike.
- Graceful shutdown that drains in-flight work

Load `references/09-queues-and-async.md`.

---

## Step 2 — The transactional outbox

The pattern: the state change and the event row are written in **one**
transaction. A separate publisher reads the outbox and delivers. A sweeper
finds anything stuck.

Non-negotiable properties, from `references/08-transactional-outbox.md`:

- Atomic, lease-based claiming, so two publishers never take the same row
- A per-attempt audit record, so "why did this not deliver" is answerable
- Exponential backoff with jitter
- The **worker** is the retryable unit, not the event — this is the key design
  insight, because a fan-out event with three consumers needs each consumer to
  retry independently
- Parent status rolled up from child workers
- A sweeper that queries the *invariant*, not a status column, so it catches
  rows no status transition could describe
- Acknowledge the broker at the durability boundary — after the work is durable,
  never before

Use it whenever losing the event would corrupt state, whenever a state change
requires an external side effect, and **always** where money moves or a
resource is provisioned. Skip it for transient UI notifications.

---

## Step 3 — Queue with a dead-letter queue

- A managed queue. Broker operations are not differentiating work.
- **One dead-letter queue per queue**, not one shared. A shared DLQ makes
  triage guesswork.
- A redrive policy with a maximum receive count
- Poison-message handling: a message that fails deterministically must leave
  the queue, or it blocks everything behind it
- Publish through an exchange or topic, not directly to a queue, so adding a
  second consumer is a configuration change
- Keep observability traffic off the business broker — a metrics flood must not
  be able to starve order processing
- An alarm on DLQ depth greater than zero, and an alarm on queue age

Every handler is idempotent. Every handler restores tenant context. Every
handler has a timeout.

---

## Step 4 — The first integration adapter

One adapter per external provider, behind an interface the domain owns. Load
`references/17-integrations-and-webhooks.md`.

Every external call gets the full pipeline:

| Control | Without it |
|---|---|
| **Timeout** | A call with no timeout is a resource leak; enough of them exhaust the pool and take down the process |
| **Retry, transient only, with backoff and jitter** | Either no recovery from a blip, or a retry storm that synchronises across replicas |
| **Circuit breaker** | Retry without a breaker amplifies the provider's outage into yours |
| **Bulkhead** | One slow dependency consumes all concurrency and starves the others |
| **Kill switch** | No way to disable a misbehaving provider without a deploy |
| **Per-tenant rate limit** | One customer's usage exhausts a shared quota |

Never use a default timeout. Defaults in most HTTP clients are unbounded or
far too generous.

One client instance per process, configured explicitly, with connection reuse.
Creating a client per request defeats connection pooling and leaks sockets.

---

## Step 5 — The webhook receiver

Untrusted write paths need all five steps, in this order:

1. **Verify the signature.** A shared secret in a header is not a signature —
   it cannot bind the request body, so it does not prevent tampering. Use HMAC
   over the raw body with a timestamp, compared in constant time, with a
   replay window.
2. **Deduplicate** on the provider's event id.
3. **Persist** the raw event before processing.
4. **Acknowledge** with a 2xx immediately.
5. **Process asynchronously.**

Processing before acknowledging causes the provider to time out and redeliver,
which turns one event into many.

There are **no unauthenticated write paths.** If an endpoint must be
unauthenticated, it goes in the inventory described in
`references/22-security-architecture.md` with written compensating controls.

For outbound webhooks: sign them, include a timestamp, retry with backoff,
expose delivery attempts to the customer, and constrain the destination — a
customer-supplied URL is a server-side request forgery vector, covered in
`references/24-security-appsec-controls.md`.

---

## Gate

- [ ] `worker` deploys and scales independently of `api`
- [ ] No background loop runs in the `api` process
- [ ] A state change and its outbox event commit atomically — demonstrate with
      a crash injected between them
- [ ] The publisher claims rows atomically; two publishers do not double-send —
      demonstrate under concurrency
- [ ] A failing delivery retries with backoff and lands in the DLQ after the
      maximum receive count
- [ ] The sweeper recovers a row left in flight by a killed process
- [ ] Every handler is idempotent — demonstrate by delivering the same message
      twice and asserting one effect
- [ ] Every external call has an explicit timeout; no default is relied on
- [ ] The circuit breaker opens under sustained provider failure and recovers
- [ ] The kill switch disables a provider without a deploy
- [ ] Webhook signature verification rejects a tampered body and a replayed
      timestamp
- [ ] A duplicate webhook produces one effect
- [ ] Graceful shutdown drains in-flight work — demonstrate it
- [ ] Alarms exist on DLQ depth and queue age

---

## References

- `references/08-transactional-outbox.md`
- `references/09-queues-and-async.md`
- `references/17-integrations-and-webhooks.md`
- `references/07-distributed-consistency.md` — reconciliation as the backstop
- `references/06-transactions-and-integrity.md` — never place I/O in a transaction
- `references/20-concurrency.md` — claiming without a race
- `references/19-reliability.md` — graceful shutdown
- `references/24-security-appsec-controls.md` — SSRF on outbound webhooks
