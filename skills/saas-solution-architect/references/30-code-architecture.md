# Code architecture and design patterns

## Part 29 — Code architecture

### 29.1 The layered structure

Every service follows an identical layout, applied consistently at scale:

```
src/
  api/           routes + controllers      (REST and webhooks)
  graphql/       typeDefs + resolvers      (the primary API)
  middlewares/   guards · validators · error handlers
  services/      business logic · transaction boundaries
  repositories/  data access
  models/        ORM definitions
  dto/           API-shaped data transfer objects
  interfaces/    contracts and types
  validators/    input schemas
  providers/     external system adapters
  producers/     message publishers
  consumers/     message consumers
  factories/     construction and registries
  packages/      shared internal libraries
  helpers/ utils/ enums/ constants/ config/ locales/
  migrations/ seeders/
  tests/
```

**This consistency is a genuine achievement.** An engineer who learns one service can navigate any of them. In a system this size, that property is worth more than most individual design decisions.

### 29.2 Judging the structure against design criteria

| Criterion | Assessment | Evidence |
|---|---|---|
| **Cohesion** | Good within layers | Each directory has one clear responsibility |
| **Coupling — code** | Good | Business logic depends on adapters, not vendor SDKs |
| **Coupling — data** | Poor | Shared tables and duplicated models across services |
| **Single responsibility** | Good at the layer level; some very large service files | Files exceeding 2,000 lines exist |
| **Dependency inversion** | Partial | Adapters invert external dependencies; the data layer exposes ORM types |
| **Testability** | Mixed | Direct instantiation (`new Service()`) inside methods hinders substitution |
| **Navigability** | Excellent | Identical structure everywhere |
| **Type safety** | Good, with escape hatches | Broad typing; some `any` and suppression comments in generic code |

### 29.3 Reference flow: dependency injection without a framework

The dominant construction pattern is direct instantiation inside methods:

```ts
✗ class OrderService {
    async place(input: Input) {
      const inventory = new InventoryService();     // constructed inside the method
      const payments  = new PaymentService();       // cannot be substituted in a test
      // …
    }
  }
```

This makes unit testing require module-level mocking, and it hides the dependency graph. Constructor injection with defaults solves it without adopting a framework:

```ts
✓ class OrderService {
    constructor(
      private readonly orders    = new OrderRepository(),
      private readonly inventory = new InventoryService(),
      private readonly payments  = new PaymentService(),
) {}

    async place(input: Input) { /* uses this.inventory, this.payments */ }
  }

  // Production: new OrderService()                       — defaults apply
  // Test:       new OrderService(fakeRepo, fakeInv, fakePay)   — no module mocking
```

The pattern is already used in parts of a system of this class and is worth making universal. It costs nothing, requires no container, keeps the production call site unchanged, and makes the dependency graph visible in the constructor signature.

**For a larger system**, a lightweight container (`tsyringe`, `typedi`) adds lifecycle scopes and removes the manual wiring — the closest Node equivalent to Spring's application context. Adopt it when the number of collaborators per service makes manual wiring the bottleneck, not before.

**Principle.** *Dependencies are declared in the constructor with production defaults. A class that constructs its own collaborators inside a method cannot be tested without mocking the module system.*

### 29.4 Reference flow: a data-access abstraction that actually abstracts

A generic base repository over the ORM provides uniformity, type parameters, and transaction awareness — real value. Its limitation is that when the interface exposes ORM types (`WhereOptions`, `IncludeOptions`, `Order`, `Transaction`), the abstraction does not *insulate* callers from the ORM; it only relocates the coupling.

```ts
✗ findAll(opts: { where?: WhereOptions; include?: IncludeOptions[]; order?: Order }): Promise<T[]>
    // every caller now depends on Sequelize's type vocabulary

✓ // Domain-shaped query objects, translated inside the repository
  interface Page<T> { items: T[]; nextCursor: string | null }
  interface RecordQuery {
    status?:  RecordStatus[];
    since?:   Date;
    search?:  string;
    cursor?:  string;
    limit?:   number;
  }

  class RecordRepository extends TenantRepository<Record> {
    // The signature is domain vocabulary. The ORM lives entirely inside.
    async search(tenantId: TenantId, q: RecordQuery): Promise<Page<Record>> { … }
  }
```

Two further refinements:

- **Prefer explicit, purpose-named finders** (`findActiveByOwner`) over a generic pass-through `findAll`. A named finder documents the access pattern, can be indexed deliberately, and is a place to enforce invariants.
- **Type generics precisely.** `Partial<T[]>` describes a partial *array*, not an array of partial objects (`Partial<T>[]`) — a distinction that silently disables type checking on bulk-write inputs.

**Principle (Ousterhout's deep modules).** *A module's value is the ratio of the complexity it hides to the size of its interface. A data-access layer that re-exports its ORM's types is a shallow module: it adds a layer without hiding anything. Design the interface in the domain's vocabulary and keep the ORM entirely behind it.*

### 29.5 Reference flow: shared code as a published package

Shared libraries — authorization, pagination, query building, resilience, task chains — are vendored into each service by copying the directory. Copying is the fastest way to share code once and the most expensive way to share it thereafter: each copy evolves independently, so a fix in one does not reach the others, and the copies drift apart in ways nothing detects.

```
✗ service-a/src/packages/authorize/…      ← copy 1
  service-b/src/packages/authorize/…      ← copy 2, now different
  …
     · a security fix must be applied N times, correctly, by hand
     · behaviour differs between services in ways nothing reports
     · no version history for the shared component itself

✓ @company/authorize      v2.3.1     ← one source, versioned, changelogged
  @company/pagination     v1.4.0
  @company/resilience     v3.0.2
  @company/db-contracts   v5.1.0     ← generated types for shared tables

     · published to a private registry (npm, CodeArtifact, GitHub Packages)
     · semantic versioning: a breaking change is visible in the version
     · services upgrade deliberately, one at a time
     · a security fix is one release plus N dependency bumps — and the
       bump is mechanical and auditable
     · the package has its own tests, run once, in its own pipeline
```

Two intermediate options, if a registry is genuinely not yet available:

| Approach | Cost | Benefit |
|---|---|---|
| **Monorepo with workspaces** | Restructuring | One source of truth; atomic cross-service changes |
| **Git submodule / subtree** | Awkward workflow | One source of truth without a registry |
| **Private registry package** | Registry setup (~1 day on AWS CodeArtifact) | Independent versioning and adoption pace |

The registry option is the recommended one, and the setup cost is small relative to maintaining N copies of security-critical code.

**Principle.** *Shared code is a versioned package with its own tests and its own release process. Copying is acceptable only for genuinely single-use scaffolding — never for anything security-relevant, and never for anything with more than two copies.*

### 29.6 Reference flow: keeping generic infrastructure type-safe

Generic infrastructure — pipelines, task managers, dispatchers — is where type safety is most often abandoned, and it is where the consequences are largest because the code is reused everywhere.

```ts
✗ async execute(id: string, ctx: C): Promise<R> {
    for (const i in this.tasks) {              // for…in over an array: string keys
      result = await this.tasks[i].execute(id, ctx);
      // @ts-ignore
      if (result.stop) return result;          // reading an untyped property
    }
  }

✓ // Constrain the result type so control-flow properties are part of the contract
  interface TaskResult { stop?: boolean; success: boolean }

  class TaskManager<C, R extends TaskResult> {
    private readonly tasks: Task<C, R>[] = [];

    add(task: Task<C, R>): this { this.tasks.push(task); return this; }   // append
    set(tasks: Task<C, R>[]): this { this.tasks.splice(0, Infinity, ...tasks); return this; }

    async execute(id: string, ctx: C): Promise<R | undefined> {
      let result: R | undefined;
      for (const task of this.tasks) {          // for…of: real elements, in order
        result = await task.execute(id, ctx);
        if (result.stop) return result;         // typed; no suppression needed
      }
      return result;
    }
  }
```

Three details worth generalising:

1. **Constrain generics** (`R extends TaskResult`) so the infrastructure can read what it needs without suppression.
2. **`for…of` over arrays**, never `for…in`, which iterates string keys and includes any enumerable properties.
3. **Name mutation clearly.** `add` appends; `set` replaces. A method whose name suggests one and does the other loses work silently.

### 29.7 Code architecture principles

1. **One structure, applied everywhere.** Consistency beats local optimality at scale.
2. **Layer responsibilities are strict**: transport / policy / orchestration / business logic / data access / adapters.
3. **Dependencies are constructor parameters with production defaults.**
4. **The data layer's interface uses domain vocabulary, not ORM types.**
5. **Shared code is a versioned package.**
6. **Generic infrastructure is fully typed**; suppression comments are a review blocker there specifically.
7. **A file over ~500 lines is a design signal**; over 1,000 it is a refactor candidate.
8. **`for…of` over arrays; never `for…in`.**
9. **Method names state their effect** — `add` versus `set`, `find` versus `findOrFail`.
10. **Prefer named, purpose-specific finders** to generic query pass-throughs.

---

## Part 30 — Design patterns actually in use

Included only where the implementation genuinely embodies the pattern.

| Pattern | Implementation | Problem solved | Benefit | Trade-off | General SaaS applicability |
|---|---|---|---|---|---|
| **Transactional Outbox** | Outbox tables + fan-out + claim + sweeper | Atomicity between state change and event publication | No lost events; full audit trail | Extra tables, a sweeper deployable, latency | **DEFAULT** for any state change with a side effect |
| **Repository** | Generic base + per-entity repositories | Uniform data access | Consistency; a place to enforce tenant scope | Shallow if it exposes ORM types | **DEFAULT** |
| **Adapter** | One `providers/` module per external system | Isolating vendor contracts | Vendor churn confined to one module | An extra indirection | **DEFAULT** |
| **Strategy** | Pluggable routing and handling strategies selected from configuration | Behaviour varying by tenant configuration | New behaviour without touching callers | Indirection; harder to trace | **DEFAULT** where behaviour is configurable |
| **Chain of Responsibility / Pipeline** | Task and filter managers with ordered steps | Composing multi-step operations | Small testable steps; declarative order | Control flow spread across classes | **DEFAULT** for multi-step flows |
| **Saga with compensation** | Task chains implementing `rollback` | Multi-system operations that cannot be transactional | Partial failures are cleaned up | Compensation is best-effort and needs care | **OPTIONAL** — when a flow spans systems |
| **Factory / Registry** | Worker, producer, and consumer factories; a database-backed handler registry | Dynamic resolution of implementations | Extension without modifying callers | Indirection; needs CI validation | **DEFAULT** for plugin-style extension |
| **Singleton** | Connection, client, and pub/sub instances | Expensive resource reuse | Connection pooling; bounded resources | Hidden global state; test isolation | **DEFAULT** for connections; avoid for logic |
| **State Machine** | Enumerated statuses with atomic transitions | Preventing invalid lifecycle transitions | Reviewable, observable lifecycles | Ceremony for simple entities | **DEFAULT** beyond three states |
| **Circuit Breaker** | Implemented in the resilience package | Retry storms against a failing dependency | Fast failure; recovery headroom | Tuning; false trips | **DEFAULT** wherever there is retry |
| **Bulkhead** | Implemented in the resilience package | One dependency starving all concurrency | Fault isolation | Capacity planning per dependency | **OPTIONAL** — when dependencies vary in latency |
| **Middleware / Decorator** | The resilience pipeline; Express and Apollo plugins | Composable cross-cutting concerns | Orthogonal, ordered, reusable | Order-dependence must be documented | **DEFAULT** |
| **DTO** | Explicit DTO layer | Decoupling the API contract from the schema | Schema changes do not leak to clients | Mapping code | **DEFAULT** |
| **Cursor pagination** | Shared pagination package | Deep-page cost; unstable page boundaries | Constant cost; stable results | Opaque cursors; no random page access | **DEFAULT** |
| **Ambient context** | `AsyncLocalStorage` request context | Propagating correlation and tenant | Clean signatures; reliable correlation | Implicit data flow | **DEFAULT** |
| **Double-entry ledger** | Journal entries with credit/debit | Auditable, reconstructable money | Full auditability; immutable history | Balance requires projection | **DEFAULT** for any money movement |
| **Idempotent consumer** | Idempotency key with a unique constraint | Duplicate message delivery | Safe at-least-once processing | An extra table and constraint | **DEFAULT** for every consumer |
| **Lease-based claim** | Conditional `UPDATE … RETURNING` with lease expiry | Distributed work coordination | Exactly-once execution without a lock service | Lease tuning | **DEFAULT** for job processing |

### Patterns not present, and where each would earn its place

| Pattern | Where it would help | Priority |
|---|---|---|
| **CQRS (read models)** | Analytics and reporting currently query the write primary | High |
| **Event sourcing** | Not warranted; the outbox plus an append-only journal already provides the audit properties that matter | Low |
| **API gateway policy layer** | Centralised rate limiting, request size limits, and standard security headers | Medium |
| **Service mesh** | Not warranted at this service count | Low |
| **Feature flags** | Decoupling release from deploy; safe progressive rollout | High |
| **Dependency injection container** | Testability; explicit dependency graphs | Medium |
| **Schema registry for messages** | Queue payloads are currently unversioned contracts | Medium |
| **Backend-for-frontend** | Not needed; federation already fulfils this role | Low |
