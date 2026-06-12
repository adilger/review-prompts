# LDLM lock ordering and nesting

LDLM (the Lustre Distributed Lock Manager) deadlocks are among the most damaging
regressions in Lustre: a lock-ordering mistake can hang or cascade-evict a whole
cluster, and the bug is timing-dependent so tests rarely catch it. Treat any
change that enqueues an LDLM lock, changes lock order, lock mode, or lock bits,
or runs inside a lock callback, as deadlock-sensitive. Ordering/nesting
violations are `(defect)`.

The deadlock model: an LDLM lock holder must always be able to make progress so
it can answer the blocking AST (callback) and drop the lock. Anything that makes
a lock holder wait on a second lock — or on an RPC, or on a peer that is itself
blocked on the first lock — can close a cycle. Holding locks on two different
servers at once is especially dangerous: it "introduces inter-server dependency
and can lead to cascading evictions" (`cl_object.h`).

## Client side

### Do not enqueue a lock while holding another lock

The default rule: a client thread should not request (enqueue) a new LDLM lock
while it already holds an LDLM lock. `cl_object.h` states it is "generally
unsafe to actively use (hold) two locks on the different OST servers at the same
time". The CLIO design avoids this by collecting all locks an IO needs **up
front** (sorted), enqueuing them, and only then running the IO — rather than
acquiring locks lazily mid-IO while others are held.

Flag code that calls `cl_lock_request()` / `ldlm_cli_enqueue()` / an enqueue
path from inside an already-locked region, a lock callback, or an IO step that
runs after locks are taken.

### Lock callbacks must stay non-blocking and must not enqueue

Blocking, completion, and glimpse ASTs (`l_blocking_ast`, `l_completion_ast`,
`l_glimpse_ast`, e.g. `__osc_dlm_blocking_ast`) run in response context and must:
- never enqueue a new LDLM lock and never wait on an RPC reply;
- never take a page lock — "lock order is page lock -> object lock"
  (`osc_cache.c`); a callback that already holds the object lock must not then
  grab page locks. The blocking AST flushes/writes back pages without holding
  them (`osc_lock_flush`) and releases, it does not re-lock.
- The glimpse AST deliberately does **not** take the `cl_lock` mutex (see
  LU-1274 comment in `osc_lock.c`); it matches an existing lock with
  `LDLM_FL_TEST_LOCK` and queries size without lock protection.

This overlaps the general ptlrpc callback rule — see ptlrpc.md.

### Documented exceptions and their required ordering

These take a lock that may span the whole file/all stripes, but they take it
**once at IO setup**, not lazily while holding others:
- **O_APPEND write** — must take a `[0, EOF]` extent lock for atomicity
  (`cl_object.h`).
- **ftruncate** — must take a `[offset, EOF]` lock (`cl_object.h`).
- **Group locks / lockahead** — manually requested locks use
  `LDLM_FL_NO_EXPANSION` (`CEF_LOCK_NO_EXPAND`) so the server won't expand them
  into a conflicting range; each is its own IO, not nested.
- **Glimpse / speculative locks** — asynchronous, no anchor, do not wait for the
  server reply (`async = true`), so they cannot block an IO thread. Speculative
  requests "must be either no_expand or glimpse" (`osc_lock.c`).

For everything else, the LOV layer **sub-divides** a multi-stripe IO so each
piece "is fully contained within a single stripe ... This avoids cascading
evictions" (`Documentation/clio.txt`). Flag a change that makes a normal
read/write hold extent locks on more than one stripe/OST at once.

### Ordering and flags

- When an IO legitimately collects several locks, they are sorted (by object
  `lu_fid` then offset) before enqueue (`cl_io_lock`). A change that enqueues a
  set of locks in an unsorted/ad-hoc order is a defect.
- Flags that mark a legitimately non-deadlocking request:
  `LDLM_FL_TEST_LOCK` (no grant, just probe), `LDLM_FL_NO_EXPANSION` (no server
  expansion), `LDLM_FL_BLOCK_NOWAIT` (fail fast instead of queueing),
  speculative/glimpse (async, no waiter). If a nested or callback-context
  enqueue lacks one of these, question it.

## Server side (MDT / MDD)

### Canonical rule: lock multiple objects in FID order

To avoid ABBA deadlock when locking two unrelated objects, lock them in
ascending `lu_fid_cmp()` order (seq, then oid, then ver). `mdt_reint.c` does
exactly this at its `order_by_fid:` fallback: "To avoid AB/BA deadlock given two
phase locking, if `lu_fid_cmp(fid1, fid2) > 0` ... lock the greater FID second."

### …but ordering is now more nuanced — use the helpers

Pure FID order is only the fallback. Rename computes order in
`mdt_rename_determine_lock_order()` with this precedence, and new multi-object
code must follow the same model rather than re-deriving it:
1. **Parent before child** — if one object is an ancestor of the other (or one is
   the root), the ancestor/root is locked first.
2. **Striped-directory stripe index** — sibling stripes of the same striped
   (LMV) directory are locked in ascending master-MDT stripe index order.
3. **Same-directory rename** — order the two name locks by **PDO name hash**
   (`mlh_pdo_hash`, from `full_name_hash(name)`), not FID, to avoid ABBA between
   concurrent `a→b` / `b→a`.
4. **Children / hardlinks** — old vs new child are locked in FID order (the
   explicit reason is the hardlink deadlock fixed by LU-15491).
5. Otherwise fall back to FID order.

Don't hand-roll lock acquisition of two objects. Use the existing helpers and
flag code that bypasses them:
- `mdt_object_lock()` — object known to be local.
- `mdt_object_check_lock()` — when LOOKUP must be combined with other ibits and
  the parent/child may live on different MDTs (it splits the LOOKUP lock).
- `mdt_object_lock_try()` — non-blocking, for cases that can ABBA (link, migrate
  link parents); on failure, drop all locks, revoke the blocked one, and restart.
- `mdt_parent_lock()` — picks PDO vs regular parent lock automatically.
- `mdt_object_stripes_lock()` — for striped directories.

### The rename / big-filesystem lock (BFL)

Some operations serialize on the root-FID "rename lock" (`mdt_rename_lock()`,
`LUSTRE_BFL_FID`, `MDS_INODELOCK_UPDATE`): cross-parent directory rename, remote
source/target, etc. When BFL is needed it must be taken **before** the per-object
locks, and skipped on replay to avoid recovery deadlock. Flag a new
multi-object/cross-dir path that should serialize but skips the BFL, or takes it
after per-object locks.

### DNE / cross-MDT / remote objects

- Check `mdt_object_remote()` before choosing the lock type; remote objects use
  `mdt_remote_object_lock*` and must use try-lock + restart patterns, not
  blocking parent→child locks, to avoid cross-MDT ABBA.
- Never take the **same ibit on the same local object twice** (e.g. a separate
  LOOKUP lock on a local object you also lock for UPDATE) — that self-deadlocks;
  combine bits in one call instead.
- Remote directories split bits across MDTs: the master grants `UPDATE|PERM`, the
  MDT holding the name entry grants `LOOKUP`. Code assuming LOOKUP and UPDATE are
  always co-located is wrong under DNE.

### ibits and lock modes

- `MDS_INODELOCK_DOM` must **not** be combined with other ibits in a local lock
  (`LASSERT(!(*ibits & MDS_INODELOCK_DOM && *ibits & ~MDS_INODELOCK_DOM))` in
  `mdt_handler.c`) — it conflicts with the GROUP lock. Take DOM separately, often
  via trylock.
- Watch lock-mode correctness (LCK_EX / LCK_PW / LCK_PR) and that the ibits
  requested (LOOKUP / UPDATE / PERM / LAYOUT / XATTR / OPEN / DOM) match what the
  operation actually touches; over-broad modes/bits hurt parallelism, too-narrow
  ones miss protection.

## What to flag (summary)

- Client: enqueuing a lock while holding one (outside the documented exceptions);
  any enqueue, RPC wait, or page-lock acquisition inside a lock callback;
  a normal IO holding locks across multiple stripes/OSTs; unsorted lock sets.
- Server: locking two objects without going through the ordering helpers / FID
  order; wrong ordering precedence for rename/striped/same-dir/DNE; missing or
  mis-sequenced BFL; double-locking the same local object; combining DOM with
  other ibits locally; blocking (non-try) locks on paths that can ABBA across
  MDTs.

When a change adds a new lock or reorders locking, ask the reviewer's core
question explicitly: against every other path that locks these same objects,
is there a consistent global order? If you can construct two paths that take the
same two locks in opposite orders, that is the deadlock — report it.
