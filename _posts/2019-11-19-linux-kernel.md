---
title: 深入Linux内核架构:学习笔记
subtitle: 具体的实现原理，巩固现代操作系统知识点
layout: post
tags: [linux, os]
---

深入Linux内核架构学习笔记，具体的实现原理，巩固现代操作系统知识点。

大佬出品，好好研究一下。

# 进程管理和调度

The allocation of CPU time can be portrayed in much simplified form as in Figure 2-1. Processes are
spread over a time slice, and the share of the slice allocated to them corresponds to their relative impor-
tance. The time flow in the system corresponds to the turning of the circle, and the CPU is represented by
a ‘‘scanner‘‘ at the circumference of the circle. The net effect is that important processes are granted more
CPU time than less important processes, although all eventually have their turn.

![](../img/linux-kernel-cpu-allocation.png)

*preemptive multitasking*: 抢占式多任务。 each process is allocated a certain time period during which it may execute. Once this period has expired, the kernel withdraws control from the process and lets a different process run — regardless of the last task performed by the previous process. 

Its **runtime environment** — essentially, **the contents of all CPU registers and the page tables** — is, of course, **saved so that results are not lost and the process environment is fully reinstated when its turn comes around again.** The length of the time slice varies depending on the importance of the process (and therefore on the priority assigned to it). Figure 2-1 illustrates this by allocating segments of different sizes to the individual processes.

## 进程生命周期

**The scheduler must know the status of every process in the system when switching between tasks**; it
obviously doesn’t make sense to assign CPU time to processes that have nothing to do. Of equal impor-
tance are the **transitions between individual process states**. 

A process may have one of the following states: 

- ❑  **Running** — The process is executing at the moment. 
- ❑  **Waiting** — The process is able to run but is not allowed to because the CPU is allocated to another process. The scheduler can select the process, if it wants to, at the next task switch. 
- ❑  **Sleeping** — The process is sleeping and cannot run because **it is waiting for an external event**. The scheduler *cannot* select the process at the next task switch. 



**The system saves all processes in a <u>process table</u>** — regardless of whether they are running, sleeping, or waiting.

 However, **sleeping processes are specially ‘‘marked‘‘ so that the scheduler knows they are not ready** to run (see how this is implemented in Section 2.3).

 **There are also a number of queues that group sleeping processes** so that **they can be woken at a suitable time** — when, for example, an external event that the process has been waiting for takes place. 

Figure 2-2 shows several process states and transitions.

![](../img/linux-kernel-transition.png)

#### 僵尸状态

A special process state not listed above is the **‘‘zombie‘‘state**. As the name suggests, such processes are defunct（解散） but are somehow still alive. **In reality, they are dead because <u>their resources (RAM, connections to peripherals, etc.) have already been released</u> so that they cannot and never will run again. However, they are still alive because <u>there are still entries for them in the process table</u>.** 

#####  1. How do zombies come about? 

The reason lies in the process creation and destruction structure under Unix. **A program terminates when two events occur**：

-   **first**, the program must be killed by another process or by a user (this is usually done by sending a `SIGTERM` or `SIGKILL` signal, which is equivalent to terminating the process regularly); 
- **second**, the **parent process** from which the process originates **must invoke or have already invoked the wait4 (read: *wait for*) system call when the child process terminates.** This confirms to the kernel that **the parent process has acknowledged the death of the child**. **The system call enables the kernel to free resources reserved by the child process.** 

A zombie occurs when only the first condition (the program is terminated) applies but not the second (wait4). **A process always switches briefly to the zombie state between termination and removal of its data from the process table.** In some cases (if, e.g., the parent process is badly programmed and does not issue a wait call), **a zombie can firmly lodge 保留 itself in the process table** and remain there until the next reboot. This can be seen by reading the output of process tools such as `ps` or `top`. This is hardly a problem as the residual data take up little space in the kernel.

## 抢占式多任务

The structure of Linux process management requires two further process state options — `user mode` and
`kernel mode`. 

If a process wants to access system data or functions (the latter manage the resources shared between all
processes, e.g., filesystem space), it must switch to kernel mode. Obviously, this is possible only under
control and via clearly defined paths. Chapter 1 mentioned briefly that ‘‘**system calls**‘‘ are one way to switch between modes. Chapter 13 discusses the implementation of such calls in depth.

**A second way** of switching from user mode to kernel mode is **by means of interrupts** — switching is
then triggered automatically. 

Unlike system calls, which are invoked intentionally by user applications, interrupts occur more or less arbitrarily. Generally, the actions needed to handle interrupts have nothing to do with the process executing when the interrupt occurred. For example, an interrupt is raised when an external block device has transferred data to RAM, although these data may be intended for any process running on the system. Similarly, incoming network packages are announced by means of aninterrupt. Again, it is unlikely that the inbound package is intended for the process currently running.
For this reason, Linux performs these actions in such a way that the running process is totally unaware
of them.



The **preemptive scheduling model** of the kernel establishes a hierarchy that determines which process states may be interrupted by which other states. 

- ❑  **Normal processes may always be interrupted** — even by other processes. **When an important process becomes runnable (for example, an editor receives long-awaited keyboard input —) ,the scheduler can decide whether to execute the process immediately, even if the current process is still happily running.** This kind of preemption makes an important contribution to good interactive behavior and low system latency. 
- ❑  **If the system is in <u>kernel mode</u> and is <u>processing a system call</u>, no other process in the system is able to cause withdrawal of CPU time.** The scheduler is forced to wait until execution of the system call has terminated before it can select another process. **However, the system call can be suspended by an interrupt**.
- ❑  **Interrupts can suspend processes in user mode** and **in kernel mode**. They have the highest priority because it is essential to handle them as soon as possible after they are issued. 



## 进程描述

**All algorithms of the Linux kernel concerned with processes and programs are built around a data structure named `task_struct` and defined in `include/sched.h`.** This is one of the central structures in the system. Before we move on to deal with the implementation of the scheduler, **it is essential to examine how Linux manages processes**.



The **task structure includes a large number of elements that link the process with the kernel subsystems** which I discuss below. I therefore make frequent reference to later chapters because **it is difficult to explain the significance of some elements without detailed knowledge of them**.

```c
<sched.h>
struct task_struct { 
/* -1 unrunnable, 0 runnable, >0 stopped */
volatile long state; 
void *stack;
atomic_t usage; 
/* per process flags, defined below */
unsigned long flags; 
unsigned long ptrace; 
/* BKL lock depth */
int lock_depth;

int prio, static_prio, normal_prio; 
struct list_head run_list;
const struct sched_class *sched_class; 
struct sched_entity se;

unsigned short ioprio;

unsigned long policy; 
cpumask_t cpus_allowed; 
unsigned int time_slice;

#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT) 
struct sched_info sched_info;
#endif

struct list_head tasks; 
/*
* ptrace_list/ptrace_children forms the list of my children 
* that were stolen by a ptracer.
*/
struct list_head ptrace_children;
struct list_head ptrace_list;

struct mm_struct *mm, *active_mm;

/* task state */
struct linux_binfmt *binfmt;
long exit_state;
int exit_code, exit_signal;
/* The signal sent when the parent dies */
int pdeath_signal; 

unsigned int personality; 
unsigned did_exec:1; 
pid_t pid;
pid_t tgid;

/*
* pointers to (original) parent process, youngest child, younger sibling,
* older sibling, respectively. (p->father can be replaced with
* p->parent->pid) */
struct task_struct *real_parent; /* real parent process (when being debugged) */

struct task_struct *parent; /* parent process*/

/*
* children/sibling forms the list of my children plus the
* tasks I’m ptracing. */
struct list_head children; 
struct list_head sibling; 
struct task_struct *group_leader;/* threadgroup leader */


/* PID/PID hash table linkage. */ 
struct pid_link pids[PIDTYPE_MAX]; 
struct list_head thread_group;

struct completion *vfork_done; /* for vfork() */
int __user *set_child_tid;     /* CLONE_CHILD_SETTID */
int __user *clear_child_tid;   /* CLONE_CHILD_CLEARTID */

unsigned long rt_priority;
cputime_t utime, stime, utimescaled, stimescaled;
unsigned long nvcsw, nivcsw; /* context switch counts */ 

struct timespec start_time; /* monotonic time */
struct timespec real_start_time; /* boot based time */

/* mm fault and swap info: this can arguably be seen as either
mm-specific or thread-specific */ 
unsigned long min_flt, maj_flt;

cputime_t it_prof_expires, it_virt_expires; 
unsigned long long it_sched_expires;
struct list_head cpu_timers[3];

/* process credentials */
uid_t uid,euid,suid,fsuid;
gid_t gid,egid,sgid,fsgid;
struct group_info *group_info;
kernel_cap_t cap_effective, cap_inheritable, cap_permitted;
unsigned keep_capabilities:1; 
struct user_struct *user;
char comm[TASK_COMM_LEN]; /* executable name excluding path
														- access with [gs]et_task_comm (which lock
                            it with task_lock())
                            - initialized normally by flush_old_exec */

/* file system info */
int link_count, total_link_count; 
/* ipc stuff */
struct sysv_sem sysvsem;
/* CPU-specific state of this task */
struct thread_struct thread; 
/* filesystem information */
struct fs_struct *fs; 
/* open file information */
struct files_struct *files; 
/* namespace */
struct nsproxy *nsproxy; 
/* signal handlers */
struct signal_struct *signal; 
struct sighand_struct *sighand;
sigset_t blocked, real_blocked; 
sigset_t saved_sigmask;
struct sigpending pending;
unsigned long sas_ss_sp; 
size_t sas_ss_size;
int (*notifier)(void *priv); 
void *notifier_data; 
sigset_t *notifier_mask;
#ifdef CONFIG_SECURITY 
void *security;
#endif
/* Thread group tracking */ 
u32 parent_exec_id;
u32 self_exec_id;
/* journalling filesystem info */ 
void *journal_info;
/* To be restored with TIF_RESTORE_SIGMASK */

/* VM state */ 
struct reclaim_state *reclaim_state;
struct backing_dev_info *backing_dev_info;
struct io_context *io_context;
unsigned long ptrace_message;
siginfo_t *last_siginfo; /* For ptrace use. */
...
};

```



Admittedly诚然, it is difficult to digest消化 the amount of **information in this structure**. However, the **structure contents can be broken down into sections**, each of which represents a specific aspect of the process: 

❑  **State and execution information** such as 

- **pending signals**
- **binary format** used (and any emulation information for binary formats of other systems), 
- process identification number (**pid**)
- **pointers** to <u>parents and other related processes</u>
- **priorities**,
- **time information** on program execution (e.g., CPU time). 

❑  Information on **allocated virtual memory** . 

❑  **Process credentials** such as <u>user and group ID, capabilities</u> and so on. **System calls can be used to query (or modify) these data**; I deal with these in greater detail when describing the specific subsystems. 

❑  **Files used**: Not only **the binary file** with the program code but also filesystem information on **all files handled by the process** must be saved. 

❑  **Thread information**, which **records the CPU-specific runtime data of the process** (the remaining fields in the structure are not dependent on the hardware used). 

❑  Information on **interprocess communication（IPC）** required **when working with other applications**. 

❑  **Signal handlers** used by the process to respond to incoming signals. 



Many members of the task structure are not simple variables but pointers to other data structures examined and discussed in the following chapters.

 **In the present chapter, I consider some elements of task_struct that are of particular significance in process management implementation.** 

`state` specifies` the current state of a process` and `accepts the following values` (these are pre-processor constants defined in `<sched.h>`): 

❑  **TASK_RUNNING** means that a task is in a runnable state. It does not mean that a CPU is actually allocated. The task can wait until it is selected by the scheduler. This state guarantees that the process really is ready to run and is not waiting for an external event. 

❑  **TASK_INTERRUPTIBLE** is set for a sleeping process that is waiting for some event or other. When the kernel signals to the process that the event has occurred, it is placed in the TASK_RUNNING state and may resume execution as soon as it is selected by the scheduler. 

❑  **TASK_UNINTERRUPTIBLE** is used for sleeping processes disabled on the instructions of the kernel. They may not be woken by external signals, only by the kernel itself. 

❑  **TASK_STOPPED** indicates that the process was stopped on purpose — by a debugger, for example. 

❑  **TASK_TRACED** is not a process state per se — it is used to distinguish stopped tasks that are currently being traced (using the ptrace mechanism) from regular stopped tasks.



The **following constants** can be used both in the `task state` field of struct `task_struct`, but also in the field `exit_state`, which is specifically for exiting processes. 

- ❑  **EXIT_ZOMBIE** is the zombie state described above. 
- ❑  **EXIT_DEAD** is the state after an appropriate wait system call has been issued and before the task is completely removed from the system. This state is only of importance if multiple threads issue wait calls for the same task. 



**Linux provides the *resource limit* (rlimit) mechanism to impose certain system resource usage limits on processes. The mechanism makes use of the `rlim` array in `task_struct`, whose elements are of the `struct rlimit` type.** 

``` c
<resource.h> 

struct rlimit { 
unsigned long rlim_cur;

unsigned long rlim_max;

} 
```

The definition is purposely kept very general so that it can accept many different resource types. 

❑  `rlim_cur` is the current resource limit for the process. It is also referred to as the *soft limit*. 

❑ ` rlim_max `is the maximum allowed value for the limit. It is therefore also referred to as the *hard limit*. 

The `setrlimit` system call is used to increase or decrease the current limit. However, the value specified in rlim_max may not be exceeded. `getrlimits` is used to check the current limit. 

The limitable resources are identified by reference to their position in the `rlim` array, **which is why the kernel defines pre-processor constants to associate resource and position**. Table 2-1 lists the possible constants and their meanings. Textbooks on system programming provide detailed explanations on best use of the various limits in practice, and the manual page setrlimit(2) contains more detailed descriptions of all limits. 

### 进程类型

进程生成方式：

- ❑  **fork** generates an identical copy of the current process; this copy is known as a ***child process*.** All resources of the original process are copied in a suitable way so that after the system call there are two independent instances of the original process. These instances are not linked in any way but have, for example, the same set of open files, the same working directory, the same data in memory (each with its own copy of the data), and so on.
- ❑  **exec** <u>replaces a running process with another application loaded from an executable binary file</u>. In other words, a new program is loaded. Because exec does not create a new process, an old program must first be duplicated using fork, and then exec must be called to generate an additional application on the system. 

**Linux also provides the <u>clone</u> system call** in addition to the two calls above that are available in all Unix flavors and date back to very early days. **In principle, clone works in the same way as fork, but the new process is not independent of its parent process and can share some resources with it. It is possible to specify *which* resources are to be shared and which are to be copied — for example, data in memory, open files, or the installed signal handlers of the parent process.** 



**clone** is used to **implement *threads*.** However, the system call alone is not enough to do this. Libraries are also needed in userspace to complete implementation. Examples of such libraries are *Linuxthreads* and *Next Generation Posix Threads*. 

### 命名空间

**Namespaces provide a lightweight form of virtualization by allowing us to view the global properties of a running system under different aspects.** The mechanism is similar to *zones* in Solaris or the *jail* mech- anism in FreeBSD. After a general overview of the concept, I discuss the infrastructure provided by the namespace framework. 

#### 概念



#### 实现

The implementation of namespaces requires two components: 

- **per-subsystem namespace structures** ： wrap all formerly global components on a per-namespace basis
- a **mechanism**： associates a given process with the individual namespaces to which it belongs. Figure 2-4 illustrates the situation. 

![](../img/linux-ns-impl.png)

Formerly global properties of subsystems are wrapped up in namespaces, and each process is associated
with a particular selection of namespaces. Each kernel subsystem that is aware of namespaces must
provide a data structure that collects all objects that must be available on a per-namespace basis. struct
nsproxy is used to collect pointers to the subsystem-specific namespace wrappers:

```c
<nsproxy.h>
struct nsproxy { 
    atomic_t count;
    struct uts_namespace *uts_ns; 
    struct ipc_namespace *ipc_ns; 
    struct mnt_namespace *mnt_ns; 
    struct pid_namespace *pid_ns; 
    struct user_namespace *user_ns; 
    struct net *net_ns;
};

```

Currently the following areas of the kernel are aware of namespaces: 

- ❑  The **UTS namespace** contains the name of the running kernel, and its version, the underlying 

  architecture type, and so on. UTS is a shorthand for Unix *Timesharing System*. 

- ❑  All information related to **inter-process communication (IPC)** is stored in struct `ipc_namespace`. 

- ❑  The view on the **mounted filesystem** is given in struct `mnt_namespace`. 

- ❑  struct `pid_namespace` provides **information about process identifiers**. 

- ❑  struct `user_namespace` is required to **hold per-user information that allows for limiting resource usage for individual users**. 

- ❑  struct `net_ns contains` all **networking-related namespace parameters**. There is, however, still quite a lot of effort required to make this area fully aware of namespaces as you will see in Chapter 12. 



**In this chapter, we will be concerned about UTS and user namespaces**.

 Since` fork` can be instructed to open a new namespace when a new task is created, appropriate flags to control the behavior must be provided. One flag is available for each individual namespace: 

```c
<sched.h>
#define CLONE_NEWUTS 0x04000000  /* New utsname group? */
#define CLONE_NEWIPC 0x08000000  /* New  ipcs */
#define CLONE_NEWUSER 0x10000000 /* New user namespace */
#define CLONE_NEWPID 0x20000000  /* New pid namespace */ 
#define CLONE_NEWNET 0x40000000  /* New  network namespace */
```

**Each task is associated with his own view of the namespaces**:

```c
<sched.h>
struct task_struct { 
    ...
    /* namespaces */
    struct nsproxy *nsproxy;
    ... 
}
```



**Because a pointer is used, a collection of sub-namespaces can be shared among multiple processes.** This way, **changes in a given namespace will be visible in all processes that belong to this namespace**.

Notice that support for namespaces must be enabled at compile time on a per-namespace basis. Generic
support for namespaces is, however, always compiled in. This allows the kernel to avoid using different
code for systems with and without namespaces.

The initial global namespace is defined by `init_nsproxy`, which keeps pointers to the initial objects of the per-subsystem namespaces:



```c
<kernel/nsproxy.c>
struct nsproxy init_nsproxy = INIT_NSPROXY(init_nsproxy);
<init_task.h>
#define INIT_NSPROXY(nsproxy) { \ 
.pid_ns = &init_pid_ns, \
.count = ATOMIC_INIT(1), \
.uts_ns = &init_uts_ns, \ 
.mnt_ns = NULL, \ 
INIT_NET_NS(net_ns) \ 
INIT_IPC_NS(ipc_ns) \ 
.user_ns = &init_user_ns, \
}
```



#### The UTS Namespace





#### The User Namespace

The user namespace is handled similarly in terms of data structure management: **When a new user**
**namespace is requested, a copy of the current user namespace is generated and associated with the**
**nsproxy instance of the current task**. However, the representation of a user namespace itself is slightly
more complex:

```c
<user_namespace.h>
struct user_namespace { 
struct kref kref;
struct hlist_head uidhash_table[UIDHASH_SZ]; 
struct user_struct *root_user;
};

```

As before, kref is a reference counter that tracks in how many places a user_namespace instance is
required. For each user in the namespace, an instance of struct user_struct keeps track of the individ-
ual resource consumption, and the individual instances are accessible via the hash table uidhash_table.



**The exact definition of user_struct is not interesting for our purposes.** It suffices to know that some statistical elements like the number of open files or processes a user has are kept in there. What is much more interesting is that each user namespace accounts resource usage for its users completely detached from other namespaces — including accounting for the root user. This is possible because a new user_struct both for the current user and the root is created when a user namespace is cloned:



```c
kernel/user_namespace.c
static struct user_namespace *clone_user_ns(struct user_namespace *old_ns) {
...
struct user_namespace *ns; 
struct user_struct *new_user;
... 
ns = kmalloc(sizeof(struct user_namespace), GFP_KERNEL);
ns->root_user = alloc_uid(ns, 0);
/* Reset current->user with a new one */ 
new_user = alloc_uid(ns, current->uid);
switch_uid(new_user); return ns;
}

```

`alloc_uid` is a helper function that allocates an instance of user_struct for a user with a given UID in the current namespace if none exists yet. Once an instance has been set up for both root and the current user, switch_uid ensures that the new user_struct will be used to account resources from now on. This essentially works by setting the user element of struct task_struct to the new user_struct instance. 

Notice that if support for user namespaces is not compiled in, cloning a user namespace is a null opera- tion: The default namespace is always used. 

### Process Identification Numbers

Unix processes are always assigned a number to uniquely identify them in their namespace. This number
is called the *process identification number* or *PID* for short. Each process generated with fork or clone is
automatically assigned a new unique PID value by the kernel.



#### Process Identifiers 

Each process is, however, not only characterized by its PID but also by other identifiers. Several types are possible: 

- ❑  All processes in a thread group (i.e., different execution contexts of a process created by call-ing clone with CLONE_THREAD as we will see below) have a uniform *thread group id* (TGID). If a process does not use threads, its PID and TGID are identical. 

  The main process in a thread group is called the *group leader*. The group_leader element of the task structures of all cloned threads points to the task_struct instance of the group leader. 

- ❑  Otherwise, independent processes can be combined into a *process group* (using the setpgrp sys- tem call). The pgrp elements of their task structures all have the same value, namely, the PID of the process group leader. Process groups facilitate the sending of signals to all members of the group, which is helpful for various system programming applications (see the literature on sys- tem programming, e.g., [SR05]). Notice that processes connected with pipes are contained in a process group. 

- ❑  Several process groups can be combined in a session. All processes in a session have the same **session ID** which is held in the session element of the task structure. The **SID** can be set using the `setsid` system call. It is used in terminal programming but is of no particular relevance to us here. 



Namespaces add some additional complexity to how PIDs are managed. Recall that PID namespaces are organized in a hierarchy. 

**When a new namespace is created, all PIDs that are used in this namespace are visible to the parent namespace, but the child namespace does not see PIDs of the parent namespace**. 

We **have to distinguish** between **local and global IDs**:

❑  *Global IDs* are identification numbers that are valid within the kernel itself and in the initial namespace to which the init tasks started during boot belongs. For each ID type, a given global identifier is guaranteed to be unique in the whole system. 

❑  *Local IDs* belong to a specific namespace and are not globally valid. For each ID type, they are valid within the namespace to which they belong, but identifiers of identical type may appear with the same ID number in a *different* namespace. 

The **global PID** and **TGID** are directly stored in the **task struct**, namely, in the elements **pid** and tgid: 

```c
<sched.h> 

struct task_struct { 

... 

pid_t pid; 

pid_t tgid; 

... 

} 
```

Both are of type `pid_t`, which resolves to the type `__kernel_pid_t`; this, in turn, has to be defined by each architecture. Usually an int is used, which means that 232 different IDs can be used simultaneously. 

The **session and process group IDs** are not directly contained in the task structure itself, but **in the struc- ture used for signal handling**. `task_struct->signal->__session` denotes the global SID, while the global PGID is stored in `task_struct->signal->__pgrp`. The auxiliary functions `set_task_session` and `set_task_pgrp` are provided to modify the values. 

####  Managing PIDs 

In addition to these two fields, the kernel needs to find a way to **manage all local per-namespace quantities**, as well as the other identifiers like TID and SID. This requires several interconnected data structures and numerous auxiliary functions that are discussed in the following. 

##### Data Structures

Below I use the term ID to refer to *any* process identifier. I specify the identifier type explicitly (e.g., TGID for ‘‘thread group identifier’’) where this is necessary. 

A small subsystem known as a *pid allocator* is available to speed up the allocation of new IDs. 

Besides, the kernel needs to provide auxiliary functions :

-  functions allow for finding the `task` structure of a process by reference to an` ID` and its` type`,
-  functions that **convert** between the **in-kernel representation of IDs** and **the numerical values visible to userspace**. 

I need to discuss how PID namespaces are represented. The elements required for our purposes are as follows: 

```c
<pid_namespace.h>
struct pid_namespace { 
...
struct task_struct *child_reaper;
...
int level;
struct pid_namespace *parent;
};

```

In reality, the structure **also contains elements that are needed** by the PID allocator to produce a stream of unique IDs, but **these do not concern us now**. What is interesting are the following elements: 

- ❑  **Every PID namespace is equipped with a task** that assumes the role taken by `init` in the global picture. One of the purposes of` init` is to call `wait4` for orphaned  tasks, and this must likewise be done by the namespace-specific init variant. A pointer to the task structure of this task is stored in `child_reaper`. 

- ❑  `parent` is a pointer to the parent namespace, and `level` denotes **the depth in the namespace hier- archy.** The initial namespace has level 0, any children of this namespace are in level 1, children of children are in level 2, and so on. **Counting the levels is important because IDs in higher levels must be visible in lower levels.** From a given level setting, the kernel can infer how many IDs must be associated with a task. 

  Recall from Figure 2-3 that namespaces are hierarchically related. This clarifies the above definitions. 

**PID management** is centered around two data structures: struct `pid` is the kernel-internal representation of a PID, and struct `upid` represents the information that is **visible in a specific namespace.** The definition of both structures is as follows: 



```c
<pid.h>
struct upid { 
  int nr;
  struct pid_namespace *ns; 
  struct hlist_node pid_chain;
};
struct pid {
  atomic_t count;
  /* lists of tasks that use this pid */ 
  struct hlist_head tasks[PIDTYPE_MAX]; 
  int level;
  struct upid numbers[1];
};

```

Since these and some other data structures are comprehensively interconnected, Figure 2-5 provides an overview about the situation before I discuss the individual components. 

As for struct `upid`, `nr` represents the numerical value of an ID, and `ns` is a pointer to the namespace to which the value belongs. All `upid` instances are **kept on a hash table** to which we will come in a moment, and `pid_chain` allows for implementing hash overflow lists with standard methods of the kernel. 

The definition of struct `pid` is headed by a reference `counter count`. `tasks` is **an array with a hash list head for every ID type. This is necessary because an ID can be used for several processes.** **All task_struct instances that share a given ID are linked on this lis**t. `PIDTYPE_MAX` denotes the number of ID types: 

```c
<pid.h>
enum pid_type {
PIDTYPE_PID, 
PIDTYPE_PGID, 
PIDTYPE_SID, 
PIDTYPE_MAX
};

```



![](../img/linux_kernal_pid_manager.png)

Notice that thread group IDs are *not* contained in this collection! This is because **the thread group ID is**
**simply given by the PID of the thread group leader**, so a separate entry is not necessary.

**A process can be visible in multiple namespaces**, and the local ID in each namespace will be different.
`level` denotes in how many namespaces the process is visible (in other words, this is the depth of the
containing namespace in the namespace hierarchy), and `numbers` contains an instance of `upid` for each
level. Note that the array consists formally of one element, and this is true if a process is contained only
in the global namespace. Since the element is at the end of the structure, additional entries can be added
to the array by simply allocating more space.

Since all task structures that share an identifier are kept on a list headed by `tasks`, a list element is
required in struct `task_struct`:

```c
<sched.h>
struct task_struct { 
...
/* PID/PID hash table linkage. */ 
struct pid_link pids[PIDTYPE_MAX];
... 
};
```

The auxiliary data structure `pid_link` permits linking of `task` structures on the lists headed from struct `pid`: 

```c
<pid.h> 
struct pid_link { 
struct hlist_node node; 
struct pid *pid; 

};
```

 pid points to a pid instance to which the task belongs, and node is used as list element. 

A hash table is used to find the pid instance that belongs to a numeric PID value in a given namespace: 

``` c
kernel/pid.c 

static struct hlist_head *pid_hash;
```

` hlist_head` is a kernel standard data element used to create doubly linked hash lists (Appendix C 

describes the structure of such lists and introduces several auxiliary functions for processing them). 

`pid_hash` is used as **an array of hlist_heads**. The number of elements is determined by the RAM con- figuration of the machine and lies between 2^4 = 16 and 2^12 = 4, 096. `pidhash_init` computes the apt size and allocates the required storage. 

Suppose that a new instance of **struct pid** has been allocated and set up for a given ID type type. It is
attached to a task structure as follows:

```c
kernel/pid.c
int fastcall attach_pid(struct task_struct *task, enum pid_type type, struct pid *pid)
{
struct pid_link *link;
link = &task->pids[type];
link->pid = pid;
hlist_add_head_rcu(&link->node, &pid->tasks[type]);
return 0;
}
```

A connection is made in both directions: The` task structure` can access the `pid` instance via  `task_struct->pids[type]->pid`.

 Starting from the `pid` instance, the` task` can be found by iterating over the `tasks[type] `list. `hlist_add_head_rcu` is a standard function to traverse a list that additionally ensures as per the RCU mechanism (see Chapter 5) that the iteration is safe against race conditions that could arise when other kernel components manipulate the list concurrently.

> RCU（Read-Copy Update）是数据同步的一种方式，在当前的Linux内核中发挥着重要的作用。
>
> RCU主要针对的数据对象是链表，目的是提高遍历读取数据的效率，为了达到目的使用RCU机制读取数据的时候不对链表进行耗时的加锁操作。这样在同一时间可以有多个线程同时读取该链表，并且允许一个线程对链表进行修改（修改的时候，需要加锁）。

#### Functions

**The kernel provides a number of auxiliary functions to manipulate and scan the data structures described above.** Essentially **the kernel must be able to fulfill two different tasks**: 

1. **Given a local numerical ID** and **the corresponding namespace**, **find the task structure that is described by this tuple**. 
2. **Given a task structure, an ID type, and a namespace**, **obtain the local numerical ID.** 

Let us first concentrate on the case in which a `task_struct` instance must be <u>converted into a numerical</u> 

<u>ID</u>. This is a two-step process: 

1. **Obtain the pid instance associated with the task structure**. The auxiliary functions `task_pid`, `task_tgid`,` task_pgrp`, and` task_session` are provided for the different types of IDs. This is simple for PIDs: 

   ``` c
   <sched.h> 
   
   static inline struct pid *task_pid(struct task_struct *task) { 
   
   return task->pids[PIDTYPE_PID].pid; 
   
   } 
   ```

   

Obtaining a TGID works similarly because it is nothing other than the PID of the tread group leader. The element to grab is `task->group_leader->pids[PIDTYPE_PID].pid`. 

Finding out a process group ID requires using **PIDTYPE_PGID** as array index. However, it must again be taken from the pid instance of the process group leader: 

``` c
<sched.h> 

static inline struct pid *task_pgrp(struct task_struct *task) { 

return task->group_leader->pids[PIDTYPE_PGID].pid; 

}
```



2. Once the pid instance is available, the numerical ID can be read off from the uid information 

available in the numbers array in struct pid: 

``` c
kernel/pid.c 

pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns) { 

struct upid *upid; 
pid_t nr = 0; 
if (pid && ns->level <= pid->level) {
	upid = &pid->numbers[ns->level]; 
  if (upid->ns == ns) 
    nr = upid->nr; 
}
return nr;
}
```

Because a parent namespace sees PIDs in child namespaces, but not vice versa, the kernel has to ensure that the current namespace level is less than or equal to the level in which the local PID was generated. 

It is also important to note that the kernel need only worry about generating global PIDs: All other ID types in the global namespace will be mapped to PIDs, so there is no need to generate, for instance, global TGIDs or SIDs. 

