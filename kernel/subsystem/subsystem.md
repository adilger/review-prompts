# Subsystem Guide Index

Load subsystem guides from the prompt directory based on what the code
touches. Each guide contains subsystem-specific invariants, API
contracts, and common bug patterns. Each subsystem guide may reference
additional pattern files to load conditionally.

A change can match multiple rows. Load **every** matching guide, not
just the deepest or most specific. For example, code touching
`lustre/llite/rw.c` matches CLIO, VFS and MM Folio rows - all three guides apply.

The triggers column below includes both path names, function calls,
and symbols regexes.

## Subsystem Guides

> **Path resolution:** every filename in the "File" column below — and in the
> "Optional Patterns" section — is relative to **this file's directory**:
> `review-prompts/kernel/subsystem/`. For example,
> `networking-core.md` resolves to
> `review-prompts/kernel/subsystem/networking-core.md`.

| Subsystem | Triggers | File |
|-----------|----------|------|
| Networking Core | lnet/, skb_, sockets, xfrm, dst_, sock_put, release_sock, pskb_may_pull, SNMP_*_STATS | networking-core.md |
| Netlink | `genl_`, `nla_`, `NLA_`, `NLM_F_`, `nlmsg_`, `netlink_callback`, Documentation/netlink/specs/, files marked `YNL-GEN` | netlink.md |
| Alignment Helpers | `ALIGN`, `ALIGN_DOWN`, `IS_ALIGNED`, `PAGE_ALIGN`, `PAGE_ALIGN_DOWN`, `pageblock_align`, `pageblock_aligned`, `pageblock_start_pfn`, `pageblock_end_pfn` | alignment.md |
| MM Folio/Page Cache | `folio_*`, `page_folio`, `compound_head`, `filemap_*`, `xa_*`, `xas_*`, `page_cache_*`, lustre/llite/rw.c, lustre/llite/rw26.c, lustre/llite/rvvp_page.c, lustre/llite/llite_mmap.c, | mm-folio.md |
| MM Large Folios/THP/Hugetlb | `huge_memory`, `hugetlb`, `split_huge_*`, `folio_test_large`, `hstate`, PMD sharing | mm-largepage.md |
| MM VMA Operations | `vma_*`, `mmap_*`, `vm_area_struct`, `vm_flags`, `anon_vma`, `maple_tree` | mm-vma.md |
| MM Allocation | `alloc_pages`, `__GFP_*`, `kmalloc`, `kmem_cache_*`, `slub`, `vmalloc`, `zone_watermark`, `mempool`, `memblock` | mm-alloc.md |
| MM Reclaim/Swap/Migration | `vmscan`, `shrink_*`, `lru_*`, `swap_*`, `shmem_*`, `mem_cgroup_*`, `writeback`, `migrate_*` | mm-reclaim.md |
| VFS | inode, dentry, vfs_, lustre/llite/*.c | vfs.md |
| CLIO | *clio*, lustre/llite/*.c, lustre/osc/*.c, lustre/lov/*.c | clio.md |
| Locking | spin_lock*, mutex_*, rwsem*, seqlock*, *seqcount* | locking.md |
| Scheduler | kernel/sched/, sched_, schedule, *wakeup* | scheduler.md |
| Timers | timer_list, timer_setup, mod_timer, del_timer, hrtimer, delayed_work | timers.md |
| BPF | kernel/bpf/, tools/lib/bpf/, tools/testing/selftests/bpf, bpf, verifier | bpf.md |
| BTF Fields | `map_check_btf`, `check_and_init_map_value`, `bpf_obj_free_fields`, `BPF_SPIN_LOCK`, `BPF_TIMER`, `BPF_KPTR` | btf.md |
| Libbpf API | tools/lib/bpf/, `LIBBPF_API`, `libbpf_err`, `libbpf_err_ptr` | libbpf.md |
| RCU | rcu*, call_rcu, synchronize_rcu, kfree_rcu, kvfree_call_rcu | rcu.md |
| Encryption | crypto, fscrypt_ | fscrypt.md |
| Tracing | trace_, tracepoints | tracing.md |
| Workqueue | kernel/workqueue.c, work_struct | workqueue.md |
| Syscalls | `SYSCALL_DEFINE`, `copy_from_user`, `copy_to_user`, `get_user`, `put_user`, any change to syscall parameter validation | syscall.md |
| Block/NVMe | block layer, nvme | block.md |
| io_uring | io_uring/, io_uring_, io_ring_, io_sq_, io_cq_, io_wq_, IORING_ | io_uring.md |
| Cleanup API | `__free`, `guard(`, `scoped_guard`, `DEFINE_FREE`, `DEFINE_GUARD`, `no_free_ptr`, `return_ptr` | cleanup.md |
| RCU lifecycle | `call_rcu(`, `kfree_rcu(`, `synchronize_rcu(`, `rhashtable_*` + `call_rcu`, `hlist_del_rcu` + `call_rcu`, `list_del_rcu` + `call_rcu` | rcu.md |
| Sysfs | sysfs_create_group, sysfs_update_group, attribute_group, is_visible | sysfs.md |
| Perf Tools | tools/perf/, openat, fdopendir, closedir | perf.md |
| Build System | Kbuild, Makefile, Makefile.am, Makefile.in, *.m4, `gnu11`, `-funsigned-char`, `-fno-strict-aliasing` | build.md |
| Selftests | tools/testing/selftests/, TEST_PROGS, TEST_FILES, TEST_GEN_FILES | selftests.md |
| OSD API | lustre/osd-*/*.c | osd.md |
| Lustre utils | lustre/utils/*, lnet/utils/* | lustre-utils.md |
| Rust | any Rust code | rust.md |

## Optional Patterns

Load only when explicitly requested in the prompt:

- **Subjective Review** (subjective-review.md): Subjective general assessment
