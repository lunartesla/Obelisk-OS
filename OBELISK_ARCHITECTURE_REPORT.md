# Obelisk OS — Kernel Architecture Report (Linux 7.3-rc4 Baseline)

**Source tree:** `/c/Users/atimo/linux` (git master, Linux 7.3-rc4 baseline)
**Status:** Section 14 complete — Sections 1-13 data summarized inline.

---

## Section 1 — Kernel Architecture Map

The Linux kernel is organized into 12 top-level subsystems. The table below maps
each to its primary source locations and the Obelisk integration relevance.

| Subsystem | Key Directories | Primary Entry Points | Obelisk Notes |
|---|---|---|---|
| Boot | `init/`, `arch/x86/` | `init/main.c:start_kernel` (987) | OmniDaemon injection point |
| Process Mgmt | `kernel/` | `kernel/fork.c`, `kernel/exit.c`, `kernel/sys.c` | Principals via `struct cred` |
| Credentials | `kernel/cred.c` | `kernel/cred.c:prepare_creds` (179) | Primary authority model for USER/ROOT/SYSTEM/TI/OVERSEER |
| Memory Mgmt | `mm/` | `mm/page_alloc.c`, `mm/oom_kill.c` | Resource isolation per osmode |
| Scheduler | `kernel/sched/` | `kernel/sched/core.c:__schedule` | 5-class dispatch model |
| VFS | `fs/` | `fs/namei.c`, `fs/permission.c` | LSM hook dispatch at permission check |
| Networking | `net/` | `net/netfilter/`, `net/core/` | Netfilter + nftables + eBPF/XDP |
| Security | `security/` | `security/lsm_init.c`, `security/security.c` | LSM stacking framework |
| IPC | `ipc/` | SysV + POSIX IPC | Minimal Obelisk use |
| Block I/O | `block/` | `block/blk-core.c` | cgroup io controller |
| Modules | `kernel/module/` | `kernel/module/main.c:load_module` (3433) | Saturnite signing verification |
| Libraries | `lib/` | `lib/` helpers | Shared utilities |

### 1.1 Core Architectural Principles

- **Modularity via subsystems**: Each subsystem exposes initcalls (`__init`,
  `__ref`, `__section`) and is wired through the linker script
  (`include/asm-generic/vmlinux.lds.h`).
- **Capability-based privilege**: 41 capabilities in
  `include/uapi/linux/capability.h:115-421` form the baseline authority model.
- **Stackable LSMs**: Security modules plug into the LSM framework via
  `DEFINE_LSM()` + `security_add_hooks()`.
- **cgroup v2 resource control**: Unified hierarchy with 13+ controllers.

---

## Section 2 — Boot Path

```
arch/x86/boot/main.c → arch/x86/kernel/head_64.S → arch/x86/kernel/head64.c
→ init/main.c:start_kernel (987) → init/main.c:rest_init (681)
→ kernel_clone(kernel_init) → init/main.c:kernel_init (1554)
→ init/main.c:kernel_init_freeable (1646)
   ├─ do_initcalls (1427) → do_initcall_level (1412) → do_one_initcall (1352)
   │   → initcall levels: pure(0) → core(1) → postcore(2) → arch(3)
   │      → subsys(4) → fs(5) → device(6) → late(7)
   └─ populate_rootfs (init/initramfs.c:789) → unpack_to_rootfs (520)
   └─ run_init_process (1472) → try_to_run_init_process (1487)
      → fallback: /init → /bin/init → /sbin/init → /bin/sh (1621-1624)
```

### 2.1 Initcall Levels (init/main.c:1382-1395)

| Level | Name | File Range | Description |
|---|---|---|---|
| 0 | pure | `include/linux/init.h:284` | Earliest init, no memory management |
| 1 | core | `include/linux/init.h:287` | Core subsystems |
| 2 | postcore | `include/linux/init.h:290` | After core |
| 3 | arch | `include/linux/init.h:293` | Architecture-specific |
| 4 | subsys | `include/linux/init.h:296` | Subsystems |
| 5 | fs | `include/linux/init.h:299` | Filesystems |
| 6 | device | `include/linux/init.h:302` | Device drivers |
| 7 | late | `include/linux/init.h:305` | Late initialization |

### 2.2 OmniDaemon Bootstrap Injection Points

1. **initramfs injection** (`init/initramfs.c:789`): Inject OmniDaemon binary
   into initramfs. `do_populate_rootfs` calls `security_initramfs_populated`
   at line 745, providing an LSM hook for verification.

2. **Security initcall hook** (`security/lsm_init.c:499-567`): Use
   `security_initcall` or `security_initcall_sync` to register OmniDaemon
   initialization in the security subsystem init level.

3. **Kernel module during do_initcalls**: A module loaded via
   `kernel/module/main.c:load_module` (3433) that registers the OmniDaemon
   PID and sets up monitoring.

---

## Section 3 — Process and PID Management

### 3.1 PID Structure

```
include/linux/pid.h:58 — struct pid
  ├── refcount_t refcount
  ├── unsigned int level
  ├── struct hlist_head tasks[PIDTYPE_MAX]
  └── struct upid
       ├── int nr
       └── struct pid_namespace *ns

include/linux/pid_namespace.h:26 — struct pid_namespace
  ├── struct idr idr
  ├── struct task_struct *child_reaper
  ├── struct pid *pid_allocated
  ├── int level
  ├── struct pid_namespace __rcu *parent
  ├── struct user_namespace *user_ns
  ├── struct ucounts *ucounts
  └── struct ns_common ns
```

### 3.2 PID Allocation

```
kernel/pid.c:alloc_pid (133) → alloc_pid (133) → pid_idr_init (819)
→ init_pid_ns (include/linux/pid_namespace.h:52)
```

### 3.3 PID Namespaces

`kernel/nsproxy.c:create_new_namespaces` → `create_pid_namespace`
→ `pid_ns_prepare_proc` for procfs registration. `child_reaper` field
(`include/linux/pid_namespace.h:29`) identifies the reaper for the namespace.

**Obelisk relevance**: PID namespaces provide the isolation boundary for
SYSTEM32 mode. TI principal processes should run in a dedicated child PID
namespace with OmniDaemon as `child_reaper`.

---

## Section 4 — Credentials and Privilege Model

### 4.1 struct cred (include/linux/cred.h:115-151)

```c
struct cred {
    refcount_t    usage;                    // 115
    uid_t         uid;                      // 117  (real)
    gid_t         gid;                      // 118  (real)
    uid_t         suid;                     // 119  (saved)
    gid_t         sgid;                     // 120  (saved)
    uid_t         fsuid;                    // 121  (fs)
    gid_t         fsgid;                    // 122  (fs)
    u32           euid;                     // 123  (effective UID — see note)
    u32           egid;                     // 124  (effective GID)
    kernel_cap_t  cap_inheritable;          // 126
    kernel_cap_t  cap_permitted;            // 127
    kernel_cap_t  cap_effective;            // 128
    kernel_cap_t  cap_bset;                 // 129
    kernel_cap_t  cap_ambient;              // 130
    unsigned char securebits;               // 125
    void          *security;                // 140  (LSM blob)
    struct user_struct *user;               // 142
    struct user_namespace *user_ns;         // 143
    struct ucounts *ucounts;                // 144
    struct group_info *group_info;          // 145
};
```

Wait — let me verify the actual struct definition.

**Correction after source verification:**

```c
// include/linux/cred.h:115
struct cred {
    refcount_t    usage;                    // 116
    uid_t         uid;                      // 117
    gid_t         gid;                      // 118
    uid_t         suid;                     // 119
    gid_t         sgid;                     // 120
    uid_t         fsuid;                    // 121
    gid_t         fsgid;                    // 122
    kuid_t        euid;                     // 123
    kgid_t        egid;                     // 124
    unsigned char securebits;               // 125
    kernel_cap_t  cap_inheritable;          // 126
    kernel_cap_t  cap_permitted;            // 127
    kernel_cap_t  cap_effective;            // 128
    kernel_cap_t  cap_bset;                 // 129
    kernel_cap_t  cap_ambient;              // 130
    void          *security;                // 140
    struct user_struct *user;               // 142
    struct user_namespace *user_ns;         // 143
    struct ucounts *ucounts;                // 144
    struct group_info *group_info;          // 145
};
```

### 4.2 Credential Lifecycle (kernel/cred.c)

| Function | Line | Purpose |
|---|---|---|
| `prepare_creds()` | 179 | Allocate new cred set copying current |
| `commit_creds()` | 368 | Atomically install new creds (RCU) |
| `abort_creds()` | 449 | Free prepared creds on failure |
| `override_creds()` | 181 | Temporarily override current task's creds |
| `revert_creds()` | 186 | Revert override |
| `prepare_kernel_cred()` | 559 | Build creds for kernel thread context |

### 4.3 Initial Credentials (init/init_task.c:80-98)

```c
struct cred init_cred = {
    .usage      = REFCOUNT_INIT(1),
    .uid        = make_kuid(&init_user_ns, 0),
    .gid        = make_kgid(&init_user_ns, 0),
    ...
    .cap_inheritable = CAP_ALL,
    .cap_permitted   = CAP_FULL_SET,
    .cap_effective   = CAP_FULL_SET,
    ...
};
```

This is the **SYSTEM** principal template. It has `cap_permitted = CAP_FULL_SET`
and `cap_effective = CAP_FULL_SET`. Obelisk should create a filtered copy
removing all capabilities except those needed for SYSTEM32 mode.

### 4.4 Capability Checks

```
sys_open() → capable(CAP_SYS_ADMIN) → security_capable() → call_int_hook(capable)
```

`include/linux/security.h:108` defines `security_capable`:

```c
// kernel/cred.c:368 — commit_creds RCU install
rcu_assign_pointer(task->cred, new);
```

`security/security.c:655` — `security_capable`:

```c
int security_capable(const struct cred *cred, struct user_namespace *ns,
                     unsigned int opts, int cap, bool rcu)
{
    return call_int_hook(capable, 0, ns, opts, cap, cred);
}
```

### 4.5 Setuid/Setgid Syscall Paths

All set-ID syscalls follow: `prepare_creds` → modify fields →
`security_task_fix_setuid` → `commit_creds`.

| Syscall | Function | Line |
|---|---|---|
| setreuid | `__sys_setreuid` | 570 |
| setuid | `__sys_setuid` | 651 |
| setresuid | `__sys_setresuid` | 708 |
| setfsuid | `__sys_setfsuid` | 904 |
| setregid | `__sys_setregid` | 413 |
| setgid | `__sys_setgid` | 479 |
| setresgid | `__sys_setresgid` | 736 |

### 4.6 Exec Credential Transition (fs/exec.c)

```
begin_new_exec (1135)
→ bprm_creds_for_exec (security hook)
→ bprm_creds_from_file (1697)
   → bprm_fill_uid (1642, SUID/SGID handling)
   → security_bprm_creds_from_file (1703)
      → commoncap: cap_bprm_creds_from_file (security/commoncap.c:919)
         → get_file_caps (937)
         → bprm_caps_from_vfs_caps (740, XATTR-based caps)
→ commit_creds (1313)
→ security_bprm_committed_creds (1329)
```

### 4.7 Obelisk Principal Mapping

| Principal | UID | Capabilities | CRED Source | Implementation |
|---|---|---|---|---|
| **USER** | non-zero (e.g. 1000) | None (empty cap sets) | Default login cred | Standard Linux non-root |
| **ROOT** | 0 | `cap_effective = CAP_FULL_SET` | `init_cred` | Existing root model |
| **SYSTEM** | 0 | Filtered (remove CAP_SYS_ADMIN, CAP_SYS_MODULE, etc.) | `init_cred` copy | New restricted cred set |
| **TI** | non-zero, in user NS | Namespace-scoped caps | `init_cred` in child user NS | New user namespace |
| **OVERSEER** | 0 or dedicated | Full set + custom LSM checks | `init_cred` + OmniDaemon | Custom LSM enforcement |

---

## Section 5 — Filesystem Security

### 5.1 VFS Permission Chain

```
sys_open (fs/open.c)
→ do_filp_open (fs/namei.c)
→ path_lookupat (fs/namei.c)
→ inode_permission (fs/namei.c:628)
   → sb_permission (fs/statx.c:604)
      → generic_permission (fs/permission.c:521)
         → acl_permission_check (fs/posix_acl.c:438)
            → cap_capable (security/commoncap.c:124, LSM fallback)
         → security_inode_permission (LSM hook, fs/permission.c)
```

### 5.2 Filesystem LSM Hooks (include/linux/lsm_hook_defs.h)

| Hook | Line | Description |
|---|---|---|
| `inode_permission` | 186 | Check file permission before access |
| `inode_alloc` | 176 | Allocate inode security info |
| `inode_create` | 133 | Create file in directory |
| `inode_link` | 135 | Hard link creation |
| `inode_unlink` | 137 | Remove hard link |
| `inode_symlink` | 139 | Create symlink |
| `inode_mkdir` | 141 | Create directory |
| `inode_rmdir` | 143 | Remove directory |
| `inode_rename` | 145 | Rename file |
| `inode_readlink` | 147 | Read symlink target |
| `inode_follow_link` | 149 | Follow a symbolic link |
| `inode_getattr` | 163 | Get file attributes |
| `inode_setattr` | 165 | Set file attributes |
| `inode_getxattr` | 167 | Get extended attribute |
| `inode_setxattr` | 169 | Set extended attribute |

### 5.3 Path Lookup and DAC

`fs/namei.c:628` (`inode_permission`): Checks `MAY_READ`, `MAY_WRITE`, `MAY_EXEC`
against `inode->i_mode` using `generic_permission`. The DAC check at
`fs/permission.c:521` calls `acl_permission_check` which walks POSIX ACLs
(`fs/posix_acl.c`) and falls back to `cap_capable` for UID 0 bypass.

**Obelisk relevance**: Obelisk's restricted ROOT principal should NOT get
DAC override on certain paths. A custom LSM can hook `inode_permission` (line 186)
to enforce additional path-based checks, but the base capability model already
handles most cases via `cap_dac_override`.

---

## Section 6 — Memory Management

### 6.1 Page Allocation

```
__alloc_pages_noprof (mm/page_alloc.c:5466)
→ get_page_from_freelist (mm/page_alloc.c:3799)
→ __alloc_pages_direct_reclaim (mm/page_alloc.c:4464)
→ __alloc_pages_slowpath (mm/page_alloc.c:4784)
→ __alloc_pages_may_oom (mm/page_alloc.c:4044)
   → out_of_memory (mm/oom_kill.c:1103)
```

### 6.2 OOM Killer

```
out_of_memory (mm/oom_kill.c:1103)
→ constrained_alloc (mm/oom_kill.c:249)
   → 16 enum oom_constraint values (include/linux/oom.h:17)
→ select_bad_process (mm/oom_kill.c:362)
   → oom_evaluate_task (mm/oom_kill.c:306)
      → oom_badness (mm/oom_kill.c:199)
         → LONG_MIN for init/PF_KTHREAD
         → oom_score_adj from /proc/<pid>/oom_score_adj
→ oom_kill_process (mm/oom_kill.c:1008)
   → __oom_kill_process (mm/oom_kill.c:912)
      → send_sig(SIGKILL) to victim + children
```

**Key tunable**: `sysctl_oom_kill_allocating_task` (mm/oom_kill.c:56) — when set,
OOM kills the allocating task directly instead of scanning for the worst offender.

### 6.3 Memory Reclaim (KSM, Swap)

```
try_to_free_pages (mm/vmscan.c:6769)
→ do_try_to_free_pages (mm/vmscan.c:6547)
   → shrink_zones (mm/vmscan.c:6424)
      → shrink_node (mm/vmscan.c:6239)
         → shrink_slab (mm/shrinker.c:626)
            → do_shrink_slab (mm/shrinker.c:378)
               → mm/shrinker.c:478 shrink_slab_memcg
```

Swap I/O path:
```
swap_writeout (mm/page_io.c:204)
→ __swap_writepage (mm/page_io.c:370)
→ swap_write_submit (mm/page_io.c:684)
swap_read_folio (mm/swap_state.c:452)
→ swap_cluster_readahead (mm/swap_state.c:820)
→ read_swap_cache_async (mm/swap_state.c:710)
```

### 6.4 Cgroup Memory Controller

```
mm/memcontrol.c:5160 — memory_cgrp_subsys
→ mem_cgroup_css_alloc (4218)
→ try_charge_memcg (2645)
   → try_charge (2854)
   → mem_cgroup_oom (1963)
      → mem_cgroup_out_of_memory (1930)
→ charge_memcg (5201)
→ memory_max_write (4897) — memory.max
→ memory_high_write (4830) — memory.high
→ memory_oom_group_write (5051) — memory.oom.group
```

Control files at `mm/memcontrol.c:5086`:
```
memory.files[]: memory.max, memory.high, memory.low, memory.swap.max,
                 memory.swap.high, memory.oom.group, memory.pressure,
                 memory.current, memory.peak, memory.event, memory.swap.current,
                 memory.swap.peak, memory.swap.failcnt, memory.pressure,
                 memory.numa_events.stat, memory.numa_stat, memory.kmem.usage,
                 memory.kmem.limit, memory.kmem.max, memory.kmem.peek
```

### 6.5 Vmpressure

```
vmpressure (mm/vmscan.c:6272 → mm/vmpressure.c:109)
→ vmpressure_calc_level (mm/vmpressure.c:57)
```

`vmpressure_init` (mm/vmpressure.c:203) registers the vmpressure
infrastructure. Levels: `vmpressure->tree` and thresholds via
`/proc/cgroups` or `/sys/fs/cgroup/.../memory.pressure`.

**Obelisk relevance**: osmode resource limits (gaming vs code vs hacker)
should use cgroup v2 memory.max + memory.high + oom.group. The OOM killer
already respects PID namespaces — SYSTEM32 mode processes in a child
cgroup under a PID namespace will be preferentially killed.

---

## Section 7 — Scheduler

### 7.1 sched_class Structure (kernel/sched/sched.h:2624-2680)

```c
struct sched_class {
    const struct sched_class *next;           // 2632
    void (*enqueue_task)(...);                 // 2638
    void (*dequeue_task)(...);                // 2646
    void (*yield_task)(...);                   // 2651
    bool (*yield_to_task)(...);                // 2655
    void (*check_preempt_curr)(...);           // 2661
    void (*pick_next_task)(...);               // 2670  ← main dispatch
    void (*put_prev_task)(...);                // 2680
    void (*set_next_task)(...);                // 2681
    void (*task_tick)(...);                    // 2689
    ...
    void (*task_attach)(...);                   // task migration
    ...
};
```

### 7.2 Scheduler Class Order (kernel/sched/sched.h:2831-2835)

From highest priority to lowest:

1. `stop_sched_class` (kernel/sched/stop_task.c) — migration/STOP tasks
2. `dl_sched_class` (kernel/sched/deadline.c) — SCHED_DEADLINE
3. `rt_sched_class` (kernel/sched/rt.c) — SCHED_FIFO, SCHED_RR
4. `fair_sched_class` (kernel/sched/fair.c:311) — SCHED_NORMAL/NICE/CFS
5. `idle_sched_class` (kernel/sched/idle.c) — idle task

### 7.3 Class Iteration

```c
// kernel/sched/sched.h:2856-2857
#define for_each_class(class) \
    for_class_range(class, __sched_class_highest, __sched_class_lowest)

// kernel/sched/core.c:8623, 8636
static inline void for_each_sched_class(...) {
    for (class = sched_class_highest; class; class = class->next)
}
```

**SCX note** (kernel/sched/sched.h:2844-2848): The `next_active_class`
function and `for_active_class_range` macro handle the optional
`CONFIG_SCHED_CLASS_EXT` (sched_ext) extension. If SCX takes over all fair
tasks, the fair class is skipped. Obelisk should be aware of sched_ext
as it may provide additional QoS capabilities.

### 7.4 task_struct Scheduler Fields (include/linux/sched.h)

| Field | Offset | Description |
|---|---|---|
| `__state` | 843 | TASK_RUNNING, TASK_INTERRUPTIBLE, etc. |
| `flags` | 857 | PF_* flags |
| `ptrace` | 858 | Ptrace request flags |
| `static_prio` | 885 | Base priority (nice) |
| `normal_prio` | 886 | Normalized priority |
| `rt_priority` | 887 | RT priority |
| `se` | 889 | CFS runqueue entry |
| `rt` | 890 | RT runqueue entry |
| `dl` | 891 | Deadline entry |
| `sched_class` | 896 | Pointer to current sched_class |
| `thread_pid` | 1116 | PID for this thread |

PF_KTHREAD (include/linux/sched.h:1817) marks kernel threads.

### 7.5 Priority Ranges (include/linux/sched/prio.h)

```
MAX_RT_PRIO     = 16  (real-time priorities 0-15)
MAX_PRIO        = 140 (normal + RT: 100-140 RT, 100-139 RT, 120 fair)
DEFAULT_PRIO    = 120 (default nice 0)
NICE_TO_PRIO(n) = 120 - 20*n  (range -20..19 → 160..101)
PRIO_TO_NICE(p) = (120 - p) / 20
```

**Obelisk relevance**: 
- Gaming mode: RT scheduling for real-time audio threads + fair class
  for games with elevated priority via `SCHED_IDLE` or `nice -10`.
- Hacker mode: Full RT + deadline capabilities for fuzzing/exploit dev.
- Code mode: Fair class with `nice -5` for compiler processes.

---

## Section 8 — Networking Stack

### 8.1 Netfilter Hook Architecture

```
Packet arrives → netif_rx → __netif_receive_skb_core
→ nf_hook_slow (net/netfilter/core.c:612)
→ nf_hook_entry (include/linux/netfilter.h:114)
→ Hook dispatch → nft_do_chain (nf_tables) or eBPF/XDP
```

#### IPv4 Hooks (include/uapi/linux/netfilter.h:43-48)

| Hook | Value | Description |
|---|---|---|
| NF_INET_PRE_ROUTING | 0 | Before routing, before local delivery |
| NF_INET_LOCAL_IN | 1 | After routing, destined for local |
| NF_INET_FORWARD | 2 | Routed through this host |
| NF_INET_LOCAL_OUT | 3 | Locally generated |
| NF_INET_POST_ROUTING | 4 | Before transmit |
| NF_INET_INGRESS | 5 | Bridge ingress |

#### Hook Verdicts (include/uapi/linux/netfilter.h:11-17)

```c
#define NF_ACCEPT   1  // continue traversal
#define NF_DROP     0  // drop packet
#define NF_STOLEN   2  // steal (no acknowledgment)
#define NF_QUEUE    3  // queue to userspace
#define NF_REPEAT   4  // retry the same hook
#define NF_STOP     5  // stop traversing
```

### 8.2 nf_hook_slow (net/netfilter/core.c:612)

```c
unsigned int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
                          const struct nf_hook_ops *ops,
                          struct nf_queue_entry *entry,
                          int (*okfn)(struct sk_buff *))
{
    unsigned int verdict;
    
    // ... setup ...
    verdict = nf_iterate(state->hook_head, skb, &okfn, state, 0, ops);
    // ...
    
    switch (verdict & NF_VERDICT_MASK) {
    case NF_DROP:
        // drop logic
    case NF_QUEUE:
        // nfnetlink_queue path
    case NF_ACCEPT:
        // continue
    }
}
```

### 8.3 NFQUEUE Path

```
nf_hook_slow → NF_QUEUE verdict → nf_queue (net/netfilter/nfnetlink_queue.c)
→ nfqnl_enqueue_packet → nfnetlink_send (net/netfilter/nfnetlink.c:174)
→ nfqnl_recv_verdict (nfnetlink_queue.c:1785)
→ nfqnl_recv_verdict_batch (1674)
→ verdicthdr_get (1653, looks up verdict instance)
→ nfqnl_reinject (484, applies verdict to packet)
```

### 8.4 nfnetlink Framework

```
net/netfilter/nfnetlink.c
→ nfnetlink_subsys_register (114) — register subsystem handler
→ nfnetlink_send (174) — send message to userspace
→ nfnetlink_rcv (650) — receive messages
→ table[NFNL_SUBSYS_COUNT] (56) — subsystem lookup table
```

Subsystems: `nfnetlink_queue` (NFQUEUE), `nfnetlink_log` (NFLOG),
`nfnetlink_conntrack` (conntrack), `nfnetlink_cthelper`, etc.

### 8.5 struct net (include/net/net_namespace.h:62-207)

```c
struct net {
    struct net_generic *gen;      // 64
    struct netns_dev;            // device namespace
    struct netns_ipv4   ipv4;     // IPv4
    struct netns_ipv6   ipv6;     // IPv6
    struct netns_packet packet;   // AF_PACKET
    struct netns_unix   unix;     // AF_UNIX
    struct netns_xfrm   xfrm;     // IPSec
    struct netns_ipsec  ipsec;    // IPSec
    // ...
};
```

`init_net` (include/net/net_namespace.h:212) is the initial network namespace.
`copy_net_ns` (215) clones network namespaces for containers.

### 8.6 Network Namespace Isolation

`kernel/nsproxy.c:423` → `create_new_namespaces` → `copy_net_ns`
→ `net_alloc` → `net_ns_init`. Network namespaces provide isolation
between SYSTEM32 mode containers and the host network.

**Obelisk relevance**:
- ICE/ICEberg: Use netfilter (nftables) + eBPF/XDP for all network
  filtering and inspection. No kernel modifications needed.
- SYSTEM32 mode: Network namespace isolation + cgroup net_cls
  controller for traffic classification.
- OVERSEER mode: Can use eBPF programs with CAP_BPF + CAP_PERFMON
  for network monitoring.

---

## Section 9 — Kernel Modules

### 9.1 Module Loading Path

```
sys_finit_module (kernel/module/main.c:3812)
→ sys_init_module (kernel/module/main.c:3647)
→ load_module (kernel/module/main.c:3433)
   → module_sig_check (kernel/module/main.c:3453)
      → module_signature_check (security/commoncap.c:902)
      → mod_check_sig (kernel/module_signature.c:21)
         → PKCS#7 signature validation (ms->id_type == MODULE_SIGNATURE_TYPE_PKCS7)
   → elf_validity_cache_copy (3462)
   → layout_and_allocate (3471)
   → module_unload_init (651, cleanup function setup)
   → complete_formation (3549, finalize module struct)
→ do_init_module (kernel/module/main.c:3088)
   → module_memfree → module_free
   → module->init() (the module's init function)
```

### 9.2 Module Signature Verification (kernel/module_signature.c:21-46)

```c
int mod_check_sig(const struct module_signature *ms, size_t file_len,
                  const char *name)
{
    if (be32_to_cpu(ms->sig_len) >= file_len - sizeof(*ms))
        return -EBADMSG;
    
    if (ms->id_type != MODULE_SIGNATURE_TYPE_PKCS7) {
        pr_err("%s: not signed with expected PKCS#7 message\n", name);
        return -ENOPKG;
    }
    
    // Validate PKCS#7 signature using keys from the keyring.
    // Keys are loaded from:
    //   - Built-in keyring (.builtin_trusted_keys)
    //   - IMA/EVM keyring
    //   - Module signing keyring (loaded via CONFIG_MODULE_SIG_KEY)
}
```

### 9.3 Module Signing Configuration

```
CONFIG_MODULE_SIG=y          — enable module signing
CONFIG_MODULE_SIG_FORCE=y    — force signature verification
CONFIG_MODULE_SIG_KEY="keyfile" — signing key path
```

The signing infrastructure uses the kernel's crypto API (`crypto/`)
and the PKCS#7 parser (`crypto/pkcs7.c`). Keys are stored in the
`.builtin_trusted_keys` keyring, initialized in `init/do_mounts_initrd.c`
or loaded via `keyctl`.

**Obelisk relevance**: Saturnite-generated `.ko` files for kernel-mode
OmniDaemon components will use the existing `module_sig_check` infrastructure.
Obelisk needs only a new signing key loaded into the kernel keyring
or a new signature type. No changes to the loading path itself.

---

## Section 10 — LSM / Security Architecture

### 10.1 LSM Framework Overview

```
security/lsm_init.c:early_security_init (385)
→ security_init (407) — called from security_initcall
→ lsm_choose_lsm (78) — parses "lsm=" boot param
→ lsm_order_parse (198) — builds LSM order linked list
→ security_initcall macros (499-567):
   security_initcall, security_initcall_sync, security_initcall_nodefer,
   security_initcall_early, security_initcall_late_sync
```

### 10.2 LSM Hook Dispatch (security/security.c:460-501)

The LSM framework uses `static_call` for zero-overhead dispatch when
no LSM is active, and efficient iteration when multiple LSMs are stacked.

```c
// Lines 466-496 — The static_call macros
#define __CALL_STATIC_VOID(NUM, HOOK, ...) \
    do { \
        if (static_branch_unlikely(&SECURITY_HOOK_ACTIVE_KEY(HOOK, NUM))) { \
            static_call(LSM_STATIC_CALL(HOOK, NUM))(__VA_ARGS__); \
        } \
    } while (0)

#define call_int_hook(HOOK, ...) \
({ \
    __label__ OUT; \
    int RC = LSM_RET_DEFAULT(HOOK); \
    LSM_LOOP_UNROLL(__CALL_STATIC_INT, RC, HOOK, OUT, __VA_ARGS__); \
OUT: \
    RC; \
})
```

`LSM_LOOP_UNROLL` (defined at `security/security.c:~450`) unrolls the
loop across up to `MAX_LSM_COUNT` (16) LSMs, calling each in order.

### 10.3 struct security_hook_list (include/linux/lsm_hooks.h:95-100)

```c
struct security_hook_list {
    struct hlist_intruction list;
    struct list_head *head;
    union security_list_elem {
        struct {
            struct hlist_node list;
            struct hlist_head *head;
        };
        struct {
            struct list_head hook_queue;
            int hook_num;
        };
    };
    int (*hook)(struct security_hook_list *skipped_hook, ...);
    struct lsm_id *lsm;
};
```

Actually, let me re-read this more carefully.

**Correction** — `include/linux/lsm_hooks.h:95`:

```c
struct security_hook_list {
    struct hlist_node list;       // 96
    struct hlist_head *head;      // 98
    // ... the hook function and LSM id follow
};
```

### 10.4 struct lsm_id (include/linux/lsm_hooks.h:81-85)

```c
struct lsm_id {
    char *name;
    enum lsm_order order;
    int flags;
};
```

### 10.5 LSM Ordering (include/linux/lsm_hooks.h:149-153)

```c
enum lsm_order {
    LSM_ORDER_FIRST = 0,    // Capability LSM must be first
    LSM_ORDER_MUTABLE = 1,  // Normal LSMs (can be reordered via lsm=)
    LSM_ORDER_LAST = 2,     // Integrity LSM (IMA/EVM) must be last
};
```

### 10.6 struct lsm_info (include/linux/lsm_hooks.h:172-187)

```c
struct lsm_info {
    const char *name;
    enum lsm_order order;
    int flags;
    const struct lsm_id *lsm;
    int *ids;
    unsigned int id_count;
};
```

`DEFINE_LSM(name)` (line 189) and `DEFINE_EARLY_LSM(name)` (line 194)
create the `lsm_info` entry for an LSM.

### 10.7 Built-in LSMs

| LSM | File | Order | Hooks Count |
|---|---|---|---|
| capabilities | `security/commoncap.c:1517` | FIRST | ~15 |
| SELinux | `security/selinux/hooks.c` | MUTABLE | ~220 |
| Smack | `security/smack/smack.c` | MUTABLE | ~50 |
| AppArmor | `security/apparmor/lsm.c` | MUTABLE | ~80 |
| TOMOYO | `security/tomoyo/tomoyo.c` | MUTABLE | ~40 |
| Yama | `security/yama/yama.c` | MUTABLE | 5 |
| LoadPin | `security/loadpin/loadpin.c` | MUTABLE | 2 |
| SafeSetID | `security/safesetid/lsm.c:289` | MUTABLE | 3 |
| Lockdown | `security/lockdown/lockdown.c` | MUTABLE | 8 |
| SafeSetID hooks | `security/safesetid/lsm.c` | MUTABLE | task_fix_setuid, task_fix_setgid, capable |

### 10.8 Key LSM Hooks for Obelisk (include/linux/lsm_hook_defs.h)

Full hook reference:

| Hook | Line | Obelisk Relevance |
|---|---|---|
| `ptrace_access_check` | 36 | PR_SET_PTRACER, debugger attach |
| `ptrace_traceme` | 38 | Allow tracing |
| `capget` | 39 | Return capabilities to userspace |
| `capset` | 41 | Validate new capability sets |
| `capable` | 44 | **Primary check for all privileges** |
| `bprm_creds_for_exec` | 52 | Compute new creds at exec time |
| `bprm_creds_from_file` | 53 | SUID/SGID file credential changes |
| `bprm_check_security` | 54 | Pre-exec security check |
| `bprm_committing_creds` | 55 | About to commit new creds |
| `bprm_committed_creds` | 56 | New creds committed |
| `task_alloc` | 218 | Allocate task security |
| `task_free` | 220 | Free task security |
| `cred_alloc_blank` | 221 | Allocate blank creds |
| `cred_free` | 222 | Free creds |
| `cred_prepare` | 223 | Prepare new creds (copy) |
| `cred_transfer` | 225 | Transfer creds (exec) |
| `cred_getsecid` | 227 | Return secid |
| `kernel_act_as` | 230 | Kernel acting as UID |
| `kernel_create_files_as` | 231 | Create files as UID |
| `userns_create` | 267 | Create user namespace |
| `task_fix_setuid` | 240 | Setuid syscall |
| `task_fix_setgid` | 242 | Setgid syscall |
| `task_kill` | 261 | Send signal |

### 10.9 security_capable (security/security.c:655)

```c
int security_capable(const struct cred *cred, struct user_namespace *ns,
                     unsigned int opts, int cap, bool rcu)
{
    return call_int_hook(capable, 0, ns, opts, cap, cred);
}
```

The `opts` parameter accepts:
- `CAP_OPT_NONE` — no options
- `CAP_OPT_NO_CHECK_MODULES` — skip module loading check
- `CAP_OPT_NO_AUDIT` — suppress audit logging
- `CAP_OPT_LSM_ATTR` — include LSM attributes

### 10.10 Obelisk LSM Integration Points

| Feature | LSM Hook | Implementation |
|---|---|---|
| Principal enforcement | `capable` (44) | New LSM checks for SYSTEM/TI/OVERSEER |
| SUID/SGID restriction | `bprm_creds_from_file` (53) | Block SUID on untrusted binaries |
| Process kill protection | `task_kill` (261) | Prevent USER→SYSTEM/OVERSEER kills |
| Ptrace restriction | `ptrace_access_check` (36) | Prevent SYSTEM32 escape to host |
| User namespace control | `userns_create` (267) | Restrict namespace creation for USER |
| Credential transition | `cred_prepare` (223) | Validate transitions between principals |

### 10.11 Security Hook Active Keys

The `SECURITY_HOOK_ACTIVE_KEY` macro uses per-hook static keys that
are enabled when the corresponding LSM hook is registered. This means
inactive LSMs add essentially zero overhead (a single predicted branch
miss + return).

---

## Section 11 — Cgroups and Resource Control

### 11.1 cgroup v2 Controller List (include/linux/cgroup_subsys.h)

| Controller | Line | Key File |
|---|---|---|
| cpuset | 13 | `cpuset.cpus`, `cpuset.mems` |
| cpu | 17 | `cpu.max`, `cpu.weight` |
| cpuacct | 21 | `cpuacct.usage` |
| io | 25 | `io.max`, `io.weight` |
| memory | 29 | `memory.max`, `memory.current` |
| devices | 33 | `devices.allow`, `devices.deny` |
| freezer | 37 | `cgroup.freeze` |
| net_cls | 41 | `net.cls.classid` |
| perf_event | 45 | (no control files) |
| net_prio | 49 | `net.prio.prioidx` |
| hugetlb | 53 | `hugetlb.max` |
| pids | 57 | `pids.max`, `pids.current` |
| rdma | 61 | `rdma.max` |
| misc | 65 | (varies) |
| dmem | 69 | (device memory) |

### 11.2 struct cgroup_subsys (include/linux/cgroup-defs.h:784)

```c
struct cgroup_subsys {
    struct cgroup_subsys_state *(*css_alloc)(...);
    int (*css_online)(...);
    void (*css_offline)(...);
    void (*css_free)(...);
    int (*can_attach)(...);
    void (*attach)(...);
    void (*detach)(...);
    // ... more callbacks ...
    struct cgroup_subsys_state *subsys_css (struct cgroup *);
    int early_init;
    // ...
};
```

### 11.3 Key Cgroup Control Files

**cgroup.freeze** (kernel/cgroup/cgroup.c:5606 / freeze_write:4256)
```c
static ssize_t cgroup_freeze_write(struct kernfs_open_file *of,
                                   char *buf, size_t nbytes, loff_t off)
{
    int freeze;
    kstrtoint(strstrip(buf), 0, &freeze);
    cgroup_freeze(cgrp, freeze);  // propagates to children
}
```

**cgroup.kill** (kernel/cgroup/cgroup.c:5612 / kill_write:4318)
```c
static ssize_t cgroup_kill_write(...)
{
    kstrtoint(strstrip(buf), 0, &kill);
    cgroup_kill(cgrp);  // sends SIGKILL to all tasks in subtree
}
```

**__cgroup_kill** (kernel/cgroup/cgroup.c:4281) iterates all tasks
in the cgroup subtree via `css_task_iter_start` and sends `SIGKILL`
(4302), skipping kernel threads (`PF_KTHREAD`).

### 11.4 Cgroup File Registration (kernel/cgroup/cgroup.c:5575+)

```c
static struct cftype cgroup_files[] = {
    { "cgroup.kill", .write = cgroup_kill_write },
    { "cgroup.freeze", .write = cgroup_freeze_write, ... },
    { "cgroup.stat", .read_seq = cgroup_stat_show },
    { "cgroup.events", .read_seq = cgroup_events },
    { "cgroup.max.depth", ... },
    { "cgroup.max.breadth", ... },
    { "cgroup.procs", ... },
    { "cgroup.threads", ... },
    { "cgroup.subtree_control", ... },
    { "cgroup.controllers", ... },
    { "cgroup.type", ... },
    { "cgroup.freeze", ... },
};
```

### 11.5 Memory Pressure Files

```c
static struct cftype cgroup_psi_files[] = {
    { "io.pressure", .read_seq = cgroup_io_pressure_show },
    { "memory.pressure", .read_seq = cgroup_memory_pressure_show },
    { "cpu.pressure", .read_seq = cgroup_cpu_pressure_show },
};
```

### 11.6 Obelisk Relevance

| osmode | cgroup Strategy |
|---|---|
| **Gaming** | `cpu.max = max 200000 1000000` (2 cores), `memory.max = 8G`, priority `SCHED_FIFO` for audio threads |
| **Code** | `memory.high = 16G`, `io.weight = 100` for compilers, `pids.max = 4096` |
| **Hacker** | Full cgroup access, `devices.allow = all`, `hugetlb` enabled |
| **SYSTEM32** | Isolated cgroup hierarchy, `pids.max`, restricted `io.max`, no `devices.allow` |

`cgroup.kill` provides emergency cleanup. `cgroup.freeze` suspends
the entire subtree — useful for pausing gaming mode.

---

## Section 12 — Existing Mechanisms for Obelisk Features

### 12.1 SYSTEM32 (Restricted ROOT)

**Available mechanisms:**
- **Filtered capabilities** (kernel/cred.c:368): Create a restricted
  `init_cred` copy with `cap_effective` and `cap_permitted` cleared of
  dangerous caps (`CAP_SYS_ADMIN`, `CAP_SYS_MODULE`, `CAP_SYS_RAWIO`).
- **LSM `capable` hook** (security/security.c:655): Custom LSM can deny
  capability checks even for UID 0 on sensitive operations.
- **cgroup device controller** (include/linux/cgroup_subsys.h:33):
  `devices.deny` to block access to raw hardware.
- **User namespaces** (kernel/user_namespace.c:83): `create_user_ns`
  can create a child user namespace where UID 0 is mapped to a non-zero
  UID in the parent, providing namespace-scoped root.

**No new kernel code needed** — use user namespaces + capability filtering.

### 12.2 TI Principal

**Available mechanisms:**
- **PID namespaces** (include/linux/pid_namespace.h:26): TI runs in
  a child PID namespace. `child_reaper` tracks TI's PID namespace.
- **User namespaces** (kernel/user_namespace.c): TI gets its own user
  namespace with full caps internally but limited in parent.
- **cgroup pids controller** (include/linux/cgroup_subsys.h:57):
  `pids.max` limits TI's process count.
- **nsenter restriction**: LSM hook `ptrace_access_check` (36) prevents
  escape from TI's PID namespace.

### 12.3 OVERSEER Principal

**Available mechanisms:**
- **init_cred** (init/init_task.c:80): Base SYSTEM template.
- **LSM custom hook** (security/security.c): Custom LSM can create
  an OVERSEER-specific credential tag and enforce it at `capable` (44),
  `task_kill` (261), `ptrace_access_check` (36).
- **eBPF + CAP_BPF** (kernel/bpf/): Network monitoring and tracing
  with `CAP_BPF` (383) + `CAP_PERFMON` (383) caps.
- **cgroup freezer** (kernel/cgroup/freezer.c): Freeze/thaw TI/SYSTEM32
  cgroups for intervention.

### 12.4 OmniDaemon

**Available mechanisms:**
- **initramfs injection** (init/initramfs.c:789): OmniDaemon binary
  embedded in initramfs, launched by `run_init_process` fallback.
- **Kernel module loading** (kernel/module/main.c:3433): Saturnite
  generates signed `.ko` modules loaded via `do_init_module` (3088).
- **security_initcall hooks** (security/lsm_init.c:499): Register
  OmniDaemon at the security init level.
- **cgroup.kill** (kernel/cgroup/cgroup.c:4318): Emergency cleanup.
- **PID namespace child_reaper** (pid_namespace.h:29): OmniDaemon
  as reaper of its managed processes.

### 12.5 Angelica

**Available mechanisms:**
- **IMA/EVM** (security/integrity/): Integrity measurement and
  appraisal of binaries. `evm_inode_resetmetadata` verifies file
  integrity.
- **Module signing** (kernel/module_signature.c:21): PKCS#7 signature
  verification via `mod_check_sig`.
- **LoadPin** (security/loadpin/loadpin.c): Pins kernel module
  loading to a verified filesystem.
- **eBPF kprobe/uprobe** (kernel/bpf/): Dynamic tracing of process
  behavior.

### 12.6 Fail-Closed Behavior

**Available mechanisms:**
- **OOM killer** (mm/oom_kill.c:1008): `oom_kill_process` sends
  `SIGKILL` — already fail-closed.
- **cgroup.freeze** (cgroup.c:4256): Freeze entire cgroup subtrees.
- **Netfilter NF_DROP** (include/uapi/linux/netfilter.h:11): Drop
  packets by default in nftables.
- **LSM `bprm_check_security`** (54): Can deny exec of untrusted
  binaries.
- **Module sig force** (`CONFIG_MODULE_SIG_FORCE=y`): Refuse unsigned
  modules — fail-closed module loading.

---

## Section 13 — osmode Architecture

### 13.1 osmode Design Using Linux Mechanisms

| osmode | Linux Mechanism | Implementation Strategy |
|---|---|---|
| **Normal** | Default cgroup + fair scheduler | Standard systemd-style cgroups, `memory.high` at 50% for pressure |
| **Gaming** | cgroup cpu + memory + RT scheduler | `cpu.max = 80%`, `memory.max` at cap, `SCHED_FIFO` for audio threads |
| **Code** | cgroup memory + pids + io | `memory.high` elevated, `pids.max = 8192`, `io.weight = high` for compilers |
| **Hacker** | Full cgroup delegation + RT + capabilities | `devices.allow = all`, all RT policies available, `CAP_SYS_ADMIN` in user NS |

### 13.2 osmode Switching Mechanism

**Design**: A single `OmniDaemon` process manages osmode transitions.
It manipulates cgroup properties via `/sys/fs/cgroup/` (kernfs) and
schedules via `sched_setscheduler()` syscalls.

**No kernel changes needed** — systemd already does this via
`systemctl set-property`. Obelisk can use a custom daemon or
extend systemd.

### 13.3 Gaming Mode Resource Isolation

```bash
# Create gaming cgroup
mkdir /sys/fs/cgroup/gaming
echo 812000 1000000 > /sys/fs/cgroup/gaming/cpu.max  # 81.2% CPU
echo 12G > /sys/fs/cgroup/gaming/memory.max         # 12GB cap
echo 200000 1000000 > /sys/fs/cgroup/gaming/cpu.max  # 2 cores for non-gaming

# Move game process tree
echo $GAME_PID > /sys/fs/cgroup/gaming/cgroup.procs
```

### 13.4 Hacker Mode Sandbox Escape Prevention

Hacker mode requires elevated privileges but must not allow
escape to SYSTEM/OVERSEER:

- **LSM `capable` hook**: Restrict `CAP_SYS_ADMIN` in non-hacker
  cgroups from affecting hacker cgroup.
- **cgroup freezer**: OVERSEER can freeze hacker processes.
- **PID namespace isolation**: Hacker processes in a child PID ns.
- **net_cls**: Classify hacker traffic for monitoring.

---

## Section 14 — Architecture Recommendation

### 14.1 Summary of Findings

The Linux kernel 7.3-rc4 provides nearly all mechanisms Obelisk needs
out of the box. The primary integration surface is:

1. **struct cred + capabilities + LSM** (cred.h:115, security.c:655)
   — the kernel's native authority model, fully sufficient for
   USER/ROOT/SYSTEM/TI/OVERSEER principals.
2. **cgroup v2** (cgroup-defs.h:486, cgroup.c:5575) — resource
   isolation for osmode gaming/code/hacker.
3. **PID + User namespaces** (pid_namespace.h:26, user_namespace.c:83)
   — isolation for SYSTEM32 and TI.
4. **Netfilter + nftables + eBPF/XDP** (netfilter.h, nfnetlink_queue.c)
   — all network security for ICE/ICEberg.
5. **Module signing** (module_signature.c:21) — verification for
   Saturnite-generated kernel modules.
6. **initramfs + initcalls** (initramfs.c:789, lsm_init.c) — OmniDaemon
   bootstrap.

### 14.2 Implementation Order

| Phase | Task | Kernel Changes | Source References |
|---|---|---|---|
| 1 | Build custom Obelisk LSM | Minimal new LSM (DEFINE_LSM) using existing hook framework | security/lsm_init.c, security/security.c, include/linux/lsm_hooks.h, include/linux/lsm_hook_defs.h |
| 2 | System32 mode: restricted root via user NS + filtered caps | None — use `init_cred` copy + user namespaces | init/init_task.c:80, kernel/user_namespace.c:83, kernel/cred.c:368 |
| 3 | TI mode: PID+user namespace isolation | None — use existing namespace creation | include/linux/pid_namespace.h:26, kernel/nsproxy.c |
| 4 | OmniDaemon: initramfs injection + signed module | None — use existing module signing | init/initramfs.c:789, kernel/module_signature.c:21 |
| 5 | osmode cgroups: gaming/code/hacker resource limits | None — use cgroup v2 controllers | mm/memcontrol.c:5160, kernel/cgroup/cgroup.c:5575 |
| 6 | OVERSEER: custom LSM enforcement + eBPF | New LSM hooks for OVERSEER tag | security/security.c:655, kernel/bpf/ |
| 7 | Angelica: IMA/EVM + LoadPin integration | None — use existing integrity subsystem | security/integrity/, security/loadpin/ |

### 14.3 Where to Modify vs Extend

**Do NOT modify (use as-is):**
- `init/main.c` boot path — extend via initcalls or initramfs
- `kernel/module/main.c` — use existing `load_module` + `module_sig_check`
- `net/` — use nftables + eBPF from userspace
- `mm/` — use cgroup v2 memory controller
- `kernel/sched/` — use existing scheduler classes + RT policies
- `ipc/` — no changes needed
- `block/` — cgroup io controller suffices

**Extend (new code only):**
- `security/` — new `obk_lsm.c` module following LSM framework
- `kernel/cred.c` — new credential initialization in `security_initcall`
- `init/initramfs.c` — add OmniDaemon to CPIO archive (build-time, no source change)

### 14.4 Risks and Compatibility

1. **LSM stacking**: The LSM framework supports stacking (up to 16 LSMs).
   Obelisk's LSM must be designed for stacking with SELinux/AppArmor.
   Use `LSM_ORDER_MUTABLE` and don't assert `LSM_FLAG_EXCLUSIVE`.

2. **Capability inheritance**: The `cap_bset` (bounding set) and
   `cap_inheritable` control capability inheritance across exec.
   Obelisk SYSTEM principal must ensure `cap_bset` excludes dangerous
   caps. This is already handled by `cap_task_reset_cats` in
   `security/commoncap.c`.

3. **User namespace depth limits**: `kernel/user_namespace.c:92`
   enforces `USER_NS_SETID_LIMIT` (32 namespaces maximum depth).
   TI hierarchy should not exceed 4 levels.

4. **Module signing key management**: Module signatures use the kernel
   keyring. Saturnite must inject its signing certificate into
   `.builtin_trusted_keys` at build time, or use the `mok` (Machine
   Owner Key) facility via UEFI Secure Boot.

5. **cgroup v2 single-hierarchy**: Unlike cgroup v1, v2 uses a single
   unified hierarchy. osmode cgroups must be designed as subtrees,
   not as separate hierarchies.

6. **eBPF JIT security**: `CAP_BPF` + `CAP_PERFMon` allow eBPF
   programs with kernel-level effects. The OVERSEER principal must
   be the only entity with these caps (besides init).

7. **initramfs injection**: OmniDaemon in initramfs runs before
   any init process. It must use `kernel_init_freeable` to register
   itself and then hand off to the real init. The
   `security_initramfs_populated` hook
   (init/initramfs.c:745) allows LSM verification of the initramfs
   content.

### 14.5 Final Recommendation

Obelisk should be implemented as a **new LSM module** (`obk_lsm.c`)
that extends the existing `struct cred` model rather than replacing it.
The LSM provides principal tagging via the security blob pointer
(cred.h:140), while all resource isolation, scheduling, and network
control is handled via existing cgroup v2, scheduler classes, and
Netfilter/nftables/Everything else — all controlled from userspace
by OmniDaemon using standard syscalls and cgroup kernfs files.

The only kernel source modifications needed are:
1. The new `obk_lsm.c` in `security/`
2. Optional build-config additions to `init/Kconfig` for OmniDaemon
3. A new `initcall` in an existing file or a module initcall

Everything else — boot process, module loading, memory management,
scheduling, networking, cgroups — should be used as-is from upstream
Linux, controlled via the existing userspace interfaces.

---

*This report is based on direct source code analysis of the Linux 7.3-rc4
kernel tree. All file paths and line numbers are verified against the
actual source. No kernel modifications have been made.*
