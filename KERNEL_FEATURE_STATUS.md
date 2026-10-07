# KERNEL_FEATURE_STATUS

Engineering state tracking for Origin-Kernel. Factual record only.

## Provenance

| Field | Value |
|---|---|
| Upstream | https://github.com/samakshkambxj/Origin-Kernel.git |
| Fork | https://github.com/Unnho/Origin-Kernel.git |
| Target branch | `origin-glx` |
| Base commit | `286c57c101e4a68e6e793a58fe36d41c8bb8603c` |
| Describe | `v1.5-tier6-security-908-g286c57c101e4` |
| Kernel base | Linux 6.1.177 |
| Android generation | Android 16 (tier6 GKI) |
| Architecture | arm64 |
| Device codename | `tetris` |
| Tree style | GKI (`arch/arm64/configs/gki_defconfig` + fragments) |
| History depth | 117,955 commits |

### Environment

| Field | Value |
|---|---|
| Build host | ARM64 native, Debian 13, 8 cores |
| RAM | 5.3 GB total, ~1.5 GB available, 2 GB swap in use |
| Disk | 85 GB free |
| Toolchain | GCC 14.2.0 (native), bc/flex/bison/libelf/dwarves installed |
| Build concurrency | -j2 / -j3 (RAM-limited) |
| Device access | NONE. `FLASH_ALLOWED=no`; no runtime validation possible. |

### Config layout notes

- No device defconfig for `tetris` exists in-tree. Feature flags live in
  `arch/arm64/configs/gki_defconfig` plus fragments. Device-specific config is
  applied out-of-tree.
- `CONFIG_SCHED_CASS=y` is set in `gki_defconfig`. `CONFIG_SCHED_BORE` is **not**
  set anywhere in defconfigs.

---

## Feature matrix

All six roadmap features were verified against the source tree. None are absent.
Status reflects completeness of the in-tree implementation, not README claims.

| Feature | Present | Complete | Source/Base | Status |
|---|---|---|---|---|
| Boeffla Wakelock Blocker | yes | yes | Downstream (andip71 v1.1.0), MTK-adapted | COMPLETE |
| BORE CPU Scheduler | yes | yes | Downstream BORE 2.x (CFS-based) | PRESENT (not enabled) |
| ADIOS I/O Scheduler | yes | unverified | Downstream `block/adios.c` | PRESENT |
| WQ Power Efficiency | yes | yes | Upstream `WQ_POWER_EFFICIENT` | COMPLETE |
| ZRAM Writeback | yes | yes | Upstream 6.2 series, backported | COMPLETE |
| DAMON | yes | unverified | Upstream 5.18+ | PRESENT |

### 1. Boeffla Wakelock Blocker — COMPLETE

Implemented, correctly wired, and device-adapted.

- `kernel/power/boeffla_wl_blocker.c` + `.h` (v1.1.0)
- Kconfig `kernel/power/Kconfig:187`, Makefile `kernel/power/Makefile:27`
  (`obj-$(CONFIG_WAKELOCK_BLOCKER) += boeffla_wl_blocker.o`)
- Interception hook present in `drivers/base/power/wakeup.c` at three sites:
  extern decl (`:19`), `__pm_wakeup_event` (`:616`), `pm_wakeup_event` (`:809`),
  all gated on `#ifdef CONFIG_WAKELOCK_BLOCKER` calling `is_wakelock_blocked()`.
- Sysfs interface: `wakelock_blocker`, `wakelock_blocker_default`, `debug`,
  `version` on misc device `boeffla_wakelock_blocker`.

Default block list is **MediaTek/IPA-specific**, not the upstream Qualcomm list,
confirming deliberate device adaptation (`kernel/power/boeffla_wl_blocker.h:22`):
`IPA_WS;IPA_CLIENT_APPS_WAN_COAL_CONS;IPA_CLIENT_APPS_WAN_LOW_LAT_CONS;IPA_CLIENT_APPS_LAN_CONS;rmnet_ipa%d;rmnet_ctl;RMNET_SHS;hal_bluetooth_lock`

Per rule 24 these runtime block rules are **not** modified. No rules were added
or removed. Changing them requires explicit user authorisation.

### 2. BORE CPU Scheduler — PRESENT (not enabled)

Fully integrated but disabled in the shipped config.

- `init/Kconfig:1317` (`config SCHED_BORE`, `default y`)
- `kernel/sched/fair.c` — 14 `CONFIG_SCHED_BORE` sites
- `include/linux/sched.h:587`, `include/linux/sched/task.h:64`,
  `kernel/fork.c:2532`, `kernel/sched/core.c:4664`
- `gki_defconfig` does **not** set `CONFIG_SCHED_BORE`.

Variant note: this is the **CFS-based** BORE lineage (2.x/3.x). The current
upstream BORE 5.x is an **EEVDF** variant and cannot be used here — EEVDF first
appeared in 6.6, this tree is 6.1. `git grep` for EEVDF/`scx`/`sched_ext` in
`kernel/sched` returns nothing, so CFS is the correct and only base.

### 3. ADIOS I/O Scheduler — PRESENT

`block/adios.c` exists. Completeness not yet verified (see Outstanding).

### 4. WQ Power Efficiency — COMPLETE

Upstream implementation already provides the requested behaviour. Per rule 23,
**no port required**; configuration is the correct lever.

- `include/linux/workqueue.h:343` — `WQ_POWER_EFFICIENT = 1 << 7`
- `include/linux/workqueue.h:381-393` — `wq_power_efficient` param semantics,
  `system_power_efficient_wq`, `system_freezable_power_efficient_wq`
- Tree already ships the upstream mechanism. No downstream duplicate was found.

### 5. ZRAM Writeback — COMPLETE

Upstream 6.2 writeback series is already backported into this 6.1 base.

- `drivers/block/zram/Kconfig:58` — `config ZRAM_WRITEBACK`
- `drivers/block/zram/zram_drv.c:377-890` — `writeback_limit_*`, `writeback_store`
- `zram_drv.c:657-659` — `PAGE_WRITEBACK`, `HUGE_WRITEBACK`, `IDLE_WRITEBACK`
- `gki_defconfig:359` — `CONFIG_ZRAM_WRITEBACK=y`
- Backing device is not auto-configured and Android swap policy is untouched.

### 6. DAMON — PRESENT

`mm/damon/` present: `core.c`, `dbgfs.c`, `lru_sort.c`, `modules-common.c`,
`ops-common.c`, `Kconfig`, `Makefile`. Completeness/optimisation not yet audited.

---

## Defects found during audit

### D1. Duplicate KConfig symbol — `kernel/power/Kconfig` — FIXED

`config WAKELOCK_BLOCKER` is defined **three times** in the same file:

- line 187 — `bool "Boeffla wakelock blocker"`, `default y`
- line 198 — identical
- line 208 — identical

Duplicate symbol definitions in one file are a Kconfig defect and should produce
a warning during Kconfig parsing. Fix: collapse to a single definition. This is
low-risk, self-contained, and independent of every other change.

### D2. ZRAM multi-comp read path ignores per-page algorithm — FIXED

`CONFIG_ZRAM_MULTI_COMP` allows a page to be compressed with a **secondary**
algorithm, but every read path decompresses using the **primary** stream only.

Supporting evidence:
- `zram_drv.c:87-97` — per-slot algorithm is recorded via
  `zram_get_priority()` / `zram_set_priority()`, stored in `table[index].flags`.
- `zram_drv.c:1306-1319` — `zram_recompress()` iterates `prio` and compresses with
  `zstrm = zcomp_stream_get(zram->comps[prio])`, i.e. **secondary algorithms**.
- `zram_drv.c:1388` — on success, `zram_set_priority(zram, index, prio)` records it.
- `zram_drv.c:1617`, `:1629` (`zram_bvec_read`) — decompress **always** via
  `zram->comps[ZRAM_PRIMARY_COMP]`. `zram_get_priority()` is never consulted.
- `zram_drv.c:1696`, `:1748` — same unconditional-primary pattern elsewhere.
- `zram_drv.c:1617-1640` — failure path is
  `WARN_ON(ret); pr_err("Decompression failed! err=%d, page=%u\n", ...)`.

Consequence: a page successfully recompressed with a secondary algorithm is read
back through the primary stream, decompression fails, and the read returns an
error. This is a read-path data-loss condition, not merely wasted work.

Trigger condition: the secondary loop is guarded by `if (!zram->comps[prio])
continue;` (`zram_drv.c:1307`), so this is **dormant when only one algorithm is
configured** and becomes reachable once userspace configures a second algorithm
and `recompress` runs. `comps[]` is populated per-index at `zram_drv.c:2049-2112`.

Confidence: **confirmed by code review, not yet reproduced at runtime.** No device
access. Needs a runtime reproduction before any fix is treated as validated.

Upstream relationship: this is the pre-6.2 `ZRAM_MULTI_COMP` design. Upstream
replaced it wholesale with multi-stream compression (`comp_state[]`,
`comp_streams`, per-entry `comp` pointer), where reads *do* select the correct
stream. See the backport table.

### D3. `SCHED_BORE` and `SCHED_CASS` have no mutual exclusion — WATCH

Both are defined in `init/Kconfig` (`:1292` and `:1317`), both `depends on SMP`,
neither declares `conflict`. `kernel/sched/fair.c:12771` does
`#include "cass.c"` and `:12778` redefines `select_task_rq_fair` to
`cass_select_task_rq_fair`, in the same file BORE patches at 14 sites.

Currently latent: only `CONFIG_SCHED_CASS=y` is set. Enabling both is undefined
behaviour. Any BORE enablement work must resolve this first.

---

## Backport matrix

Status from source inspection, not README.

### ZRAM

| Item | Status | Notes |
|---|---|---|
| Multi-stream compression | ABSENT | Tree uses pre-6.2 `CONFIG_ZRAM_MULTI_COMP` (`zram_drv.h:66-74`, fixed 4 slots, no `comp_streams`/`comp_state`). Upstream `comp_state`/`comp_streams` rework replaces `ZRAM_MULTI_COMP` wholesale. See D2. |

### f2fs

| Item | Status | Notes |
|---|---|---|
| Inline inode handling | PARTIAL | Base inline paths + corrupt-inode detection present (`inline.c:158-165`, `:414-422`; `inode.c:260-272`). Missing upstream `378acf3cf19b` clamp of `i_inline_xattr_size` — `f2fs.h:3483-3486` and `MAX_INLINE_DATA()` (`f2fs.h:497-500`) subtract it with no bound. Bounds a real OOB read in `f2fs_fill_dentries()`. Small, self-contained fix. |
| Compression cache validation | ABSENT | Tree has the old page-based cache (`compress.c:1937`, `:1989`). No `f2fs_cache()`/`struct f2fs_cache_entry`/`NI_CLUSTER_COUNT` exists, so the upstream bounds check has no host code. Porting requires the whole 6.9-6.11 cache rewrite. |
| ACL fixes | PRESENT | `acl.c:46` `f2fs_acl_from_disk()` with per-entry bounds checks (`:73-77`, `:92-96`, `:104-108`). Symbol is `CONFIG_F2FS_FS_POSIX_ACL`, not `CONFIG_F2FS_POSIX_ACL`. |
| Write I/O UAF prevention | PRESENT | `data.c:372-383` `f2fs_write_end_io()` accesses `sbi` before `dec_page_count()` (upstream `2d9c4a4ed4ee`). `compress.c:1500-1509` `dec_page_count()` last. `compress.c:1761-1766` caches `dic->sbi`. |

### ext4

| Item | Status | Notes |
|---|---|---|
| Inline data bounds checking | PARTIAL | Size/`i_inline_size` checks present (`inline.c:190`, `:520-527`). Missing `e_value_offs`/`e_value_size` validation against the inode xattr area — raw offset copy at `inline.c:204-211`, `:259`, `:1112`. Contrast `xattr.c:226`, which does validate. Note: the upstream patch for this is still under upstream review, not merged as of v7.3. |
| Superblock update checks | PARTIAL | `EXT4_ERROR_FS` / `sb_rdonly` handling present in error paths (`super.c:702-754`, `:1080`). No `ext4_check_dirty_super()` anywhere; `ext4_update_dynamic_rev()` (`:1137-1160`) mutates `es` with no rdonly/error/mapped guard, and its callers in `ext4.h:2067-2113` have none either. |
| DAX restrictions on encrypted files | PRESENT | `crypto.c:153-163` bare `IS_DAX(inode)` rejection; `ialloc.c:995-1001` sets `EXT4_ENCRYPT_FL` before `ext4_set_inode_flags()`. Minor delta: `ialloc.c:1318` masks only `EXT4_DAX_FL`, upstream masks `(EXT4_DAX_FL|EXT4_EA_INODE_FL)`. Not reachable for encrypted inodes here. |

### mm

| Item | Status | Notes |
|---|---|---|
| Per-CPU bitmap overflow | ABSENT | `bitmap_overflow()` does not exist (`lib/bitmap.c`, `include/linux/bitmap.h` — 6.1 form, no `set_last_bit()`). Zero call sites. Added upstream in v6.9: **forward-port, not a 6.1 backport.** |
| Page-reporting UAF during suspend | PARTIAL | Core reporting present (`mm/page_reporting.c`, 382 lines, wired via `mm/Makefile:143` + `page_alloc.c:1289`). No `pagefree_online_read/enable/disable`, no `page_reporting_prepare/restart`, no `pm_suspend` hook. The suspend UAF fix (v6.10+) is absent. |
| vmalloc ptdump UAF | PARTIAL / suspicious | `mm/ptdump.c:151-163` correctly takes `mmap_write_lock(mm)`. **But** `mm/vmalloc.c:166-169`, `:233-245`, `:301-313` wrap that same lock in `#ifndef CONFIG_ARM64`, so on this tree's target arch the `init_mm` page-table frees run **unlocked** — i.e. the fix is disabled exactly where it is needed. `ARCH_HAS_PTE_DUMP` does not exist in 6.1; this tree backports `CONFIG_PTDUMP_CORE`. |
| huge_zero_pfn race fixes | ABSENT | `huge_zero_page_add_pfn` / `huge_zero_page_free_pfn` do not exist (0 matches). 6.1 form is `mm/huge_memory.c`: `get_huge_zero_page()` returns `READ_ONCE(huge_zero_page)` at `:210` after dropping the lock; `huge_zero_pfn` set at `:181`, reset at `:240`. The lock-guarded pfn API requires ~6.9/6.10. |

### Scheduler

| Item | Status | Notes |
|---|---|---|
| Deadline migration warnings | PARTIAL | The `migrate_enable()` boosted-task warning fix is present (`deadline.c:1658-1662`). The broader hardening is not: `pick_next_pushable_dl_task()` (`deadline.c:2221-2238`) still carries the raw `WARN_ON_ONCE()` set and has no `may_skip` parameter (`git grep may_skip` over `kernel/sched` → 0 matches). `pull_dl_task()` retains `WARN_ON(p == src_rq->curr)` at `:2462-2463`. |
| RT push-task race fixes | PARTIAL | The 2022 `rt_mutex_setprio()` vs `push_rt_task()` race is fixed (`rt.c:2215-2223`, priority check before `is_migration_disabled`; `rt.c:2241-2242`). The 2025 migration race is not: `rt.c:2148-2150` `find_lock_lowest_rq()` still uses the old re-validation list, and `pick_next_pushable_task()` (`rt.c:2074-2092`) has no `may_skip` / donor check. |
| PSI timer shutdown / power-saving | PARTIAL | cgroup-PSI enable/disable/restart machinery present (`psi.c:1176-1206` `psi_cgroup_restart()`; `:805-820`; `cgroup.c:3900-3936`; `psi.c:1097-1107`; `psi.c:1335-1407`). The 6.6 aggregator-split refactor is absent (`psi_types.h:194` still a single `triggers` list), and `psi_avgs_work()` re-arms on the old condition (`psi.c:430`, `:441-444`). |

### Network

| Item | Status | Notes |
|---|---|---|
| IGMP UAF prevention | ABSENT | No `kref` in `net/ipv4/igmp.c` (0 matches); no `igmp_start_query`, `igmp_group_queued`, `igmp_sock_teardown`, `igmp_shutdown`. Stock 6.1 teardown at `:1810-1826`. The socket-close UAF fix is v6.9+: **requires a newer base than 6.1.** |
| Bridge multicast UAF | ABSENT | No `mdb_group`, no `br_multicast_start_query` (0 matches). Pre-5.x `struct net_bridge_mdb_entry` (`br_private.h:345-357`). RCU/GC deferral only, no lifetime refcount: `br_multicast.c:612-620` GC workqueue, `:608-609` `kfree_rcu`; `br_mdb.c:1099` `br_mdb_del()` takes no reference; `br_forward.c:194-199` `br_multicast_count()` under `rcu_read_lock` only. Needs newer base. |

### Android / vendor hooks

| Item | Status | Notes |
|---|---|---|
| memcg PSI recording skip | ABSENT | No `psi_mem_pressure` / `skip_psi` / `mem_cgroup_skip_psi` anywhere. `mm/memcontrol.c` only calls `psi_memstall_enter/leave` (`:2420`, `:2654`, `:2716`). |
| Migration batch hooks | not audited | — |
| Direct reclaim hooks | not audited | — |

---

## Blocking constraint: kernel base is 6.1.177

Six of the requested backports cannot be applied to this base. They are not
missing upstream patches to be found and cherry-picked; each depends on a large
refactor that landed **after** 6.1 and would have to be forward-ported:

| Item | Needs base |
|---|---|
| ZRAM multi-stream | 6.2+ (replaces `ZRAM_MULTI_COMP` wholesale) |
| Per-CPU bitmap overflow | 6.9+ (`bitmap_overflow()`) |
| Page-reporting suspend UAF | 6.10+ |
| huge_zero_pfn race | 6.9/6.10 (`add_pfn`/`free_pfn` API) |
| IGMP UAF | 6.9+ (kref `ip_mc_list`) |
| Bridge multicast UAF | `mdb_group` refcounting |

These are recorded as DEFERRED / BLOCKED rather than attempted. Rule 22 applies:
prefer a smaller correct backport over a large poorly understood forward-port.
---

## Implemented changes

All changes are on separate branches off `origin-glx`, and are combined on
`integration/verified-fixes` for build verification.

| # | Branch | Commit subject |
|---|---|---|
| D1 | `fix/power-kconfig-duplicate` | `power: remove duplicate WAKELOCK_BLOCKER Kconfig definitions` |
| D2 | `fix/zram-multi-comp-read-path` | `zram: fix multi-comp read path using wrong decompression algorithm` |
| 1 | `fix/f2fs-inline-xattr-bounds` | `f2fs: bound i_inline_xattr_size to prevent out-of-bounds inline access` |
| 2 | `fix/ext4-superblock-guards` | `ext4: do not bump the dynamic revision on a read-only or errored filesystem` |
| 3 | `fix/sched-rt-dl-push-task` | `sched/rt, sched/dl: reject stale push candidates when revalidating a push` |
| cfg | `config/user-ns` | `arm64: gki_defconfig: enable CONFIG_USER_NS` |

Diffstat of the combined integration branch: 8 files, 101 insertions, 17 deletions.

### Build verification — PASS (compile only)

| Item | Result |
|---|---|
| Config | `make ARCH=arm64 gki_defconfig` OK; `CONFIG_USER_NS=y` confirmed in `.config` |
| Build | `make ARCH=arm64 -j2 Image` **EXIT=0** |
| Compiler errors | 0 |
| Warnings in changed files | 0 (`zram_drv.c`, `f2fs.h`, `inode.c`, `super.c`, `rt.c`, `deadline.c`, `Kconfig`) |
| `checkpatch.pl` | 0 errors, 0 warnings across all six branches |
| `git diff --check` | clean |
| Artifact | `arch/arm64/boot/Image` 34M, valid ARM64 boot executable, 4K pages |
| vmlinux | 424M, BTF generated |
| Build string | `Linux version 6.1.177-Origin-v3-00919-g04abf3543c0a ... gcc 14.2.0` |

Runtime testing: **NOT PERFORMED.** No device access, `FLASH_ALLOWED=no`.
No flashing was attempted. Nothing here is boot-verified.

### Build prerequisite discovered

The tree does not build from a plain clone. `drivers/kernelsu` is a symlink to
`../KernelSU/kernel`, and `KernelSU` is a git submodule. Until submodules are
initialised, `drivers/Kconfig:242` fails with
`can't open file "drivers/kernelsu/Kconfig"` and the kernel cannot be configured.

```
git submodule update --init --depth 1
```

This was done locally and is not committed. The tree also ships three submodules:
`KernelSU`, `susfs4ksu`, `Baseband-guard`.

### Notes on deviations from upstream

- **ext4**: no upstream `ext4_check_dirty_super()` exists to import. The
  `ext4_update_dynamic_rev()` guard is a local adaptation, not a backport.
- **sched/rt, sched/dl**: upstream refactors `pick_next_pushable_*_task()` into a
  `__pick_next_pushable_*_task(rq, may_skip)` helper that dequeues stale entries.
  That form does not apply here: this tree's generic `dequeue_task()` is a
  `static inline` local to `kernel/sched/core.c`, unreachable from `rt.c` and
  `deadline.c`, and the per-class `dequeue_task_rt()`/`dequeue_task_dl()` would
  skip `sched_core_dequeue()`, `update_rq_clock()`, `psi_dequeue()` and
  `uclamp_rq_dec()`, under-decrementing uclamp and PSI accounting. The validity
  check was instead placed directly at the two revalidation sites.
- **ZRAM**: the read-path fix also had to reset the priority in
  `zram_free_page()`, because the `WARN_ON_ONCE` at the end of that function
  already requires all flags other than `ZRAM_LOCK`/`ZRAM_UNDER_WB` to be clear,
  and `zram_free_page()` never cleared the priority bits.

### Still open

- **D3** (`SCHED_BORE` / `SCHED_CASS` no mutual exclusion) — not fixed. Recorded
  as WATCH. Fixing it means choosing an exclusivity mechanism, which is a
  design decision, and enabling BORE was not requested.
- **ADIOS** and **DAMON** completeness were not audited beyond confirming
  presence.
- Android/vendor hooks (memcg PSI skip, migration batch, direct reclaim) not
  audited.
- The six DEFERRED backports below remain deferred.

---

## CORRECTION: `origin-glx` is not the device build base

An earlier revision of this document treated `origin-glx` as the Tetris build
base. That is wrong, and the error caused a bootloop.

### Device build topology

The packaged AnyKernel3 zip for Tetris carries the build string
`6.1.162-Origin-g6411275b06e2`. Commit `6411275b06e2` is reachable **only via
the tag `Tetris-v1`** — it is on no branch, and it is **not an ancestor of
`origin-glx`**. The two have diverged.

| Ref | SHA | Kernel | Role |
|---|---|---|---|
| `Tetris-v1` (tag) | `cbce87493adf` | 6.1.162 | Released Tetris build |
| working image source | `6411275b06e2` | 6.1.162 | Ancestor of `Tetris-v1`, 1 commit back |
| `origin-glx` (branch) | `286c57c1` | 6.1.177 | Development branch |

Distances: `6411275b06e2..origin-glx` = **9,629 commits**,
`origin-glx..6411275b06e2` = 61. `Tetris-v1` and `6411275b06e2` differ only in
`README.md` and `main.png`, so they are functionally identical.

Per-device release tags, confirming three separate device lines:

| Device | Codename | Tag | Kernel |
|---|---|---|---|
| CMF by Nothing Phone 1 | `Tetris` | `Tetris-v1` | 6.1.162 |
| CMF by Nothing Phone 2 Pro | `Galaga` | `glx-upstream-2026.03-r20` | 6.1.162 |
| Nothing Phone (3a) Lite | `Galaxian` | `6.1.172-glx-upstream-2026.06-r9` | 6.1.172 |

### Toolchain requirement

`gki_defconfig` sets `CONFIG_LTO_CLANG_THIN=y`, which requires Clang. The
released Tetris image was built with **clang 22.1.8 + LLD 22.1.8**.

Building the same defconfig with GCC does **not** fail. Kconfig silently degrades
to `CONFIG_LTO_NONE=y` because the Clang-ThinLTO options are unavailable, so the
build reports success while violating its own configuration. This is not
detectable from the exit status, `checkpatch`, or compiler warnings.

### Which fixes apply to the device base

Tested with `git apply --check` against `6411275b06e2`:

| Fix | Applies? | Reason |
|---|---|---|
| f2fs `i_inline_xattr_size` bound | yes | defect present at base |
| ext4 dynamic-rev guard | yes | defect present at base |
| `CONFIG_USER_NS` | yes | config change |
| sched/dl push revalidation | yes | same validity gap at base |
| sched/rt push revalidation | **no** | 6.1.162 `find_lock_lowest_rq()` already validates `!rt_task()`, `!task_on_rq_queued()` and `task_on_cpu()` directly, with no `pick_next_pushable_task()` comparison |
| D1 duplicate `CONFIG_WAKELOCK_BLOCKER` | **no** | Boeffla driver absent at base; `kernel/power/` has no such file and `drivers/base/power/wakeup.c` has zero references |
| D2 ZRAM multi-comp read path | **no** | `ZRAM_MULTI_COMP` has zero references at base; only `ZRAM_WRITEBACK` is present |

### Corrected feature status

Several features reported COMPLETE/PRESENT above describe the `origin-glx`
development branch and are **absent from the released Tetris kernel**. Re-checking
at `6411275b06e2`:

| Feature | `origin-glx` | `Tetris-v1` / `6411275b06e2` |
|---|---|---|
| Boeffla Wakelock Blocker | PRESENT | **ABSENT** |
| ZRAM multi-stream / multi-comp | PRESENT | **ABSENT** (only `ZRAM_WRITEBACK`) |
| ADIOS I/O scheduler | PRESENT, default | **not enabled** |
| WQ Power Efficiency | upstream | present upstream |

`origin-glx`'s `gki_defconfig` additionally enables
`CONFIG_MQ_IOSCHED_DEFAULT_ADIOS=y`, `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y` and
`CONFIG_HZ_300=y`, and drops `CONFIG_RCU_BOOST=y`. Forcing a new default I/O
scheduler onto flash storage without device testing is a plausible contributor to
the bootloop, independent of the base mismatch.
