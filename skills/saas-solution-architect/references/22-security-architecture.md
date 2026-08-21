# Security architecture

## 25.1 Required controls

| Control | Requirement |
|---|---|
| Managed identity provider (JWT, MFA, SRP) | **Required** |
| Token revocation denylist | **Required** — must fail closed |
| Four-layer authorization | **Required** |
| Field-level encryption of third-party credentials (KMS) | **Required** |
| Webhook signature verification (HMAC, per-tenant) | **Required** |
| Payment webhook signature verification (HMAC + timestamp) | **Required** |
| CORS allow-list | **Required** |
| Input validation at the boundary | **Required** |
| SQL parameter binding | **Required** |
| Distributed rate limiting | **Required** |
| Bot and abuse defence at signup and login | **Required** |
| Audit logging | **Required** |
| Row-level tenancy | **Required** — plus RLS as the backstop |
| Row-Level Security enforcement | **Required — commonly missing** |
| Secrets manager | **Required — commonly missing** |
| Platform identity instead of static AWS keys | **Required — commonly missing** |
| Dependency scanning / SAST / SBOM | **Required — commonly missing** |
| Security headers (HSTS, CSP, etc.) at the app layer | **Required — commonly missing** |
| Containers running as a non-root user | **Required — commonly missing** |

## 25.2 What each control buys

- **Credential storage is outsourced** to a managed identity provider. No password hashes, no MFA implementation, no reset-token lifecycle to get wrong.
- **Third-party credentials are encrypted with a managed KMS** rather than stored in plaintext.
- **Webhook signatures are verified with per-tenant secrets**, so one tenant's compromised credential cannot forge another tenant's events.
- **Authorization is layered and explicit**, with entitlement modelled separately from permission.
- **Revocation fails closed.**

Together these form a reasonable foundation. The controls marked commonly missing are **platform hardening** rather than application redesign, which is why they are cheap to add early and awkward to add late.

## 25.3 Reference flow: secrets management on AWS

```
✗ .env files on hosts, static IAM access keys in environment variables
     · rotation is a manual, coordinated, error-prone operation
     · secrets appear in process listings, crash dumps, and shell history
     · no audit trail of access
     · a host compromise yields every credential at once

✓ Layered, with rotation built in:

  AWS credentials       → instance profile / ECS task role / IRSA
                          (rotated automatically; never stored)
  Application secrets   → Secrets Manager (rotation-capable) or
                          SSM Parameter Store SecureString (cheaper)
  Non-secret config     → SSM Parameter Store / environment variables
  Injection             → fetched at startup, cached in memory,
                          refreshed on a timer; never written to disk
  Access control        → IAM policy per service, scoped to its own
                          secret paths only
  Audit                 → CloudTrail records every secret access
```

**Principle.** *Rotation must be possible without a deploy. That single requirement rules out secrets baked into images, committed to repositories, or copied into caches — and it is the requirement to design against.*

## 25.4 Reference flow: container hardening

```dockerfile
# ── Build stage ──────────────────────────────────────────────
FROM node:22.17-slim AS builder
WORKDIR /app
# Copy BOTH manifest and lockfile, then install from the lockfile:
# this is what makes the build reproducible.
COPY package.json package-lock.json ./
RUN npm ci                                   # ← ci, not install: honours the lockfile
COPY . .
RUN npm run build

# ── Production dependencies, separately ──────────────────────
FROM node:22.17-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# ── Runtime ──────────────────────────────────────────────────
FROM node:22.17-slim AS runner
ENV NODE_ENV=production
WORKDIR /app
# Copy ONLY what runs: built output and production dependencies.
COPY --from=deps    --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/dist./dist
COPY --from=builder --chown=node:node /app/package.json ./

USER node                                    # ← never root
ENTRYPOINT ["/usr/bin/dumb-init", "--"]      # ← PID 1 that forwards signals
CMD ["node", "dist/server.js"]               # ← the app, not a wrapper
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s \
  CMD node -e "require('http').get('http://localhost:'+process.env.PORT+'/health/live',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"
```

Each line addresses a specific property:

| Practice | Property gained |
|---|---|
| Copy the lockfile; `npm ci` | **Reproducible builds** — the same commit yields the same dependency tree |
| Pinned base image tag (not `latest`) | The build cannot change underneath you |
| Separate production-dependency stage | Runtime image excludes devDependencies, sources, and build tooling |
| `USER node` | A container escape does not begin as root |
| `dumb-init` as PID 1 | Signals reach the app; zombies are reaped; graceful shutdown works |
| Direct `node` command | No wrapper swallowing signals |
| `NODE_ENV=production` | Framework production paths; no dev-only middleware |
| `HEALTHCHECK` | The system can detect a wedged process |

For a Go service the equivalent is a static binary in `scratch` or `distroless`:

```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                                  # cached, verified against go.sum
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath \
      -ldflags="-s -w -X main.version=${VERSION}" -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

This reduces a ~1 GB toolchain image to a few tens of megabytes with no shell, no package manager, and no root — which removes most of the attack surface rather than defending it.

## 25.5 Reference flow: securing metered and unauthenticated surfaces

Two endpoint classes need explicit review beyond the standard authorization check:

**1. Endpoints that trigger billable external calls.** These are cost-amplification targets even when they expose no data. Covered in `17-integrations-and-webhooks.md` §8: authenticate, rate-limit at three levels, cap daily spend, meter, alert, and cache stable answers.

**2. Endpoints deliberately exposed without authentication.** Every one needs a written justification and compensating controls:

```
For each unauthenticated endpoint, document:
  □ Why it cannot be authenticated
  □ What it can cause (data exposure? cost? state change?)
  □ Rate limits: per IP, per path, global
  □ Whether it triggers any external metered call (if so: it must be
    authenticated, signed, or cached)
  □ Whether it can enumerate anything (identifiers, existence, validity)
  □ What monitoring exists for abnormal volume
```

**Principle.** *Maintain an explicit inventory of unauthenticated endpoints, reviewed on every change. An endpoint becomes unauthenticated by omission far more often than by decision, and nothing in a normal review surfaces it.*

## 25.6 Reference flow: supply chain

```
□ Lockfiles committed and used by the build (`npm ci`, `go mod download`)
□ Dependency vulnerability scanning in CI, failing on high severity
□ Automated dependency update PRs
□ SBOM generated per build and retained with the artifact
□ Base images pinned by digest, rebuilt on a schedule for OS patches
□ Container image scanning in the registry
□ Third-party CI actions pinned to a commit SHA, never to a mutable ref
□ Deployment credentials scoped to deployment only, with an audit trail
□ Signed images and verification at deploy time (ADVANCED)
```

**On CI actions specifically:** an action referenced by a mutable ref such as `@master` executes whatever that reference points to *at run time*, with access to whatever secrets the step is given. Pinning to a commit SHA converts a trust relationship into a verifiable one.

## 25.7 Reference flow: HTTP security headers

Even for an API, headers are cheap and worth setting explicitly rather than assuming the load balancer provides them:

```ts
app.use(helmet({
  hsts: { maxAge: 31_536_000, includeSubDomains: true, preload: true },
  contentSecurityPolicy: { directives: { defaultSrc: ["'none'"], frameAncestors: ["'none'"] } },
  referrerPolicy: { policy: 'no-referrer' },
  frameguard: { action: 'deny' },
  noSniff: true,
}));
app.disable('x-powered-by');     // stop advertising the framework and its version
```

## 25.8 General SaaS security principles

1. **Outsource credential handling** to a managed identity provider.
2. **Secrets live in a secrets manager; rotation requires no deploy.**
3. **Platform identity over static keys** for all cloud API access.
4. **Tenant isolation is enforced by a mechanism** — required parameters plus RLS.
5. **Fail closed on every security decision.**
6. **Validate at the boundary; reject unknown fields.**
7. **Bind every SQL value; never accept a caller-supplied identifier position.**
8. **Verify every webhook signature; deduplicate every webhook.**
9. **Containers run as non-root, from pinned bases, with lockfile-driven builds.**
10. **Least-privilege IAM per service**, scoped to its own resources.
11. **Audit-log every security-relevant action**, with longer retention and stricter access.
12. **Never log secrets or personal data beyond an identifier.**
13. **Classify endpoints by both data sensitivity and cost exposure.**
14. **Maintain an inventory of unauthenticated endpoints.**
15. **Uniform validation across environments.**
16. **Constant-time comparison for every secret.**
17. **Scan dependencies and images continuously; pin CI actions by digest.**
18. **Encryption in transit and at rest, asserted at startup, not assumed.**
