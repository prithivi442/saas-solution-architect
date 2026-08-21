# Phase 9 — Launch and operate

**Output:** a completed readiness checklist, a proven rollback, a disaster-recovery drill, an on-call setup, `docs/saas/09-runbooks.md`
**Writes code:** only what the checklist forces
**Depends on:** all prior phases

---

## What this phase is for

Explain it to the user like this:

> Before this goes live I'm going to work through a full checklist, area by
> area, and show you the result of each item rather than just ticking it. Two of
> them I want to actually perform rather than confirm: rolling back a deployment,
> and restoring the database from a backup. A backup nobody has restored is not
> a backup, and a rollback nobody has performed is a hope.

---

## Step 1 — The readiness checklist as a blocking gate

Load `references/36-production-readiness.md` and work every area:
architecture, database, transactions and consistency, API, authentication and
authorization, multi-tenancy, caching, queues and async, billing, background
jobs, reliability, security, observability, performance, infrastructure and
CI/CD, testing, disaster recovery.

**Rules for this gate:**

- Every item is answered with evidence — a command output, a test result, a
  screenshot, a document reference. Not an assertion.
- Any unmet item is either fixed, or accepted in writing by the user with a
  stated risk and a date to revisit.
- Do not mark an item complete because the code exists. Mark it complete
  because you ran the check and can show the output.

Record the result in `docs/saas/10-readiness-signoff.md` with the evidence
alongside each item.

## Step 2 — Perform the two drills

**Rollback.** Deploy a change, then roll back to the previous artifact by
digest. Time it. Confirm no data loss and no manual step. If the rollback needs
a human to remember something, it will fail during the incident when it is
needed.

**Restore.** Restore the database from backup into a scratch environment.
Measure how long it took and how much data was lost. Compare against the
recovery objectives — and if there are no stated objectives, set them now with
the user, in business terms: how much data can you afford to lose, and how long
can you afford to be down.

Also verify: backups are encrypted, tested on a schedule, and stored so that a
compromise of the primary account cannot delete them.

## Step 3 — Migrations under real conditions

Confirm the migration path is: automated, locked so two deploys cannot migrate
concurrently, a blocking pipeline step, and every migration
backward-compatible so a rollback of application code does not break against
the new schema.

Practise one expand-and-contract change end to end.

## Step 4 — On-call

- Alert routing to a person, tested by firing one
- Every alert has a runbook — collect them into `docs/saas/09-runbooks.md`
- Escalation path
- A status page, and who updates it
- A support path for customers, and who answers

If the Phase 0 answer to "who is on call" was "nobody", say so plainly now and
agree what happens when it breaks at 2am. That is a business decision, not a
technical one, but it must be a decision rather than an omission.

## Step 5 — Launch

- Smoke tests after deploy, automated, with automatic rollback on failure
- Deploy during a window when someone is watching
- Watch the four dashboards and the error budget for the first hours
- Keep the first cohort small if the product allows it

## Step 6 — The first week

- Review every alert that fired: was it real, was it actionable, did the
  runbook help? Delete or fix the ones that were neither.
- Review the saturation dashboards against the load test predictions
- Confirm reconciliation has run and diverged zero times
- Confirm the audit log is being written and is queryable
- Book the first access review and the first dependency update cycle

---

## Gate

- [ ] Every readiness checklist item is answered with evidence
- [ ] Every unmet item is explicitly accepted in writing, with a revisit date
- [ ] A rollback has been performed, timed, and lost no data
- [ ] A restore from backup has been performed, timed, and measured for loss
- [ ] Recovery objectives are stated in business terms and met by the drills
- [ ] Backups are encrypted and cannot be deleted by a compromise of the
      primary account
- [ ] Migrations are automated, locked, blocking and backward-compatible, and
      one expand-and-contract change has been practised
- [ ] Alert routing has been tested by firing a real alert
- [ ] Every alert has a runbook
- [ ] Smoke tests run after deploy with automatic rollback
- [ ] Someone is named for on-call, or the absence is accepted in writing

---

## After launch

Hand the user a short standing list, because the system now needs maintenance
rather than construction:

| Cadence | Activity |
|---|---|
| Weekly | Review alerts that fired; dependency update PRs |
| Monthly | Access review; error budget review; cost review |
| Quarterly | Restore drill; threat model review against new features; degradation matrix review |
| Annually | Penetration test; disaster recovery exercise; review every accepted risk |

Then point them at `references/37-principles-and-sequencing.md` for what to
invest in next, and `references/34-decision-framework.md` for deciding when
growth genuinely justifies more complexity.

The platform now enforces correctness by default. From here, the work is
features.

---

## References

- `references/36-production-readiness.md` — the checklist
- `references/31-infrastructure-and-deployment.md` — rollback, migrations, IaC
- `references/32-cicd.md` — smoke tests and automatic rollback
- `references/19-reliability.md` — recovery objectives, degradation
- `references/37-principles-and-sequencing.md` — what to fund next
- `references/34-decision-framework.md` — when to add complexity later
