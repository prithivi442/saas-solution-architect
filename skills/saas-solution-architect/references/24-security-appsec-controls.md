# Application security controls

The controls in this document are the ones that a general security checklist
tends to miss, and that a multi-tenant commercial product specifically needs.
Each is presented as a reference flow with the failure it prevents.

---

## 1 Server-side request forgery and egress control

**Why this is load-bearing here.** A SaaS product with outbound webhooks,
integrations, "import from URL", link previews, avatar-by-URL, or PDF rendering
accepts a customer-supplied URL and then fetches it *from inside your network*.
That converts your server into a request proxy positioned behind your own
firewall.

The high-value target is the cloud instance-metadata service. On an
unhardened instance it hands out role credentials to anything that can make a
local HTTP request.

### The wrong shapes

```
✗ fetch(customerSuppliedUrl)
     · reaches 169.254.169.254 (cloud metadata → role credentials)
     · reaches 127.0.0.1 and ::1 (admin interfaces, debug endpoints)
     · reaches 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 (internal services)
     · reaches your own database, cache and queue management ports
     · follows redirects out of any check you did on the original URL

✗ Blocklist of literal strings ("169.254.169.254", "localhost")
     · 0x7f.0.0.1, 2130706433, 017700000001, [::ffff:127.0.0.1] all resolve to loopback
     · attacker-controlled DNS returns a private address for a public-looking host
     · a shortener or an open redirect hides the destination entirely

✗ Validate the hostname, then fetch by hostname
     · the check resolves DNS once, the fetch resolves again
     · between the two, the attacker's DNS answer changes  ← DNS rebinding
```

### The correct shape

```
✓ Layered, in this order:

  1. Parse strictly. Reject anything that is not http/https. Reject
     credentials in the URL, and reject non-default ports unless allowlisted.

  2. Prefer an ALLOWLIST of destinations. For outbound webhooks a customer
     registers and verifies a destination once; ad-hoc URLs are the exception,
     not the rule.

  3. Resolve DNS YOURSELF. Reject every resolved address that is loopback,
     link-local, private, unique-local, multicast, reserved, or IPv4-mapped
     IPv6 pointing at any of those. Check EVERY address returned, not the
     first.

  4. Connect to the RESOLVED IP, passing the hostname for TLS and Host.
     This is what closes DNS rebinding: the address you validated is the
     address you connect to.

  5. Do not follow redirects automatically. Follow manually, at most a few
     hops, re-running steps 1-4 on every hop.

  6. Bound it: connect timeout, total timeout, maximum response size, and a
     restricted content type where the use case allows.

  7. Egress at the network layer: run the fetcher where its outbound access is
     restricted by a NAT gateway or proxy, so a bypass in code is still
     contained.

  8. Harden the metadata service: require the hardened token-based version
     (IMDSv2 on AWS) and set the hop limit to 1 so a proxied request cannot
     reach it.
```

```ts
// Reference: fetch a customer-supplied URL safely
import { lookup } from 'node:dns/promises';
import net from 'node:net';
import ipaddr from 'ipaddr.js';

const ALLOWED_PROTOCOLS = new Set(['http:', 'https:']);
const MAX_REDIRECTS = 3;

function assertPublicAddress(ip: string): void {
  const parsed = ipaddr.parse(ip);
  // IPv4-mapped IPv6 (::ffff:127.0.0.1) must be unwrapped before classifying
  const addr = parsed.kind() === 'ipv6' && (parsed as ipaddr.IPv6).isIPv4MappedAddress()
    ? (parsed as ipaddr.IPv6).toIPv4Address()
    : parsed;
  const range = addr.range();
  if (range !== 'unicast') {
    throw new SsrfBlocked(`destination ${ip} is in reserved range '${range}'`);
  }
  // 'unicast' still includes some ranges worth refusing explicitly
  if (addr.kind() === 'ipv4') {
    const [a, b] = (addr as ipaddr.IPv4).octets;
    if (a === 100 && b >= 64 && b <= 127) throw new SsrfBlocked('carrier-grade NAT');
  }
}

async function resolveAndValidate(hostname: string): Promise<string> {
  // A literal IP must be validated too — it never reaches DNS
  if (net.isIP(hostname)) { assertPublicAddress(hostname); return hostname; }

  const answers = await lookup(hostname, { all: true, verbatim: true });
  if (answers.length === 0) throw new SsrfBlocked('no address');
  for (const { address } of answers) assertPublicAddress(address);  // ALL of them
  return answers[0].address;
}

export async function safeFetch(rawUrl: string, opts: { maxBytes: number }) {
  let url = new URL(rawUrl);

  for (let hop = 0; hop <= MAX_REDIRECTS; hop++) {
    if (!ALLOWED_PROTOCOLS.has(url.protocol)) throw new SsrfBlocked(url.protocol);
    if (url.username || url.password)         throw new SsrfBlocked('credentials in URL');

    const ip = await resolveAndValidate(url.hostname);

    // Connect to the validated IP; keep the hostname for TLS/SNI and Host.
    // This is the step that defeats DNS rebinding.
    const res = await undiciRequest(`${url.protocol}//${ip}${url.pathname}${url.search}`, {
      headers: { host: url.host },
      servername: url.hostname,
      maxRedirections: 0,                 // we follow manually, re-validating
      headersTimeout: 3_000,
      bodyTimeout: 10_000,
    });

    if (res.statusCode >= 300 && res.statusCode < 400 && res.headers.location) {
      url = new URL(String(res.headers.location), url);   // re-validate next loop
      continue;
    }
    return readCapped(res.body, opts.maxBytes);           // bound the response
  }
  throw new SsrfBlocked('too many redirects');
}
```

**Principle.** *Validate the resolved address, then connect to that address.
Any design that validates a name and connects by name has a window between the
two, and that window is the vulnerability.*

---

## 2 Object-level authorization, and IDOR inside a tenant

The generated cross-tenant matrix proves tenant A cannot reach tenant B. It
does **not** prove that inside tenant A, an ordinary member cannot reach a
record belonging to another member — a colleague's private note, another
employee's salary record, a document shared with a subset of the team.

This is a distinct bug class with a distinct test matrix, and it is the single
most commonly exploited web vulnerability class.

```
✗ The pattern that produces it:

    const doc = await docs.findById(tenantId, params.id);   // tenant checked
    return toDto(doc);                                       // ownership NOT checked

  Tenant scoping made this feel safe. It is not: every member of the tenant
  can now read every document in the tenant by iterating identifiers.
```

```
✓ Authorize the OBJECT, not just the collection:

    const doc = await docs.findById(tenantId, params.id);
    if (!doc) return notFound();
    await authorizeObject(ctx, 'document:read', doc);   // owner? shared? role?
    return toDto(doc);
```

Rules:

- **Every read of a single object by identifier runs an object-level check.**
  Collection endpoints filter; single-object endpoints authorize.
- **Never rely on an unguessable identifier as the control.** Use opaque
  sortable identifiers to remove enumeration as a *capability*, but treat that
  as defence in depth, never as the authorization decision.
- **The check is on the loaded object**, using its actual owner and sharing
  state — not on parameters the caller supplied.
- **Nested and expanded fields are separate decisions.** A root object the
  caller may read can expose a nested object they may not. In GraphQL this is
  per-field authorization; in REST it is per-expansion.
- **Write, delete and state-transition checks are separate from read.** Being
  able to see something is not permission to change it.

### The test matrix

```
for each object type
  for each operation (read, update, delete, each transition)
    for each relationship the caller may have to the object:
        owner · explicitly shared · same team · same tenant, unrelated ·
        tenant admin · different tenant
      assert the expected allow or deny
```

Generate it. A hand-written matrix omits the object type someone added last
week, which is exactly the one with the bug.

**Principle.** *Tenant scoping is a filter, not an authorization decision.
Every object fetched by identifier is authorized against the object that was
actually loaded.*

---

## 3 Support impersonation and break-glass access

"Log in as this customer" is a feature nearly every SaaS builds, and it is one
of the most reliable routes to a serious incident — either through misuse, or
through an attacker who compromises one support account and inherits access to
every customer.

```
✓ Required properties:

  Authorization      a dedicated permission, held by few, never bundled into
                     a general "support" role
  Justification      a ticket reference or written reason, captured before the
                     session starts, not selected from a dropdown of one option
  Approval           for restricted data: a second person approves
                     (break-glass), with the approval recorded
  Time-boxed         a short expiry; renewal requires a new justification
  Scoped             read-only by default. Write, and especially credential,
                     billing and permission changes, blocked or separately
                     approved
  Attributed         the audit log records BOTH identities: the acting support
                     user and the impersonated user. Never only the customer.
  Visible            an unmistakable banner in the UI for the whole session,
                     so an operator cannot forget which account they are in
  Disclosed          tenant admins can see impersonation history for their own
                     tenant. This is a trust feature, and enterprise buyers ask
  Alerted            volume anomalies, out-of-hours use, and access to
                     restricted-classification data all alert
  Excluded           impersonated sessions never count toward product analytics
                     or trigger customer-facing notifications
```

```ts
// The token carries both identities; there is no way to render one invisible.
interface ImpersonationClaims {
  sub: UserId;              // the customer being acted as
  act: {                    // RFC 8693 'actor' claim — who is really here
    sub: UserId;            // the support engineer
    reason: string;
    ticket: string;
    approvedBy?: UserId;
  };
  scope: 'read-only' | 'read-write';
  exp: number;              // short
}

// Blocked during impersonation regardless of scope:
const FORBIDDEN_WHILE_IMPERSONATING = new Set([
  'user:credentials:change', 'user:mfa:disable', 'apikey:create',
  'billing:payment-method:change', 'tenant:member:role:change',
  'tenant:delete', 'data:export:bulk',
]);
```

**Principle.** *An impersonated action has two actors and the log records both.
A system that can produce an audit entry indistinguishable from the customer's
own action has no audit trail for its most privileged operation.*

The same properties apply to any direct production access — a database console,
a runbook script, an admin panel. Named accounts, no shared credentials,
approval for restricted data, time-boxed, logged, and alerted.

---

## 4 Audit log integrity

An audit log that a compromised administrator can edit is not evidence. Two
properties are needed: it must be append-only, and tampering must be
*detectable*.

```
✓ Schema — every entry answers six questions:

    who       actor id, actor type (user · service · support · system),
              and the real actor if impersonating
    what      action from a closed vocabulary, not free text
    to what   target type, target id, tenant id
    when      server time, monotonic sequence within the tenant
    from where source IP, user agent, request id, session id
    under what authority
              the permission and entitlement that allowed it, and the
              approval reference if one was required

    plus: outcome (allowed · denied), and a before/after diff for changes
```

Denied attempts matter as much as successful ones — a burst of denials is one
of the few reliable early signals of an active attack.

### Tamper evidence by hash chaining

```
Each entry stores the hash of the previous entry, so any modification or
deletion breaks the chain from that point forward:

    entry[n].prev_hash = H( canonical(entry[n-1]) )
    entry[n].hash      = H( canonical(entry[n]) || entry[n].prev_hash )

Per tenant, so one tenant's volume cannot slow another's writes.

Periodically publish a checkpoint — the latest hash and sequence number —
somewhere the application cannot rewrite: a separate account's object store
with object-lock, or a write-once log. Without an external anchor, an attacker
with full database access can recompute the whole chain consistently.

A scheduled verifier walks the chain and alerts on the first break.
```

Operational rules:

- Written by the system only. No application code path issues an `UPDATE` or
  `DELETE` against the audit table; enforce with database permissions, not
  convention.
- Retention longer than everything else, and longer than the time it typically
  takes to *discover* a breach. Ninety days of logs cannot investigate an
  intrusion found after four months. One year minimum; check the compliance
  target.
- Separate access control. Reading the audit log is itself an audited action.
- Never a substitute for application logs, and never containing the sensitive
  values themselves — reference them.
- Availability of the audit path must not be optional: if the audit write
  fails, the action fails. An unlogged privileged action is worse than a
  rejected one.

**Principle.** *An audit log is only evidence if tampering is detectable and
the chain is anchored outside the system that writes it.*

---

## 5 Account and session security

The identity provider handles credential storage. These remain your problem.

### Enumeration

Every endpoint accepting an email address must respond identically whether the
account exists or not — same status, same body, same timing.

```
✗ Sign-up:         "That email is already registered"
✗ Reset:           404 for unknown, 200 for known
✗ Login:           "No account with that email" vs "Incorrect password"
✗ Invite:          "Already a member"
✗ Timing:          200ms when the hash is computed, 20ms when it is skipped

✓ One response:    "If an account exists, we've sent an email"
✓ Constant work:   compute a dummy hash on the not-found path
✓ Sign-up:         always "check your email"; the email differs, not the response
```

Timing is the one people forget. If the password hash is only computed when the
user exists, response time answers the question the message refused to.

### Credential stuffing and brute force

Layered, because each layer alone is bypassable:

```
□ Per-account throttle with exponential backoff, then temporary lock
□ Per-IP throttle
□ Per-ASN or per-subnet throttle — stuffing arrives distributed, so a
  per-IP limit alone barely registers
□ Device and session reputation
□ Proof-of-work or CAPTCHA escalation on anomalous volume, not on everyone
□ Breached-password check at set and at change, against a known-compromised
  corpus
□ Notify the user on login from a new device or location
□ Alert on a spike in failed logins across many accounts — the signature of
  stuffing, which per-account limits do not surface
```

Never lock permanently on failed passwords alone: that converts a nuisance into
a denial-of-service against your own customers.

### Multi-factor

- Enforceable as a **tenant policy**, not only a user preference. Enterprise
  buyers require the tenant admin to mandate it.
- TOTP and platform passkeys. Treat SMS as a fallback only; it is subject to
  SIM swap.
- Recovery codes generated once, shown once, stored hashed.
- Re-authenticate before security-sensitive changes: password, MFA
  enrolment, email, API keys, payment method, member roles.
- MFA reset is a privileged support operation, and one of the most abused
  social-engineering paths. It requires identity verification, approval, and an
  audit entry.

### Sessions

```
□ Short access token lifetime; refresh rotation with reuse detection —
  a replayed refresh token invalidates the whole family
□ A revocation denylist with a TTL matching the access-token lifetime, which
  FAILS CLOSED when unreachable
□ Bind sessions to a device fingerprint where the product allows
□ Full session invalidation on: password change, MFA change, role change,
  removal from tenant, suspected compromise
□ A user-visible session list, with individual and global revoke
□ Idle and absolute lifetimes; the absolute one is what limits a stolen token
□ Rotate the session identifier on privilege change, to prevent fixation
```

Removal from a tenant must terminate sessions immediately. A departed employee
holding a valid token until it expires is the gap that "revocation fails
closed" exists to close.

### API keys and machine credentials

```
□ Stored hashed, shown once at creation
□ Prefixed with a recognisable, scannable identifier so a leaked key can be
  detected in public repositories and revoked automatically
□ Scoped: a key can never exceed its creator's own permissions
□ Tenant-bound and, ideally, expiring by default
□ Last-used timestamp and source, so unused keys can be found and removed
□ Rotatable with an overlap window, so rotation needs no downtime
□ Listed and revocable by tenant admins
```

**Principle.** *Every response that varies by whether an account exists is an
enumeration oracle, and timing is a response.*

---

## 6 Enterprise identity: SSO and SCIM

Required to sell to larger companies, and usually gated behind a higher tier —
which makes it an entitlement, not a global feature.

```
□ SAML 2.0 and OIDC, configured per tenant
□ Domain verification before a tenant may claim email-domain-based routing.
  Without it, one tenant can claim a domain and intercept another's users.
□ Signature verification on every assertion; validate issuer, audience,
  recipient, conditions, and NotOnOrAfter. Reject unsigned assertions
  absolutely.
□ Replay prevention on assertion IDs
□ Enforce a signing algorithm allowlist; reject 'none' and weak algorithms
□ Just-in-time provisioning maps to a DEFAULT role, never an admin role
□ Attribute-to-role mapping is tenant-configured and validated
□ SSO-enforced mode: when a tenant requires SSO, password login is disabled
  for that tenant — including for existing users
□ A documented break-glass path for when the customer's identity provider is
  misconfigured, itself audited
```

**SCIM matters more for deprovisioning than provisioning.** Onboarding a user
late is an inconvenience; offboarding late is a security incident.

```
□ SCIM 2.0 user and group endpoints
□ Deactivation, not deletion, on the customer's side — preserve the audit trail
□ Deactivation terminates sessions and revokes tokens immediately
□ Group membership drives role assignment
□ Reconcile periodically: if SCIM calls stop arriving, drift is silent, so
  alert on unexpected quiet
□ Rate-limited and authenticated with a per-tenant credential
```

---

## 7 File uploads and user-generated content

Two separate risks: the file harming your infrastructure, and the file harming
other users when served back.

```
✓ Upload path:

  □ Presigned, direct-to-object-storage upload. Never proxy bytes through the
    application: it turns a large upload into memory pressure and a denial of
    service.
  □ Size limit enforced by the presigned policy, not only in the client
  □ Content type determined by INSPECTING the bytes, never from the
    Content-Type header or the file extension
  □ Extension and type allowlist per feature, never a blocklist
  □ Filename generated server-side. Never use the client's filename in a
    path — it carries traversal sequences, null bytes and control characters
  □ Malware scanning before the file becomes available, with a quarantine state
  □ Archive handling bounded: depth, entry count, total expanded size. An
    unbounded extractor is a decompression bomb target
  □ Image and document processing in a sandbox with no network egress and a
    memory cap. Media parsers are a historically rich source of
    remote-code-execution bugs
  □ Strip metadata such as EXIF location before serving, unless the product
    needs it — otherwise you are publishing your users' home addresses
```

```
✓ Serving path — this is where multi-tenant products get hurt:

  □ A SEPARATE ORIGIN for user content. Not a path on the application domain.
    An HTML or SVG file served from your application's origin executes with
    your application's cookies and same-origin privileges.
  □ Content-Disposition: attachment for anything not deliberately rendered
  □ X-Content-Type-Options: nosniff, and an explicit Content-Type
  □ A restrictive CSP on the content origin
  □ Short-lived signed URLs; storage never publicly listable
  □ Access authorized by the application on every request — a signed URL is a
    bearer token, so its lifetime is its blast radius
  □ SVG either rejected, sanitised, or rasterised. SVG is a script container.
  □ Tenant in the storage key prefix, with policies scoped to the prefix
```

**Principle.** *User-controlled bytes are served from an origin that has no
privileges to lose. Sharing an origin between your application and your users'
files makes every upload a potential cross-site scripting vector.*

---

## 8 Per-tenant encryption and crypto-shredding

Envelope encryption gives two things a single database key cannot: a credible
answer to "prove our data is isolated", and a real deletion mechanism.

```
      ┌──────────────────────────────────────────────────────┐
      │ Key management service                               │
      │   customer master key (CMK) — never leaves the KMS    │
      └───────────────────────┬──────────────────────────────┘
                              │ encrypts
      ┌───────────────────────▼──────────────────────────────┐
      │ per-tenant data encryption key (DEK)                  │
      │   stored encrypted alongside the tenant record        │
      │   decrypted into memory, cached briefly, never on disk │
      └───────────────────────┬──────────────────────────────┘
                              │ encrypts
      ┌───────────────────────▼──────────────────────────────┐
      │ restricted-classification fields for that tenant      │
      └──────────────────────────────────────────────────────┘

Deleting a tenant → destroy the DEK → every ciphertext for that tenant is
permanently unrecoverable, including in backups already written.
```

This last property is what makes deletion provable. Erasing rows does not
address the backups; destroying the key does.

```
□ Field-level encryption for restricted classifications: third-party
  credentials, government identifiers, health data, payment tokens
□ One DEK per tenant, wrapped by the CMK
□ DEK rotation supported, with a key version on every ciphertext so old data
  stays readable during re-encryption
□ Encryption at the application layer for these fields, so a database
  compromise alone yields ciphertext
□ Deterministic encryption ONLY where equality search is required, with the
  understood loss of semantic security — never for low-cardinality values
□ Search over encrypted fields via blind indexes, not by decrypting in bulk
□ Every key operation audited via the KMS trail
□ Bring-your-own-key as an enterprise tier feature: the customer controls the
  CMK, and revoking it removes your access. Model the availability consequence
  before offering it.
□ Backups encrypted with a separately managed key, in a separate account,
  with object-lock so a compromise of production cannot delete them
```

---

## 9 Log redaction

Logs travel widely — aggregators, third-party vendors, support tooling, laptops.
They are frequently the least-protected copy of your most sensitive data.

```
✗ Denylist redaction — redact fields named 'password', 'token', 'ssn'
     · misses 'passwd', 'pwd', 'authorization', 'apiKey', 'card_number'
     · misses the field someone added last week
     · misses nested structures and arrays
     · misses whole-object logging: logger.info({ user })
     · misses error objects that carry the request that caused them

✓ Allowlist serialisation — a value reaches a log ONLY if a serialiser was
  written for its type.

    type Loggable = string | number | boolean | null | LoggableId | Redacted;

    // Domain objects get explicit serialisers exposing identifiers, not content
    const logUser = (u: User) => ({ userId: u.id, tenantId: u.tenantId, role: u.role });
    // Note what is absent: email, name, phone, address.

    // The logger REFUSES unknown object types rather than stringifying them
    logger.info('user.updated', logUser(user));       // ok
    logger.info('user.updated', { user });            // compile error
```

```
□ Allowlist, enforced by types so violations fail the build
□ Structured fields only; never interpolate a value into a message string,
  because interpolation escapes the redaction layer entirely
□ Redact at the point of serialisation, not in the aggregation pipeline —
  once it has left the process it has already been written somewhere
□ Error and exception paths redacted too. Stack traces and error objects
  routinely carry the offending request body, and error paths are the
  least-tested code in any system.
□ Request and response body logging off by default; when enabled for
  debugging, time-boxed and field-filtered
□ Personal data reduced to an identifier. Never an email, never a name
□ Never a secret, token, key, session identifier, card number or full
  authorization header — not even truncated, if the truncation still narrows
  the search space usefully
□ URLs scrubbed of query parameters; tokens routinely appear there
□ A CI check scanning for known secret shapes in log statements
□ A periodic scan of the aggregator for anything that got through
```

**Principle.** *Redaction is an allowlist enforced at serialisation. Any
denylist is a list of the leaks somebody has already thought of.*

---

## 10 Browser-side controls

```
□ Content-Security-Policy with per-response nonces. No 'unsafe-inline',
  no 'unsafe-eval'. Add frame-ancestors, base-uri and form-action —
  default-src alone does not cover them.
□ Tokens in httpOnly, Secure, SameSite cookies rather than local storage.
  Local storage is readable by any script that runs on the page, which makes
  every cross-site scripting bug an immediate account takeover instead of a
  contained defacement.
□ CSRF defence: SameSite=Lax or Strict, plus an explicit token for
  state-changing requests on cookie-authenticated endpoints
□ X-Frame-Options: DENY or frame-ancestors 'none', unless the product is
  deliberately embeddable — in which case an explicit tenant-configured
  allowlist
□ Strict-Transport-Security with a long max-age, includeSubDomains, preload
□ Referrer-Policy: no-referrer or strict-origin-when-cross-origin, so URLs
  containing identifiers do not leak to third parties
□ Subresource integrity on any third-party script, and as few as possible.
  Every third-party script on an authenticated page can read that page.
□ CORS as an explicit allowlist. Never reflect the Origin header, and never
  combine a wildcard with credentials.
□ Cross-Origin-Opener-Policy and Cross-Origin-Resource-Policy to isolate the
  browsing context
□ Autocomplete and paste behaviour left alone on credential fields; blocking
  paste measurably weakens passwords by defeating password managers
□ No sensitive data in the URL. It reaches history, referrers, logs and
  analytics.
```

---

## 11 AI feature controls

If the product has generative features, they introduce a boundary with
properties the rest of the system does not have: the model treats data and
instructions as the same channel.

```
□ Treat model output as untrusted input. Never place it directly into a
  database query, a shell command, a file path, a URL to fetch, or raw HTML.
□ Prompt injection cannot be reliably prevented by instructions — content the
  model reads can override the instructions it was given. Constrain what the
  model can DO instead: no tool should be reachable through model output that
  the requesting user could not invoke directly, with their own permissions.
□ Tenant isolation inside prompts and retrieval. A retrieval index is a
  datastore: tenant is a mandatory filter applied server-side, never a hint
  in the prompt.
□ Never place another tenant's data, secrets, or system credentials in a
  context window.
□ Per-tenant token and spend caps, with alerts. Generative endpoints are
  metered third-party calls and therefore cost-amplification targets — the
  controls for metered endpoints apply in full.
□ Log prompts and completions under the same redaction rules as everything
  else, with a retention period, and disclose the practice.
□ Human confirmation before any model-initiated action that moves money,
  changes permissions, sends external communication, or deletes data.
□ Output filtering for the injection classes relevant to where the output is
  rendered.
□ Declare in the data processing record whether customer data is used for
  training. For business customers the answer is normally no, and it must be
  contractually true.
```

---

## 12 Mapping to external standards

Useful during procurement: enterprise buyers ask against these, and having the
mapping ready shortens a security review considerably.

| OWASP API Security Top 10 | Covered by |
|---|---|
| API1 Broken object level authorization | §2 |
| API2 Broken authentication | §5 |
| API3 Broken object property level authorization | §2, explicit DTOs |
| API4 Unrestricted resource consumption | Rate limits, complexity limits, spend caps |
| API5 Broken function level authorization | Four-layer model, declared policies verified in CI |
| API6 Unrestricted access to sensitive business flows | §1 abuse cases in the threat model |
| API7 Server-side request forgery | §1 |
| API8 Security misconfiguration | §10, container and cloud hardening |
| API9 Improper inventory management | Endpoint inventory, published contract |
| API10 Unsafe consumption of third-party APIs | Adapter layer with validation, timeouts, breakers |

| OWASP ASVS area | Covered by |
|---|---|
| V1 Architecture and threat modelling | Threat model reference |
| V2 Authentication | §5 |
| V3 Session management | §5 |
| V4 Access control | §2, §3 |
| V5 Validation and encoding | Boundary validation, unknown-field rejection |
| V6 Cryptography | §8 |
| V7 Error handling and logging | §4, §9 |
| V8 Data protection | §8, classification |
| V9 Communication | TLS everywhere, asserted at startup |
| V10 Malicious code | Supply chain reference |
| V11 Business logic | Abuse cases, idempotency, conditional writes |
| V12 Files and resources | §7 |
| V13 API and web service | §10, API standard |
| V14 Configuration | §10, configuration validation |

---

## Principles

1. **Validate the resolved address, then connect to it.** Name-based validation
   has a rebinding window.
2. **Tenant scoping is a filter, not an authorization decision.** Authorize the
   object you loaded.
3. **An impersonated action has two actors, and the log records both.**
4. **An audit log is evidence only if tampering is detectable** and the chain is
   anchored outside the system that writes it.
5. **If the audit write fails, the action fails.**
6. **Every response that varies by account existence is an oracle** — timing
   included.
7. **Deprovisioning is the security-critical half of identity integration.**
8. **User-controlled bytes are served from an origin with no privileges.**
9. **Destroying a key is the only deletion that reaches your backups.**
10. **Redaction is an allowlist at serialisation.** Denylists enumerate known
    leaks.
11. **Treat model output as untrusted input**, and constrain capability rather
    than relying on instructions.
12. **Every control here is verified by a test**, because a security control
    nobody tests is a security control nobody has.
