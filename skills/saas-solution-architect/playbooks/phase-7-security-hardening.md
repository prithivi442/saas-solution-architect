# Phase 7 — Security hardening

**Output:** `docs/saas/05-threat-model.md`, `docs/saas/06-security-controls.md`, `docs/saas/07-compliance-pack.md`, `docs/saas/08-incident-response.md`, and the controls themselves
**Writes code:** yes
**Depends on:** Phases 2–6

---

## What this phase is for

Explain it to the user like this:

> Security work has been happening in every phase already — tenant isolation,
> fail-closed authorization, webhook signatures, secrets handling. This phase
> does the part that can only be done once the system exists: work out
> systematically how someone would attack it, check that each of those paths is
> closed, and produce the documents your enterprise customers will ask for
> before they sign.
>
> One thing worth saying plainly: the most common way a SaaS product leaks data
> is not a clever exploit. It is one customer being able to see another
> customer's records because a query forgot a filter. That is why so much of
> the earlier work went into making that structurally impossible rather than a
> thing someone has to remember.

Security is continuous, not a phase. What is scheduled here is the systematic
review and the artifacts. Do not let its position in the sequence imply that
anything before it was insecure by design.

---

## Step 1 — Threat model

Load `references/23-security-threat-model.md` and produce
`docs/saas/05-threat-model.md`.

- Inventory assets and classify data
- Draw trust boundaries — every point where data crosses from less trusted to
  more trusted
- Apply STRIDE at each boundary: spoofing, tampering, repudiation, information
  disclosure, denial of service, elevation of privilege
- Write abuse cases alongside the use cases: for each feature, how would
  someone misuse it?
- Produce a risk register with an owner and a decision per risk: mitigate,
  accept, transfer, or avoid

A threat model that lists only mitigated risks is incomplete. Accepted risks,
written down with a reason and an owner, are the valuable part.

---

## Step 2 — Application security controls

Load `references/24-security-appsec-controls.md` and work the list. The ones
most often missing in a SaaS at this stage:

**Server-side request forgery.** The product has outbound webhooks and
integrations, which means user-supplied URLs reach the server. Without egress
control, a customer can make the server call internal addresses, including the
cloud instance-metadata endpoint. Needs: destination allowlisting, resolved-IP
validation against private ranges, protection against DNS rebinding
re-resolution, and instance-metadata hardening.

**Object-level authorization within a tenant.** The cross-tenant matrix from
Phase 3 proves tenant A cannot reach tenant B. It does not prove that user A
cannot reach user B's records *inside* tenant A. That is a separate test matrix
and a separate class of bug.

**Support impersonation.** "Log in as this customer" is a feature almost every
SaaS eventually builds and one of the most common paths to a serious incident.
It needs: a separate audit trail, time-boxing, an explicit approval or consent
step, a clear visual indicator, and no ability to change credentials or billing
while impersonating.

**Audit log integrity.** An audit log an attacker can edit is not evidence.
Append-only, hash-chained, with a schema covering who did what to whom, when,
from where, and under what authority.

**Enterprise identity.** SSO, SAML or OIDC, and SCIM for provisioning. The
deprovisioning path matters more than the provisioning path: when an employee
leaves the customer's company, their access must end.

**Account security.** MFA enforcement per tenant policy, session management and
revocation, protection against account enumeration on every endpoint that
takes an email, and credential-stuffing defence on sign-in.

**File uploads and user content.** Serve user content from a separate origin so
a malicious file cannot script against the application, force download
disposition, validate content type by inspection rather than by the declared
header, and scan.

**Per-tenant encryption.** Envelope encryption with per-tenant data keys, which
makes crypto-shredding a real deletion mechanism when a customer leaves.

**Log redaction.** An allowlist, not a denylist. A denylist misses the field
somebody added last week.

**Frontend.** Content Security Policy with nonces, token storage in httpOnly
cookies rather than local storage, CSRF defence with SameSite, and
clickjacking protection.

**AI features**, if the product has any: prompt injection boundaries, output
handling, tenant data isolation in prompts and any retrieval index, and cost
controls.

Record every control and its status in `docs/saas/06-security-controls.md`,
mapped to OWASP ASVS and the OWASP API Security Top 10 — so that when a
customer sends a security questionnaire, the answer already exists.

---

## Step 3 — Cloud posture and supply chain

Load `references/25-security-cloud-and-supply-chain.md`.

- Least-privilege roles per service, scoped to their own resources, with
  permission boundaries
- **No long-lived cloud keys anywhere**, including CI — use federated identity
- Network and egress topology; private subnets; service endpoints
- SBOM per build, retained with the artifact
- Base images pinned by digest and rebuilt on a schedule for OS patches
- Third-party CI actions pinned to a commit SHA, never a mutable ref. An action
  referenced by a branch executes whatever that branch points to at run time,
  with access to whatever secrets the step is given.
- Blocking CI gates: dependency audit, secret scanning, static analysis
- Image signing and verification at deploy, if the compliance target requires it

---

## Step 4 — Compliance and response

Load `references/26-security-compliance-and-response.md`.

Produce `docs/saas/07-compliance-pack.md` covering only the targets named in
the Phase 0 brief — do not build for a certification nobody asked for:

- Control mapping for the target framework, showing which phase built each
  control
- Data processing record, lawful basis, sub-processor register
- Data subject request mechanics: access, export, correction, erasure — and
  whether the system can actually perform them today
- Retention and deletion schedule, including what happens when a tenant leaves
- Residency posture

Produce `docs/saas/08-incident-response.md`:

- Severity definitions and who declares
- Roles during an incident, and who talks to customers
- The notification clock for the applicable regime, and who starts it
- Forensic readiness: is log retention long enough to investigate a breach
  discovered two months late?
- A published vulnerability disclosure path and a `security.txt`
- Post-incident review process

Then **run a tabletop exercise.** A response plan that has never been walked
through is a document, not a capability.

---

## Gate

- [ ] Threat model complete, with an owner and a decision per risk
- [ ] Accepted risks explicitly accepted in writing by the user
- [ ] SSRF controls in place and tested against private ranges, metadata
      endpoints, and a rebinding attempt
- [ ] Object-level authorization matrix passes within a tenant, not just across
- [ ] Impersonation is audited, time-boxed, indicated, and restricted
- [ ] Audit log is append-only and tamper-evident — demonstrate detection
- [ ] Account enumeration is not possible on any endpoint taking an email
- [ ] MFA can be enforced per tenant; SCIM deprovisioning removes access
- [ ] User content is served from a separate origin, with forced download
- [ ] Log redaction is allowlist-based; a scan finds no secrets or personal data
- [ ] Security headers set explicitly, CSP included
- [ ] No long-lived cloud credentials exist anywhere, including CI
- [ ] SBOM generated per build; images pinned by digest; actions pinned by SHA
- [ ] Dependency, secret and static-analysis gates block a merge — demonstrate
- [ ] Encryption in transit and at rest is **asserted at startup**, not assumed
- [ ] Unauthenticated endpoint inventory is complete, each with justification
      and compensating controls
- [ ] Compliance pack covers every target from the Phase 0 brief
- [ ] Data subject requests can actually be executed — demonstrate an export
      and an erasure
- [ ] Incident response tabletop has been run
- [ ] A penetration test is booked or complete, with findings tracked

---

## References

- `references/23-security-threat-model.md`
- `references/24-security-appsec-controls.md`
- `references/25-security-cloud-and-supply-chain.md`
- `references/26-security-compliance-and-response.md`
- `references/22-security-architecture.md` — secrets, containers, headers,
  unauthenticated surfaces, and the general principles
- `references/16-multi-tenancy.md` — isolation verification
- `references/15-auth-and-authorization.md` — fail-closed revocation
