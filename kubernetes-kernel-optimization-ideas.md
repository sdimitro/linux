# Linux Kernel Optimization Ideas for Kubernetes/Container Workloads

Research notes identifying bottlenecks and optimization opportunities across kernel subsystems relevant to high-density container environments.

---

## 1. Network Namespace Creation/Teardown

**File: `net/core/net_namespace.c`**

### `setup_net()` (line 436)
- Every registered `pernet_operations` runs its `.init` callback **sequentially** — no parallelism.
- `pernet_ops_rwsem` (read) held across the entire initialization, blocking concurrent `register_pernet_subsys()` callers.

### `copy_net_ns()` (line 551)
- `pernet_ops_rwsem` held for the entire duration of namespace setup (line 577-583).
- On busy Kubernetes nodes with many containers starting simultaneously, any concurrent module load (`register_pernet_subsys()`) takes a write lock and blocks **all** new namespace creations.

### `cleanup_net()` (line 664) — Critical Bottlenecks
- **`rcu_barrier()` at line 711**: Waits for ALL outstanding RCU callbacks globally — not scoped to this namespace. On a 100-container node, teardown of one namespace can be delayed by callbacks from all others.
- **`synchronize_rcu_expedited()`** (called from `ops_undo_list()` at line 243): Forces an expedited grace period — expensive CPU cycles.
- **`unhash_nsid()` at line 697**: Iterates **every** entry of `net_namespace_list` with per-namespace `spin_lock(&tmp->nsid_lock)`, making teardown **O(n)** in the number of live namespaces.

### Single-threaded workqueue (line 1272)
```c
netns_wq = create_singlethread_workqueue("netns");
```
All namespace destructions are serialized through a single-thread workqueue — a serialization point during pod termination storms.

### `net_alloc_generic()` (lines 68-81)
- Called on every `copy_net_ns()` via `net_alloc()`. No caching of the generic array; a new allocation is always made even for short-lived namespaces.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| `rcu_barrier()` global wait | Per-namespace RCU domain |
| O(n) `unhash_nsid()` | Lazy/deferred unhashing |
| Single-threaded `netns_wq` | Multi-threaded workqueue |
| `pernet_ops_rwsem` write lock blocks all creations | Fine-grained per-subsystem locking |

---

## 2. cgroup Performance

**File: `kernel/cgroup/cgroup.c`**

### Lock Architecture (Major Contention Points)
```c
DEFINE_MUTEX(cgroup_mutex);                        // line 89  — master lock for hierarchy changes
DEFINE_SPINLOCK(css_set_lock);                     // line 90  — protects task->cgroups
DEFINE_SPINLOCK(cgroup_idr_lock);                  // line 108 — protects cgroup_idr and css_idr
DEFINE_PERCPU_RWSEM(cgroup_threadgroup_rwsem);     // line 116
```

The `cgroup_mutex` is a **single global mutex** that serializes all cgroup creation and destruction. In high-density Kubernetes environments starting/stopping many pods simultaneously, this is the central bottleneck.

### `cgroup_create()` (lines 5837-5936)
- Every cgroup creation holds `cgroup_mutex` (via `cgroup_mkdir()` at line 5986) and modifies all ancestors under `css_set_lock` — **O(depth)** spinlock-protected work.
- `percpu_ref_init()` allocates per-CPU reference counter storage — a memory allocation per cgroup.

### `css_create()` (lines 5784-5831)
- Called for **each enabled subsystem** per cgroup. With 10+ subsystems active (memory, cpu, io, net_cls, etc.), there are 10+ `percpu_ref_init()` + `kmalloc()` calls per pod creation, **all under `cgroup_mutex`**.

### `cgroup_destroy_locked()` (lines 6146-6218)
- `percpu_ref_kill_and_confirm()` (line 6119 in `kill_css()`) uses `percpu_ref` slowpath with an **IPI to all CPUs** to confirm the ref is seen as killed — expensive and blocking per CSS.
- Two-stage destruction process.

### `cgroup_migrate_execute()` (line 2691)
- Task migration holds `percpu_down_write(&cgroup_threadgroup_rwsem)` (line 2545) which blocks **ALL** other tasks from doing fork/exit during migration.

### rstat flushing — `kernel/cgroup/rstat.c`
```c
static DEFINE_SPINLOCK(rstat_base_lock);  // line 12 — global spinlock for base stats
```
All base stat (CPU time etc.) flushes across all cgroups contend on `rstat_base_lock`.

### Known TODO
**`kernel/cgroup/cpuset.c:3038`**:
```c
/* TODO: have a better way to handle failure here */
WARN_ON_ONCE(set_cpus_allowed_ptr(task, cpus_attach));
```

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| `cgroup_mutex` global mutex | Sharding or RCU-based hierarchy |
| `percpu_ref_kill` IPI to all CPUs per CSS | Batched refcnt kill |
| `cgroup_threadgroup_rwsem` blocks fork/exit during migration | Narrower critical section |
| `rstat_base_lock` global spinlock | Per-subtree lock or RCU |

---

## 3. Container Networking: Bridge, veth, netfilter

### veth — `drivers/net/veth.c`

**`veth_xmit()` (lines 347-418)**
- Non-XDP, non-GRO path calls `__netif_rx()` which enqueues to the softirq receive queue — one extra context-switch/scheduling event per packet vs a direct in-kernel delivery.
- XDP is the fast path but requires an XDP program attached.

**`u64_stats_sync` per-queue stats (lines 56-59)**
- Uses seqcount — readers must retry, writers acquire. At high PPS this becomes a hotspot.

### Bridge — `net/bridge/br_fdb.c`

**Single per-bridge spinlock** (`br->hash_lock`) at lines 374, 466, 510, 572, 709, 869, 976...
- Protects the FDB (MAC address table). All FDB lookups and modifications contend on this **one lock** regardless of how many cores are running containers on the same bridge.
- Container networking typically uses a single `docker0` or CNI bridge — all pods on the same node share this one lock.

**`br_forward_finish()` (lines 63-70)**
- Every forwarded frame passes through netfilter hooks even if no bridge netfilter rules are configured.

**FIXME in `br_netfilter_hooks.c:314`**
```c
/* FIXME Need to refragment */
```
DNATed packets that require refragmentation after bridge processing are not handled.

### netfilter conntrack — `net/netfilter/nf_conntrack_core.c`

**Lock structure (lines 57-81)**
```c
__cacheline_aligned_in_smp spinlock_t nf_conntrack_locks[CONNTRACK_LOCKS];
DEFINE_SPINLOCK(nf_conntrack_expect_lock);
```

**`__nf_conntrack_confirm()` (lines 1203-1350)**
- Uses a double-lock pattern with seqcount for hash table resizing. Under high connection rate, the `do..while` retry loop can spin repeatedly on seqcount collisions.

**`__nf_conntrack_alloc()` (lines 1658-1714)**
- Every new connection allocates from `nf_conntrack_cachep`. The conntrack count check uses `atomic_inc_return(&cnet->count)` — atomic per-namespace per new connection.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| Bridge FDB `br->hash_lock` | RCU hash table for reads |
| Conntrack double-lock seqcount retry | Lock-free lookup where possible |
| Netfilter hooks on every bridge frame | Bypass when no rules configured |

---

## 4. Memory Management for Containers

**File: `mm/memcontrol.c`**

### `try_charge_memcg()` (lines 2355-2500+)
- **Fast path** (`consume_stock`): uses per-CPU stock to avoid atomic operations — lock-free.
- **Slow path**: `page_counter_try_charge()` uses atomic operations on the hierarchical counter chain — one atomic per hierarchy level per charge event.
- On limit breach: calls `drain_all_stock()` which is a **cross-CPU IPI operation**.

### Slab charging TODO (line 3246)
```c
/* TODO: we could batch this until slab_pgdat(slab) changes
 * between iterations, with a more complicated undo */
```
Slab object charging happens one-by-one in `__memcg_slab_alloc_hook()` even when bulk-allocating slabs.

### `mem_cgroup_css_alloc()` (lines 3823-3872)
- `mem_cgroup_alloc()` allocates per-CPU data (stats vectors). On NUMA machines this is expensive.
- `static_branch_inc()` is a patching operation — requires stopping execution on the CPU temporarily.

### `mem_cgroup_css_free()` (lines 3949-3971)
- Freeing a memcg is **synchronous and blocking** — waits for writeback completion via `wb_wait_for_completion()`.
- `static_branch_dec()` again requires patching.

### mmap.c lock hierarchy
```
mm->mmap_lock (rwsem)
  -> anon_vma->lock (rwsem)
    -> i_mmap_lock_write(mapping)
```
- `mmap_lock` is per-process but under fork-heavy workloads write-locks can serialize.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| `drain_all_stock()` cross-CPU IPI | Smarter stock watermarks |
| Per-object slab charging in bulk alloc | Batch charging by pgdat |
| Blocking memcg free (writeback wait) | Async/deferred free path |

---

## 5. Scheduler: CPU Bandwidth Throttling

**File: `kernel/sched/fair.c`**

### `assign_cfs_rq_runtime()` (lines 5711-5721)
- Called every time a CFS run queue runs out of its slice (default: 5000 us).
- Acquires `cfs_b->lock` per bandwidth slice refresh — on a heavily loaded system with many cgroups this lock is acquired O(CPUs) times per period per task group.

### `throttle_cfs_rq()` (lines 6005-6044)
- `walk_tg_tree_from()` walks the entire task group subtree — **O(depth)** RCU walk on throttle.

### `unthrottle_cfs_rq()` (lines 6046-6130)
- Same `walk_tg_tree_from()` cost on unthrottle.

### `do_sched_cfs_period_timer()` (lines 6257-6310)
- Period timer must release and re-acquire `cfs_b->lock` for each unthrottle batch during distribution — lock/unlock cycling under timer callback.
- `distribute_cfs_runtime()` unthrottles queues one at a time while dropping the lock.

### Throttle notification (line 5824)
- Throttle notifications to individual tasks use `task_work` — deferred until the task returns from a syscall. A task can run past its quota briefly before throttle takes effect.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| `cfs_b->lock` contended per period/throttle event | Hierarchical slack distribution |
| O(depth) tree walk per throttle/unthrottle | Cached/flattened hierarchy |
| Lock/unlock cycling in period timer | Batched unthrottle |

---

## 6. Filesystem/Overlay

**File: `fs/overlayfs/copy_up.c`**

### `ovl_copy_up_one()` (lines 1123-1199)
- Copy-up is a **serialized operation** per dentry (`ovl_copy_up_start()` acquires the overlay inode lock).
- Concurrent writes to the same file in the same container are serialized.

### `ovl_copy_up_workdir()` (lines 758-840)
- Two `sb_start_write()` / `sb_end_write()` cycles per copy-up.
- Lock ordering forces unlock/relock across `ovl_copy_up_data()` and a TOCTOU check (lines 803-805).

### Known TODOs
**`copy_up.c:536-537`**:
```c
/* TODO: implement create index for non-dir, so we can call it when
 * encoding file handle for non-dir in case index does not exist. */
```

**`util.c:1161-1162`**:
```c
/* TODO: implement metadata only index copy up when called with
 *       ovl_copy_up_flags(dentry, O_PATH). */
```
Metadata-only copy-up (`O_PATH`) is not implemented for index cases — forces full copy-up where only metadata is needed, wasting I/O.

**`readdir.c:1066`**:
```c
WRAP_DIR_ITER(ovl_iterate) // FIXME!
```

### Lookup — `fs/overlayfs/namei.c`
- `ovl_lookup()` (line 1382) must traverse all lower layers in order, acquiring dentries at each layer.
- On a deep layer stack (many overlay mounts), each lookup multiplies VFS overhead.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| Full copy-up for O_PATH | Implement metadata-only index copy-up |
| Serialized per-dentry copy-up | Parallel copy-up for independent dentries |
| Multi-layer lookup overhead | Layer caching / shortcircuit |

---

## 7. PID Namespace and IPC Namespace

### PID Namespace — `kernel/pid_namespace.c`

**`create_pid_namespace()` (lines 76-138)**
- `register_pidns_sysctls(ns)` at line 111 registers a per-namespace sysctl table under `sysctl_lock` (a global mutex).
- `create_pid_cachep()` (lines 40-62) uses `pid_caches_mutex` and `kmem_cache_create()` which can be slow (cached per nesting level).

**`zap_pid_ns_processes()` (lines 192-283)**
- Acquires `tasklist_lock` (a **global `rwlock_t`**) at line 226 to iterate and signal all processes — contends with all fork/exit operations globally.

### IPC Namespace — `ipc/namespace.c`

**`create_ipc_ns()` (lines 39-109)**
- Creates a mq mount (`mq_init_ns`) — allocates a new superblock and vfsmount per namespace.
- Three separate `setup_*_sysctls()` calls, each registering sysctl tables under global lock.

**`free_ipcs()` (lines 127-148)**
- Holds `ids->rwsem` for the entire teardown of all IPC objects — blocks any concurrent IPC access during teardown.

**`free_ipc()` workqueue (lines 170-184)**
```c
synchronize_rcu();  // line 180 — global RCU barrier per destroyed IPC namespace
```

**`put_ipc_ns()` (lines 202-212)**
- Uses `mq_lock` (global spinlock from `ipc/mqueue.c`) to protect the reference count drop.

### Optimization Ideas
| Bottleneck | Opportunity |
|-----------|-------------|
| Global sysctl lock on PID ns create | Pre-allocate or lazy register |
| `tasklist_lock` read lock for zap | Per-namespace process list |
| `synchronize_rcu()` per IPC ns free | Batch or use `call_rcu` |
| `mq_lock` global spinlock on ns refcount drop | Per-ns lock or lockless refcount |

---

## Top Priority Optimization Targets (Summary)

| Priority | Area | File:Line | Bottleneck | Opportunity |
|----------|------|-----------|-----------|-------------|
| **P0** | Net ns teardown | `net_namespace.c:711` | `rcu_barrier()` global | Per-namespace RCU domain |
| **P0** | Net ns teardown | `net_namespace.c:1272` | Single-threaded `netns_wq` | Multi-threaded WQ |
| **P0** | cgroup hierarchy | `cgroup.c:89` | `cgroup_mutex` global | Sharding or RCU-based hierarchy |
| **P1** | Net ns teardown | `net_namespace.c:697` | O(n) `unhash_nsid()` | Lazy/deferred unhashing |
| **P1** | cgroup CSS destroy | `cgroup.c:6119` | IPI to all CPUs per CSS | Batched refcnt kill |
| **P1** | Bridge FDB | `br_fdb.c:374` | Single `hash_lock` for all MACs | RCU hash table for reads |
| **P1** | CFS bandwidth | `fair.c:5716,6011,6068` | `cfs_b->lock` contention | Hierarchical slack distribution |
| **P2** | memcg charge | `memcontrol.c:2422` | cross-CPU `drain_all_stock()` IPI | Smarter stock watermarks |
| **P2** | memcg slab | `memcontrol.c:3246` | Per-object charging (TODO) | Batch charging by pgdat |
| **P2** | cgroup stat flush | `rstat.c:12` | `rstat_base_lock` global | Per-subtree lock or RCU |
| **P2** | ovl copy-up | `copy_up.c:536, util.c:1161` | Full copy-up for O_PATH (TODO) | Metadata-only index copy-up |
| **P3** | PID ns sysctl | `pid_namespace.c:111` | Global sysctl lock | Pre-allocate or lazy register |
| **P3** | IPC ns free | `ipc/namespace.c:180` | `synchronize_rcu()` per free | Batch or use `call_rcu` |
| **P3** | cgroup migration | `cgroup.c:2545` | `threadgroup_rwsem` blocks fork/exit | Narrower critical section |
| **P3** | Conntrack | `nf_conntrack_core.c:1237` | Seqcount retry under load | Lock-free lookup paths |

---

*Generated from Linux 7.0-rc2 kernel source tree.*
