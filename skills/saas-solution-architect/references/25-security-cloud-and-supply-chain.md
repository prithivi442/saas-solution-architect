# Cloud posture and supply chain

Two attack paths that bypass the application entirely: the cloud account, and
the build pipeline. Both hand out production access without touching a single
line of application code, which is why application-focused security work
routinely leaves them open.

---

## 1 Identity and access in the cloud account

### No long-lived credentials, anywhere

```
✗ Static access keys in CI secrets, developer laptops, environment variables
     · they do not rotate, so they accumulate
     · they are copied into scripts, chat messages and screenshots
     · a leaked key is valid until somebody notices
     · attribution is weak: many people share one identity

✓ Short-lived credentials from an identity the platform issues:

  Workloads       task or pod role, assumed automatically, rotated by the
                  platform, never stored
  CI/CD           OIDC federation from the CI provider to a cloud role,
                  scoped by repository AND by branch or environment
  Humans          identity-centre or SSO sessions, MFA-enforced, time-limited
  Emergency       one break-glass identity, MFA on a physically secured device,
                  use alerts immediately and loudly, credentials rotated after
                  every use
```

The CI federation trust policy is the one people get wrong. A policy that trusts
the CI provider without constraining the subject lets **any** repository on that
provider assume your role.

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "//": "Pin the exact repository AND ref. A wildcard here trusts every",
      "//": "repository on the provider, which is the whole internet.",
      "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
    }
  }
}
```

For deploys gated on an environment, pin `environment:production` instead of a
branch, so the role is only assumable from a run that passed the environment's
approval.

### Least privilege that survives contact with delivery pressure

```
□ One role per workload, scoped to its own resources — its own secret paths,
  its own storage prefix, its own queue
□ Permission boundaries so a role cannot grant itself more than the boundary
  allows, even if someone edits its policy
□ Separate accounts per environment. An account is the only truly hard
  boundary a cloud provider offers; a tag is not.
□ Deploy credentials that can deploy and nothing else. They cannot read
  customer data, and they cannot alter audit configuration.
□ Service control policies denying, organisation-wide: disabling audit
  logging, disabling encryption, deleting backups, creating public storage,
  and operating in unapproved regions
□ No wildcard resources on write, delete or IAM actions
□ Access review on a schedule; unused permissions removed. Use the
  provider's last-accessed data rather than opinions.
□ Nobody has standing production data access. Elevation is requested,
  approved, time-boxed and audited.
```

### The audit trail of the account itself

```
□ Control-plane audit logging on, in every region, including regions you do
  not use — an attacker will choose one you are not watching
□ Logs delivered to a separate account, with object-lock, that production
  roles cannot write to or delete
□ Alerts on: root account use, MFA disabled, audit configuration changed,
  policy changes granting broad access, new identity provider or trust
  relationship, key deletion scheduled, storage made public, security group
  opened to the internet, unusual region activity
□ Threat detection service enabled, with findings routed to a person
□ Configuration drift detection against the infrastructure-as-code baseline
```

---

## 2 Network and egress

```
□ Application workloads in private subnets. Nothing accepts inbound traffic
  except the load balancer.
□ Database and cache reachable only from application security groups, by
  reference to the group rather than by IP range
□ Security group rules narrow on both directions. Egress rules matter: they
  are what limits an attacker who has already achieved code execution.
□ Private service endpoints, so traffic to managed services does not traverse
  the internet
□ Egress through a controlled path — NAT gateway or proxy — with an allowlist
  for anything that fetches customer-supplied URLs. This is the network-layer
  half of SSRF defence, and it holds when the application-layer half has a bug.
□ Instance metadata hardened: token-based version required, hop limit 1
□ TLS everywhere, including inside the network. Internal traffic is not
  trusted traffic; it is merely traffic you did not inspect.
□ TLS 1.2 minimum, modern cipher suites, certificate rotation automated
□ Web application firewall at the edge with managed rule sets, rate-based
  rules, and a documented process for reviewing blocked traffic
□ Distributed-denial-of-service protection at the edge, and a documented
  understanding of what it does not cover
```

**Principle.** *Egress rules are a security control, not a networking detail.
Ingress restrictions stop an attacker arriving; egress restrictions stop them
leaving with anything.*

---

## 3 Encryption posture, asserted rather than assumed

```
□ Encryption at rest on every store: database, backups, snapshots, object
  storage, queues, cache, logs, container images
□ Customer-managed keys where the compliance target requires demonstrable key
  control, with key policies restricting use to specific roles
□ Automatic key rotation enabled
□ Encryption in transit everywhere, including to the database. A database
  connection without TLS is a plaintext channel carrying every record.
□ ASSERTED AT STARTUP. The application verifies at boot that its database
  connection is encrypted and refuses to start otherwise.
```

That last item is the one that distinguishes a real control from an intention.
Encryption configured once and never verified is encryption that a
configuration change silently removed.

```ts
// Verify, do not assume. A driver that silently falls back to plaintext on a
// TLS failure is common, and the failure is invisible without this check.
const [{ ssl_in_use }] = await db.query(
  `SELECT ssl AS ssl_in_use FROM pg_stat_ssl WHERE pid = pg_backend_pid()`
);
if (!ssl_in_use) {
  logger.fatal('database connection is not encrypted');
  process.exit(1);
}
```

---

## 4 Supply chain

Your dependencies run with your privileges. So does your build system.

### Dependencies

```
□ Lockfile committed, and the build installs FROM the lockfile
  (npm ci · pip-sync · go mod download · cargo --locked). A lockfile the
  build ignores is a file, not a control.
□ Vulnerability scanning in CI, failing the build on high severity, with a
  documented exception process that has an expiry date
□ Automated update pull requests, reviewed and merged on a cadence rather
  than in a panic after an announcement
□ Install scripts disabled where the ecosystem allows. A postinstall script
  is arbitrary code execution at install time, on developer machines and in CI.
□ A cooling-off period before adopting a brand-new version — most malicious
  package versions are caught within days
□ Dependency count treated as a cost. Every addition is new code running with
  your privileges, and transitive dependencies are the majority of it.
□ Private registry or proxy, so a public package cannot shadow an internal
  package name (dependency confusion)
□ Internal package names claimed on the public registry, or scoped to your
  organisation, so nobody else can claim them
□ Licence scanning, because a licence problem is a commercial problem
```

### Build artifacts

```
□ Base images pinned by DIGEST, not by tag. A tag is mutable; today's
  'node:22-slim' is not tomorrow's.
□ Base images rebuilt on a schedule for OS patches, so pinning does not mean
  rotting
□ Multi-stage builds: build tooling never reaches the runtime image
□ Non-root user; read-only root filesystem; no shell in the runtime image
  where the base allows it
□ Image scanning in the registry, blocking promotion on high severity
□ SBOM generated per build, in a standard format, retained WITH the artifact.
  The question it answers is "are we affected", asked under time pressure.
□ Build provenance attestation — what was built, from which commit, by which
  workflow
□ Images signed, and signatures verified at deploy time, so only artifacts
  your pipeline produced can run
□ Reproducible builds where practical: the same commit produces the same
  digest, which is what makes an attestation meaningful
```

### The pipeline itself is production infrastructure

This is the part most often overlooked. CI holds credentials that can deploy to
production, which makes it a production access path with, usually, weaker
controls than production has.

```
□ Third-party actions pinned to a full commit SHA, never a tag or branch.
  An action referenced by a mutable ref executes whatever that ref points to
  AT RUN TIME, with access to whatever secrets the step is given. Pinning
  converts a trust relationship into a verifiable one.
□ Actions restricted to an allowlist of vetted publishers
□ Least-privilege pipeline tokens; read-only by default, write only in the
  jobs that need it
□ Secrets scoped to the environment and job that needs them, never repository-
  wide, and never available to a pull-request build from a fork
□ Untrusted pull requests run in an isolated context with no secrets. A
  workflow triggered by an untrusted event that checks out the PR's code and
  runs it with secrets is a full compromise, and it is a common
  misconfiguration.
□ Branch protection: required reviews, required status checks, no force-push
  to protected branches, no self-approval
□ Signed commits or tags for release artifacts
□ Environment approval gates for production deploys
□ Pipeline definition changes reviewed as security changes, because they are
□ Pipeline logs treated as potentially sensitive and scanned for leaked
  secrets
□ Secret scanning with push protection, so a credential is rejected rather
  than merely reported after it is public
□ A documented, practised procedure for rotating every credential CI holds
```

**Principle.** *The build system is a production access path. Every control you
apply to production access applies to it, and pinning by digest is what turns
"we trust this action" into "we verified this code".*

---

## 5 Secure development lifecycle gates

Blocking, or they are advisory and will be skipped under deadline.

| Gate | Runs on | Blocks on | Notes |
|---|---|---|---|
| Secret scanning | Every push | Any finding | Push protection, not post-hoc detection |
| Dependency audit | Every PR, plus daily | High severity | Daily catches newly published advisories against unchanged code |
| Static analysis | Every PR | New high-confidence findings | Baseline existing findings; block only regressions, or the gate is ignored |
| Licence check | Every PR | Disallowed licence | |
| Container scan | Every build | High severity | |
| IaC scan | Every PR touching infrastructure | Misconfiguration | Catches public storage and open security groups before they exist |
| Authorization policy check | Every PR | Any operation without a declared policy | The most valuable gate in this table |
| Cross-tenant matrix | Every PR | Any failure | |
| Dynamic scan | Nightly against staging | High severity | |
| Penetration test | Annually, and before a major launch | Findings tracked to closure | |

Two notes that determine whether these gates survive:

**Baseline your static analysis.** A gate that fails on two thousand
pre-existing findings gets disabled in a week. Baseline the existing set, block
on new findings, and burn down the baseline separately.

**Give exceptions an expiry.** A permanent exception is a removed control. An
exception with a date returns as a task.

---

## 6 Vulnerability management

```
□ An owner for security findings — a named person, not a rota
□ Severity-based remediation targets, agreed and measured:
     critical  · days
     high      · weeks
     medium    · a release cycle
     low       · backlog with review
□ Findings tracked in the normal work tracker, visible alongside features,
  so security work competes for time openly rather than invisibly
□ A published security contact and a security.txt at the well-known path
□ A vulnerability disclosure policy stating scope, safe harbour, and expected
  response times. Researchers will find things; the policy decides whether
  they tell you or somebody else.
□ Monitoring advisories for every component in the SBOM
□ An emergency patch path that has been practised, so a critical dependency
  advisory does not require inventing a process on the day
```

---

## Principles

1. **No long-lived cloud credentials anywhere, including CI.** Federate.
2. **Constrain the federation subject to a repository and a ref.** An
   unconstrained trust policy trusts the whole provider.
3. **An account is the only hard boundary.** Separate environments by account.
4. **Egress rules limit the damage of a compromise** the way ingress rules limit
   its likelihood.
5. **Assert encryption at startup.** Configuration decays silently.
6. **Install from the lockfile**, or the lockfile is decoration.
7. **Pin by digest**, and rebuild on a schedule so pinning does not become
   rotting.
8. **The pipeline is production.** Secrets scoped, forks isolated, actions
   pinned, changes reviewed as security changes.
9. **Generate an SBOM per build** — the question it answers is asked under time
   pressure.
10. **Gates block, or they are decoration.** Baseline the old findings, block
    the new ones.
11. **Exceptions expire.** A permanent exception is a removed control.
