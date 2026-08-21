# Phase 0 — Product and constraints intake

**Output:** `docs/saas/01-product-brief.md`
**Writes code:** no
**Typical length:** one conversation

---

## What this phase is for

Every later decision depends on facts only the user has. This phase collects
them once, writes them down, and turns them into engineering constraints. Skip
it and you will guess — and the guesses that hurt most are the ones about
compliance and pricing, because both are structural and both are expensive to
retrofit.

Say this to the user in your own words:

> Before I build anything, I need to understand your business — not your
> technology. I'll ask about what you sell, who buys it, and what rules you
> operate under. I'll handle every technical decision myself and explain each
> one as I make it.

---

## Ask these. One at a time.

Do not present this as a form. Ask, listen, follow up. If an answer is "I don't
know", say what you will assume and move on — an assumption on the record is
worth more than a blocked conversation.

### 1. The product

- In one or two sentences, what does the product do for the person using it?
- Who is the buyer, and who is the day-to-day user? Are they the same person?
- Is the customer an individual, or a company with several people in it?

*Why it matters:* if the customer is a company with multiple users, you need
tenants, roles and invitations from the start. Retrofitting an organisation
concept onto a single-user product touches every table and every endpoint.

### 2. Pricing

- How do you plan to charge? Per user, per month, per usage, per site, a flat
  fee, or a mix?
- Will there be tiers? Roughly what does each tier include?
- Free trial, freemium, or paid only?
- Will you ever need to change pricing without waiting for a software release?

*Why it matters:* this determines the entitlement model. The answer to the last
question is almost always yes, which is why plan entitlements are stored as
data rather than written into code.

### 3. Scale

- How many customer companies do you expect in year one? In year three?
- How many users inside a typical customer?
- Is usage steady through the day, or spiky?
- Is there a largest-imaginable customer who is much bigger than the rest?

*Why it matters:* tenant count drives the isolation strategy. Spikiness drives
autoscaling. A single outsized customer is the usual reason to consider
isolating one tenant's resources.

### 4. Data sensitivity and regulation

- What is the most sensitive thing the system will store? Names, payment
  details, health information, children's data, government identifiers?
- Do your buyers work in a regulated industry — healthcare, finance,
  education, government?
- Will enterprise buyers ask for a security questionnaire, SOC 2, or ISO 27001?
- Does any customer need their data held in a specific country?

*Why it matters:* this is the answer that most often changes the architecture.
Residency can force a deployment topology. A compliance target changes what
must be logged, how long it is kept, and who may access it. Ask early.

### 5. Integrations

- What other systems must this connect to? Payment, email, calendars,
  accounting, single sign-on?
- Will customers want to receive notifications of events in your system?
- Will customers want an API of their own?

*Why it matters:* every external dependency is a failure domain, and outbound
notifications mean webhook delivery, signing, and retry infrastructure.

### 6. Real-time and AI

- Does anything need to update on screen instantly for multiple people at once?
- Will the product use AI features — generation, summarisation, chat?

*Why it matters:* live updates justify a separate deployable, and only then.
AI features bring their own security surface and their own cost controls.

### 7. Team and operations

- Who will maintain this after launch? A team, one engineer, an agency, nobody
  yet?
- Does anyone already have a language or cloud they are committed to?
- Who is on call if it breaks at 2am?

*Why it matters:* operational capacity is a hard constraint on architecture.
A design nobody can run is a design that fails. If the answer to on-call is
"nobody", weight every choice toward managed services.

### 8. Commercial constraints

- What is the target launch date, and is it fixed or aspirational?
- What is the monthly infrastructure budget you would be comfortable with?
- Is there an existing product or data to migrate from?

---

## Then write the brief

Create `docs/saas/01-product-brief.md`:

```markdown
# Product brief

## The product
## Customers and users
## Pricing model and tiers
## Expected scale, year 1 and year 3
## Data sensitivity classification
## Regulatory and compliance targets
## Data residency requirements
## Required integrations
## Real-time requirements
## AI features
## Team and operational capacity
## Commercial constraints

## Derived engineering constraints
| Constraint | Consequence | Phase |
|---|---|---|

## Open questions and assumptions made
| Question | Assumption used | Revisit when |
|---|---|---|
```

The **derived engineering constraints** table is the part that matters. It is
where you translate. Examples of the translation:

| They said | Constraint |
|---|---|
| "Customers are companies with several staff" | Tenants, roles, invitations from Phase 3 |
| "We'll charge per site, tiers may change" | Entitlements as data, not code — Phase 3 |
| "Enterprise buyers will ask for SOC 2" | Audit log, access review, retention policy — Phase 7 |
| "One customer must be hosted in Germany" | Residency assessment before Phase 1 topology |
| "Nobody is on call" | Managed services everywhere; alert only on user-visible harm |
| "We need to notify customers of events" | Outbound webhooks with signing and retry — Phase 4 |
| "Usage is spiky around month end" | Autoscale on a measured signal; queue depth alarms |

---

## Gate

Present the brief and confirm:

- [ ] The product description is right in the user's own words
- [ ] The pricing model is right, including how tiers might change
- [ ] The sensitivity classification and every compliance target is named
- [ ] Residency requirements are explicit, including "none"
- [ ] Every assumption made on the user's behalf is written down
- [ ] The user has read the derived constraints and agrees with the
      consequences

Then stop and ask for approval to proceed to Phase 1.

---

## References

Load only if a question above needs grounding:

- `references/00-glossary.md` — if the user asks what a term means
- `references/34-decision-framework.md` — if the user pushes on a
  scale-related decision now rather than in Phase 1
- `references/26-security-compliance-and-response.md` — if a compliance
  target is named and you need to state its consequences accurately
