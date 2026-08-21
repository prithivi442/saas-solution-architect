# Multi-tenancy

The one area where a mistake is not a bug but a breach. Everything here exists
to convert tenant isolation from something the team remembers into something the
system enforces.

---

## 1 The model

| Concern | Requirement |
|---|---|
| Tenant unit | One customer organisation — name it once and use that name everywhere |
| Isolation strategy | **Shared database, shared schema, row-level `tenant_id`** |
| Tenant identification | An opaque tenant reference, never an enumerable integer, resolved server-side |
| Membership verification | Principal → membership → tenant, verified on every request |
| Tenant context | Established once in the authorization wrapper, stored in ambient context |
| Query scoping | A required parameter in the data layer, **not** a predicate added per call site |
| Cache key scoping | Tenant identifier in every key |
| Queue message scoping | Tenant identifier in the message, **re-verified** by the consumer |
| Row-level security | Enabled, with a policy on every scoped table |
| Per-tenant encryption keys | For restricted-classification fields — see `24-security-appsec-controls.md` |

Shared-database, row-level tenancy is the right default for this class of
product: one schema to migrate, one connection pool, one backup, and per-tenant
cost that scales with usage rather than with tenant count. Schema-per-tenant
collapses at a few thousand tenants; database-per-tenant collapses far sooner.

**Use an opaque reference rather than a numeric tenant id at the transport
layer.** It removes tenant enumeration as a capability.

---

## 2 The central risk

With row-level tenancy, **the tenant predicate is the only thing separating
customers**, and it is applied by whichever code happens to be writing the
query. The failure mode is not dramatic. It is a single lookup that takes an
identifier and omits the tenant filter:

```ts
// The shape to eliminate: a primary-key lookup with no tenant predicate.
async findById(id: number) {
  return this.model.findByPk(id);          // ← returns ANY tenant's row
}

// A caller that authenticates, authorizes, checks the plan — and then reads
// an object it never confirmed belongs to the caller's tenant:
const record = await service.findById(args.id);   // args.id is client-supplied
return record;
```

Every layer of authorization passed. The response contains another tenant's
data.

This is why the object-scope layer described in
`15-auth-and-authorization.md` is the layer that decides whether the others
matter. It is compounded whenever an **enumerable integer identifier** is the
public identifier, because guessing `id + 1` becomes a viable enumeration
strategy.

**The same hazard applies to identifiers passed through to external
providers.** A handler that accepts a provider-side identifier from the client
and retrieves it directly from the provider's API inherits the provider's
tenancy model — which is usually "everything in one account". The ownership
check must happen locally, against your own records, before the provider is
called.

---

## 3 Reference flow: making tenant scope structural

Four mechanisms, in increasing strength. Adopt at least the first two; the third
is the one that makes the guarantee absolute.

### Mechanism 1 — the tenant is a required parameter of the data layer

```ts
// Every tenant-scoped repository takes the tenant as a NON-OPTIONAL argument.
// There is no overload without it, so omission is a compile error.
abstract class TenantRepository<T> {
  constructor(private model: ModelStatic<any>) {}

  async findById(tenantId: TenantId, id: EntityId): Promise<T | null> {
    return this.model.findOne({ where: { id, tenantId, deletedAt: null } });
  }

  async findMany(tenantId: TenantId, where: Where = {}): Promise<T[]> {
    return this.model.findAll({ where: { ...where, tenantId, deletedAt: null } });
  }

  async update(tenantId: TenantId, id: EntityId, patch: Partial<T>): Promise<number> {
    // The tenant predicate is in the WHERE clause of the write, too —
    // an update is as much a cross-tenant risk as a read.
    const [n] = await this.model.update(patch, {
      where: { id, tenantId, deletedAt: null },
    });
    return n;                                  // 0 ⇒ not found OR not yours: same answer
  }
}
```

Two supporting details do most of the work:

- **A branded type for the tenant id** —
  `type TenantId = number & { readonly __brand: 'TenantId' }` — so a plain
  `number` cannot be passed by accident and argument order cannot be transposed
  silently.
- **Returning the same result for "absent" and "not yours"** — `null` or `0`
  rows — so the handler cannot accidentally distinguish them and leak existence.

### Mechanism 2 — separate global data explicitly

Not every table is tenant-scoped: country lists, currency tables, plan
catalogues and price books are global. Make the distinction explicit in the type
system so "no tenant filter" is a deliberate, visible choice:

```ts
abstract class GlobalRepository<T> { /* no tenant parameter, by design */ }
abstract class TenantRepository<T> { /* tenant parameter required */ }
```

A reviewer now sees which one a repository extends. A repository that should be
tenant-scoped but extends `GlobalRepository` is a visible anomaly rather than an
invisible omission.

### Mechanism 3 — Row-Level Security as the backstop

This is the mechanism that converts the guarantee from *disciplined* to
*enforced*, because the database applies it regardless of what the application
forgets:

```sql
ALTER TABLE records ENABLE ROW LEVEL SECURITY;
ALTER TABLE records FORCE ROW LEVEL SECURITY;      -- applies to the table owner too

CREATE POLICY tenant_isolation ON records
  USING      (tenant_id = current_setting('app.tenant_id')::bigint)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
      -- USING      → filters reads and the rows an UPDATE/DELETE can see
      -- WITH CHECK → prevents INSERT/UPDATE from writing another tenant's id
```

```ts
// Set the tenant for the duration of the transaction, once, in one place.
await sequelize.transaction(async (t) => {
  await sequelize.query('SELECT set_config($1, $2, true)',       // true ⇒ transaction-local
    { bind: ['app.tenant_id', String(tenantId)], transaction: t });
  return work(t);
});
```

With RLS active, a query that forgets its tenant predicate returns **zero rows**
instead of another tenant's data. The failure mode changes from a silent breach
to a visible, testable bug — exactly the trade you want.

Two operational notes: use `set_config(..., true)` so the setting is
transaction-scoped rather than session-scoped, which is critical when
connections are pooled and reused; and ensure the application's database role is
**not** `BYPASSRLS` and not the table owner unless `FORCE ROW LEVEL SECURITY` is
set.

### Mechanism 4 — verify isolation in CI

```
□ A schema test asserts every tenant-scoped table has a tenant_id column,
  a NOT NULL constraint, and an index leading with it
□ A schema test asserts every such table has RLS enabled and a policy
□ A lint rule forbids primary-key lookups without a tenant predicate outside
  GlobalRepository subclasses
□ An integration test suite runs every read endpoint as tenant A against
  a resource owned by tenant B and asserts 404 — this catches the class
  of bug that review does not
□ A test asserts every unique index that must be per-tenant includes
  tenant_id (a global unique index is a cross-tenant collision waiting
  to happen, and a cross-tenant existence oracle)
```

**Principle.** **Tenant isolation must be enforced by a mechanism, not
maintained by discipline.** Required parameters catch most omissions at compile
time; RLS catches the rest at run time; a cross-tenant test suite catches what
neither does. Any one of the three alone is insufficient at scale.

---

## 4 Isolation across every layer

Tenancy is not only a database concern. Each layer needs its own scoping rule.

| Layer | Requirement | Failure if omitted |
|---|---|---|
| Database rows | `tenant_id` on every scoped table + RLS | Cross-tenant read or write |
| Unique indexes | Include `tenant_id` in every per-tenant uniqueness constraint | One tenant's data blocks another's insert; existence becomes observable |
| Cache keys | Tenant identifier in the key prefix | Cross-tenant cache poisoning or leakage |
| Queue messages | Tenant identifier in headers *and* re-validated by the consumer | A consumer acting on the wrong tenant |
| Object storage | Tenant segment in the key path; presigned URLs scoped and short-lived | Cross-tenant file access |
| Search indexes | Tenant filter applied at query time, not post-filtered | Cross-tenant search results |
| Logs and traces | Tenant identifier as a structured field | Cannot investigate per-tenant incidents |
| Metrics | Tenant as a **low-cardinality bucket** (tier, size class) — never a raw id | Metric cardinality explosion |
| Rate limits | Keyed per tenant | One tenant exhausts everyone's budget |
| Background jobs | Tenant carried in the job payload; context established on execution | Jobs operating on the wrong tenant |
| External provider calls | Ownership verified **locally** before the provider is called | Inheriting the provider's flat tenancy |
| Exports and reports | Tenant filter applied at generation, and re-asserted at download | Bulk cross-tenant disclosure — the highest-impact variant |

---

## 5 Choosing an isolation strategy

```
Tenant count > ~100, or tenants are small relative to your infrastructure?
  → Shared database, shared schema, row-level tenancy + RLS.       [DEFAULT]

A regulatory requirement mandates physical separation for SOME tenants?
  → Hybrid: pooled by default, dedicated for those tenants.
    Same code path; different connection target. Never a second codebase.

Fewer than ~50 tenants, each very large, with per-tenant scaling needs?
  → Database per tenant. Accept: N migrations, N backups, N monitors,
    and a control plane you now have to build and operate.

Never:
  → Schema per tenant at scale. It looks like isolation and behaves like
    a migration outage: thousands of schemas to alter in lockstep.
```

---

## Principles

1. **Row-level tenancy with RLS is the default.**
2. **The tenant identifier is a required, branded parameter in the data layer.**
3. **Global reference data is a different, explicitly-typed repository class.**
4. **RLS is the backstop.** Application discipline is the first line, not the
   only line.
5. **"Not found" and "not yours" produce identical responses.**
6. **Every per-tenant unique index includes the tenant column.**
7. **Ownership is verified locally before any external provider call.**
8. **Every layer scopes**: cache, queue, storage, search, logs, exports.
9. **Tenant appears in logs as a field, and in metrics only as a
   low-cardinality bucket.**
10. **Cross-tenant access attempts are tested in CI and metered in
    production.** A rising rate is either an attack or a bug, and you want to
    know which.
11. **Never expose an enumerable identifier** as a public resource identifier.
