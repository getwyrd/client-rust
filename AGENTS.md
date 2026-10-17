# AGENTS.md — client-rust (getwyrd fork, `integration` branch)

Context for coding agents working on this tree, including the leaves of the
`client-rust-pdca` loop. This file lives on the fork's `integration` branch only; it is
not part of any upstream PR.

## Review rubric & protocol

The standing rubric the loop's builder, reviewer and adversary apply
(`client-rust-pdca` `pdca.toml` `rubric_file`). The client rules are condensed from
client-rust-test's [`docs/client-rules.md`](https://github.com/getwyrd/client-rust-test/blob/main/docs/client-rules.md),
the normative source: on any disagreement that file wins, and this section is stale. The
same section lives in client-rust-test's `AGENTS.md`; keep the two in step.

### Hard conventions (MUST)

- **One concern per change, upstream-shaped.** A loop bundle must stay cherry-pickable onto
  `tikv/client-rust` `master` on its own: no unrelated reformatting, no drive-by refactors.
- **Never emulate a missing capability.** A gap closes by sending the request the server
  implements (the same kvproto request client-go sends), never by computing the answer on
  the client side.
- **Errors stay errors.** An unrecognized or unexpected server error must surface as an
  error; it never becomes `Ok`, `None` or a default value (client rule ER-1).
- **Every await on a network path is bounded.** A new wait without a deadline is a defect.
- **No new dependency, no build-time code, no `unsafe`** without a human: those are R2
  changes the loop parks for review, and `build.rs` / `proto-build/` changes are refused.

### Client rules a change to tikv-client must not break (MUST; source: client-rust-test's `docs/client-rules.md`)

One line per rule, kept to the source's conditions. `*` marks a conditional MUST: it binds
only if the client uses that capability. Where a line and the source differ, the source
wins and this line is a defect.

- **TS-1** — every `start_ts` and `commit_ts` comes from PD's TSO, or from a
  TiKV-sanctioned derivation such as `min_commit_ts` inference — never a local clock.
- **TS-2** — `commit_ts > start_ts`, always.
- **WR-1** — exactly one primary key; its lock is the source of truth for the transaction,
  and every secondary lock points to it.
- **WR-2** — prewrite everything before committing anything; commit the primary before any
  secondary; until the primary commit is durable the transaction stays rollbackable.
- **WR-3** — if the primary commit's outcome is unknowable (e.g. an RPC timeout after
  send), report *undetermined*, never clean failure.
- **WR-4** — when the client knows a transaction failed, its prewritten and pessimistic
  locks are actively cleaned up, not abandoned to TTL.
- **WR-5\*** — async commit: list **all** secondaries in the primary's prewrite; adopt the
  server-derived commit ts; fall back to 2PC when the server declines (a
  `min_commit_ts == 0` response).
- **WR-6\*** — 1PC only for single-region transactions; when the server declines, fall
  back to 2PC.
- **WR-7\*** — 1PC / async commit: set and respect `max_commit_ts`; on
  `CommitTsTooLarge`, abort or fall back.
- **PL-1** — each locking statement carries its own `for_update_ts`, refreshed from TSO on
  conflict retry (statement retry, not transaction retry).
- **PL-2** — locks acquired for a statement that then fails are pessimistic-rolled-back
  promptly, not left to block others for a TTL.
- **RD-1** — a transaction reads its own uncommitted writes.
- **RD-2** — a read at ts T never ignores a lock with `start_ts ≤ T`: back off or clean up
  the lock, then retry.
- **RD-3\*** — stale read only against replicas whose `safe_ts ≥ start_ts`; on
  `DataIsNotReady`, retry elsewhere or fall back to the leader — never serve the read
  anyway.
- **LR-1** — a foreign lock is resolved through its primary: `CheckTxnStatus` on the primary
  decides, with an exact tri-state decode — locked (`lock_ttl > 0`), committed
  (`commit_version > 0`), rolled back (both zero).
- **LR-2** — never roll back a lock before its TTL expires unless the transaction's status
  is already known.
- **LR-3** — an expired primary with no lock and no write record is escalated with
  `rollback_if_not_exist`, leaving a rollback tombstone.
- **LR-4** — async-commit locks are never unilaterally rolled back: check all secondaries
  (`CheckSecondaryLocks`), then commit or roll back by the actual outcome.
- **LR-5** — `ResolveLock` carries the transaction's true outcome: `commit_version == 0`
  rolls back, `> 0` commits at that ts. Never guessed.
- **RE-1** — routing-stale errors (`NotLeader`, `EpochNotMatch`, `StoreNotMatch`,
  `RegionNotFound`, `StaleCommand`, `MismatchPeerId`) refresh the region cache using the
  hints in the error and retry internally; they never surface as data errors.
- **RE-2** — back-off-and-resend errors (`ServerIsBusy`, honouring its `backoff_ms`;
  `MaxTimestampNotSynced`) are retried with backoff.
- **RE-3** — non-retryable errors fail fast (`FlashbackInProgress`: fail ASAP, never
  blind-retry).
- **RE-4** — retries are bounded and backed off; on exhaustion the region error surfaces as
  a region error, distinct from data errors.
- **ER-1** — never misreport an outcome: failure is not success, unknown is not known, and
  an unrecognized server error stays an error.
- **ER-2** — "key absent" is a result, not an error, and is distinguishable from "read
  failed".
- **ER-3** — honour `KeyError` routing: `locked` → back off / resolve and retry;
  `retryable` → the client may restart the txn; `abort` → the client aborts the txn.
- **GC-1** — before advancing a GC safepoint, scan and resolve **all** locks with
  `start_ts ≤ safepoint` (at or below it, the boundary included).
- **GC-2** — never advance the cluster safepoint past a snapshot some service still reads;
  on shared clusters that means service safepoints or the GC controller.
- **RAW-1** — no RawKV alongside TxnKV in API V1; API V2's keyspace prefixes make
  coexistence safe.
- **RAW-2** — a V1 client never sends V1TTL.
- **RAW-3** — CAS only in atomic mode, and atomic and non-atomic writes are never mixed on
  the same keys.

### Recurring defect classes (MUST check when the diff touches the surface)

- **Vacuous assertion** — a test that passes without binding the property (e.g. asserts
  the call returned, not what it returned). Mutants on the diff catch these.
- **Converging by the wrong mechanism** — a gap that closes because both sides now fail
  the same way, or because the subject computes locally what the oracle asks the server
  for. Check that the subject sends the same kvproto request the oracle sends.
- **Evidence loss** — a projection or driver change that removes a field from a trace.
- **Unbounded await** — a new await on a network path without a deadline.
- **Keys or values in logs** — log lines carry `start_ts` and identifiers, never user
  key or value bytes.

### Reviewer protocol

- Cite `path:line` on the target tree for every finding.
- Do not raise findings about `Signed-off-by` or commit metadata.
- A finding answered by a tracked issue is settled; do not re-raise it.
