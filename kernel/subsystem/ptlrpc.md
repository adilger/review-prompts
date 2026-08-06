# ptlrpc and RPC callback rules

ptlrpc is the RPC engine. Code that runs in RPC completion / event context is
latency-critical and runs in constrained contexts. Analyze any new or changed
callback for blocking behavior.

## Callbacks must terminate quickly

RPC reply interpreters and the various ptlrpc callbacks
(`rq_interpret_reply` / `ptlrpc_interpterer_t`, `set_interpret`,
completion/commit callbacks, LNet event callbacks, ldlm completion/blocking/glimpse
ASTs) run on shared service threads or in soft-IRQ-like context. They must:

- return quickly and do bounded work
- **never wait for another RPC to complete**, synchronously or by blocking on a
  reply, lock, or condition that itself depends on RPC progress — this can
  deadlock the RPC pipeline (the thread you are blocking is the one that would
  make progress)
- never block on an unbounded task: no waiting on user space, no unbounded
  loops, no `wait_event` on a far-away condition, no synchronous I/O

Scheduling deferred work is acceptable (though not ideal): handing the slow
work to a workqueue, a ptlrpcd, or a separate thread, and returning, is fine.
Doing the slow/blocking work *inline* in the callback is a regression.

Questions to raise:
- Does this callback call into something that can wait for an RPC reply
  (`ptlrpc_queue_wait`, a synchronous `*_sync` send, `l_wait_event` on RPC
  completion, taking a lock held across an RPC)?
- Does it allocate with a blocking/`GFP_KERNEL` flag in a context that must not
  enter reclaim or sleep?
- Could it loop for an unbounded number of iterations driven by remote input?

If the work cannot be made bounded and non-blocking, the correct pattern is to
defer it; flag callbacks that block inline instead.

## Request handlers must pack a reply on every return path

A server request handler that returns 0 (success) **must** have packed its reply
buffer. `tgt_handle_request0()` LBUGs with "handler did not pack reply but
returned no error" otherwise — a 100% crash that has been shipped and reverted.

- Trace every return path of a new/modified handler: each path that returns 0
  must reach the `req_capsule_server_pack()` / reply-fill code. An early
  `RETURN(0)` that skips packing is a defect.
- When adding a field to a reply, the server must grow/repack the reply buffer
  (the request format / `req_capsule` size) to match; a reply written larger
  than the packed buffer is corruption. (See wire-protocol.md for the format
  side.)

## Reply interpreters must validate the reply before using it

A client reply interpreter must confirm a reply actually arrived and is the
expected type before dereferencing it:
- check `ptlrpc_client_replied()` / `req->rq_repmsg` is non-NULL and
  `lustre_msg_get_type()` is not `PTL_RPC_MSG_ERR` before
  `req_capsule_server_get()`. Flag an interpreter that reads the reply body on
  the error/no-reply path.

## Allocation context on server / callback paths

- Use `GFP_NOFS` (not `GFP_KERNEL`) for allocations on MDT/OST/target request
  handling, ptlrpc, and lock-callback paths. This is the default for
  `OBD_ALLOC_*` and `LIBCFS_ALLOC_*` functions.
  Using `GFP_KERNEL` there can re-enter the
  filesystem through direct reclaim and deadlock. Flag `GFP_KERNEL` reached from
  a server handler or an LDLM/RPC callback.

## Import / export lifetime

- An import can be torn down and reconnected (failover, umount races). Code that
  caches or dereferences `imp` / `exp` across a wait, reconnect, or umount must
  re-check it for NULL/staleness. Flag an `obd_import` dereference on a path that
  can race `class_destroy_import()` / umount without a guard.

## Guard against double-completion / re-invocation

- When work can be triggered more than once on the same object, or a cleanup can
  run from both a synchronous path and an async callback, use a single state flag
  (set-and-test) to ensure it runs once. Recurring bugs: double `*_putref` /
  unlock after an async deref, and recursive callback storms (e.g. repeated DOM
  discard) that overflow the stack. Flag an async deref/cleanup with no
  "already-done" guard.

## Reachability of RPC paths

When the commit message describes a feature negotiated at connect time, confirm
the changed RPC path is actually reachable for the relevant connect flags and
peer versions (see wire-protocol.md). A handler gated behind a flag the peer
never sets is effectively dead for that workload.
