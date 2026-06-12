# OSD / dt-object API Details

- OSD API guidelines are located in `Documentation/osd-api.txt` file.

## Transaction declare/execute pairing — `(defect)`

The dt/OSD layer is two-phase: every modification done inside a transaction must
first be **declared** so credits and quota are reserved. This is the single most
common OSD bug class in the fix history.

- For every `dt_<op>()` that runs in the transaction (`dt_insert`, `dt_delete`,
  `dt_ref_add`/`dt_ref_del`, `dt_create`/`dt_destroy`, `dt_xattr_set`,
  `dt_write`, `dt_attr_set`, `dt_declare_*` index ops, etc.) there must be a
  matching `dt_declare_<op>()` on the same object **before** `dt_trans_start()`.
  Flag any execute-phase operation with no corresponding declare — it under-
  reserves credits and can fail or corrupt mid-transaction.
- The declare must match the execute in object, key, and magnitude. Watch for:
  - declaring against the wrong object (e.g. declaring on the source but writing
    the target — only the declared object is guaranteed reserved);
  - quota/credit estimates that assume a fixed extent-tree depth, non-sparse
    layout, or that uid/gid won't change between declare and execute;
  - a new operation added to the execute phase without updating the declare.
- Conversely, if `dt_trans_create()` itself fails there is nothing to clean up —
  return directly rather than `goto` a cleanup label that frees an unacquired
  handle.

## Block allocation / IO races — `(defect)`

- A newly allocated block must be zeroed (or fully written) before async IO is
  submitted against it, or stale disk contents leak. Check the
  allocate→memset/zero→submit ordering and the serialization (`i_append_sem`)
  when a write meets a not-yet-allocated block.
- Preallocation position (`pa_pstart`) must be preserved across mballoc cache
  eviction; flag logic that recomputes it from a possibly-evicted cache.

## ldiskfs kernel-patch drift — `(defect)`

The bundled ext4→ldiskfs patch series under `ldiskfs/kernel_patches/` is rebased
per kernel version and is a recurring source of typos.

- When a patch is added for a new kernel, diff it against the adjacent kernel's
  copy: a single wrong operator (`=` vs `+=`) or feature-flag check
  (inode-local flag vs superblock feature) silently corrupts behavior.
- A new autoconf feature test for an ext4/VFS API must include the right headers
  (e.g. `#include <linux/fs.h>`), or it fails to compile and the feature is
  silently treated as absent — see review-core.md TASK 2.2.
