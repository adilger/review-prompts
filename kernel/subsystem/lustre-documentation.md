# Lustre and LNet documentation

- When updating any of the user visible functionality in utilities, make sure relevant manual pages are correspondingly updated/added/removed in Documentation/man* directories.
- When updating lustre/utils/lib* for liblustre library, make sure the API manual pages in Documentation/man3 are kept up to date.
- When new tunable parameter is added with LDEBUGFS_SEQ_FOPS* or LPROC_SEQ_FOPS* or LUSTRE_{RW,RO,WO}_ATTR or modified, the corresponding manual page in Documentation/man4 should be added or modified as appropriate.

## Man pages belong in the same patch — `(minor)`

- A new user-visible tunable, command, or option should add/update its man page in the **same** patch that adds the functionality:
  - man1 for `lfs` sub-commands and other regular user tools
  - man3 for `llapi` library API functions
  - man4 for parameters added in procfs, sysfs, debugfs, modules
  - man8 for `lctl` sub-commands and other administrator tools
  so the documentation is reviewed against the implementation and not forgotten.  Flag new parameter/command/option with no man page change.
- If manual pages are added after a parameter or command change has landed, they should be marked with a `Fixes:` line referencing the original change.
- The `Added in commit` comment in the `AVAILABILITY` section can only have an accurate commit hash in a patch landed **after** the command or parameter has landed.  For man pages newly added in a patch this should only have the most recent parent tag, like `2.17.56`.
- For man pages created or modified after a feature or parameter has landed, the specific `git describe` hash should be used for `Added in commit`.

## Generic rules for Lustre man pages
- man pages should follow generally-accepted formatting (see `man-pages.md`)
- man pages should document a single command or parameter, unless there is substantial overlap between related commands/APIs.  They should **not** have large internal sub-sections for multiple sub-commands, those should be split into independent man pages and cross-referenced.
- if a man page contains descriptions for multiple related commands, the exact names should be accessible as separate pages and use a `.so` reference to the main page
- the `DATE` for any new/modified man page should have a recent date in it
- there should almost always be an `EXAMPLES` section in a Lustre man page.
- the `EXAMPLES` section should contain realistic parameter values and usage, not just random values that might match the syntax of the function.
- the `AVAILABILITY` section should list which Lustre release it was added in and have a comment with the exact commit (if landed, otherwise a placeholder)
- the `SEE ALSO` section should list related commands ordered first by numeric man page subsection, then alphabetically within the subsection
-- since Lustre man pages are still incomplete, it is acceptable to reference an external man page for a real command that does not yet exist but generate a `(minor)` warning.

## `lctl` and `lfs` sub-commands
- both `lctl` and `lfs` have many sub-commands and sub-sub-commands for different purposes
- the sub-commands and usually sub-sub-commands should get their own pages
- the `SYNOPSIS` section should use semantic variable names that describe the argument like `KILOBYTES` or `POOL_NAME`, not just `NUMBER` or `VALUE`
- the `OPTIONS` section should list all of the command line options and how to use them
- the `EXAMPLES` section should have at least one of each of the major ways of using a sub-command
- the `SEE ALSO` section should reference related (sub-)sub-commands and the parent command

## `llapi_` library API man pages
- should show the Lustre-specific header required to use this function, almost always `#include <lustre/lustreapi.h>` but may require others
- should properly format the `SYNOPSIS` section to highlight variable names
-- unlike other man page sections, function variable names should be in lowercase
- should describe enumerated constants, though not their specific values
- should describe public data structures and their fields
- should describe `RETURN VALUES` and `ERRORS` for the function(s)
- should always have a usage `EXAMPLE` section that show Lustre-specific headers and a reasonable example of its usage
- code examples should be technically correct, but do not need to contain a complete compileable program
