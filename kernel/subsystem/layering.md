# Client-side layering rules

The Lustre client is layered. The layering rule constrains which component is
allowed to touch kernel VFS / MM structures directly, and violating it is a
regression even when the code compiles and "works".

## The rule

On the client side, only `lustre/llite/` may directly touch kernel
VFS/MM-facing structures and call VFS/MM kernel functions — `struct inode`,
`struct dentry`, `struct file`, `struct address_space`, `struct page` /
`folio` cache manipulation, page-cache and writeback helpers, `iov_iter`,
mmap/`vm_area_struct`, etc.

The lower client layers — `lustre/mdc/`, `lustre/osc/`, `lustre/lov/`,
`lustre/lmv/` (and similarly `lustre/mgc/`, `lustre/fld/`, `lustre/fid/`) — must
**not** reach into VFS/MM kernel structures directly. They are supposed to call
*up* into llite through the established Lustre methods/operations (the CLIO
slot operations, `cl_*` interfaces, lu_object operations, registered
callbacks) and let llite perform the kernel-facing work.

General kernel primitives are exempt: spinlocks, mutexes, atomics, lists,
workqueues, timers, basic allocators, string/bit helpers, etc. may be used
anywhere. The restriction is specifically about VFS- and MM-level objects and
their operations.

## What to flag

Raise a regression / question when a lower layer reaches across the boundary:

- `mdc/`, `osc/`, `lov/`, or `lmv/` code that dereferences or mutates
  `inode->i_*`, `dentry`, `file`, `address_space`/`i_pages`, or calls VFS/MM
  helpers (`filemap_*`, `truncate_*`, `set_page_dirty`, `folio_*`,
  `generic_*`, `mark_inode_dirty`, page-cache locking) directly.
- A new entry point that hands a raw `struct page`/`folio`, `inode`, or
  `address_space` down into mdc/osc/lov/lmv for it to manipulate, instead of
  going through a `cl_*`/CLIO method.
- Duplicated VFS/MM logic in a lower layer that should have been a call back up
  into llite.

When you see such a cross-layer access, identify the specific structure being
touched and ask whether it should go through a CLIO/llite method instead.
See clio.md for the CLIO interfaces that exist for this purpose.
