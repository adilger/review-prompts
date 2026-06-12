# Lustre and LNet utils, liblustre library documentation info

- When updating any of the user visible functionality in utilities, make sure relevant manual pages are correspondingly updated/added/removed in Documentation/man* directories.
- When updating lustre/utils/lib* for liblustre library, make sure the API manal pages in Documentation/man3 are kept up to date.
- wirecheck.c, wiretest.c and wirehdr.c are special test files taht don't need to be documented.

## Man pages belong in the same patch — `(minor)`

- A new user-visible tunable, command, or option should add/update its man page
  (man4 for `lctl` parameters, man8 for tools) in the **same** patch that adds
  the functionality, so the documentation is reviewed against the implementation
  and not forgotten. Flag a new parameter/command/option with no man page change.
- Add the `.so` man-page reference/symlink files for each new parameter
  (e.g. a `man4/<module>.<param>.4` that does `.so man8/<tool>.8`).
- This applies to module parameters added anywhere in the tree, not only to code
  under `{lnet,lustre}/utils/` — wherever a user-visible knob is introduced.

## Userspace tool checkpoints — `(defect)` unless noted

Distilled from two years of utils `Fixes:` commits:

- **getopt / command tables must be NULL-terminated.** `struct option
  long_opts[]` must end with `{ .name = NULL }`, and `command_t` arrays passed to
  `cfs_parser()` must end with a `{ .pc_name = NULL }` sentinel — a missing
  sentinel segfaults on an unknown option/command.
- **Use `ssize_t`, not `size_t`, for `read()`/`write()` results**, and handle the
  negative error before any unsigned comparison (an error return compared as
  unsigned reads as a huge positive).
- **Clean up on every error path**: each `malloc`/`calloc` needs a matching
  `free()` on all returns (use `goto out_<var>` labels). Only `close(fd)` when
  `fd >= 0`.
- **errno sign discipline**: store `rc = -errno`, and pass the matching sign to
  `strerror()` (`strerror(-rc)`); a negative argument to `strerror()`/an
  unsigned compare is a real bug.
- **Parse NIDs with the `cfs_nidstr_*` helpers**, not `strchr(':')` — IPv6 and
  large/multi-rail NIDs contain colons; reserve `MAXNIDSTR`/`LNET_NIDSTR_SIZE`.
- **Verify each dispatch-table entry calls its intended `jt_*` handler** — a
  copy-pasted row that points at the wrong function silently runs the wrong
  subcommand.
- `(style)` **Output consumed by scripts must keep stable delimiters**: don't drop
  the field separator for a single value, and print a header only when there is
  content (`count > 0`, not `>= 0`). Tools and tests parse this output.
- `(minor)` Handle `-V`/`--version` before option parsing so it works without the
  otherwise-mandatory arguments.
