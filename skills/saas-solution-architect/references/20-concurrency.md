# Concurrency and race conditions

## 23.1 The reference implementationbase

The outbox worker claim is a textbook lease-based atomic claim and is the pattern to generalise:

```sql
UPDATE workers
   SET status = 'processing', locked_by = :instanceId,
       locked_at = now(), last_attempted_at = now(), updated_at = now()
 WHERE id = :id
   AND deleted_at IS NULL
   AND status IN ('pending', 'failed', 'limited')          -- valid source states
   AND (locked_at IS NULL
        OR locked_at < now() - (:leaseTtlMs * INTERVAL '1 millisecond'))
RETURNING id, event_id, worker_type, attempt_count, max_attempts, …;
-- Zero rows ⇒ another runner owns it. No lock service, no coordination protocol.
```

Four properties make this correct:

1. **The claim and the read are one statement.** There is no window between deciding and acting.
2. **The lease expires**, so a runner that dies mid-work does not block the row forever.
3. **`RETURNING` provides the work item**, so the claim doubles as the fetch.
4. **Valid source states are in the predicate**, so an already-terminal row cannot be re-claimed.

**Principle.** *A distributed claim is a conditional `UPDATE … RETURNING` with a lease expiry. This handles the overwhelming majority of distributed-coordination needs in a SaaS, requires no additional infrastructure, and is correct by construction.*

## 23.2 The recurring race shapes, and the fix for each

| Shape | Where it appears | Correct mechanism |
|---|---|---|
| **Check-then-act on a limit** | Quota checks, seat limits, rate limits | Atomic increment with the limit in the predicate (`ON CONFLICT … WHERE used < limit`) |
| **Read-check-write on a state field** | Terminate, cancel, approve, publish | Conditional `UPDATE … WHERE status = :expected` |
| **Read-modify-write on a balance** | Charge, refund, credit | Append-only entry with the balance predicate in the `INSERT … SELECT` |
| **Concurrent read-decide on two counters** | Completion detection | Single atomic operation returning the new value, or a Lua script |
| **Duplicate message delivery** | Every queue consumer | Idempotency key with a unique constraint |
| **Duplicate webhook delivery** | Every webhook receiver | Provider event id with a unique constraint |
| **Duplicate scheduled execution across replicas** | In-process cron | Distributed lease with compare-and-delete release |
| **Duplicate client submission** | Any create mutation | Client-supplied `Idempotency-Key` |
| **Terminal action running twice** | Teardown, finalise, settle | Idempotency latch (`SET NX`) around the terminal action |
| **Lost update on a mutable list** | Any read-append-write of a serialised collection | A real collection type with atomic append, or a child table |

**The last shape deserves emphasis** because it is easy to introduce accidentally. Any pattern that reads a delimited or serialised value, appends to it in application code, and writes it back is a lost-update race: two concurrent appends each read the same base value and one overwrites the other.

```
✗ value = read(key)                       # "a|b"
  write(key, value + "|c")                # concurrent writer's "|d" is lost

✓ atomic append:   RPUSH key "c"          # Redis list / set / sorted set
✓ or normalise:    INSERT INTO items (parent_id, value) VALUES (:parent, 'c')
                   # a child row per element: no read-modify-write at all
```

**Principle.** *A collection stored as a single mutable serialised value cannot be appended to concurrently. Use a data structure with an atomic append, or normalise it into rows.*

## 23.3 Choosing a concurrency-control mechanism

```
Is the operation expressible as one statement whose predicate
encodes the precondition?
  → YES: conditional UPDATE / INSERT … WHERE. Use this. It is free and correct.

Must you read, compute in application code, then write the same rows?
  → SELECT … FOR UPDATE, inside a transaction, with a consistent lock
    ordering across all code paths to avoid deadlocks.

Is the resource outside the database (an external provider, a file)?
  → Distributed lease: SET NX with a TTL exceeding the maximum work
    duration, released by compare-and-delete, and safe if it expires early.

Is contention high and conflicts rare?
  → Optimistic concurrency: a version column, retry on conflict.

Is contention high and conflicts common?
  → Serialise deliberately: partition the work so one worker owns a key
    range, or queue per-entity so the same entity is never processed twice
    concurrently.

Is the operation naturally idempotent?
  → Do nothing. This is the best possible answer, and it is worth
    restructuring an operation to achieve it.
```

## 23.4 Concurrency principles

1. **Every write path is examined for concurrent execution during review.** The question is asked explicitly, not assumed.
2. **Prefer atomicity to locking.** A conditional write beats a lock; a lock beats a distributed lock.
3. **Preconditions belong in predicates**, not in preceding reads.
4. **Every distributed lock has a TTL** and is safe to lose.
5. **Release a lease only if you still hold it** — compare-and-delete, never blind delete.
6. **Every consumer is idempotent.** At-least-once delivery is the only delivery guarantee available.
7. **Every terminal action is guarded by an idempotency latch.**
8. **Atomic increments do not make read-decide-act atomic.** Use the returned value, or a script.
9. **Consistent lock ordering** across all code paths, documented, to prevent deadlocks.
10. **Test concurrency explicitly**: fire N simultaneous requests at every limit, transition, and charge, and assert the invariant. This class of bug is invisible to sequential tests and is the reason it survives review.
