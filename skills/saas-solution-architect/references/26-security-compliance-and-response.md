# Compliance readiness and incident response

Two things every commercial SaaS needs before it is asked for them: evidence
that its controls exist, and a rehearsed plan for the day one fails.

Both are cheap to design in and expensive to retrofit. A compliance programme
started during an enterprise sales cycle delays the deal by months; one started
in Phase 1 mostly consists of writing down what already exists.

---

## 1 Decide what you are actually building for

Do not build for a certification nobody has asked for. Do find out early which
one they will ask for, because the answer changes what must be logged, how long
it is kept, and who may see it.

| Trigger | What it usually means |
|---|---|
| Selling to mid-market or enterprise businesses | SOC 2 Type II, eventually ISO 27001 |
| Any European users or customers | GDPR, with a lawful basis per purpose |
| California or several other US states | State privacy law; overlaps heavily with GDPR mechanics |
| Handling card details directly | PCI DSS — avoid by never touching card data; use the provider's hosted fields |
| Health data in the US | HIPAA, and a business associate agreement |
| Selling to schools | FERPA, COPPA if under-13 users |
| Selling to government | Region-specific frameworks with long lead times |
| Customers in regulated finance | Their own supervisory requirements flow down to you |

**The cheapest compliance decision available: stay out of scope.** Never storing
card numbers removes most of PCI DSS. Never storing health data removes HIPAA.
Scope reduction beats control implementation every time.

---

## 2 SOC 2 and ISO 27001 readiness

Most of what an auditor wants, this standard has already built. The gap is
usually evidence and process, not engineering.

| Control area | Already built | Still needed |
|---|---|---|
| Access control | Four-layer authorization, least-privilege roles | Documented access request and approval process; quarterly review with records |
| Change management | Blocking CI gates, branch protection, reviews | A written policy; evidence that it was followed |
| Encryption | In transit and at rest, asserted at startup | Key management policy |
| Logging and monitoring | Structured logs, metrics, audit log, alerts | Retention policy; evidence of review |
| Vulnerability management | Scanning gates, dependency updates | Remediation targets, tracked and measured |
| Incident response | — | Plan, roles, tabletop evidence — section 5 |
| Business continuity | Backups, multi-AZ, rollback | Recovery objectives, tested restores with records |
| Risk management | Threat model | Risk register with owners and review cadence |
| Vendor management | Adapter-per-provider isolation | Sub-processor register, diligence records, contracts |
| Personnel | — | Background checks, onboarding and offboarding checklists, security training records |
| Physical security | Inherited from the cloud provider | Provider's own report, referenced in yours |
| Secure development | Threat model, review gates, tests | A written secure-development policy |

Practical guidance:

- **Evidence is a byproduct of automation, or it is manual work forever.** A
  control performed by a pipeline produces logs an auditor can sample. A control
  performed by a person produces a spreadsheet somebody has to maintain.
- **Type II covers a period**, typically three to twelve months. The clock
  starts when the controls are operating, so starting the controls early is what
  shortens the calendar.
- **Scope tightly.** One product, one environment, the controls that matter.
- **Pick the framework once.** ISO 27001 and SOC 2 overlap heavily; mapping one
  to the other later is much cheaper than running both from the start.

---

## 3 Data protection

### The record of processing

Maintain it as a living document. It is the first thing a regulator or an
enterprise privacy team asks for, and it takes an afternoon if the system was
designed with it in mind.

```markdown
| Purpose | Data categories | Subjects | Lawful basis | Retention | Location | Sub-processors |
|---|---|---|---|---|---|---|
| Provide the service | Account, usage, content | Customer users | Contract | Life of contract + 30d | eu-west-1 | Hosting, email |
| Billing | Name, address, tax id, payment token | Billing contacts | Contract, legal obligation | 7 years (tax) | eu-west-1 | Payment provider |
| Product analytics | Pseudonymised events | Customer users | Legitimate interest | 14 months | eu-west-1 | Analytics vendor |
| Support | Correspondence, impersonation logs | Customer users | Contract | 2 years | eu-west-1 | Helpdesk vendor |
| Security | Audit log, IPs, device data | All users | Legal obligation, legitimate interest | 1-2 years | eu-west-1 | Log vendor |
| Marketing | Contact details, preferences | Prospects | Consent | Until withdrawn | eu-west-1 | Email vendor |
```

Two roles matter and they are frequently confused. For your customers' data you
are usually the **processor** and they are the controller — you act on their
instructions, and the data processing agreement is the instruction. For your own
prospect and employee data you are the **controller**. The obligations differ,
and the distinction determines who answers a data subject's request.

### Data subject requests, as an engineering requirement

This is where privacy law becomes a system design problem. Each right implies a
capability, and it must actually work.

| Right | Capability required | The usual failure |
|---|---|---|
| Access | Produce everything held about one person | Data spread across database, logs, search index, analytics, backups, vendors — and nobody has the full list |
| Portability | Export in a machine-readable format | Only a PDF exists |
| Rectification | Correct data, including in downstream systems | Corrected locally, stale in the search index and at the analytics vendor |
| Erasure | Delete, and prove it | Backups, replicas, logs and vendors retain it |
| Restriction | Stop processing without deleting | No such state exists in the model |
| Objection | Stop a specific purpose | Purposes are not modelled, so they cannot be switched off individually |

```
✓ Design for these from the start:

  □ A data inventory that is generated, not maintained by hand. A hand-kept
    inventory is wrong within a quarter.
  □ A person-to-records mapping — every table holding personal data reachable
    from a subject identifier
  □ An export job producing a complete, machine-readable bundle
  □ Deletion that is a real capability:
        hard delete where possible
        crypto-shredding for anything in immutable stores and backups
        an anonymisation path where records must survive for financial or
          legal reasons — a ledger entry cannot be deleted, but it can stop
          identifying a person
  □ Retention as configuration with an enforcing job, not as a policy document
  □ Downstream propagation: deletion reaches the search index, the cache, the
    analytics vendor and the support tool, with evidence per destination
  □ Backup handling stated explicitly: either backups age out inside the
    stated window, or the key for that tenant is destroyed
  □ Audit entries for every request, its scope, and its completion
  □ A response clock, tracked. One month under GDPR, extendable once.
```

**Principle.** *Every data subject right implies a capability. A privacy policy
promising erasure that the system cannot perform is a liability, not a
protection.*

### Residency and transfers

```
□ Know where every store physically is — including backups, logs, search
  indexes, analytics, error tracking, and every vendor
□ Error tracking and log aggregation are the most commonly missed. They often
  default to a US region and receive personal data continuously.
□ For an EU residency commitment: the whole processing chain must be in
  region, vendors included
□ Transfer mechanism documented where data does leave the region
□ Sub-processor register published, with a notification process for changes.
  Enterprise data processing agreements require advance notice.
□ If residency is a tier feature, model the cost before selling it: it may
  mean a separate deployment, which multiplies operational work
```

---

## 4 Retention, deletion and offboarding

```
□ A retention period for every data category, enforced by a job that runs and
  is monitored
□ Log retention long enough to investigate a breach discovered late.
  Ninety days cannot investigate an intrusion found after four months.
  A year is a reasonable floor for audit and security logs.
□ Backup retention aligned with the recovery objective and the retention
  policy — a seven-year backup of data you promised to delete in one year is a
  contradiction
□ Tenant offboarding as a defined, tested sequence:
      1. account suspended, capability restricted, data intact
      2. grace period, published and honoured, for export and reinstatement
      3. export made available on request
      4. deletion executed, including crypto-shredding
      5. records that must survive (financial, legal) anonymised, not kept
         whole
      6. certificate of deletion issued if the contract requires it
      7. every step audited
□ Deletion is tested. A deletion path nobody has run does not work.
```

The grace period is a commercial decision with a security consequence: data kept
"just in case" is data that can still be breached. Set it, publish it, and
enforce it with a job.

---

## 5 Incident response

The plan's purpose is to remove decision-making from the worst possible moment.

### Severity, defined in advance

| Severity | Definition | Response |
|---|---|---|
| **SEV1** | Confirmed unauthorised access to customer data, or complete outage | Immediate, all hands, executive and legal engaged, notification clock starts |
| **SEV2** | Suspected breach, major degradation, or a critical vulnerability actively exploitable | Immediate, on-call plus security owner |
| **SEV3** | Contained security issue, no data exposure | Same business day |
| **SEV4** | Low-risk finding | Normal backlog |

**Anyone may declare.** Over-declaring costs an hour; under-declaring costs the
notification deadline. Say this explicitly in the plan, because the default
human behaviour is to hesitate.

### Roles

```
Incident commander   runs the response, makes decisions, does NOT do the
                     technical work — a commander with their hands in a
                     terminal is not commanding
Technical lead       investigates and remediates
Communications lead  customers, status page, internal updates
Legal and privacy    notification obligations, regulator contact
Scribe               timeline as it happens; the single most undervalued
                     role, because the timeline is what the post-incident
                     review and any regulator will need
```

For a small team one person may hold several roles — but the roles are still
named, so nothing is unowned.

### The sequence

```
1  DETECT      alert, report, customer, or researcher
2  DECLARE     assign a severity and a commander; open a channel and a
               timeline document
3  CONTAIN     stop the bleeding before understanding it fully.
               Revoke credentials · disable the account or feature ·
               block the path · isolate the host.
               PRESERVE EVIDENCE FIRST — snapshot before you terminate.
               A terminated instance is a destroyed crime scene.
4  ASSESS      what was accessed, by whom, when, how much, whose.
               "We cannot determine the scope" is the answer that forces the
               widest possible notification — which is why logging and
               retention are an incident-response control.
5  ERADICATE   remove the access path, patch the cause
6  RECOVER     restore service, verify integrity, watch for recurrence
7  NOTIFY      per the obligations below
8  REVIEW      blameless post-incident review within a week, with actions
               that have owners and dates
```

**Contain before you fully understand.** The instinct to diagnose first is
strong and wrong; every minute of an active intrusion is more data.

### Notification obligations

```
GDPR         · to the supervisory authority within 72 HOURS of becoming
               aware, where there is a risk to individuals
             · to affected individuals without undue delay where the risk is
               high
             · the clock starts at AWARENESS, not at certainty. A partial
               notification followed by an update is expected and permitted.
             · as a processor, notify your controller — your customer —
               without undue delay, so they can meet their own deadline

Contracts    · enterprise data processing agreements routinely require 24 or
               48 hours, which is stricter than the law. Know your worst
               contractual commitment; it is the one that governs.

US states    · vary by state, by data type, and by resident count

Sector       · HIPAA, financial supervisors and others have their own clocks
```

Prepare in advance: notification templates, the regulator's submission route,
the list of which customers have which contractual notification windows, and
who is authorised to speak. Drafting these during a SEV1 wastes the hours that
matter.

### Forensic readiness

An incident is investigated with the data you already collected. Decide now:

```
□ Audit log retention exceeds realistic discovery time (a year, minimum)
□ Access logs at the edge, in the application, and in the cloud control plane
□ Logs stored where a compromised production role cannot delete them
□ Immutable audit chain, so the timeline can be trusted — section 4 of the
  application security controls reference
□ A documented snapshot procedure that preserves before it remediates
□ Retained infrastructure-as-code history, so "what was the configuration on
  that date" is answerable
□ An external incident response retainer if the team is small. Negotiating one
  mid-incident is expensive and slow.
□ Cyber insurance reviewed against the actual notification obligations
```

### Tabletop exercises

Run one before launch and then annually. A plan that has never been walked
through is a document, not a capability.

Scenarios worth rehearsing, in rough order of likelihood:

1. A dependency has a critical vulnerability, actively exploited in the wild
2. A cross-tenant data exposure reported by a customer
3. Leaked credentials found in a public repository
4. A researcher reports a vulnerability with a disclosure deadline
5. A compromised support account used to impersonate customers
6. Ransomware in the cloud account, including backups targeted
7. A sub-processor announces their own breach

Each exercise ends with a list of things that did not work. Those are the
deliverable, not the exercise.

---

## 6 Answering security questionnaires

Enterprise deals include one, and they are a significant time sink handled
reactively.

```
□ Maintain a completed answer bank, sourced from the controls document
□ Keep a trust page: architecture summary, control list, sub-processors,
  certifications, uptime history
□ Publish a security overview so most questions are answered before they are
  asked
□ Track which answers are aspirational, and never ship an aspirational answer
  — a control claimed and absent is a misrepresentation that survives into
  the contract
□ Note the questions you cannot yet answer well; they are a prioritised
  security backlog written by your own customers
```

---

## Principles

1. **Find out which framework will be asked for before designing around one.**
2. **Scope reduction beats control implementation.** Never touching card data
   removes most of PCI DSS.
3. **Evidence is a byproduct of automation, or it is manual work forever.**
4. **Every data subject right implies a capability** that must actually work.
5. **A privacy promise the system cannot keep is a liability.**
6. **Log retention is an incident-response control.** "We cannot determine
   scope" forces the widest notification.
7. **Contain before you fully understand.** Preserve evidence before you
   remediate.
8. **The notification clock starts at awareness**, not at certainty.
9. **Know your strictest contractual notification window** — it is usually
   tighter than the law.
10. **Anyone may declare an incident.** Under-declaring is the expensive error.
11. **A plan never rehearsed is a document.** Run the tabletop.
12. **Never ship an aspirational questionnaire answer.**
