# Phase 6 — Observability

**Output:** structured logs, metrics, tracing, SLOs with burn-rate alerts, a runbook per alert, `docs/saas/04-degradation-matrix.md`
**Writes code:** yes
**Depends on:** Phase 2 platform layer, which already contains the modules

---

## What this phase is for

Explain it to the user like this:

> Right now, if the product got slow, nobody could tell you where. This phase
> fixes that. I'm going to make the system able to answer three questions
> without guessing: is it broken, is it slow, and where. Then I'll set targets
> for how reliable it should be, and make it alert you only when customers are
> actually being harmed — not every time a server has a bad second.

The three pillars go in together. Retrofitting tracing after the fact is far
harder than adding it now, because it has to be threaded through every
boundary the system already has.

---

## Step 1 — The three pillars, minimally

Load `references/21-observability.md`.

**Logs.** Structured JSON to stdout. Collection is a platform concern, not an
application one — an application that ships its own logs to a vendor has
coupled itself to that vendor and added a failure mode. Level INFO in
production. Correlation id on every line, taken from ambient context, never
passed as a parameter through function signatures.

**Metrics.** RED per endpoint — rate, errors, duration. USE per dependency —
utilisation, saturation, errors. Saturation gauges for every pool and queue.
**Enforce low-cardinality labels at the helper**: a tenant id or user id in a
metric label will eventually take down the metrics backend, and the fix is
expensive because dashboards and alerts depend on the label.

Add business metrics too, because these are the ones that detect the failures
that matter commercially: money movements, reconciliation divergences, metered
spend, authorization denials.

**Tracing.** Propagate across HTTP **and** queue boundaries. A trace that stops
at the queue is a trace that cannot answer "where did this go".

**Correlation comes from ambient context, not from a function signature.**
Threading a correlation id through every call is how it gets dropped in the one
path that mattered.

---

## Step 2 — SLOs before dashboards

This ordering is deliberate and usually reversed.

A dashboard shows what happened. An SLO decides whether it mattered. Building
dashboards first produces a wall of graphs nobody knows how to read and alerts
that fire on conditions no customer noticed.

For each user-facing journey:

1. Define what "working" means, from the customer's point of view
2. Set a target — availability and latency
3. Derive the error budget
4. Alert on **burn rate**, not on instantaneous threshold breach

Burn-rate alerting is what stops the pager firing for a single slow second
while still catching a slow degradation that will exhaust the month's budget by
Thursday.

---

## Step 3 — A runbook per alert

An alert with no runbook is a page to somebody who then has to improvise. Each
runbook states: what fired, what the customer is experiencing, the first three
things to check, how to mitigate, and how to escalate.

If an alert cannot be given a runbook, it is not an alert. Delete it or turn it
into a dashboard panel.

---

## Step 4 — Dashboards

Four, no more, to start: request health, dependency health, queue health,
saturation.

---

## Step 5 — The degradation matrix

Produce `docs/saas/04-degradation-matrix.md`. For every dependency, state what
happens when it fails — and make it a decision rather than a discovery.

| Dependency | If it is slow | If it is down | Customer sees | Alert |
|---|---|---|---|---|
| Database | | | | |
| Cache | | | | |
| Queue | | | | |
| Identity provider | | | | |
| Payment provider | | | | |
| Email provider | | | | |
| Object storage | | | | |
| Each third-party integration | | | | |

Filling this in usually reveals at least one dependency whose failure takes the
whole product down when it should only degrade a feature. That discovery is the
point of the exercise.

Load `references/19-reliability.md`.

---

## Gate

- [ ] Every log line has a correlation id, in every entrypoint including workers
- [ ] A request can be followed end to end across an HTTP call and a queue hop
- [ ] RED metrics exist per endpoint; USE per dependency; saturation per pool
- [ ] Business metrics exist for money, reconciliation and authorization denials
- [ ] No metric label carries a tenant id, user id, or other unbounded value —
      verify by inspecting the label sets
- [ ] One error-tracking system, with release tagging
- [ ] An SLO exists for every user-facing journey, with an error budget
- [ ] Alerts fire on burn rate
- [ ] Every alert has a runbook
- [ ] No secrets and no personal data beyond an identifier appear in logs —
      verify with a scan, not an assumption
- [ ] The degradation matrix is complete, and any dependency that takes the
      whole product down has been either accepted in writing or fixed

---

## References

- `references/21-observability.md`
- `references/19-reliability.md` — degradation matrix, health endpoints
- `references/27-performance.md` — what to measure before optimising
- `references/28-resource-management.md` — saturation signals worth alarming on
- `references/24-security-appsec-controls.md` — log redaction standard
