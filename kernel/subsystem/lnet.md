# LNet and LND review checkpoints

LNet (`lnet/`, `libcfs/`) and the LNDs (`o2iblnd`, `socklnd`/`ksocklnd`,
`gnilnd`, `kfilnd`) are networking code with their own recurring bug classes,
distilled from two years of `Fixes:`-tagged commits and reverts. Most are
`(defect)`.

## NID parsing and formatting

- A NID is not a bare IP: it is `address@net`. Parsing that splits on `:` or
  `@` naively breaks IPv6 (colons in the address) and large/multi-rail NIDs. Use
  the `cfs_nidstr_*` / `cfs_nidstr_find_delimiter()` helpers and `struct
  lnet_nid` (the large-NID type), not `strchr(':')` or a fixed 4-byte address.
- Reserve enough buffer for NID lists (`LNET_NIDSTR_SIZE`, `MAXNIDSTR`); flag a
  fixed stack buffer that assumes one short NID.

## Concurrency / context

- **No sleeping allocation in a Netlink dump callback or an RCU read section.**
  These run in atomic/RCU context; use `GFP_ATOMIC` or move the allocation
  outside the locked region. Flag `GFP_KERNEL` / `GFP_NOFS` / a blocking alloc
  reached from a `*_show_dump` / `genl` dump callback or inside
  `rcu_read_lock()`.
- **Don't nest `rtnl_lock()` inside `rcu_read_lock()`** — only the `*_rcu`
  variants (`dev_get_by_name_rcu`) may run under RCU; take `rtnl_lock()` outside.
- Reads of peer / NI state (`lp_state`, `ni_state`) must hold the right lock;
  flag a lockless read of state that another thread mutates.

## Teardown / lifetime

- On teardown, **restore kernel callbacks before freeing the resource**: e.g.
  restore `sk_data_ready`/`sk_write_space` before `sock_release()`, so a late
  callback can't touch freed memory. Flag free-then-restore order.
- Refcount discipline on peers, routes, connections, and message descriptors:
  every `*_decref` must match an addref, and error paths must not decref
  something never referenced (asymmetric-route decref was a real bug). Flag an
  unbalanced or error-path-only decref.
- LNet/LND callback changes must not hang or leak on `lustre_rmmod` — a callback
  that defers cleanup must still let module unload complete.

## Correctness traps that recur

- **Set `rc` before `goto`/`return` on the error path.** A path that detects
  failure but leaves `rc` at 0 (or at a stale positive value from
  `nla_strscpy`/`*_rcu` returning a length) reports success. Flag it.
- **`strncpy` does not NUL-terminate** a truncated string — use `strscpy`.
- **Symmetric counters/buffers get swapped**: PUT vs GET stats, rx vs tx buffer
  sizes (`sk_rcvbuf` vs `sk_sndbuf`), local vs global NI index. When two
  near-identical assignments differ only by direction, check they aren't
  transposed.
- **Validation in the wrong brace scope** becomes dead code — a check placed
  after the `else`/`switch` it was meant to guard never runs. Verify a new
  bounds/permission check is actually on the path it protects.
- Check zerocopy/sendpage preconditions (`sendpage_ok()`) before using
  `sendpage`/`MSG_SPLICE_PAGES`; non-sendpage-able pages must take the copy path.

## Peer/discovery state machine

- Peer discovery and health transitions are flag-driven; check flags in the
  right order and clear them at the right time (a mis-ordered `FORCE_PUSH` /
  `PUSH_SENT` check sent the push twice). When adding a state, trace every
  transition that reads or clears it.
- A NID/peer filter (e.g. dropping "non-uptodate" peers, or `nettype` filtering)
  must always leave at least one usable peer/route, or failover breaks. Flag a
  filter with no guaranteed fallback.
