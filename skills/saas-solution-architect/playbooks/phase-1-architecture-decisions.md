# Phase 1 — Architecture decisions

**Output:** `docs/saas/02-architecture-decisions.md`, `docs/saas/03-target-architecture.md`
**Writes code:** no
**Depends on:** Phase 0 brief

---

## What this phase is for

Turn the product brief into a written set of decisions, each with a reason and
each reversible on a stated condition. The document exists so that in six
months nobody has to reconstruct why the system looks the way it does — and so
that when a decision needs revisiting, the trigger is already written down.

Say this to the user:

> I'm going to write down the architecture and the reason for each choice. You
> don't need to evaluate the technical merits — read the "what this costs you"
> and "when we'd change it" columns. If any of those read wrong for your
> business, tell me.

---

## Run the decision trees

Load `references/34-decision-framework.md` and run all twelve against the
Phase 0 answers. **Default to the simpler branch every time.** The uniform goal
is: do not add architectural complexity unless the problem requires it.

For a new product the honest outcome is almost always the same:

| Decision | Answer for a new product | Reconsider when |
|---|---|---|
| Monolith or microservices | Modular monolith, two entrypoints | A domain owns its data, has a measured independent scaling curve, and CI, IaC, observability and on-call all exist |
| Datastore | PostgreSQL | An access pattern, durability need, retention need **and** cost all diverge together |
| Redis | Only if more than one replica must share state, or a read is measurably hot | Never for anything whose loss corrupts business state |
| Queue | Yes, managed, from Phase 4 | — |
| Outbox | Yes, from Phase 4 | Never skipped where money moves or a resource is provisioned |
| Sync or async | Sync only where the user must be told before acting again | — |
| Read replica | Not yet | Reporting competes with transactional load |
| Partitioning | No | A single table makes maintenance painful |
| Distributed locks | No — use conditional writes | Scheduled work runs in a replicated service, or the resource is external |
| Events between services | Not applicable, there is one service | — |
| CQRS | No | Read and write loads conflict after replicas and indexes |
| Another service | No | Every box in the twelve-point checklist is yes |

If you find yourself choosing the complex branch, write down the specific
measured number that forced it. If there is no number, choose the simple
branch.

---

## Set the component tier

For every component, classify it as **default**, **optional** or **advanced**
using `references/01-golden-path.md`. Then justify anything that is not a
default. The classification is not about popularity; it is about whether the
problem has been demonstrated.

The classifying question for anything advanced: *what specific, measured
problem does this solve that the tier below cannot?* If it cannot be answered
with a number, it is not yet time.

---

## Handle stack deviation

Deviate from the default stack only for a reason in the brief:

| Reason in the brief | Legitimate deviation |
|---|---|
| The team is committed to a language | Use it. Read `references/38-stack-swapouts.md` and map the platform layer onto it. |
| The company has a cloud commitment | Use that cloud. Map the managed services. |
| Data must stay in a named country | Assess topology now; it may force a deployment decision |
| A measured CPU-bound workload exists | A compiled language for that path only |
| Nobody is on call | Push harder toward managed services than the default already does |

"A team member prefers it" is not a reason. Neither is novelty. Record any
deviation with its trigger so it can be revisited.

---

## Then write the decision record

Create `docs/saas/02-architecture-decisions.md`. One entry per decision:

```markdown
## D-001 — <the decision, stated in one line>

**Choice.** What we are doing.

**Why.** The reason, tied to a specific line in the product brief.

**What this costs you.** In money, time, or capability. Plainly.

**What we are giving up.** The alternative and what it would have bought.

**When we would change this.** The specific, observable trigger — a number
where possible.

**Reversibility.** Cheap / moderate / expensive, and why.
```

Cover at minimum: application shape, datastore, tenancy strategy, API style,
identity provider, authorization model, queue and scheduler, billing provider,
cache, hosting and orchestration, region topology, environment topology.

Then create `docs/saas/03-target-architecture.md` with:

- The architecture diagram, adapted from `references/01-golden-path.md`
- A component table: what each part does, in plain language
- The **connection budget** from `references/28-resource-management.md` — do
  this now, because replica count multiplied by pool size is a ceiling people
  discover during their first traffic spike
- The environment topology from `references/32-cicd.md`
- A first-cut cost estimate per environment, so the budget answer from
  Phase 0 is tested against reality rather than assumed

---

## Compliance check

If the brief named any compliance target or residency requirement, load
`references/26-security-compliance-and-response.md` now and add a section
stating which controls that target requires and which phase builds each. Doing
this in Phase 1 rather than Phase 7 prevents the common and expensive failure:
discovering during an enterprise sales cycle that audit logging, retention or
access review was never designed in.

---

## Gate

- [ ] All twelve decision trees run, with the simple branch taken unless a
      measured number forced otherwise
- [ ] Every non-default component has a written justification
- [ ] Every decision has a change trigger
- [ ] The connection budget is calculated and within the datastore's ceiling
- [ ] Every compliance target from Phase 0 maps to controls and phases
- [ ] The cost estimate is inside the stated budget, or the gap is flagged
- [ ] The user has read the "what this costs you" column for every decision

Stop and ask for approval to begin building.

---

## References

- `references/34-decision-framework.md` — the twelve trees
- `references/01-golden-path.md` — target architecture and tiers
- `references/02-architecture-and-boundaries.md` — boundary shapes to avoid
- `references/28-resource-management.md` — the connection budget worksheet
- `references/38-stack-swapouts.md` — only if deviating from the default stack
- `references/26-security-compliance-and-response.md` — if a target was named
- `references/35-anti-patterns.md` — read once before finalising, as a check
