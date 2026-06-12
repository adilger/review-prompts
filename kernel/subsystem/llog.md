# llog (Lustre log) review checkpoints

The llog is an on-disk record log (`obdclass/llog*.c`, `lustre_log.h`) used for
config, changelogs, update logs (DNE), and unlink/setattr replay. Its records
are on-disk and travel between nodes, so it has both endianness and
crash-consistency constraints. Distilled from two years of `Fixes:` commits.

## Record format and endianness

- llog record headers (`llog_rec_hdr`, `llog_rec_tail`: `lrh_len`, `lrt_len`,
  `lrh_index`, `lrh_type`) come from disk/network and must be **swabbed before
  use**, especially before they are used to compute an offset or length. Flag a
  record length/index consumed before the swab. (This is the wire-protocol swab
  rule applied to on-disk records — see wire-protocol.md.)
- A new llog record type must keep `wiretest`/`wirecheck` and any swab routine in
  sync, like any wire structure.

## Bitmap and index crash-consistency

- The header bitmap, `llh_count`, and `llh_cat_idx` must be updated in an order
  that survives a crash and matches reader expectations. Persist `llog_cat_idx`
  to disk when it changes (e.g. inside the cancel path), not just in memory, or
  it is lost across restart.
- Watch the ordering between decrementing `llh_count` and reading/updating
  `llh_cat_idx`; a reader racing a cancel must not see an inconsistent pair.
- Handle zero-filled gaps in a sparse llog file (records lost to `ENOSPC` or a
  thread race) by skipping the gap, not by misreading zeros as a record.

## Handle lifecycle and remote objects

- Protect a `llog_handle` across unlock/refresh boundaries with the
  get/put reference helpers; a pointer cached across `llog_cat_refresh()` can be
  freed under you.
- For a **remote** llog object (DNE update logs), creation is asynchronous: do
  the object create/`prep_log` in the **declare** phase and re-check `LOHA_EXIST`
  rather than assuming the local synchronous path. Flag a remote llog object
  used in the execute phase without a declare-phase create.

## Error-code semantics

- EOF from `llog_*_next_block()` is reported differently across versions
  (`-EBADR` vs `-EIO`); a reader must accept both to stay interop-compatible.
  Flag code that treats only one as end-of-log.
- Expected, benign errors (`-ENOENT`, `-ESTALE` on a missing/stale catalog entry)
  should be logged at `CDEBUG(D_HA)` level, not `CERROR`. Flag a noisy `CERROR`
  on an expected-missing record.
