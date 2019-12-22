---
title: 深入Linux内核架构（1）:进程管理
subtitle: 具体的实现原理，巩固现代操作系统知识点
layout: post
tags: [linux, os ,process]
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

...





### PID生成

除了管理PIDs，内核还要负责生成唯一的PIDs。

To keep track of which PIDs have been allocated and which are still free, **the kernel uses a large bitmap** in which each PID is identified by a bit. The value of the PID is obtained from the position of the bit in the bitmap. 

Allocating a free PID is then restricted essentially to looking for the first bit in the bitmap whose value is 0; this bit is then set to 1. Conversely, freeing a PID can be implemented by ‘‘toggling‘‘ the corresponding bit from 1 to 0. These operations are implemented using 

```c
kernel/pid.c 

static int alloc_pidmap(struct pid_namespace *pid_ns) 
```

to reserve a PID, and 

```c
kernel/pid.c 

static fastcall void free_pidmap(struct pid_namespace *pid_ns, int pid) 
```

to free a PID. How they are implemented does not concern us here, but naturally, they must work on a per-namespace basis. 

When a new process is created, it may be visible in multiple namespaces. For each of them a local PID must be generated. This is handled in `alloc_pid`: 

```c
kernel/pid.c
struct pid *alloc_pid(struct pid_namespace *ns) {
    struct pid *pid; 
    enum pid_type type; 
    int i, nr;
    struct pid_namespace *tmp; 
    struct upid *upid;
    ...
    tmp = ns;
    for (i = ns->level; i >= 0; i--) {
        nr = alloc_pidmap(tmp);
        ...
        pid->numbers[i].nr = nr; 
        pid->numbers[i].ns = tmp; 
        tmp = tmp->parent;
    }
    pid->level = ns->level;
		...
}
```

All upids that are contained in struct pid are filled with the newly generated PIDs. Each upid instance must be placed on the PID hash: 

```c
kernel/pid.c 

for (i = ns->level; i >= 0; i--) { 
    upid = &pid->numbers[i]; 
    hlist_add_head_rcu(&upid->pid_chain, &pid_hash[pid_hashfn(upid->nr, upid->ns)]); 
} 
... 
return pid; 
} 
```

### Task 关系

**In addition to the relationships resulting from ID links**, the kernel is also responsible for **managing the ‘‘family relationships‘‘ established on the basis of the Unix model of process creation 进程创建模型.** The following **terminology 术语** is used in this context: 

❑  If process *A* forks to generate process *B*, *A* is known as the *parent* process and *B* as the *child* process.

If process *B* forks again to create a further process *C*, the relationship between *A* and *C* is sometimes referred to as a *grandparent* and *grandchild* relationship. **父子**

❑  If process *A* forks several times therefore generating several child processes *B*1 , *B*2 , . . . , *Bn* , the relationship between the *Bi* processes is known as a *siblings* relationship. **兄弟**

Figure 2-6 illustrates the possible family relationships graphically.
 The `task_struct task` data structure **provides two list heads to help implement these relationships**: 

```c
<sched.h>
struct task_struct { 
    ...
    struct list_head children; /* list of my children */
    struct list_head sibling; /* linkage in my parent’s children list */
    ...
}

```

❑  children is the list head for the list of all child elements of the process. 

❑  siblings is used to link siblings with each other. 

![](../img/linux-kernel-pid-family.png)

New children are placed at the *start* of the siblings list, meaning that the chronological sequence of
forks can be reconstructed.

## 进程管理的系统调用

I discuss the implementation of the `fork` and `exec` system call families. Normally, **these calls are not issued directly by applications** but are **invoked via an intermediate layer** — the C standard library — that is responsible for communication with the kernel.

The methods used to switch from user mode to kernel mode differ from architecture to architecture.

### 进程复制

The traditional Unix system call to duplicate a process is `fork`. However, it is not the only call imple-
mented by Linux for this purpose — in fact, there are three:

1. **fork** is the **heavy-weight** call because it creates a full copy of the parent process that then executes as a child process. **To reduce the effort associated with this call, Linux uses the *copy- on-write* technique, discussed below.** 

2. **vfork** is similar to fork but **does not create a copy of the data of the parent process**. Instead, **it shares the data between the parent and child process.** This saves a great deal of CPU time (and if one of the processes were to manipulate the shared data, the other would notice automatically). 

   `vfork` is designed for the situation in which a child process just generated immediately executes an `execve` system call to **load a new program**. **The kernel also guarantees that the parent process is blocked until the child process exits or starts a new program**. 

   Quoting the manual page vfork, it is ‘‘rather unfortunate that Linux revived this specter from the past.’’ **Since fork uses copy-on-write, the speed argument for vfork does not really count anymore, and its use should therefore be avoided.** 

3. **clone** generates threads and **enables a decision to be made as to exactly which elements are to be shared between the parent and the child process and which are to be copied**. 

#### Copy on Write

Not the entire address space of the process but only its page tables are copied. These establish the link between virtual address space and physical pages as described briefly in Chapter 1 and at length in Chapters 3 and 4. The address spaces of parent and child processes then point to the same physical pages.

 **parent and child processes must not be allowed to modify each other’s pages** , which is why the page tables of *both* processes indicate that only read access is allowed to the pages — even though they could be written to in normal circumstances.

Providing that both processes have only read access to their pages in memory, data sharing between the two is not a problem because no changes can be made. 

As soon as one of the processes attempts to write to the copied pages, the processor reports an access error to the kernel (errors of this kind are called *page faults*). The kernel then references additional memory management data structures (see Chapter 4) to check whether the page can be accessed in Read and Write mode or in Read mode only — if the latter is true, a *segmentation fault* must be reported to the  process. As you see in Chapter 4, the actual implementation of the page fault handler is more complicated because other aspects, such as swapped-out pages, must also be taken into account. 

The condition in which a page table entry indicates that a page is ‘‘Read Only’’ although normally it would be writable allows the kernel to recognize that the page is, in fact, a COW page. It therefore creates a copy of the page that is assigned exclusively to the process — and may therefore also be used for write operations. How the copy operation is implemented is not discussed until Chapter 4 because extensive background knowledge of memory management is required. 

**The COW mechanism enables the kernel to delay copying of memory pages for as long as possible and — more importantly — to make copying unnecessary in many cases. This saves a great deal of time.** 



#### Executing System Calls 

The entry points for the `fork`, `vfork`, and `clone` system calls are the `sys_fork`, `sys_vfork`, and `sys_clone` functions. Their definitions are architecture-dependent because the way in which parameters are passed between userspace and kernel space differs on the various architectures (see Chapter 13 for further infor- mation). **The task of the above functions is to extract the information supplied by userspace from the registers of the processors and then to invoke the architecture-*in*dependent `do_fork` function responsible for process duplication.** The prototype of the function is as follows. 

```c
kernel/fork.c 

long do_fork(unsigned long clone_flags, 
             unsigned long stack_start, 
						 struct pt_regs *regs, 
             unsigned long stack_size, 
             int __user *parent_tidptr, 
             int __user *child_tidptr) 
```

The function requires the following arguments: 

❑  A flag set (**clone_flags**) to specify duplication properties. The low byte specifies the signal num- ber to be sent to the parent process when the child process terminates. The higher bytes hold various constants discussed below. 

❑  **The start address of the user mode stack** (start_stack) to be used. 

❑  **A pointer to the register set holding the call parameters in raw form (regs).** The data type used is the architecture-specific struct pt_regs structure, which holds all registers in the order in which they are saved on the kernel stack when a system call is executed (more information is provided in Appendix A). 

❑  **The size of the user mode stack (stack_size).** This parameter is usually unnecessary and set to 0. 

❑  **Two pointers to addresses in userspace (parent_tidptr and child_tidptr) that hold the TIDs of the parent and child processes.** **They are needed for the thread implementation of the NPTL (*Native Posix Threads Lilbrary*) library**. I discuss their meaning below. 

The different fork variants are distinguished primarily by means of the flag set. On most architectures,the classical fork call is implemented in the same way as on IA-32 processors. 

```c
arch/x86/kernel/process_32.c 

asmlinkage int sys_fork(struct pt_regs regs) { 

return do_fork(SIGCHLD, regs.esp, &regs, 0, NULL, NULL); 

} 
```

The only flag used is `SIGCHLD`. **This means that the `SIGCHLD` signal informs the parent process once the child process has terminated**. Initially, the same stack (whose start address is held in the esp**(sp: stack pointer )** register on IA-32 systems) is used for the parent and child processes. However, the COW mechanism creates a copy of the stack for each process if it is manipulated and therefore written to. 

If `do_fork` was successful, the PID of the newly created task is returned as the result of the system call. Otherwise the (negative) error code is returned. 

The implementation of `sys_vfork` differs only slightly from that of `sys_fork` in that additional flags are used (CLONE_VFORK and CLONE_VM whose meaning is discussed below). 

`sys_clone` is also implemented in a similar way to the above calls with the difference that `do_fork` is invoked as follows: 

```c
arch/x86/kernel/process_32.c
asmlinkage int sys_clone(struct pt_regs regs) {
    unsigned long clone_flags;
    unsigned long newsp;
    int __user *parent_tidptr, *child_tidptr;
    clone_flags = regs.ebx;
    newsp = regs.ecx;
    parent_tidptr = (int __user *)regs.edx; 
    child_tidptr = (int __user *)regs.edi; 
    if (!newsp)
    	newsp = regs.esp;
    return do_fork(clone_flags, newsp, &regs, 0, parent_tidptr, child_tidptr);
 }
```

The `clone` flags are no longer permanently set but **can be passed to the system call as parameters in various registers**. 

Thus, **the first part of the function deals with extracting these parameters**.

 Also, **the stack of the parent process is not copied**; instead, **a new address (newsp) can be specified for it.** (**This is required to generate threads that share the address space with the parent process but use their own stack in this address space**.) 

**Two pointers (`parent_tidptr` and `child_tidptr`) in userspace are also specified for purposes of communication with thread libraries**. Their meaning is discussed in Section 2.4.1.



#### Implementation of **do_fork** 

All three `fork` mechanisms end up in `do_fork` in `kernel/fork.c` (an architecture-*in*dependent function), 

whose code flow diagram is shown in Figure 2-7. 

`do_fork` begins with an invocation of `copy_process`, which performs the actual work of **generating a new process** and **reusing the parent process data specified by the flags**. Once the child process has been generated, the kernel must carry out the following concluding operations: 

![](../img/linux-kernel-fork-impl.png)

❑  Since `fork` returns the PID of the new task, it must be obtained. This is complicated because the **fork operation could have opened a new PID namespace if the flag `CLONE_NEWPID` was set.** If this is the case, then `task_pid_nr_ns` is required to obtain the PID that was selected for the new process in the *parent* namespace, that is, the namespace of the process that issued fork. 

**If the PID namespace remains unchanged**, calling `task_pid_vnr` is enough to obtain the local PID because old and new processes will live in the same namespace. 

```c
kernel/fork.c 

nr = (clone_flags & CLONE_NEWPID) ?
 task_pid_nr_ns(p, current->nsproxy->pid_ns) : task_pid_vnr(p); 
```

❑  **If the new process is to be monitored with Ptrace** (see Chapter 13), the `SIGSTOP` signal is sent to 

the process immediately after generation to allow an attached debugger to examine its data. 

❑  The child process is woken using `wake_up_new_task`; in other words, **the `task` structure is added to the scheduler queue.** The `scheduler` also gets a chance to specifically handle newly started tasks, which, for instance, allows for implementing a policy that gives new tasks a good chance to run soon, but also prevents processes that fork over and over again to consume all CPU time. 

**If a child process begins to run before the parent process, this can greatly reduce copying effort, especially if the child process issues an exec call after fork.** However, keep in mind that **enqueuing a process in the scheduler data structures does not mean that the child process begins to execute immediately but rather that it is available for selection by the scheduler.** 

❑  If the` vfork` mechanism was used (the kernel recognizes this by the fact that the `CLONE_VFORK` flag is set), the *completions* mechanism of the child process must be enabled. The `vfork_done` element of the child process task structure is used for this purpose. **With the help of the `wait_for_completion` function, the parent process goes to sleep on this variable until the child process exits.** When a process terminates (or a new application is started with execve), the kernel automatically invokes `complete(vfork_done)`. This wakes all processes sleeping on it. In Chapter 14, I discuss the implementation of completions in greater detail. 

By adopting this approach, the kernel ensures that the parent process of a child process generated using vfork remains inactive until either the child process exits or a new process is executed. The temporary inactivity of the parent process also ensures that both processes do not interfere with each other or manipulate each other’s address space. 

#### Copying Processes 

In `do_fork` the bulk of the work is done by the `copy_process` function, whose code flow diagram is shown in Figure 2-8. Notice that **the function has to handle the main work for the three system calls fork, vfork, and clone**. 

![](../img/linux-kernel-copy-process.png)

let’s restrict our description to **a slightly simplified version of the function** so as **not to lose sight of the most important aspects** in a myriad 无数的 of details.

Quite a number of flags control the behavior of process duplication. **They are all well documented in**
**the clone(2) man page, and instead of repeating them here, I advise you to just take a look into it** — or, for that matter, any good text on Linux systems programming. More interesting is that there are some
flag combinations that do not make sense, and the kernel has to catch these. 



```c
kernel/fork.c 

static struct task_struct *copy_process(unsigned long clone_flags, 
                                        unsigned long stack_start, 
                                        struct pt_regs *regs, 
                                        unsigned long stack_size, 
                                        int __user *child_tidptr, struct pid *pid) 
{ 
		int retval;
   	struct task_struct *p;
   	int cgroup_callbacks_done = 0; 
    if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS)) 
        return ERR_PTR(-EINVAL); 
...

```

This is also a good place to recall from the introduction that Linux sometimes has to return a pointer if
an operation succeeds, and an error code if something fails. 

Unfortunately, the C language only allows a single direct return value per function, so any information about possible errors has to be encoded into the pointer. 

While pointers can in general point to arbitrary locations in memory, **each architecture supported by Linux has a region in virtual address space that starts from virtual address 0 and goes at least 4 KiB far where no senseful information can live.** The kernel can thus reuse this pointer range to encode error codes: If the return value of fork points to an address within the aforementioned range, then the call has failed, and the reason can be determined by the numerical value of the `pointer. ERR_PTR` is a helper macro to perform the encoding of the numerical constant `EINVAL` (invalid operation) into a pointer.

Some further flag checks are required: 

❑  **When a thread is created with `CLONE_THREAD`,** signal sharing must be activated with 

`CLONE_SIGHAND`. Individual threads in a thread group cannot be addressed by a signal. 

❑  **Shared signal handlers** can **only be provided if the virtual address space is shared between parent and child (`CLONE_VM`).** Transitive thinking reveals that **threads  also have to share the address space with the parent**. 



The **task structures** for parent and child differ only in one element: **A new kernel mode stack is allocated for the new process**. A pointer to it is stored in `task_struct->stack`. Usually the `stack` is stored in a union with `thread_info`, which holds all required processor-specific low-level information about the thread. 

```c
<sched.h> 
union thread_union {
  struct thread_info thread_info; 
	unsigned long stack[THREAD_SIZE/sizeof(long)]; 
};
```

In principle, individual architectures are, however, free to store whatever they like in the stack pointer if they signal this to the kernel by setting the pre-processor constant` __HAVE_THREAD_FUNCTIONS`. In this case, they must provide their own implementations of` task_thread_info` and `task_stack_page`, which allows for obtaining the thread information and the kernel mode stack for a given `task_struct` instance. Additionally, they must implement the function setup_thread_stack that is called in `dup_task_struct` to create a destination for stack. Currently, only IA-64 and m68k do not rely on the default methods of the kernel. 

On most architectures, one or two memory pages are used to hold an instance of thread_union. On IA-32, two pages are the default setting, and thus the available kernel stack size is slightly less than
 8 KiB because part is occupied by the thread_info instance. Note, though, that the configuration option `4KSTACKS` decreases the stack size to `4 KiB` and thus to one page. This is advantageous if a large number of processes is running on the system because one page per process is saved. On the other hand, it can lead to problems with external drivers that often tend to be **‘‘stack hogs,’**’ for example, use too much stack space. All central parts of the kernel that are part of the standard distribution have been designed to operate smoothly also with a stack size of 4 KiB, but problems can arise (and unfortunately have in the past) if binary-only drivers are required, which often have a tendency to clutter up the available stack space. 

`thread_info` holds process data that needs to be accessed by the architecture-specific assembly language code. Although the structure is defined differently from processor to processor, its contents are similar to the following on most systems. 

```c
<asm-arch/thread_info.h> 

struct thread_info { 
    struct task_struct *task; /* main task structure */
    struct exec_domain *exec_domain;  /* execution domain */
    unsigned long flags; /* low level flags */
    unsigned long status;/* thread-synchronous flags */
    __u32 cpu; /* current CPU */ 
    int preempt_count; /* 0 => preemptable, <0 => BUG */
    mm_segment_t addr_limit;  /* thread address space */ 
    struct restart_block  restart_block; 
}
```



❑  **task** is a pointer to the `task_struct` instance of the process. 

❑  **exec_domain** is used to implement *execution domains* with which different ABIs (*Application Binary Interface*s) can be implemented on a machine type (e.g., to run 32-bit applications on an AMD64 system in 64-bit mode). 

❑  **flags** can hold various process-specific flags, two of which are of particular interest to us: 

- ❑  **TIF_SIGPENDING** is set if the process has pending signals. 

- ❑  **TIF_NEED_RESCHED** indicates that the process should be or would like to be replaced with another process by the scheduler. 

  Other possible constants — some hardware-specific — which are, however, hardly ever used, are available in <asm-arch/thread_info.h>. 

❑  **cpu** specifies the number of the CPU on which a process is just executing (important on multi- processor systems — very easy to determine on single-processor systems). 

❑  **preempt_count** is a counter needed to implement kernel preemption, discussed in Section 2.8.3. 

❑  **addr_limit** specifies up to which address in virtual address space a process may use. As already noted, there is a limit for normal processes, but kernel threads may access the entire virtual address space, including the kernel-only portions. (This does *not* represent any kind of restric- tion on how much RAM a process may allocate.) Recall that I have touched on the separation between user and kernel address space in the Introduction, and will come back to the details in Section 4. 

❑  **restart_block** is needed to **implement the signal mechanism** (see Chapter 5). 

Figure 2-9 shows the relationship between task_struct, thread_info and the kernel stack. **When a particular component of the kernel uses too much stack space, the kernel stack will crash into the thread information**, and this will most likely lead to severe failures. Besides, this can also lead to wrong information when an emergency stack trace is printed, **so the kernel provides the function `kstack_end` to decide if a given address is within the valid portion of the stack or not.** 

![](../img/linux-kernel-relation-task-thread.png)

`dup_task_struct` copies the contents of `task_struct` and `thread_info` instances of the parent process,
but **the stack pointer is set to the new `thread_info` instance.** This means that the `task` structures of
parent and child processes are absolutely identical at this point except for the stack pointer. The child will, however, be modified in the course of `copy_process`.

There are also two symbols named `current` and `current_thread_info` that are defined as macros or functions by all architectures. Their meanings are as follows: 

❑  `current_thread_info` delivers a pointer to the `thread_info` instance of the process currently executing. The address can be determined from the kernel stack pointer because the instance is always located at the top of the stack.Because a separate kernel stack is used for each process, the process to stack assignment is unique. 

❑  `current specifies` the address of the `task_struct` instance of the current process. This function appears very frequently in the sources. The address can be determined using `get_thread_info`:` current = current_thread_info()->task`. 

Let us return to `copy_process`. After `dup_task_struct` has succeeded, **the kernel checks if the maximam number of processes allowed for a particular user are exceeded with the creation of the new task**.

```c
kernel/fork.c
if (atomic_read(&p->user->processes) >= p->signal->rlim[RLIMIT_NPROC].rlim_cur) {
      if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) && 
      p->user != current->nsproxy->user_ns->root_user)
      	goto bad_fork_free;
} 
...
```

The per-user resource counters for the user owning the current process are kept in an instance of
`user_struct` that is accessible via `task_struct->user`, and the number of processes currently held
by a particular user is stored in `user_struct->processes`. If this value exceeds the limit set by `rlimit`,
task creation is aborted — unless the current user is assigned special capabilities (`CAP_SYS_ADMIN` or
`CAP_SYS_RESOURCE`) or is the root user. **Checking for the root user is interesting: Recall from above that**
**each PID namespace has its own root user**. This must now be taken into account in the above check.

If resource limits do not prevent process creation, the interface function sched_fork is called to give
the scheduler a chance to set up things for the new task. 

Essentially, the routines initialize statistical fields and on multi-processor systems probably re-balance the available processes between the CPUs if this is necessary. Besides, the task state is set to **TASK_RUNNING** — which is not really true since the new process is, in fact, not yet running. However, this prevents any other part of the kernel from trying to change the process state from non-running to running and scheduling the new process before its setup has been completely finished.

A large number of **copy_xyz** routines are then invoked to **copy or share the resources of specific kernel subsystems**. The `task structure` contains pointers to instances of `data structures` that describe a sharable or cloneable resource. Because the `task` structure of the child starts out as an exact copy of the parent’s task structure, **both point to the same resource-specific instances initially**. This is illustrated in Figure 2-10. 

Suppose we have two resources: **res_abc** and **res_def**. Initially the corresponding pointers in the task structure of the parent and child process point to the same instance of the resource-specific data structure in memory. 

If **CLONE_ABC** is set, then both processes will share **res_abc**. This is already the case, but **it is additionally necessary to increment the reference counter of the instance to prevent the associated memory space from being freed too soon — memory may be relinquished to memory management only when it is no longer being used by a process**. **If either parent or child modifies the shared resource, the change will be visible in both processes.** 

If **CLONE_ABC** is not set, then a copy of **res_abc** is **created for the child process**, and **the resource counter of the new copy is initialized to 1**. Consequently, if parent or child modifies the resource, then changes will *not* propagate to the other process in this case. 

**As a general rule, the fewer the number of `CLONE flags` set, the less work there is to do**. However, this gives parent and child processes more opportunities to mutually manipulate(互相影响) their data structures — and this must be taken into consideration when programming applications.

![](../img/linux-kernel-copy-clone.png)

❑  `copy_semundo` uses the System V **semaphores**(信号量) of the parent process if COPY_SYSVSEM is set (see Chapter 5). 

❑  `copy_files` uses the file descriptors of the parent process if CLONE_FILES is set. Otherwise, a new files structure is generated (see Chapter 8) that contains the same information as the parent process. This information can be modified independently of the original structure. 

❑  `copy_fs` uses the filesystem context (`task_struct->fs`) of the parent process if CLONE_FS is set. This is an `fs_struct` type structure that holds, for example, the root directory and the current working directory of the process (see Chapter 8 for detailed information). 

❑  `copy_sighand` uses the signal handlers of the parent process (`task_struct->sighand`) if CLONE_SIGHAND or CLONE_THREAD is set. Chapter 5 discusses the struct sighand_struct structure used in more detail. 

❑  `copy_signal` uses the non-handler-specific part of signal handling (`task_struct->signal`, see Chapter 5) together with the parent process if CLONE_THREAD is set. 

❑  `copy_mm` causes the parent process and child process to **share the same address space** if COPY_MM is set. In this case, both processes use the same instance of `mm_struct` (see Chapter 4) to which `task_struct->mm points`. 

> If copy_mm is *not* set, it does not mean that the entire address space of the parent process is copied. The kernel does, in fact, create a copy of the page tables but does not copy the actual contents of the pages. This is done using the COW mechanism only if one of the two processes writes to one of the pages. 

❑  `copy_namespaces` has special call semantics. It is used to set up namespaces for the child process. Recall that several CLONE_NEWxyz flags control which namespaces are *shared* with the parent. However, the semantics are opposite to all other flags: If CLONE_NEWxyz is *not* specified, then the specific namespace is shared with the parent. Otherwise, a new namespace is generated. copy_namespace is a dispatcher that executes a copy routine for each possible namespace. The individual copy routines, however, are not too interesting because they essentially copy data or make already existing instances shared by means of reference counter management, so I will not discuss their implementation in detail. 

❑ `copy_thread` is — in contrast to all other copy operations discussed here — an architecture- specific function that copies the thread-specific data of a process. 

> ***Thread-specific*** **in this context does not refer to any of the** **CLONE** **flags or to the fact
> that the operation is performed for threads only and not for full processes. It simply
> means that all data that contribute to the architecture-specific execution context are
> copied (the term** ***thread*** **is used with more than one meaning in the kernel).**

What is important is to fill the elements of `task_struct->thread`. This is a structure of the `thread_struct` type whose definition is architecture-dependent. It holds all registers (plus other information) needed by the kernel to save and restore process contents during low-level switch- ing between tasks. 

Intimate knowledge of the various CPUs is needed to understand the layout of the individual thread_struct structures. A full discussion of these structures is beyond the scope of this book. However, Appendix A includes some information relating to the contents of the structures on several systems. 

Back in `copy_process`, the kernel must fill in various elements of the task structure that differ between parent and child. These include the following: 

❑  The various list elements contained in `task_struct`, for instance, sibling and children. 

❑  The interval timer elements `cpu_timers` (see Chapter 15). 

❑  The list of pending signals (pending) discussed in Chapter 5. 

After allocating a new pid instance for the task with the mechanisms described before, they are stored in the task structure. **For threads, the thread group ID is the same as that of the forking process**: 

```c
kernel/fork.c 

 p->pid = pid_nr(pid);
 p->tgid = p->pid;
 if (clone_flags & CLONE_THREAD) 
		p->tgid = current->tgid;
...
```


Recall that `pid_nr` computes the **global numerical PID for a given pid instance**. 

For regular processes, the parent process is the forking process. **This is different for threads**: Since they are seen as the second (or third, or fourth,...) line of execution *within* the generating process, **their parent is the parent’s parent**. This is easier to express in code than in words: 

```c
kernel/fork.c 

if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) 
		p->real_parent = current->real_parent; 
else 
  	p->real_parent = current; 
p->parent = p->real_parent;
```

Regular processes that are not threads can trigger the same behavior by setting CLONE_PARENT. 

Another correction is required for threads: The thread group leader of a regular process is the process itself. For a thread, the group leader is the group leader of the current process: 

```c
kernel/fork.c 

p->group_leader = p; 

if (clone_flags & CLONE_THREAD) {
  p->group_leader = current->group_leader;
  list_add_tail_rcu(&p->thread_group, &p->group_leader->thread_group); 
	... 
} 
```

**The new process must then be linked with its parent process by means of the children list**. This is handled by the auxiliary macro `add_parent`. Besides, the new process must be included in the ID data structure network as described in Section 2.3.3. 

```c
kernel/fork.c
 ...
add_parent(p);
if (thread_group_leader(p)) {
    if (clone_flags & CLONE_NEWPID)
    		p->nsproxy->pid_ns->child_reaper = p;
    set_task_pgrp(p, task_pgrp_nr(current)); 
    set_task_session(p, task_session_nr(current)); 
    attach_pid(p, PIDTYPE_PGID, task_pgrp(current)); 
    attach_pid(p, PIDTYPE_SID, task_session(current));
}
		attach_pid(p, PIDTYPE_PID, pid); 
...
		return p;
}
```

`thread_group_leader` checks only **whether pid and tgid of the new process are identical**. If so, the 

process is the leader of a thread group. In this case, some more work is necessary: 

- ❑  Recall that processes in a process namespace that is not the global namespace have their own init task. If a new PID namespace was opened by setting CLONE_NEWPID, this role must be assumed by the task that called clone. 
- ❑  The new process must be added to the current task group and session. This allows for bringing some of the functions discussed above to good use. 

Finally, the PID itself is added to the ID network. This concludes the creation of a new process! 



#### Special Points When Generating Threads 

**Userspace thread libraries** use **the `clone` system call to generate new threads**. This call supports flags (other than those discussed above) that **produce certain special effects** in the `copy_process` (and in the associated invoked functions). 

For the sake of simplicity, I omitted these flags above. 

However, **it should be remembered that the differences between <u>a classical process</u> and <u>a thread</u> in the Linux kernel are relatively fluid and both terms are often used as synonyms** (*thread* is also frequently used to mean the architecture-dependent part of a process as mentioned above). 

In this section, **I concentrate on the flags used by <u>user thread libraries</u> (above all, NPTL) to implement multithreading capabilities**.

- ❑  **CLONE_PARENT_SETTID** copies the PID of the generated thread to a point in userspace specified in the clone call (parent_tidptr, the pointer is passed to clone): 

  ```c
  kernel/fork.c 
  
  if (clone_flags & CLONE_PARENT_SETTID) 
    put_user(nr, parent_tidptr); 
  ```

  The copy operation is performed in do_fork before the task structure of the new thread is initial- ized and before its data are created with the copy operations. 

- ❑  **CLONE_CHILD_SETTID** first causes a further userspace pointer (**child_tidptr**) passed to clone to be stored in the task structure of the new process. 

  ```c
  kernel/fork.c 
  p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
  ```

   The schedule_tail function invoked when the new process is executed for the first time copies 

  the current PID to this address. 

  ```c
  kernel/schedule.c 
  
  asmlinkage void schedule_tail(struct task_struct *prev) {
   ... 
  
  if (current->set_child_tid)
   	put_user(task_pid_vnr(current), current->set_child_tid); 
  ... 
  } 
  ```

  

- ❑  **CLONE_CHILD_CLEARTID** has the initial effect in `copy_process` that the userspace pointer `child_tidptr` is stored in the task structure — but this time in a different element. 

  ```c
  kernel/fork.c 
  
  p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr: NULL;
  ```

   When the process terminates,13 0 is written to the address defined in clear_child_tid.14 

  ```c
  kernel/fork.c 
  
  void mm_release(struct task_struct *tsk, struct mm_struct *mm) { 
  
  if (tsk->clear_child_tid&& atomic_read(&mm->mm_users) > 1) { 
      u32 __user * tidptr = tsk->clear_child_tid; 
      tsk->clear_child_tid = NULL; 
      put_user(0, tidptr); 
      sys_futex(tidptr, FUTEX_WAKE, 1, NULL, NULL, 0); 
  } 
  ... 
  }
  ```


   In addition,` sys_futex`, a fast userspace mutex, is used to wake processes waiting for this event, 

  namely, the end of the thread. 

  The above flags can be used from within userspace to check when threads are generated and destroyed in the kernel. CLONE_CHILD_SETTID and CLONE_PARENT_SETTID are used to check when a thread is gen- erated; CLONE_CHILD_CLEARTID is used to pass information on the death of a thread from the kernel to userspace. **These checks can genuinely be performed in parallel on multiprocessor systems.** 

  

## 内核线程（Kernel Threads）

*Kernel threads* 由内核自己直接启动。委托一个内核函数separate process and execute it there in ‘‘parallel‘‘ to the other processes in the system (and, in fact, in parallel to execution of the kernel itself).

 Kernel threads are often referred to as *(kernel) daemons*. They are used to perform, for example, the following tasks:

❑  To **periodically synchronize modified memory pages with the block device** from which the pages originate (e.g., files mapped using mmap). 

❑  To **write memory pages into the swap area** if they are seldom used. 

❑  To **manage deferred actions**. 

❑  To implement transaction journals for filesystems. 

Basically, there are two types of **kernel thread**: 

❑  **Type 1** — The thread is **started and waits until requested by the kernel** to **perform a specific action**. 

❑  **Type 2** — Once started, **the thread runs at periodic intervals**, checks the utilization of a specific resource, and takes action when utilization exceeds or falls below a set limit value. The kernel uses this type of thread for continuous monitoring tasks. 

**The `kernel_thread` function is invoked to start a kernel thread.** Its definition is architecture-specific, but it always uses the same prototype. 

```c
<asm-arch/processor.h>
 int kernel_thread(int (*fn)(void *), void * arg, unsigned long flags) 
```

The function passed with the `fn` pointer is *executed in the generated thread*, and *the argument specified in arg* is automatically passed to the function. `CLONE flags` can be specified in flags. 

**The first task of `kernel_thread` is to construct a `pt_regs` instance in which the registers are supplied with suitable values, as would be the case with a regular `fork` system call.** Then the familiar `do_fork` function is invoked. 

```c
p = do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL); 
```

Because kernel threads are generated by the kernel itself, two special points should be noted: 

1. **They execute in the supervisor mode of the CPU, not in the user mode** (see Chapter 1). 
2. **They may access only the kernel part of virtual address space** (all addresses above **TASK_SIZE**) but **not the virtual user area**. 

Recall from above that the two pointers to `mm_structs` are contained in the` task `structure: 

```c
<sched.h> 

struct task_struct { 
... 

struct mm_struct *mm, *active_mm; 

... 
} 
```

**The total virtual address space of a system is separated into two parts on most machines**:

-  The **lower portion** is accessible by **userland programs**
-  The **upper part** is **reserved for the kernel**. 

When the kernel is running on behalf of a userland program to serve a system call, for instance, the userspace portion of the virtual address space is described by the `mm_struct` instance pointed to by `mm` (the exact content of this structure is irrelevant for now, but is discussed in Chapter 4). 

**Every time the kernel performs a context switch, the userland portion of the virtual address space must be replaced to match the then-running process.** 

This provides some room for optimization, which goes by the name *lazy TLB handling*: 

Since kernel threads are not associated with any particular userland process, the kernel does not need to rearrange the userland portion of the virtual address space and can just leave the old setting in place. 

Since any userland process can have been running before a kernel thread, the contents of the userspace part are essentially random, and the kernel thread must not modify it. To signalize that the userspace portion must not be accessed, `mm` is set to a NULL pointer. 

**However, since the kernel must know what is currently contained in the userspace, a pointer to the `mm_struct` describing it is preserved in active_mm.** 

Why are processes without an mm pointer called *lazy TLB processes*? 

**Suppose that the process that runs after a kernel thread is the same process that has run before**. In this case, **the kernel does not need to modify the userspace address tables**, and **the information in the `translation lookaside buffers` is still valid.** A switch (and a corresponding clearance of TLB data) is only required when a different userland process from before executes after the kernel thread. 

Notice that when the kernel is operating in process context, `mm` and `active_mm` have identical values. 

A kernel thread can be implemented in one of two ways. The older variant 版本 — which is still in use in some places in the kernel — is to pass a function directly to `kernel_thread`. The function is then responsible to assist the kernel in the transformation into a daemon by invoking daemonize. This results in the following actions: 

1. The function frees all resources (e.g., memory context, file descriptors, etc.) of the user pro- cess as whose child the kernel thread was started because otherwise these would be pinned until the end of the thread — this is not desirable because daemons usually run until the sys- tem is shut down. As each daemon operates only in the address area of the kernel, it does not even need these resources. 
2. **daemonize** blocks the receipt of signals. 
3. **init** is used as the parent process of the daemon. 

The more modern possibility to create a kernel thread is the auxiliary function kthread_create. 

```c
kernel/kthread.c 

struct task_struct *kthread_create(int (*threadfn)(void *data),
                                   void *data, 
																	 const char namefmt[], ...) 
```

The function creates a new kernel thread with its name given by namefmt. Initially, the thread will be stopped. To start it,` wake_up_process` needs to be used. After this, the thread function given in` threadfn` will be called with data as argument. 

As an alternative, the macro `kthread_run` (which uses the same arguments as `kthread_create`) will call `kthread_create` to create the new thread, but **will wake it up immediately.** A **kernel thread can also be bound to a particular CPU** by using `kthread_create_cpu` instead of `kthread_create`. 

Kernel threads **appear in the system process list but are enclosed in square brackets** in the output of ps to **differentiate them from normal processes**. 

**If a kernel thread is bound to a particular CPU, the CPU’s number is noted after the slash**.

```shell
root     13423   832  0 08:26 ?        00:00:00 sshd: root@pts/0
root     13432 13423  0 08:26 pts/0    00:00:00 -bash
root     13492     2  0 Dec09 ?        00:00:08 [kworker/0:0]
root     13633     2  0 Dec06 ?        00:00:08 [kworker/4:1]
root     13637     2  0 01:27 ?        00:00:00 [kworker/1:2]
root     13679     2  0 Dec08 ?        00:00:00 [kworker/0:1]
```

### 启动新的程序

**New programs are started by replacing an existing program with new code**. Linux provides the `execve`system call for this purpose.



#### execve 实现

系统调用的入口 is the architecture-dependent `sys_execve` function. This function quickly delegates its work to the system-independent `do_execve` routine. 

```c
kernel/exec.c
int do_execve(char * filename,
							char __user *__user *argv,
							char __user *__user *envp, struct pt_regs * regs)
```

Not only **the register set with the arguments** and **the name of the executable file** (filename) but also
**pointers to the arguments** and **the environment of the program** are passed as in system programming.

![](../img/do_execve.png)

First, the file to be executed is opened; in other words — as described in Chapter 8 — **the kernel finds the**
**associated inode and generates a file descriptor that is used to address the file**.

`bprm_init` then handles **several administrative 管理 tasks**: 

- `mm_alloc` generates a new instance of `mm_struct` to **manage the process address space** (see Chapter  4).
-  `init_new_context` is an architecture-specific function that **initializes the instance**,
- `__bprm_mm_init` sets up an **initial stack**.

Various parameters of the new process (e.g., euid, egid, argument list, environment, filename, etc.) that
are subsequently passed to other functions are, **for the sake of simplicity**, combined into a structure of type `linux_binprm`. `prepare_binprm` is used to supply a number of parent process values (above all, the
effective UID and GID); the remaining data — the argument list — are then copied manually into the
structure. Note that `prepare_binprm` also **takes care of handling the SUID and SGID bits**:

```c
fs/exec.c
int prepare_binprm(struct linux_binprm *bprm) {
    ...
    bprm->e_uid = current->euid; 
    bprm->e_gid = current->egid;
    if(!(bprm->file->f_vfsmnt->mnt_flags & MNT_NOSUID)) { 
        /* Set-uid? */
        if (mode & S_ISUID) {
            bprm->e_uid = inode->i_uid;
        }
        /* Set-gid? */ /*
        * If setgid is set but no group execute bit then this
        * is a candidate for mandatory locking, not a setgid
        * executable. */
        if ((mode & (S_ISGID | S_IXGRP)) == (S_ISGID | S_IXGRP)) { 
        		bprm->e_gid = inode->i_gid;
        }
    }...
}
```

Linux supports various organization formats for executable files. The standard format is **ELF** (*Executable
and Linkable Format*), which I discuss at length in Appendix E. Other alternatives are the variants shown
in Table 2-2 (which lists the names of the corresponding linux_binfmt instances in the kernel).

Even though many binary formats can be used on different architectures (ELF was designed explicitly to
be as system-independent as possible), this does not mean that programs in a specific binary format are
able to run on multiple architectures. **The assembler statements used still differ greatly from processor to processor and the binary format only indicates how the different parts of a program — data, code, and so on — are organized in the executable file and in memory**.

search_binary_handler is used at the end of do_execve to find a suitable binary format for the particular
file. Searching is possible because each format can be recognized by reference to special characteristics
(usually a ‘‘magic number‘‘ at the beginning of the file). The binary format handler is responsible for
loading the data of the new program into the old address space. Appendix E describes the steps needed
to do this when the ELF format is used. 

**Generally, a binary format handler performs the following actions:**

❑  It **releases all resources used by the old process**. 

❑  It **maps the application into virtual address space**. The following **segments** must be taken into account (**the variables specified are elements of the `task structure` and are set to the correct values by the `binary format handler`**): 
​	❑  The *text segment* contains **the executable code of the program.** `start_code` and `end_code` specify the area in address space where the segment resides. 

​	❑  The **pre-initialized data** (**variables supplied with a specific value at compilation time**) are located between `start_data` and `end_data` and are mapped from the corresponding segment of the executable file. 

​	❑  The *heap* used for **dynamic memory allocation** is placed in **virtual address space**; `start_brk` and `brk` specify its boundaries. 

​	❑  **The position of the stack** is defined by `start_stack`; the `stack` grows downward automatically on nearly all machines. The only exception is currently PA-Risc. The inverse direction of stack growth must be noted by the architecture by setting the configuration symbol `STACK_GROWSUP`. 

​	❑  The **program arguments and the environment** are mapped into the virtual address space and are located between `arg_start` and `arg_end` and `env_start` and `env_end`, respectively . 

❑ **The instruction pointer of the process** and **some other architecture-specific registers** are set so that the main function of the program is executed when the scheduler selects the process. 

How the **ELF format** populates the virtual address space will be discussed in more detail in Section 4.2.1.

![](../img/linux-kernel-binary-format.png)

### 解释二进制格式

Each binary format is represented in the Linux kernel by an instance of the following (simplified) data structure: 

```c
<binfmts.h> 

struct linux_binfmt {
	struct linux_binfmt * next; 
	struct module *module;
  int (*load_binary)(struct linux_binprm *, struct pt_regs * regs);
  int (*load_shlib)(struct file *);
  int (*core_dump)(long signr, struct pt_regs * regs, struct file * file);
  unsigned long min_coredump; /* minimal dump size */ 

};
```

Each binary format must provide three functions: 

1. `load_binary` to load normal programs. 
2. `load_shlib` to load a *shared library*, that is, a dynamic library. 
3. `core_dump` to write a core dump if there is a program error. This `dump` can subsequently be analyzed using a debugger (e.g., gdb) for troubleshooting purposes. `min_coredump` is a lower bound on the core file size from which a coredump will be generated (usually, this is the size of a single memory page). 

Each binary format must first be registered in the kernel using `register_binfmt`. The purpose of this function is to add a new binary format to a linked list whose list head is represented by the formats global variable from `fs/exec.c`. The` linux_binfmt` instances are linked with each other by means of their `next `element. 



## 调度器实现

调度器工作：共享CPU时间。这个工作可以总结为两个部分：调度策略和上下文切换。

### 概述

**The kernel must provide a method of sharing CPU time as fairly as possible between the individual**
**processes** while **at the same time taking into account differing task priorities**.

The `schedule` function is the starting point to an understanding of scheduling operations. It is defined in `kernel/sched.c` and is **one of the most frequently invoked functions in the kernel code**. The implementation of the scheduler is obscured a little by several factors: 

❑  On multiprocessor systems, several details (some very subtle) must be noted so that the scheduler doesn’t get under its own feet. 

❑  Not only *priority scheduling* but also two other soft real-time policies required by the Posix standard are implemented. 

❑  `gotos` are used to generate optimal assembly language code. These jump backward and forward in the C code and run counter to all principles of structured programming. However, this feature *can* be beneficial if it is used with great care, and the scheduler is one example where gotos make sense. 



In the following overview, I consider the **completely fair scheduler** and **neglect real-time tasks** for now. I come back to them later. **An outstanding feature of the Linux scheduler is that it does not require the concept of time slices, at least not in the traditional way**. 

**Classical schedulers** compute time slices for each process in the system and allow them to run until their time slice is used up. When all time slices of all processes have been used up, they need to be recalculated again. 

**The current scheduler**, in contrast, **considers only the wait time of a process — that is, how long it has been sitting around in the run-queue and was ready to be executed**. The task with the gravest need for CPU time is scheduled. 

**The general principle of the scheduler is to provide maximum fairness to each task in the system in terms of the computational power it is given.** Or, put differently, it tries to ensure that no task is treated unfairly. Now this clearly sounds good, but what do *fair* and *unfair* with respect to CPU time mean? Consider an ideal computer that can run an arbitrary number of tasks in parallel: If *N* processes are present on the system, then each one gets 1 of the total computational power, and all tasks really execute physically parallel. Suppose that a task requires 10 minutes to complete its work. If 5 such tasks are simultaneously present on a perfect CPU, each will get 20 percent of the computational power, which means that it will be running for 50 instead of 10 minutes. However, all 5 tasks will finish their job after exactly this time span, and none of them will have ever been inactive! 

Every time the `scheduler` is called, it picks the task with the highest waiting time and gives the CPU to
it. If this happens often enough, no large `unfairness` will accumulate for tasks, and the `unfairness` will be
evenly distributed among all tasks in the system.

Figure 2-12 illustrates **how the scheduler keeps track of which process has been waiting for how long**.
Since runnable processes are queued, the structure is known as the *run queue*.

![](../img/linux-kernel-scheduler.png)

**All `runnable tasks` are `time-ordered` in a `red-black tree`, essentially with respect to their waiting time**. The task that has been waiting for the CPU for the largest amount of time is the leftmost entry and will be considered next by the scheduler. **Tasks that have been waiting less long are sorted on the tree from left to right**.

Besides the red-black tree, a **run queue** is also **equipped** with **a *virtual* clock**.Time passes slower on
this clock than in real time, and the exact speed depends on the number of processes that are currently
waiting to be picked by the scheduler. Suppose that four processes are on the queue: Then the virtual
clock will run at one-quarter of the speed of a real clock. This is the basis to determine how much CPU
time a waiting process would have gotten if computational power could be shared in a completely fair
manner. Sitting on the run queue for 20 seconds in real time amounts to 5 seconds in virtual time. Four
tasks executing for 5 seconds each would keep the CPU occupied for 20 seconds in real time.

Suppose that the virtual time of the run queue is given by `fair_clock`, while the waiting time of a process is stored in `wait_runtime`. To sort tasks on the `red-black tree`, the kernel uses the difference `fair_clock - wait_runtime`. While `fair_clock` is a measure for the CPU time a task would have gotten if scheduling were completely fair, `wait_runtime` is a direct measure for the unfairness caused by the imperfection of real systems. 

When a task is allowed to run, the interval during which it has been running is subtracted from `wait_runtime`. This way, it will move rightward in the `time-ordered` tree at some point, and another process will be the leftmost one — and is consequently selected to run. Notice, however, that the virtual clock in `fair_clock` will increase when the task is running. This effectively means that the share of CPU time that the `task` would have received in a perfectly fair system is deducted from the time spent executing on the real CPU. This slows degradation of unfairness: Decrementing `wait_runtime` is equivalent to lowering the amount of unfairness received by the task, but the kernel must not forget that some portion of the time used to lower the unfairness would have belonged to the process in a completely fair world anyway. Suppose again that four processes sit on the run queue, and that a process has been waiting for 20 real seconds. Now it is allowed to run for 10 seconds: wait_runtime is afterward 10, but since the process would have gotten 10/4 = 2 seconds of this time span anyway, effectively only 8 time units account for the potentially new position on the run queue. 

Unfortunately, this strategy is complicated by a number of real-world issues: 

❑  **Different priority levels for tasks (i.e., `nice` values) must be taken into account**, and **more important processes must get a higher share of CPU time** than less important ones. 

❑  **Tasks must not be switched too often because a context switch, that is, changing from one task to another, has a certain overhead.** When switching happens too often, too much time is spent with exchanging tasks that is not available for effective work anymore. 

​	 On the other hand, the time that goes by between `task switches` must not be too long because large unfairness values could accumulate in this case. Letting tasks run for too long can also lead to larger latencies than desired for multimedia systems. 

We will see how the scheduler tackles these problems in the following discussion. 

A good way to understand scheduling decisions is to activate scheduler statistics at compile time. This will generate the file `/proc/sched_debug`, **which contains information on all aspects of the current state of the scheduler.** 

Finally, note that the Documentation/ directory contains some files that relate to various aspects of the scheduler. Keep in mind, however, that some of them still relate to the old O(1) scheduler and are there- fore outdated! 

### Data Structures

调度器使用了很多数据结构来排序、管理系统中的进程。调度器的工作和这些数据结构的设计有着紧密的关联。组件之间的交互关系如下图2-13.

![](../img/linux-kernel-scheduler-system.png)

Scheduling can be activated in two ways: 

- **Directly if a task goes to sleep or wants to yield the CPU** for other reasons;
- By **a periodic mechanism** that is run with constant frequency and that checks from time to time if switching tasks is necessary.

I denote these two components *generic scheduler* or *core scheduler* in the following. Essentially, the generic scheduler is a dispatcher that interacts with two other components: 

1. *Scheduling classes* 调度类型 are used to **decide which task runs next**. The kernel supports different scheduling policies (**completely fair scheduling**, **real-time scheduling**, and **scheduling of the idle task** when there is nothing to do), and scheduling classes allow for implementing these policies in a modular way: Code from one class does not need to interact with code from other classes. 

   When the scheduler is invoked, it queries the scheduler classes which task is supposed to run next. 

2. After a task has been selected to run, a low-level *task switch* must be performed. This requires close interaction with the underlying CPU. 

**Every task belongs to exactly one of the scheduling classes**, and **each scheduling class is responsible to manage their tasks**. The generic scheduler itself is not involved in managing tasks at all; this is completely delegated to the scheduler classes.

### Elements in the Task Structure

There are several scheduling-relevant elements in the task structure of each process.

```c
<sched.h>
struct task_struct { 
...
      int prio, static_prio, normal_prio;
      unsigned int rt_priority;
      struct list_head run_list;
      const struct sched_class *sched_class;
      struct sched_entity se;
      unsigned int policy;
      cpumask_t cpus_allowed;
      unsigned int time_slice;
...
}
```

❑  **Not all processes on a system are equally important**: Less urgent tasks should receive less attention, while important work should be done as quickly as possible. **To determine the importance of a particular task, it is equipped with a relative priority**. 

However, the `task` structure employs three elements to denote the priority of a process: `prio` and `normal_prio` indicate the **dynamic priorities**, `static_prio` the **static priority** of a process. **The *static priority* is the priority assigned to the process when it was started. It can be modified with the `nice` and `sched_setscheduler` system calls, but remains otherwise constant during the process’ run time**. 

**`normal_priority` denotes a priority that is computed based on the static priority and the scheduling policy of the process.** Identical static priorities will therefore result in different normal priorities depending on whether a process is a regular or a real-time process. **When a process forks, the child process will inherit the normal priority.** 

However, the priority considered by the scheduler is kept in `prio`. A third element is required because situations can arise in which the kernel needs to temporarily boost the priority of a process. Since these changes are not permanent, the static and normal priorities are unaffected by this. How the three priorities depend on each other is slightly subtle, and I discuss this in detail below. 

❑  **`rt_priority` denotes the priority of a real-time process.** Note that this does not replace the previously discussed values! **The lowest real-time priority has value 0, whereas the highest priority is 99.** Higher values correspond to higher priorities. The convention used here is different from the convention used for`nice` values. 

❑  `sched_class` denotes the scheduler class the process is in. 

❑  **The scheduler is not limited to schedule processes, but can also work with larger entities**. This allows for implementing *group scheduling*: This way, **the available CPU time can first be distributed between general process groups** (e.g., all processes can be grouped according to their owner), and **the assigned time is then again distributed within the group**. 

**This generality requires that the scheduler does not directly operate on processes but works with *schedulable entities*.** An `entity` is represented by an instance of `sched_entity`. 

In the simplest case, scheduling is performed on a per-process level, and this is the case we concentrate on initially. **Since the scheduler is designed to work on schedulable entities, each process must look to it like such an entity.** `se` therefore embeds an instance of `sched_entity` on which the scheduler operates in each `task` struct (notice that `se` is *not* a pointer because the entity is embedded in the task!). 

❑  `policy` holds the scheduling policy applied to the process. Linux supports five possible values: 

​		**SCHED_NORMAL** is used for normal processes on which our description focuses. They are handled by the **completely fair scheduler**. **SCHED_BATCH** and **SCHED_IDLE** are also handled by **the completely fair scheduler** but **can be used for less important tasks**. 

​		**SCHED_BATCH** is for CPU-intensive batch processes that are not interactive. **Tasks of this type are disfavored in scheduling decisions**: They will never preempt another process handled by the CF scheduler and will therefore not disturb interactive tasks. The class is well suited for situations in which the static priority of a task is not desired to be decreased with nice, but when the task should nevertheless not influence the interactivity of a system. 

​		**SCHED_IDLE** tasks will also be of low importance in the scheduling decisions, but this time because their relative weight is always minimal (this will become clear when I discuss how the kernel computes task weights that reflect their priority). Note that **SCHED_IDLE** is, despite its name, *not* responsible to schedule the idle task. The kernel provides a separate mechanism for this purpose. 

​		**SCHED_RR** and **SCHED_FIFO** are used to implement soft real-time processes. **SCHED_RR implements a round robin method, while SCHED_FIFO uses a first in, first out mechanism.** These are not handled by the **completely fair scheduler class**, but by the **real-time scheduler class**, which is discussed in Section 2.7 in greater length. 

The auxiliary function `rt_policy` is used to decide if a given scheduling policy belongs to the real-time class (**SCHED_RR** and **SCHED_FIFO**) or not. `task_has_rt_policy` determines this property for a given task. 

```c
kernel/sched.c 

static inline int rt_policy(int policy)
static inline int task_has_rt_policy(struct task_struct *p) 
```

❑  `cpus_allowed` is a bit field used on multiprocessor systems to **restrict the CPUs on which a process may run.** 

❑  `run_list` and `time_slice` are **required** for the `round-robin` real-time scheduler, but not for the completely fair scheduler. `run_list` is a list head used to hold the process on a run list, while `time_slice` **specifies the remaining time quantum during which the process may use the CPU**. 

###  Scheduler Classes

**Scheduler classes provide the connection between the `generic scheduler` and `individual scheduling methods`.** They are represented by several function pointers collected in a special data structure. Each operation that can be requested by the global scheduler is represented by one pointer. This allows for **creation of the generic scheduler without any knowledge about the internal working of different scheduler classes.** 

```c
<sched.h>
struct sched_class {
const struct sched_class *next;
 void (*enqueue_task) (struct rq, struct task_struct *p, int wakeup); 
 void (*dequeue_task) (struct rq, struct task_struct *p, int sleep); 
 void (*yield_task) (struct rq *rq);
 void (*check_preempt_curr) (struct rq *rq, struct task_struct *p);
 struct task_struct * (*pick_next_task) (struct rq *rq);
 void (*put_prev_task) (struct rq *rq, struct task_struct *p);
 void (*set_curr_task) (struct rq *rq);
 void (*task_tick) (struct rq *rq, struct task_struct *p);
 void (*task_new) (struct rq *rq, struct task_struct *p);
};
```

**An instance of struct `sched_class` must be provided for each scheduling class**. Scheduling classes are related in a flat hierarchy: **Real-time processes are most important, so they are handled before completely fair processes, which are, in turn, given preference to the idle tasks that are active on a CPU when there is nothing better to do**.

The next element connects the `sched_class` instances of the different scheduling classes in the described order. Note that **this hierarchy is already set up at compile time: There is no mechanism to add new scheduler classes dynamically at run time.** 

The operations that can be provided by each scheduling class are as follows: 

The operations that can be provided by each scheduling class are as follows: 

❑  enqueue_task adds a new process to the run queue. This happens when a process changes from a sleeping into a runnable state. 

❑  dequeue_task provides the inverse operation: It takes a process off a run queue. Naturally, this happens when a process switches from a runnable into an un-runnable state, or when the kernel decides to take it off the run queue for other reasons — for instance, because its priority needs to be changed. 

Although the term *run queue* is used, the individual scheduling classes need not represent their processes on a simple queue. In fact, recall from above that the completely fair scheduler uses a red-black tree for this purpose. 

❑  When a process wants to relinquish control of the processor voluntarily, it can use the sched_yield system call. This triggers yield_task to be called in the kernel. 

❑  check_preempt_curr is used to preempt the current task with a newly woken task if this is necessary. The function is called, for instance, when a new task is woken up with wake_up_new_task. 

❑  pick_next_task selects the next task that is supposed to run, while put_prev_task is called before the currently executing task is replaced with another one. Note that these operations are *not* equivalent to putting tasks on and off the run queue like enqueue_task and dequeue_task. Instead, they are responsible to give the CPU to a task, respectively, take it away. Switching between different tasks, however, still requires performing a low-level context switch. 

❑  set_curr_task is called when the scheduling policy of a task is changed. There are also some other places that call the function, but they are not relevant for our purposes. 

❑  task_tick is called by the periodic scheduler each time it is activated. 

❑  new_task allows for setting up a connection between the fork system call and the scheduler. 

Each time a new task is created, the scheduler is notified about this with new_task.


The standard functions activate_task and deactivate_task are provided to enqueue and dequeue a 

task by calling the aforementioned functions. Additionally, they keep the kernel statistics up to date. 

```c
kernel/sched.c 

static void enqueue_task(struct rq *rq, struct task_struct *p, int wakeup) 
static void dequeue_task(struct rq *rq, struct task_struct *p, int sleep) 
```

When a process is registered on a **run queue**, the `on_rq` element of the embedded `sched_entity` instance is set to 1, otherwise to 0. 

Userland applications do not directly interact with scheduling classes. They only know of the constants
**SCHED_xyz** as defined above. It is the kernel’s job to provide an appropriate mapping between these constants and the available scheduling classes. **SCHED_NORMAL**, **SCHED_BATCH**, and **SCHED_IDLE** are mapped to `fair_sched_class`, while **SCHED_RR** and **SCHED_FIFO** are associated with `rt_sched_class`. Both`fair_sched_class` and `rt_sched_class` are instances of struct `sched_class` that represent, respectively, the completely fair and the realtime scheduler. The contents of these instances will be shown
when I discuss the respective scheduler classes in detail.

### Run Queues

The central data structure of the core scheduler that is used to manage active processes is known as the
*run queue*. **Each CPU has its own run queue, and each active process appears on just one run queue.** It is not possible to run a process on several CPUs at the same time.

**The run queue is the starting point for many actions of the global scheduler**. Note, however, that **processes are not directly managed by the general elements of the run queue!** **This is the responsibility of the individual scheduler classes**, and **a class-specific sub-run queue is therefore embedded in each run queue**.

```c
kernel/sched.c
struct rq {
    unsigned long nr_running;
    #define CPU_LOAD_IDX_MAX 5
    unsigned long cpu_load[CPU_LOAD_IDX_MAX];
    ...
    struct load_weight load;
    
    struct cfs_rq cfs;
    struct rt_rq rt;
    
    struct task_struct *curr, *idle;
    u64 clock; 
		... 
};

```

❑ `nr_running` specifies the number of runnable processes on the queue — regardless of their pri-
ority or scheduling class.

❑  `load` provides **a measure for the current load on the run queue**. The queue load is essentially proportional to the number of currently active processes on the queue, where each process is additionally weighted by its priority. The speed of the virtual per-run queue clock is based on this information. Since computing the load and other related quantities is an important component of the scheduling algorithm, I devote Section 2.5.3 below to a detailed discussion of the mechanisms involved. 

❑  `cpu_load` allows for tracking the load behavior back into the past. 

❑  `cfs` and `rt` are the embedded sub-run queues for the completely fair and real-time scheduler, 

respectively. 

❑  `curr` points to the `task` structure of the process currently running. 

❑  `idle` points to the `task` structure of the idle process called when no other runnable process is available — the idle thread. 

❑  `clock` and `prev_raw_clock` are used to implement the per-run queue clock. The value of clock is updated each time the periodic scheduler is called. Additionally, the kernel provides the standard function update_rq_clock that is called from many places in the scheduler that manipulate the run queue, for instance, when a new task is woken up in wakeup_new_task. 

All run queues of the system are held in the runqueues array, which contains an element for each CPU in the system. On single-processor systems, there is, of course, just one element because only one run queue is required. 

```c
kernel/sched.c 

static DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues); 
```

### Scheduling Entities

Since the scheduler can operate with more general entities than tasks, an appropriate data structure is
required to describe such an entity. It is defined as follows:

```c
<sched.h>
struct sched_entity {
    struct load_weight load; /* for load-balancing */
    struct rb_node run_node;
    unsigned int on_rq;  ／* run queue*／

    u64 exec_start;
    u64 sum_exec_runtime;
    u64 vruntime;
    u64 prev_sum_exec_runtime;
... 
}
```

The structure can contain many more statistical elements if support for scheduler statistics has been compiled into the kernel, and also has some more elements if group scheduling is enabled. The part that is interesting for us right now, however, boils down to what you see above. The meaning of the individual elements is as follows: 

❑  `load` specifies a weight for each entity that contributes to the total load of the queue. Computing the load weight is an important task of the scheduler because the speed of the virtual clock required for CFS will ultimately depend on it, so I discuss the method in detail in Section 2.5.3. 

❑  `run_node` is **a standard tree element** that **allows the entity to be sorted** on a **red-black tree**. 

❑  `on_rq` denotes **whether the entity is currently scheduled on a run queue or no**t. 

❑  When a process is running, **the consumed CPU time needs to be recorded for the completely fair scheduler**. `sum_exec_runtime` is used for this purpose. Tracking the run time is done cumulatively, in update_curr. The function is called from numerous places in the scheduler, for instance, when a new task is enqueued, or from the periodic tick. At each invocation, the difference between the current time and exec_start is computed, and exec_start is updated to the current time. The difference interval is added to `sum_exec_runtime`. 

The amount of time that has elapsed on the virtual clock during process execution is accounted in vruntime. 

❑  When a process is taken off the CPU, its current `sum_exec_runtime` value is preserved in `prev_exec_runtime`. The data will later be required in the context of process preemption. Notice, however, that preserving the value of `sum_exec_runtime` in `prev_exec_runtime` does *not* mean that `sum_exec_runtime` is reset! The old value is kept, and `sum_exec_runtime` continues to grow monotonically. 

Since each `task_struct` has an instance of `sched_entity` embedded, a task is a schedulable entity. Notice, however, that the inverse statement is not true in general: A schedulable entity need not necessarily be a task. However in the following we are concerned only with task scheduling, so for now we can equate scheduling entities and tasks. Keep in mind that this is not true in general, though! 

## 优先级处理

**Priorities are deceptively simple from the userspace point of view:** After all, they seem to be **just a range of numbers**. The **in-kernel reality is unfortunately somewhat different**, and comparatively much effort is required to work with priorities.

### Kernel Representation of Priorities 优先级的内核表述(数字)

The static priority of a process can be set in userspace by means of the `nice` command, which internally
invokes the `nice` system call.

The nice value of a process is between −20 and +19 (inclusive). Lower values mean higher priorities. Why this strange range was chosen is shrouded in history.

The kernel uses a simpler scale ranging from 0 to 139 inclusive to represent priorities internally. Again,
lower values mean higher priorities. **The range from 0 to 99 is reserved for real-time processes**. **The nice values [−20, +19] are mapped to the range from 100 to 139, as shown in Figure 2-14**. **Real-time processes thus always have a higher priority than normal processes can ever have.**

![](../img/linux-kernel-priorities.png)

The following macros are used to convert between the different forms of representation:

```c
<sched.h>
#define MAX_USER_RT_PRIO 100
#define MAX_RT_PRIO      MAX_USER_RT_PRIO
#define MAX_PRIO         (MAX_RT_PRIO + 40)
#define DEFAULT_PRIO     (MAX_RT_PRIO + 20)

kernel/sched.c
#define NICE_TO_PRIO(nice) (MAX_RT_PRIO + (nice) + 20)
#define PRIO_TO_NICE(prio) ((prio) - MAX_RT_PRIO - 20)
#define TASK_NICE(p)       PRIO_TO_NICE((p)->static_prio)
```

### 优先级计算 Computing Priorities

**it is not sufficient to consider *just the static priority* of a process**, but that three priorities
must be taken into account: **dynamic priority** (`task_struct->prio`), **normal priority**
(`task_struct->normal_prio`), and **static priority** (`task_struct->static_prio`). These priorities
are related to each other in interesting ways, and in the following I discuss how.

`static_prio` 是计算优先级的起点。假设内核已经知道这个值，想要计算其他的优先级属性，可以通过如下方式：

```c
p->prio = effective_prio(p);
```

The auxiliary function辅助函数 `effective_prio` performs the following operations:

```c
kernel/sched.c
static int effective_prio(struct task_struct *p) {
      p->normal_prio = normal_prio(p); 
      /*
      * If we are RT tasks or we were boosted to RT priority,
      * keep the priority unchanged. Otherwise, update priority
      * to the normal priority: */
      if (!rt_prio(p->prio))
          return p->normal_prio;
      return p->prio;
}
```

helper function, `rt_prio`,checks if the normal priority is in the real-time range, that is, smaller than **RT_RT_PRIO**. Notice that the check is not related to any scheduling class, but only to the numerical value of the priority.

Things are different for real-time tasks. Observe how the normal priority is computed:

```c
kernel/sched.c
static inline int normal_prio(struct task_struct *p) {
  int prio;
  if (task_has_rt_policy(p))
      prio = MAX_RT_PRIO-1 - p->rt_priority;
  else
      prio = __normal_prio(p);
  return prio;
}
```

The normal priority needs to be computed differently for **regular tasks** and **real-time tasks**. The compu-
tation performed in `__normal_prio` is **only valid for a regular task**. **Real-time tasks, instead, compute the normal priority based on their `rt_priority` setting**. 

Because higher values of `rt_priority` denote higher real-time priorities, this runs counter to the kernel-internal representation of priorities, where *lower* values mean *higher* priorities. 

**Notice that this time, the detection of a `real-time task` is, *in contrast to* `effective_prio`, *not* based on any priority, but <u>on the scheduling policy set in the `task_struct`</u>.**

What does `__normal_priority` do? The function is really simple; it just returns the static priority: 

```c
kernel/sched.c 

static inline int __normal_prio(struct task_struct *p) { 

return p->static_prio; 

} 
```

However, one question remains: Why does the kernel base the **real-time** check in `effective_prio` on the
numerical value of the priority instead of using `task_has_rt_policy`? This is required for **non-real-time**
tasks that have been temporarily boosted to a real-time priority, which can happen when **RT-Mutexes**
are in use. 

> Real-time mutexes allow for protection of dangerous parts of the kernel against concurrent access by multiple processors. However,a phenomenon called *priority inversion* 优先级反转, in which a process with lower priority executes even though a process with higher priority is waiting for the CPU, can occur. This can be solved by temporarily boosting the priority of processes. Refer to the discussion in Section 5.2.8 for more details about this problem.

| **Task type / priority**                          | **static_prio** | **normal_prio**           | **prio**       |
| ------------------------------------------------- | --------------- | ------------------------- | -------------- |
| Non-real-time task                                | static_prio     | static_prio               | static_prio    |
| Priority-boosted（优先级激励） non-real-time task | static_prio     | static_prio               | prio as before |
| Real-time task                                    | static_prio     | MAX_RT_PRIO-1-rt_priority | prio as before |

Notice that when a process forks off a child, the current static priority will be inherited from the parent.
**The dynamic priority of the child, that is, `task_struct->prio`, is set to the normal priority of the parent.**
**This ensures that priority boosts caused by RT-Mutexes are not transferred to the child process**.



### 计算负载权值Computing Load Weights

The importance of a task is not only specified by its priority, but also by the load weight stored in
`task_struct->se.load`.

 `set_load_weight` is responsible to compute the load weight depending on the process type and its static priority.

```c
<sched.h>
struct load_weight {
unsigned long weight, inv_weight;
};
```

**The general idea is that every process that changes the priority by one nice level down gets 10 percent more CPU power, while changing one nice level up gives 10 percent CPU power less. To enforce this policy, the kernel converts priorities to weight values**.

```c
kernel/sched.c
static const int prio_to_weight[40] = {

88761, 29154, 9548, 3121, 1024,
71755, 23254, 7620, 2501, 820,
56483, 18705, 6100, 1991, 655,
46273, 14949, 4904, 1586, 526,
36291, 11916, 3906, 1277, 423,
335, 272, 110, 87, 36, 29,
215, 172, 137, 70, 56, 45, 23, 18, 15,
};
```

The array contains one entry for each nice level in the range [0, 39] as used by the kernel. The multiplier
between the entries is 1.25. To see why this is required, consider the following example. Two processes
*A* and *B* run at nice level 0, so **each one gets the same share of the CPU, namely, 50 percent**. The weight
for a nice 0 task is 1,024 as can be deduced from the table. The share for each task is 1024/(1024+1024) = 0.5, that is, 50 percent as expected.

If task *B* is **re-niced** by one priority level, it is supposed to get 10 percent less CPU share. In other words, 

this means that *A* will get 55 percent and B will get 45 percent of the total CPU time. Increasing the 

priority by 1 leads to a decrease of its weight, which is then 1, 024/1.25 ≈ 820. The CPU share *A* will 

get now is 0.55(1024/(1024+820)),  B will (820/(1024+820))=0.45 required. 

The code that performs the conversion also needs to account for real-time tasks. These will get double
of the weight of a normal task. SCHED_IDLE tasks, on the other hand, will always receive a very small
weight:

```c
kernel/sched.c
#define WEIGHT_IDLEPRIO 2 
#define WMULT_IDLEPRIO (1 << 31)

static void set_load_weight(struct task_struct *p) {
if (task_has_rt_policy(p)) {
	p->se.load.weight = prio_to_weight[0] * 2;
  p->se.load.inv_weight = prio_to_wmult[0] >> 1;
  return;
}
/*
* SCHED_IDLE tasks get minimal weight: */
if (p->policy == SCHED_IDLE) {
	p->se.load.weight = WEIGHT_IDLEPRIO;
  p->se.load.inv_weight = WMULT_IDLEPRIO;
  return;
}
p->se.load.weight = prio_to_weight[p->static_prio - MAX_RT_PRIO];
p->se.load.inv_weight = prio_to_wmult[p->static_prio - MAX_RT_PRIO];
}
```

**Recall that not only processes, but also run queues are associated with a load weight.** Every time a process is added to a run queue, the kernel calls `inc_nr_running`. This not only ensures that the run queue keeps track of how many processes are running, but also adds the process weight to the weight of the run
queue:



```c
kernel/sched.c
static inline void update_load_add(struct load_weight *lw, unsigned long inc) {
		lw->weight += inc;
}
static inline void inc_load(struct rq *rq, const struct task_struct *p) {
		update_load_add(&rq->load, p->se.load.weight);
}
static void inc_nr_running(struct task_struct *p, struct rq *rq) {
		rq->nr_running++;
    inc_load(rq, p);
}
```

Corresponding functions (`dec_nr_running`, `dec_load`, and `update_load_sub`) are called when a process 

is removed from the run queue. 

## 核心调度器 Core Scheduler

As mentioned above, scheduler implementation is based on two functions — **the periodic scheduler** and
**the main scheduler** function. **These distribute CPU time on the basis of the priorities of the available processes**; this is why the overall method can also be referred to as *priority scheduling* — although this is
a very general term, naturally. I discuss how priority scheduling is implemented in this section.

### 周期调度器 The Periodic Scheduler

The periodic scheduler is implemented in `scheduler_tick`. **The function is automatically called by the**
**kernel with the frequency HZ if system activity is going on.** If no processes are waiting to be scheduled, the tick can also be turned off to save power on computers where this is a scarce resource, for instance, laptops or small embedded systems. The mechanism underlying periodic actions is discussed in Chapter 15.
The function has two principal tasks.

1. To manage **the kernel scheduling-specific statistics** relating to the whole system and to the individual processes. The main actions performed involve incrementing counters and are of no particular interest to us. 
2. To **activate the periodic scheduling method of the scheduling class** responsible for the current process. 

```c
kernel/sched.c
void scheduler_tick(void) {
     ...
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *curr = rq->curr;
    __update_rq_clock(rq) update_cpu_load(rq);
```

The first part of the function deals with updating the run queue clock. This is delegated to
`__update_rq_clock`, which essentially advances the clock time stamp of the current instance of struct
`rq`. The function has to deal with some oddities of hardware clocks, but these are not relevant for our
purposes. `update_cpu_load` then deals with updating the`cpu_load[]` history array of the run queue.
This essentially shifts the previously stored load values one array position ahead, and inserts the present
run queue load into the first position. Additionally, the function introduces some averaging to ensure
that the contents of the load array do not exhibit large discontinuous jumps.

```c
kernel/sched.c
if (curr != rq->idle)
	curr->sched_class->task_tick(rq, curr);
}
```

**How `task_tick` is implemented depends on the underlying scheduler class.** 

The completely fair scheduler, for instance, will in this method **check if a process has been running for too long to avoid large latencies**, but I discuss this in detail below. 

Readers familiar with the old time-slice-based scheduling method should be aware, however, that this is *not* equivalent to an expiring time slice — they do not exist anymore in the completely fair scheduler. 

If the current task is supposed to be rescheduled, the scheduler class methods set the **TIF_NEED_RESCHED** flag in the task structure to express this request, and the kernel fulfills it at the next opportune moment. 

### 主调度器 The Main Scheduler

**The main scheduler function (schedule) is invoked directly at many points in the kernel to allocate the CPU to a process other than the currently active one.** After returning from system calls, the kernel also checks whether the reschedule flag **TIF_NEED_RESCHED** of the current process is set — for example, the flag is set by `scheduler_tick` as mentioned above. If it is, the kernel invokes schedule. **The function then assumes that the currently active task is definitely to be replaced with another task.** 

Before I discuss schedule in detail, I need to make one remark that concerns the `__sched` prefix. This is used for functions that can potentially call schedule, including the schedule function itself. The declaration looks as follows: 

```c
void __sched some_function(...) { 
  ...
	schedule();
	... 
}
```

The purpose of the prefix is to put the compiled code of the function into a special section of the object file,
namely, .sched.text (see Appendix C for more information on ELF sections). This information enables
the kernel to ignore all scheduling-related calls when a stack dump or similar information needs to be
shown. Since the scheduler function calls are not part of the regular code flow, they are of no interest in
such cases.

**the main scheduler schedule**. The function first determines the current run queue and saves a pointer to the `task` structure of the (still) active process in `prev`.



```c
kernel/sched.c
asmlinkage void __sched schedule(void) {
    struct task_struct *prev, *next;
    struct rq *rq;
    int cpu;
    
need_resched:
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;
    ...
```

As in **the periodic scheduler**, the kernel takes the opportunity to **update the run queue clock** and clears
the reschedule flag **TIF_NEED_RESCHED** in the `task` structure of the currently running task.

```c
__update_rq_clock(rq);
clear_tsk_need_resched(prev);
```

**Again thanks to the modular模块化 structure of the scheduler, most work can be delegated to the scheduling classes.** If the current task was in an interruptible sleep but has received a signal now, it must be promoted to a running task again. Otherwise, the task is deactivated with the scheduler-class-specific methods (`deactivate_task` essentially ends up in calling `sched_class->dequeue_task`): 

```c
kernel/sched.c 

if (unlikely((prev->state & TASK_INTERRUPTIBLE) &&
    unlikely(signal_pending(prev)))) { 

    prev->state = TASK_RUNNING;
} else { 
    deactivate_task(rq, prev, 1); 
} ... 
```

`put_prev_task` first **announces to the scheduler class** that the currently running task is going to be
replaced by another one.(当前的task会被别人取代) 

**Note that this is *not* equivalent to taking the task off the run queue, but provides the opportunity to perform some accounting and bring statistics up to date.** The next task that is supposed to be executed must also be selected by the scheduling class, and `pick_next_task` is responsible to do so:

```c
prev->sched_class->put_prev_task(rq, prev);
next = pick_next_task(rq, prev);
...
```

 **If only one task is currently able to run because all others are sleeping, it will naturally be left on the CPU**. If, however, **a new task has been selected, then task switching at the hardware level must be prepared and executed**.

```c
kernel/sched.c
if (likely(prev != next)) {
		rq->curr = next;
		context_switch(rq, prev, next);
}
...
```

`context_switch` is **the interface to the architecture-specific methods that perform a low-level context switch**.

The following code **checks if the reschedule bit of the current task is set**, and the scheduler jumps to the label described above and the **search for a new process recommences**: 

```c
kernel/sched.c 

if (unlikely(test_thread_flag(TIF_NEED_RESCHED)))
  	goto need_resched; 

} 
```

Notice that the above piece of code is executed in two different contexts: 

When **no context switch has been performed**, it is **run directly at the end of the schedule function**. 

If, however, **a context switch has been performed**, the current process will **stop running *right before* this point — the new task has taken over the CPU**. 

However, **when the previous task is reselected to run later on**, it will resume its execution directly at this point. Since `prev` will not point to the proper process in this case, the current thread needs to be found via current by `test_thread_flag`.



### Interaction with **fork** 

Whenever a new process is created using the `fork` system call or one of its variants, **the scheduler gets a chance to hook into the process with the `sched_fork` function.** On a single-processor system, the function performs essentially three actions: 

- Initialize the scheduling-related fields of the new process.
-  set up data structures (this is rather straightforward).
-  determine the dynamic priority of the process: 

```c
kernel/sched.c 

  /*
  * fork()/clone()-time setup:
  */ 
  void sched_fork(struct task_struct *p, int clone_flags) { 
  /* Initialize data structures */ 
  ...
 /* 
  * Make sure we do not leak PI boosting priority to the child: 
  */
 p->prio = current->normal_prio;
 if (!rt_prio(p->prio)) 
		 p->sched_class = &fair_sched_class; 
...
} 
```

**By using the *normal* priority of the parent process as the *dynamic* priority of the child, the kernel ensures that <u>any temporary boosts of the parent’s priority</u> are not inherited by the child.** 

Recall that **the dynamic priority of a process can be temporarily modified when RT-Mutexes are used.** **This effect must not be transferred to the child**. If the priority is not in the real-time range, the process will always start out in the completely fair scheduling class. 

**When a new task is woken up using `wake_up_new_task`, a second opportunity for the scheduler to interact with task creation presents itself**: The kernel calls the `task_new` function of the scheduling class. This gives an opportunity to enqueue the new process into the run queue of the respective class. 

### Context Switching 

Once the kernel has selected a new process, the technical details associated with multitasking must be dealt with; these details are known collectively as *context switching*. The auxiliary function `context_switch` is the dispatcher for the required architecture-specific methods. 

```c
kernel/sched.c 

static inline void
context_switch(struct rq *rq, struct task_struct *prev, 
               struct task_struct *next)
{ 
		 struct mm_struct *mm, *oldmm; 

		 prepare_task_switch(rq, prev, next);
  	 mm = next->mm;
     oldmm = prev->active_mm; 
  ..
```

Immediately before a task switch, the **prepare_arch_switch** hook that must be defined by every archi- tecture is called from **prepare_task_switch**. This enables the kernel to execute architecture-specific code to prepare for the switch. Most supported architectures (with the exception of Sparc64 and Sparc) do not use this option because it is not needed. 

The **context switch** proper is performed by invoking two processor-specific functions: 

1. `switch_mm` changes **the memory context** described in` task_struct->mm`. Depending on the processor, **this is done by loading the `page tables`, flushing the `translation lookaside buffers` (partially or fully), and supplying the `MMU` with new information.** Because these actions go deep into CPU details, I do not intend to discuss their implementation here. 

2. `switch_to` switches **the processor register contents 处理器寄存器内容** and **the kernel stack** (the virtual user address space is changed in the first step, and as it includes the user mode stack, it is not necessary to change the latter explicitly). This task also varies greatly from architecture to architecture and is usually coded entirely in **assembly language**. Again, I ignore implementation details. 

   Because **the register contents of the userspace process** are **saved on the kernel stack** when kernel mode is entered (see Chapter 14 for details), this need not be done explicitly during the context switch. And because each process first begins to execute in kernel mode (at that point during scheduling at which control is passed to the new process), the register contents are automatically restored using the values on the kernel stack when a return is made to userspace. 

Remember, however, that **kernel threads do not have their own userspace memory context and execute on top of the address space of a random task**; their `task_struct->mm` is NULL. The address space ‘‘borrowed’’ from the current task is noted in `active_mm` instead: 

```c
kernel/sched.c 

if (unlikely(!mm)) {
    next->active_mm = oldmm; 
    atomic_inc(&oldmm->mm_count); 
    enter_lazy_tlb(oldmm, next); 
} else 
	switch_mm(oldmm, mm, next);
```

 `enter_lazy_tlb` notifies the underlying architecture that **exchanging the userspace portion of the virtual address space is not required.** This speeds up the context switch and is known as the *lazy TLB* technique. 

If **the previous task was a kernel thread** (i.e., `prev->mm` is NULL), its `active_mm` pointer must be reset to  NULL to **disconnect it from the borrowed address space**: 

```c
kernel/sched.c 

if (unlikely(!prev->mm)) {
	prev->active_mm = NULL; 
	rq->prev_mm = oldmm; 

} ... 

```

Finally, the **task switch** is finished with `switch_to`, which **switches the register state and the stack** — the new process will be running after the call:



```c
kernel/sched.c

/* Here we just switch the register state and the stack. */ 
switch_to(prev, next, prev);

barrier(); 
/*
* this_rq must be evaluated again because prev may have moved
* CPUs since it called schedule(), thus the ’rq’ on its stack
* frame will be invalid.
*/
finish_task_switch(this_rq(), prev);
 }
```

The code following after `switch_to` will only be executed when the current process is selected to run
next time.

 `finish_task_switch` performs some **cleanups and allows for correctly releasing locks**. It also gives individual architectures another possibility to hook into the context switching process, but this is only required on a few machines.

The `barrier` statement is a directive for the compiler that ensures that the order in which the `switch_to` and `finish_task_switch` statements are executed is **not changed by any unfortunate optimizations** (see Chapter 5 for more details).

#### 复杂的switch_to

The interesting thing about finish_task_switch is that the cleanups are performed for the task that has
been active before the running task has been selected for execution. Notice that this is not the task that
has initiated the context switch, but some random other task in the system! The kernel must find a way
to communicate this task to the `context_switch` routine, and this is achieved with the `switch_to` **macro**.
It must be implemented by every architecture and has a very unusual calling convention: Two variables
are handed over, but in *three* parameters! This is because not only two, but three processes are involved
in a context switch. The situation is illustrated in Figure 2-16.

![](../img/linux-kernel-context-switch.png)

Suppose that three processes A, B, and C are running on the system. At some point in time, the kernel
decides to switch from A to B, then from B to C, and then from C back to A again. **Before each `switch_to`call, the pointers `next` and `prev` located on the stacks of the individual processes are set such that `prev` points to the *currently* running process, while `next` points to the process that will be running next.** 

To perform the switch from `prev` to `next`, the first two arguments are completely sufficient for` switch_to`.
For process A, prev points to A and next points to B.

**A problem arises when A is selected to execute again. Control will return to the point after `switch_to`, and if the stack were restored to the exact state it had before the switch, `prev` and `next` would still point to the same values as before the switch — namely, next=B and prev=A. In this situation, the kernel would not know that process C has actually run before process A.** 

Therefore, **the low-level task switch routine must <u>feed the previously executing task</u> to `context_switch` when a new task is selected.** Since control flow comes back to the middle of the function, this cannot be done with regular function return values, and that is why a three-parameter *macro* is used. However, the conceptional effect is the same as if switch_to were a function of two arguments that would return a pointer to the previously executing process. What switch_to essentially does is 

```c
prev = switch_to(prev,next) 
```

where the prev value returned is *not* the prev value used as the argument, but the process that executed
last in time. **In the above example, process A would feed `switch_to` with A and B, but would obtain `prev=C` as result.** 

#### Lazy FPU Mode

Because the speed of context switching plays a major role in system performance, the kernel uses a trick
to reduce the CPU time needed. **Floating-point registers are not saved unless they are actually used by the application and are not restored unless they are required**. 

This is known as the *lazy FPU* technique. Its implementation differs from platform to platform because assembly language code is used, but the basic principle is always the same. It should also be noted that, regardless of platform, **the contents of the floating-point registers** are *not* saved on the **process stack** but **in its thread data structure**. I illustrate this technique by means of an example.

For the sake of simplicity, let us assume this time that there are only two processes, A and B, on the
system. Process A is running and uses floating-point operations. When the scheduler switches to process
B, the contents of the floating-point registers of A are saved in the thread data structure of the process.
However, the values in these registers are *not* immediately replaced with the values for process B.

**If B does not perform any floating-point operations during its time slice**, A sees its former register contents when it is next activated. The kernel is therefore spared the effort of explicitly restoring register
values, and this represents a time-saving.

**If, however, B does perform floating-point operations, this fact is reported to the kernel so that it can fill the registers with the appropriate values from the thread data structure.** Consequently, the kernel saves and restores floating-point register contents only when needed and wastes no time on superfluous operations.

##  The Completely Fair Scheduling Class



All information that **the core scheduler** needs to know about the **completely fair scheduler** is contained
in `fair_sched_class`:

```c
kernel/sched_fair.c
static const struct sched_class fair_sched_class = {
    .next = &idle_sched_class,
    .enqueue_task = enqueue_task_fair,
    .dequeue_task = dequeue_task_fair,
    .yield_task = yield_task_fair,
    .check_preempt_curr = check_preempt_wakeup,
    .pick_next_task = pick_next_task_fair,
    .put_prev_task = put_prev_task_fair,
    ...
    .set_curr_task = set_curr_task_fair,
    .task_tick = task_tick_fair,
    .task_new = task_new_fair,
};
```

### 1. Data Structures

First, I need to introduce **how the CFS run queue looks**. Recall that an instance is embedded into each
per-CPU run queue of **the main scheduler**:

```c
kernel/sched.c
struct cfs_rq {
    struct load_weight load;
    unsigned long nr_running;
    
    u64 min_vruntime;
    
    struct rb_root tasks_timeline;
    struct rb_node *rb_leftmost;
    struct sched_entity *curr;
}

```

❑  `nr_running` counts **the number of runnable processes on the queue**, and `load` maintains the cumulative load values of them all. Recall that you have already encountered the load calculation in Section 2.5.3. 

❑  `min_vruntime` tracks **the minimum virtual run time of all processes on the queue**. This value forms the basis to **implement the virtual clock associated with a run queue**. The name is slightly confusing because `min_vruntime` can actually be bigger than the vruntime setting of the leftmost tree element as it needs to increase monotonically, but I will come back to this when I discuss how the value is set in detail. 

❑  `tasks_timeline` is **the base element** to **manage all processes in a time-ordered red-black tree.**` rb_leftmost` is always set to the leftmost 最左 element of the tree, that is, the element that deserves to be scheduled most. The element could, in principle, be obtained by walking through the red- black tree, but since usually only the leftmost element is of interest, this speeds up the average time spent searching the tree. 

❑  `curr` points to the schedulable entity of the currently executing process. 

### 2. CFS Operations

#### The Virtual Clock

The completely fair scheduling algorithm depends on a virtual clock that measures the amount of time a waiting process would have been allowed to spend on the CPU on a completely fair system.

However, **no virtual clock can be found anywhere in the data structures**! This is because **all required information can be inferred from the existing real-time clocks combined with the load weight associated with every process**. 

**All calculations related to the virtual clock** are performed in `update_curr`, which is called from various places in the system including the periodic scheduler. The code flow diagram in Figure 2-17 provides an overview of what the function does.

![](../img/linux-kernel-virtual-clock-code-flow.png)

First of all, the function determines **the currently executing process of the run queue** and also **obtains the real clock value of the main scheduler run queue**, which is **updated at each scheduler tick** (`rq_of` is an auxiliary function to determine the instance of struct`rq` that is associated with a CFS run queue): 

```c
static void update_curr(struct cfs_rq *cfs_rq) { 
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_of(cfs_rq)->clock;
    unsigned long delta_exec; 

    if (unlikely(!curr))
      return; 
  ...
    
```

If no process is currently executing on the run queue, there is obviously nothing to do. Otherwise, the kernel computes the time difference between the last update of the load statistics and now, and delegates the rest of the work to` __update_curr`. 

```c
kernel/sched_fair.c 

delta_exec = (unsigned long)(now - curr->exec_start); 

__update_curr(cfs_rq, curr, delta_exec);
curr->exec_start = now; 
} 

```

Based on this information, `__update_curr` has to **update the physical and virtual time** that the current process has spent executing on the CPU. This is simple for the physical time. The time difference just needs to be added to the previously accounted time: 

```c
kernel/sched_fair.c 

static inline void
 __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr, 
               unsigned long delta_exec) 
{ 
    unsigned long delta_exec_weighted;
    u64 vruntime; 

    curr->sum_exec_runtime += delta_exec; 
  ... 
```

The interesting thing is how the non-existing virtual clock is emulated using the given information. Once
more, the kernel is clever and saves some time in the common case: For processes that run at nice level
0, virtual and physical time are identical by definition. When a different priority is used, the time must
be weighted according to the load weight of the process (recall that Section 2.5.3 discussed how process
priority and load weight are connected):

```c
kernel/sched_fair.c
  delta_exec_weighted = delta_exec;
  if (unlikely(curr->load.weight != NICE_0_LOAD)) {
      delta_exec_weighted = calc_delta_fair(delta_exec_weighted, &curr->load);
  }
	curr->vruntime += delta_exec_weighted;
...
```

Neglecting some rounding and overflow checking, what calc_delta_fair does is to compute the value 

given by the following formula: 

```
delta_exec_weighted = delta_exec × NICE_0_LOAD/curr->load.weight 
```

The inverse weight values mentioned above can be brought to good use in this calculation. 

Finally, the kernel needs to set min_vruntime. Care is taken to ensure that the value is increasing mono-
tonically .

```c
kernel/sched_fair.c
/*
* maintain cfs_rq->min_vruntime to be a monotonically increasing * value tracking the leftmost vruntime in the tree.
*/
if (first_fair(cfs_rq)) {
    vruntime = min_vruntime(curr->vruntime,
    					 __pick_next_entity(cfs_rq)->vruntime);
}else
		vruntime = curr->vruntime;
cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
}
```

`first_fair` is **a helper function that checks if the tree has a leftmost element**, that is, if any process is
waiting on the tree to be scheduled. If so, the kernel obtains its vruntime, which is the smallest of all
elements in the tree. If no leftmost element is in the tree because it is empty, the virtual run time of the
current process is used instead. To ensure that the per-queue `min_vruntime` is monotonic increasing, the
kernel sets it to the larger of both values. This means that the per-queue `min_vruntime` is only updated if
it is exceeded by the vruntime of one of the elements on the tree. With this policy, the kernel ensures that
`min_vrtime` can only increase, but never decrease.

One really crucial point of **the completely fair scheduler** is that **sorting processes on the red-black tree** is based on the following key:

```c
kernel/sched_fair.c
static inline s64 entity_key(struct cfs_rq *cfs_rq, struct sched_entity *se) {
		return se->vruntime - cfs_rq->min_vruntime;
}
```

Elements with a smaller key will be placed more to the left, and thus be scheduled more quickly. This way, the kernel implements two antagonistic对抗机制 mechanisms: 

1. When a process is running, **its vruntime will steadily increase**, so it will finally move rightward in the red-black tree. 

   **Because vruntime will increase *more slowly* for more important processes**, they will also move rightward more slowly, **so their chance to be scheduled is bigger than for a less important process — just as required.** 

2. **If a process sleeps, its `vruntime` will remain unchanged.** Because the per-queue `min_vruntime` increases in the meantime (recall that it is monotonic!), the sleeper will be placed more to the left after waking up because **the key got *smaller***.

In practice, both effects naturally happen simultaneously, but this does not influence the interpretation. Figure 2-19 illustrates the different movement mechanisms on the red-black tree graphically. 

![](../img/linux-kernel-virtual-red-black.png)

#### Latency Tracking

The kernel has a built-in notion of what it considers a good scheduling latency, that is, the interval
during which every runnable task should run at least once.

It is given in `sysctl_sched_latency`, which can be controlled via `/proc/sys/kernel/sched_latency_ns` and defaults to, respectively, 20,000,000 ns and 20 ms. A second control parameter, `sched_nr_latency`, controls **the number of active processes** that are **at most handled in one latency period**. 

If the number of active processes grows larger than this bound, the latency period is extended linearly. 

`__sched_period` determines the length of the latency period, which is usually just `sysctl_sched_latency`, but is extended linearly if more processes are running. In this case, the period length is 

```c
sysctl_sched_latency × nr_running /sched_nr_latency .
```

Distribution of the time among active processes in one latency period is performed by considering the rel- ative weights of the respective tasks. **The slice length for a given process as represented by a schedulable entity is computed as follows**: 

```c
kernel/sched_fair.c
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se) {
 
      u64 slice = __sched_period(cfs_rq->nr_running);
      slice *= se->load.weight;
      do_div(slice, cfs_rq->load.weight);
      
      return slice;
 }
```

Recall that the run queue load weight accumulates the load weights of all active processes on the queue.
The resulting time slice is given in real time, but the kernel sometimes also needs to know the equivalent
in virtual time.

```c
kernel/sched_fair.c
static u64 __sched_vslice(unsigned long rq_weight, unsigned long nr_running) {
    u64 vslice = __sched_period(nr_running);
    vslice *= NICE_0_LOAD;
    do_div(vslice, rq_weight);
    return vslice;
}
static u64 sched_vslice(struct cfs_rq *cfs_rq) {
		return __sched_vslice(cfs_rq->load.weight, cfs_rq->nr_running);
}
```

Recall that a real-time interval time for a process with a given weight has the length

```
time × NICE_0_LOAD/weight
```

and this is also used to transfer the latency interval portion.

Now we have everything in place to discuss the various methods that must be implemented by CFS to
interact with the global scheduler.

### 3. Queue Manipulation 队列操作

Two functions are available to move elements to and from the run queue: `enqueue_task_fair` and
`dequeue_task_fair`. Let us concentrate on placing new tasks on the run queue first.

Besides `pointers` to the generic `run queue` and the `task structure` in question, the function accepts one more parameter: `wakeup`. This allows for specifying if the task that is enqueued has only recently been
woken up and changed into the running state (wakeup is 1 in this case), or if it was runnable before
(wakeup is 0 then). The code flow diagram for `enqueue_task_fair` is shown in Figure 2-20.

![](../img/linux-kernel-queue-mani.png)

Let us go back to `enqueue_entity`: After `place_entity` has determined the proper virtual run time for the process, it is placed on the red-black tree with `__enqueue_entity`. I have already noted before that this is a purely mechanical function that uses standard methods of the kernel to sort the task into the red-black tree.

```c
kernel/sched_fair.c
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
    u64 vruntime;
    vruntime = cfs_rq->min_vruntime;
    if (initial)
    		vruntime += sched_vslice_add(cfs_rq, se);
    if (!initial) {
        vruntime -= sysctl_sched_latency;
        vruntime = max_vruntime(se->vruntime, vruntime);
    }
    se->vruntime = vruntime;
}
```

Since the kernel has promised to run all active processes at least once within the current latency period, the `min_vruntime` of the queue is used as the base virtual time, and by subtracting `sysctl_sched_latency`, it is ensured that the newly awoken process will only run after the current latency period has been finished.

### 3. 选择下一个Task

Selecting the next task to run is performed in `pick_next_task_fair`. The code flow diagram is shown in Figure 2-21.

If no tasks are currently runnable on the queue as indicated by an empty `nr_running counter`, there is little to do and the function can return immediately. Otherwise, the work is delegated to `pick_next_entity`.

If a leftmost task is available in the tree, it can immediately be determined using the `first_fair` helper function, and `__pick_next_entity` extracts the `sched_entity` instance from the red-black tree. This is done using the container_of mechanism because the red-black tree manages instances of `rb_node` that are embedded in `sched_entitys`.

Now the task has been selected, but some more work is required to mark it as the running task. This is handled by `set_next_entity`.

```c
kernel/sched_fair.c
static void
set_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    /* ’current’ is not kept within the tree. */ 
    if (se->on_rq) {
    		__dequeue_entity(cfs_rq, se);
    }
...
```

<img src="../img/linux-kernel-next-task-fair.png" style="zoom:80%;" />

The currently executing process is not kept on the run queue, so `__dequeue_entity` removes it from the red-black tree, setting the leftmost pointer to the next leftmost task if the current task has been the leftmost one. Notice that in our case, the process has been on the run queue for sure, but this need not be the case when `set_next_entity` is called from different places.

Although the process is not contained in the red-black tree anymore, the connection between process and run queue is not lost, because `curr` marks it as the running one now:

```c
kernel/sched_fair.c

cfs_rq->curr = se;
 se->prev_sum_exec_runtime = se->sum_exec_runtime;

}
```

Because the process is now the currently active one, the real time spent on the CPU will be charged to `sum_exec_runtime`, so the kernel preserves the previous setting in `prev_sum_exec_runtime`. Note that `sum_exec_runtime` is *not* reset in the process. The difference `sum_exec_runtime - prev_sum_ exec_runtime` does therefore denote the real time spent executing on a CPU.

### 5. 处理周期性Tick

This aforementioned 上述的 difference is important **when the periodic tick is handled.** The formally responsible function is `task_tick_fair`, but the real work is done in `entity_tick`. Figure 2-22 presents the code flow diagram.

![](../img/linux-kernek-entity-tick.png)

First of all, the statistics are always updated using `update_curr`. If the `nr_running` counter of the queue indicates that fewer than two processes are runnable on the queue, nothing needs to be done. **If a process is supposed to be preempted 可被抢占的, there needs to be at least another one that *could* preempt抢占 it.** Otherwise, the decision is left to `check_preempt_tick` 检测抢占时钟tick:

```c
kernel/sched_fair.c

static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr) {

    unsigned long ideal_runtime, delta_exec;

    ideal_runtime = sched_slice(cfs_rq, curr);
		delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime)
				resched_task(rq_of(cfs_rq)->curr);
  
}
```

**The purpose of the function is to ensure that no process runs longer than specified by its share of the latency 等待周期 period.** This length of this share in real-time is computed in  `sched_slice`, and the real- time interval during which the process has been running on the CPU is given by `sum_exec_runtime`
`-prev_sum_exec_runtime` as explained above. The preemption decision **抢占机制** is thus easy: **If the task has been running for longer than the desired time interval, a reschedule is requested with `resched_task`**. This sets the **TIF_NEED_RESCHED** flag in the `task` structure, and **the core scheduler** 核心调度器 will initiate a rescheduling at the next opportune moment.

### 唤醒抢占

**When tasks are woken up in `try_to_wake_up` and `wake_up_new_task`, the kernel uses `check_preempt_curr` to see if the new task can preempt the currently running one.** Notice that <u>the **core scheduler核心调度器** is **not involved** in this process</u>! For completely fair handled tasks, the function `check_ preempt_wakeup` performs the desired check.

The newly woken task need not necessarily be handled by the completely fair scheduler. **If the new task is a real-time task, rescheduling is immediately requested because real-time tasks always preempt CFS tasks**:

```c
kernel/sched_fair.c

static void check_preempt_wakeup(struct rq *rq, struct task_struct *p) {

 struct task_struct *curr = rq->curr;
 struct cfs_rq *cfs_rq = task_cfs_rq(curr);
 struct sched_entity *se = &curr->se, *pse = &p->se;
 unsigned long gran;
//实时任务
 if (unlikely(rt_prio(p->prio))) {
   update_rq_clock(rq);
   update_curr(cfs_rq);
	 resched_task(curr);
   return;
 }
 ...
```

The most convenient cases are **SCHED_BATCH** tasks — **they do not preempt other tasks by definition 不会抢占其他任务.**

```c
kernel/sched.c

if (unlikely(p->policy == SCHED_BATCH))
  return;
...
```

**When a running task is preempted by a new task 正在跑的task被一个新的任务抢占, the kernel ensures that the old one has at least been running for a certain minimum amount of time.内核保证被抢占的task至少已经运行了一个最小的时间限额（不是随便抢占的哈）**。 

The minimum is kept in `sysctl_sched_ wakeup_granularity`, which crossed our path before. Recall that it is **per default set to 4 ms**. This refers to real time 这个依赖于实时时间, so **the kernel first needs to convert it into virtual time if required**:转换为需要的虚拟时间

```c
kernel/sched_fair.c

gran = sysctl_sched_wakeup_granularity;
if (unlikely(se->load.weight != NICE_0_LOAD))
		gran = calc_delta_fair(gran, &se->load);
```

If **the virtual run time of the currently executing task** (represented by its scheduling entity `se`) is larger than **the virtual run time of the new task** plus **the granularity 粒度** safety, a rescheduling is requested:

```c
kernel/sched_fair.c

if (pse->vruntime + gran < se->vruntime)
  resched_task(curr);
}
```

**The added ‘‘buffer’’ time ensures that tasks are not switched too frequently so that not too much time is spent in context switching instead of doing real work**.

### 处理新的Tasks

**The last operation of the completely fair scheduler that we need to consider is the hook function that is called when new tasks are created: `task_new_fair`.** The behavior of the function is controllable with the parameter `sysctl_sched_child_runs_first`. **As the name might suggest, it determined if a newly created child process should run before the parent**. This is usually beneficial, especially if the child performs an exec system call afterward. The default setting is 1, but this can be changed via `/proc/sys/kernel/sched_child_runs_first`.

Initially, the function performs the usual statistics update with `update_curr` and then employs the previously discussed place_entity:

```c
kernel/sched_fair.c

static void task_new_fair(struct rq *rq, struct task_struct *p) {

...

  struct cfs_rq *cfs_rq = task_cfs_rq(p);
  struct sched_entity *se = &p->se, *curr = cfs_rq->curr;
  int this_cpu = smp_processor_id();

  update_curr(cfs_rq);
  place_entity(cfs_rq, se, 1);
```

In this case, `place_entity` is called with initial set to 1, which amounts to computing the initial vruntime with `sched_vslice_add`. Recall that this determines the portion of the latency interval that belongs to the process, but converted to virtual time. This is the scheduler’s initial debt to the process.

```c
kernel/sched_fair.c

if (sysctl_sched_child_runs_first && curr->vruntime < se->vruntime) {
  swap(curr->vruntime, se->vruntime);

}

enqueue_task_fair(rq, p, 0);
resched_task(rq->curr);

}
```

If **the virtual run time of the parent** (represented by `curr`) is less than the virtual run time of the child, this would mean that the parent runs before the child — recall that small virtual run times favor left positions in the red-black tree. If the child is supposed to run before the parent, the virtual run times of both need to be swapped.

Afterward, the child is enqueued into the run queue as usual, and rescheduling is requested.



## 实时调度类

As mandated 被授权 by the POSIX standard, **Linux supports two real-time scheduling classes in addition to ‘‘normal‘‘ processes. T**he structure of the scheduler enables real-time processes to be integrated into the kernel without any changes in the core scheduler — this is a definitive advantage of scheduling classes.

Now is a good place to recall some of the facts discussed a long time ago. Real-time processes can be identified by the fact that they have a *higher* priority than normal processes — accordingly, their `static_prio` value is always *lower* than that of normal processes, as shown in Figure 2-14. **The `rt_task` macro is provided to establish whether a given task is a real-time process or not by inspecting its priority, and `task_has_rt_policy` checks if the process is associated with a real-time scheduling policy.**

### 1. 属性

**Real-time processes** differ from **normal processes** in one essential way: If a real-time process exists in the system and is runnable, it will always be selected by the scheduler — unless there is another real-time process with a higher priority.

The two available real-time classes differ as follows:

❑  *Round robin* processes (**SCHED_RR**) have a time slice whose value is reduced when they run if they are normal processes. Once all time quantum have expired, the value is reset to the initial value, but the process is placed at the end of the queue. This ensures that if there are several SCHED_RR processes with the same priority, they are always executed in turn.

❑  *First-in, first-out* processes (**SCHED_FIFO**) do not have a time slice and are permitted to run as long as they want once they have been selected.

It is evident that the system can be rendered unusable by badly programmed real-time processes — all that is needed is an endless loop whose loop body never sleeps. Extreme care should therefore be taken when writing real-time applications.

### 2. 数据结构

```c
kernel/sched-rt.c
const struct sched_class rt_sched_class = {
  .next = &fair_sched_class,
	.enqueue_task = enqueue_task_rt,
  .dequeue_task = dequeue_task_rt,
  .yield_task = yield_task_rt,
  
  .check_preempt_curr = check_preempt_curr_rt,
  
  .pick_next_task = pick_next_task_rt,
  .put_prev_task = put_prev_task_rt,
  
	.set_curr_task = set_curr_task_rt,
  .task_tick = task_tick_rt,
  };
```



The implementation of **the real-time scheduler class** is simpler than **the completely fair scheduler.** Only roughly 250 lines of code compared to 1,100 for CFS are required!

The core run queue also contains a sub-run queue for real-time tasks as embedded instance of struct `rt_rq`:

```c
kernel/sched.c

struct rq {
  ...
		t_rq rt;
  ...
}
```

The run queue is very straightforward — a linked list is sufficient:

```c
kernel/sched.c

struct rt_prio_array {
  DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1);/* include 1 bit for delimiter */
  struct list_head queue[MAX_RT_PRIO];

};

struct rt_rq {
 struct rt_prio_array active;
};
```

All real-time tasks with the same priority are kept in a linked list headed by `active.queue[prio]`, and the bitmap `active.bitmap` signals in which list tasks are present by a set bit. If no tasks are on the list, the bit is not set. Figure 2-23 illustrates the situation.

![](../img/linux-kernal-real-time-run-queue.png)

The analog of `update_cur` for the real-time scheduler class is` update_curr_rt`: The function keeps track of the time the current process spent executing on the CPU in `sum_exec_runtime`. **All calculations are performed with real times; virtual times are not required**. This simplifies things a lot.

### 调度操作

To enqueue and dequeue tasks is simple: The task is placed or respectively removed from the appropriate list selected by `array->queue` + `p->prio`, and **the corresponding bit in the `bitmap` is set if at least one task is present**, or removed if no tasks are left on the queue. Notice that new tasks are always queued at the end of each list.

The two interesting operations are **how the next `task` is selected** and **how `preemption` is handled**. Consider `pick_next_task_rt`, which handles selection of the next task first. The code flow diagram is shown in Figure 2-24.

![](../img/linux-kernel-rt-scheduler.png)

`sched_find_first_bit` is a standard function that finds the first set bit in `active.bitmap` — this means that higher real-time priorities (which result in lower in-kernel priorities) are handled before lower real- time priorities. The first task on the selected list is taken out, and `se.exec_start` is set to the current real-time clock value of the run queue — that’s all that is required.

The implementation of the periodic tick is likewise simple. **SCHED_FIFO** tasks are easiest to handle: They can run as long as they like and must pass control to another task explicitly by using the yield system call:

```c
kernel/sched.c

static void task_tick_rt(struct rq *rq, struct task_struct *p)
{

update_curr_rt(rq);

/*

*RR tasks need a special form of timeslice management.

*FIFO tasks have no timeslices. */
 if (p->policy != SCHED_RR)
		return;
```

If the current process is a round robin process, its time slice is decremented. When the time quantum
 is not yet exceeded, nothing more needs to be done — the process can keep running. Once the counter reverts to 0, its value is renewed to **DEF_TIMESLICE**, which is set to `100 * HZ / 1000`, that is, 100 ms. If the task is not the only task in its list, it is requeued to the end. Rescheduling is requested as usual by setting the **TIF_NEED_RESCHED** flag with `set_tsk_need_resched`:

```c
kernel/sched-rt.c
  if (--p->time_slice)
    	return;
  p->time_slice = DEF_TIMESLICE;
  /*
  * Requeue to the end of queue if we are not the only element
  * on the queue: */
  if (p->run_list.prev != p->run_list.next) {
      requeue_task_rt(rq, p);
      set_tsk_need_resched(p);
  }
}
```

The `sched_setscheduler` system call must be used to **convert a process into a real-time process**. This call is not discussed at length because it performs only the following simple tasks:

❑  It removes the process from its current queue using `deactivate_task`.

❑  It sets **the real-time priority** and **the scheduling class** in the `task` data structure.

❑  It reactivates the task.

<u>If **the process was not previously on any run queue**,</u> only **the scheduling class** and **the new priority value** need be set; **deactivation and reactivation are unnecessary**.

Note that changing the scheduler class or priority is only possible without constraints if the `sched_setscheduler` system call is performed by a process with root rights (or, equivalently, the capability **CAP_SYS_NICE**). Otherwise, the following conditions apply:

❑  **The scheduling class** can only be changed from **SCHED_NORMAL** to **SCHED_BATCH** or vice versa. A change to **SCHED_FIFO** is impossible.

❑  Only **the priority of processes** with the same **UID** or **EUID** as the **EUID** of the caller can be changed. Additionally, the priority may only be decreased, but not increased.





## 调度器增强

So far, we have only considered scheduling on real-time systems — naturally, Linux can do slightly better. **Besides support for multiple CPUs, the kernel also provides several other enhancements that relate to scheduling, discussed in the following sections**. Notice that **these enhancements add much complexity to the scheduler**, so I will mostly consider simplified situations that illuminate the essential principle, but do not account for all boundary cases and scheduling oddities.

### SMP 调度

On multiprocessor systems, the kernel must consider a few additional issues in order to ensure good scheduling:

❑  **The CPU load must be shared as fairly as possible over the available processors**. It makes little sense if one processor is responsible for three concurrent applicationswhile another has only the idle task to deal with. 如果一个processor负责三个并发应用， 

❑  **The *affinity* 亲和性of a task to certain processors in the system must be selectable**. This makes it possible, for example, to bind a compute-intensive application to the first three CPUs on a 4-CPU system while the remaining (interactive) processes run on the fourth CPU.

❑ **The kernel must be able to migrate processes from one CPU to another**. However, this option must be used with great care because it can severely impair performance. **CPU caches are the biggest problem on smaller SMP systems**. For *really* big systems, a CPU can be located literally some meters away from the memory previously used, so access to it will be very costly.

The affinity of a task to particular CPUs is defined in the `cpus_allowed` element of the `task` structure specified above. Linux provides the` sched_setaffinity` system call to change this assignment.

#### Extensions to the Data Structures

The scheduling methods that **each scheduler class must provide are augmented by two additional functions** on SMP systems:

```c
<sched.h>

struct sched_class {

 ...
 #ifdef CONFIG_SMP
unsigned long (*load_balance) (struct rq *this_rq,
                               int this_cpu,
                               struct rq *busiest,
                               unsigned long max_load_move,
															 struct sched_domain *sd,
                               enum cpu_idle_type idle,
                               int *all_pinned,
                               int *this_best_prio);

int (*move_one_task) (struct rq *this_rq,
                      int this_cpu,
 											struct rq *busiest,
                      struct sched_domain *sd,
											enum cpu_idle_type idle);

 #endif ...
 }


```

Despite their names, the functions are, however, not directly responsible to handle load balancing. **They are called by <u>the core scheduler</u> code whenever the kernel deems rebalancing necessary.** The scheduler class-specific functions then set up an iterator that allows the generic code to walk through all processes that are potential candidates to be moved to another queue, but the internal structures of the individual scheduler classes must *not* be exposed to the generic code because of the iterator. `load_balance` employs the generic function `load_balance`, while `move_one_task` uses `iter_move_one_task`. The functions serve different purposes:

❑  <u>`iter_move_one_task` picks one `task` off the busy run queue busiest and moves it to the run queue of the current CPU</u>.

❑  `load_balance` is allowed to <u>distribute multiple tasks from the busiest run queue to the current CPU</u>, but must not move more load than specified by `max_load_move`.

**How is load balancing initiated?** <u>On SMP systems, the `scheduler_tick` **periodic scheduler** function invokes the `trigger_load_balance` function on completion of the tasks required for all systems as described above.</u> 

This raises the **SCHEDULE_SOFTIRQ** softIRQ, guarantees that `run_rebalance_domains` will be run in due time. **This function finally invokes load balancing for the current CPU by calling `rebalance_domains`.** The time flow is illustrated in Figure 2-25.



To perform rebalancing, the kernel needs some more information. Run queues are therefore augmented with additional fields on SMP systems:

```c
kernel/sched.c
struct rq {
...
#ifdef CONFIG_SMP
  struct sched_domain *sd;
  /* For active balancing */
  int active_balance;
	int push_cpu;
	/* cpu of this runqueue: */
  int cpu;
	struct task_struct *migration_thread;
  struct list_head migration_queue;
#endif
  ...
}
```



![load-balance](../img/linux-kernel-smp-load-balance.png)

Run queues are CPU-specific, so `cpu` denotes the processor to which the run queue belongs. **The kernel provides one *migration thread* per run queue to which migration requests can be posted — they are kept on the list migration_queue.** 

Such **requests usually originate from the scheduler itself**, but **can also become necessary when a process is restricted to a certain set of CPUs and must not run on the one it is currently executing on anymore.** 

The kernel tries to balance run queues periodically, but if this fails to be satisfactory for a run queue, then active balancing must be used. `active_balance` is set to a nonzero value if this is required, and `cpu` notes the processor from which the request for active balancing initiates.

Furthermore, **all run queues are organized in *scheduling domains*调度域**. 

**This allows for grouping CPUs that are <u>physically adjacent</u> to each other or <u>share a common cache</u> such that processes should preferably be moved between them**. 

On ‘‘normal’’ SMP systems, however, all processors will be contained in one scheduling domain. I will therefore not discuss this structure in detail, but only mention that it contains numerous parameters that can be set via `/proc/sys/kernel/cpuX/domainY`. These include the minimal and maximal time interval after which load balancing is initiated, the minimal imbalance for a queue to be re-balanced, and so on. Besides, the structure also manages fields that are set at run time and that allow the kernel to keep track when the last balancing operation has been performed, and when the next will take place.

**So what does load_balance do?** 

The function checks if enough time has elapsed since the last re-balancing operation, and initiates a new re-balancing cycle if necessary by invoking `load_balance`. The code flow diagram for this function is shown in Figure 2-26. Notice that I describe a simplified version because **the SMP scheduler** has to deal with a very large number of corner cases that obstruct the view on the essential actions.

![](../img/linux-kernel-load-balance.png)

First of all, **the function has to identify which queue has most work to do**. This task is delegated to `find_busiest_queue`, which is called for a specific run queue. The function iterates over the queues
 of all processors (or, to be precise, of all processors in the current scheduling group) and compares their load weights. **The busiest queue is the queue with the largest value found in the end**.

Once `find_busiest_queue` has identified a very busy queue, and if at least one task is running on this queue (load balancing will otherwise not make too much sense), a suitable number of its tasks are migrated to the current queue using `move_tasks`. This function, in turn, invokes the scheduler-class- specific `load_balance` method.

When selecting potential migration candidates, the kernel must ensure that the process in question

❑  is not running at the moment or has just finished running because **this fact would cancel out the benefits of <u>the CPU caches currently filled with the process data</u>**.

❑  may execute on the processor associated with the current queue on the grounds of its CPU affinity .

If balancing failed (e.g., because all tasks on the remote queue have a higher kernel-internal priority value, i.e., a lower nice priority), **the migration thread that is responsible for the busiest run queue is woken up.** To ensure that active load balancing is performed that is slightly more aggressive than the method tried now, **`load_balance` sets the `active_balance` flag of the busiest run queue** and **also notes the CPU from which the request originates in `rq->cpu`.**



### 迁移线程

The migration thread serves two purposes: It must **fulfill migration requests** originating from the scheduler, and **it is used to implement active balancing.** **This is handled in a kernel thread that executes `migration_thread`.** The code flow diagram for the function is shown in Figure 2-27.

![](../img/linux-kernel-migration-thread.png)

**`migration_thread` runs an infinite loop and sleeps when there is nothing to do**. 

First of all, the function checks if active balancing is required, and if this is the case, `active_load_balance` is called to satisfy this request. 

The function tries to move one task from the current run queue to the run queue of the CPU that initiated the request for active balancing. It uses `move_one_task` for this purpose, which, in turn, ends up calling the scheduler-class specific `move_one_task` functions of all scheduler classes until one of them succeeds. 

Note that these functions try to move processes more aggressively than `load_balance`. For instance, they do not perform the previously mentioned priority comparison, so they are more likely to succeed.

Once the active load balancing is finished, the migration thread checks if any migration requests from the scheduler are pending in the `migrate_req` list. If none is available, the thread can reschedule. Otherwise, the request is fulfilled with `__migrate_task`, which performs the desired process movement directly without further interaction with the scheduler classes.

#### Core Scheduler Changes

Besides the additions discussed above, some changes to the existing methods are required in the core scheduler on SMP systems. While numerous small details change all over the place, the most important differences as compared to uniprocessor systems are the following:

❑  When a new process is started with the `exec` system call, a good opportunity for the scheduler to move the `task` across CPUs arises. Naturally, it has not been running yet, so there can not be any negative effects on the CPU cache by moving the task to another CPU. **`sched_exec` is the hook function invoked by the `exec` system call**, and the code flow diagram is shown in Figure 2-28.

**`sched_balance_self` picks the CPU that is currently least loaded (and on which the process is also allowed to run).** If this is not the current CPU, then `sched_migrate_task` forwards an according migration request to the migration thread using `sched_migrate_task`.

❑  The **scheduling granularity**调度的粒度 of **the completely fair scheduler** scales with the number of CPUs. The more processors present in the system, the larger the granularities that can be employed.处理器越多，粒度越大。 Both `sysctl_sched_min_granularity` and `sysctl_sched_latency` for `sysctl_sched_min_granularity` are multiplied by the correction factor **1 + log2(`nr_cpus`)**, where `nr_cpus` represents the number of available CPUs. However, they must not exceed 200 ms. `sysctl_sched_ wakeup_granularity` is also increased by the factor, but is not bounded from above.

![](../img/linux-kernel-sched-exec.png)

### 调度域和控制组 Scheduling Domains and Control Groups

In the previous discussion of the scheduler code, we have often come across the situation that the scheduler does not deal directly with processes, but with *schedulable entities*. This allows for implementing *group scheduling*: **Processes are placed into different groups, and the scheduler is first fair among these groups and then fair among all processes in the group.** 

This allows, for instance, granting identical shares of the available CPU time to each user. **Once the scheduler has decided how much time each user gets, the determined interval is then distributed between the users’ processes in a fair manner**. This naturally implies that the more processes a user runs, the less CPU share each process will get. **The amount of time for the user in total is not influenced by the number of processes, though**.

Grouping tasks between users is not the only possibility.

**The kernel also offers *control groups*,** which allow — via the special filesystem cgroups — creating arbitrary collections of tasks, which may even be sorted into multiple hierarchies. The situation is illustrated in Figure 2-29.

![](../img/linux-kernel-group-scheduling.png)

To reflect the hierarchical situation within the kernel, struct `sched_entity` is augmented with an element that allows for expressing this hierarchy:

```c
<sched.h>

struct sched_entity {
...
 #ifdef CONFIG_FAIR_GROUP_SCHED
		struct sched_entity *parent;
		...
 #endif
...
}
```

This substructure of scheduling entities must be considered by all scheduling-class-related operations. Consider, for instance, how the code to enqueue a task in the completely fair scheduler really looks:

```c
kernel/sched_fair.c

static void enqueue_task_fair(struct rq *rq, struct task_struct *p, int wakeup) {

 struct cfs_rq *cfs_rq;
 struct sched_entity *se = &p->se;

 for_each_sched_entity(se) {
   if (se->on_rq)
			break;
 	 cfs_rq = cfs_rq_of(se);
   enqueue_entity(cfs_rq, se, wakeup);
   wakeup = 1;
 }
}
```

`for_each_sched_entity` traverses the scheduling hierarchy defined by the parent elements of `sched_entity`, and each entity is enqueued on the run queue.

Notice that `for_each_sched_entity` will resolve to a trivial loop that executes the code contained in the loop body exactly once when support for group scheduling is not selected, so the behavior described in the previous discussion is regained.

### 内核抢占和低延迟的相关工作

Let us now turn our attention to **kernel preemption**, which allows for a smoother experience of the sys- tem, especially in multimedia environments. Closely related are low latency efforts performed by the kernel, which I will discuss afterward.



#### 内核抢占

**the scheduler is invoked before returning to user mode after system calls or at certain designated points in the kernel.** This ensures that the kernel, unlike user processes, cannot be interrupted unless it explicitly wants to be. This behavior can be problematic 导致问题 if the kernel is in the middle of a relatively long operation — this may well be the case with filesystem, or `memory-management- related` tasks. The kernel is executing on behalf of a specific process for a long amount of time, and other processes do not get to run in the meantime. This may result in deteriorating导致 system latency, which users experience as ‘‘sluggish‘‘ response. Video and audio dropouts may also occur in multimedia applications if they are denied CPU time for too long.

These problems can be resolved by compiling the kernel with support for *kernel preemption*. This allows not only userspace applications but also the kernel to be interrupted if a high-priority process has some things to do. Keep in mind that kernel preemption and preemption of userland tasks by other userland tasks are two different concepts!

**Kernel preemption** was added during the development of kernel 2.5. Although astonishingly few changes were required to make the kernel preemptible, the mechanism is not as easy to implement as preemption of tasks running in userspace.

 If the kernel cannot complete certain actions in a single operation — manipulation of data structures, for instance — *race conditions* may occur and render the system inconsistent. The same problems arise on multiprocessor systems discussed in Chapter 5.

The kernel may not be interrupted at all points. Fortunately, most of these points have already been identified by SMP implementation, and this information can be reused to implement kernel preemption. Problematic sections of the kernel that may only be accessed by one processor at a time are protected by so-called *spinlocks*: The first processor to arrive at a dangerous (also called critical) region acquires the lock, and releases the lock once the region is left again. Another processor that wants to access the region in the meantime has to wait until the first user has released the lock. Only then can it acquire the lock and enter the dangerous region.

If the kernel can be preempted, even uniprocessor systems will behave like SMP systems. **Consider that the kernel is working inside a critical region when it is preempted.** **The next task also operates in kernel mode, and unfortunately also wants to access the same critical region**. This is effectively equivalent to two processors working in the critical region at the same time and must be prevented. Every time the kernel is inside a critical region, kernel preemption must be disabled.

How does the kernel keep track of whether it can be preempted or not? Recall that each task in the system is equipped with an architecture-specific instance of struct `thread_info`. The structure also includes a *preemption counter*:

```c
<asm-arch/thread_info.h>

struct thread_info {
  ...
	int preempt_count; /* 0 => preemptable, <0 => BUG */
  ...
}
```

The value of this element determines whether the kernel is currently at a position where it may be interrupted. **If `preempt_count` is zero, the kernel can be interrupted**, otherwise not. The value must not be manipulated 修改 directly, but only with the auxiliary functions `dec_preempt_count` and `inc_preempt_count`, which, respectively, decrement and increment the counter.` inc_preempt_count` is invoked each time the kernel enters an important area where preemption is forbidden. When this area is exited, `dec_ preempt_count` decrements the value of the preemption counter by 1. Because the kernel can enter some important areas via different routes — particularly via nested routes — a simple Boolean variable would not be sufficient for `preempt_count`. When multiple dangerous regions are entered one after another, it must be made sure that *all* of them have been left before the kernel can be preempted again.

The `dec_preempt_count` and `inc_preempt_count` calls are integrated in the synchronization operations for SMP systems (see Chapter 5). They are, in any case, already present at all relevant points of the kernel so that **the preemption mechanism** can make best use of them simply by reusing the existing infrastructure.

Some more routines are provided for preemption handling:

- ❑  preempt_disable disables preemption by calling inc_preempt_count. Additionally, the com- piler is instructed to avoid certain memory optimizations that could lead to problems with the preemption mechanism.

- ❑  preempt_check_resched checks if scheduling is necessary and does so if required.

- ❑  preempt_enable enables kernel preemption, and additionally checks afterward if rescheduling

  is necessary with preempt_check_resched.

- ❑  preempt_disable_no_resched disables preemption, but does not reschedule.



How does the kernel know if preemption is required? First of all, the **TIF_NEED_RESCHED** flag must be set to signalize that a process is waiting to get CPU time. This is honored by `preempt_ check_resched`:

```c
<preempt.h>

#define preempt_check_resched() 
do { 

	if (unlikely(test_thread_flag(TIF_NEED_RESCHED)))
    preempt_schedule(); 

} while (0)
```

Recall that the function is called when preemption is re-enabled after it had been disabled, so this
 is a good time to check if a process wants to preempt the currently executing kernel code. If this
 is the case, it should be done as quickly as possible — without waiting for the next routine call of the scheduler.

The central function for the preemption mechanism is `preempt_schedule`. The simple desire that the kernel be preempted as indicated by **TIF_NEED_RESCHED** does not yet guarantee that this is *possible* — recall that the kernel could currently still be inside a critical region, and must not be disturbed. This is checked by `preempt_reschedule`:

```c
kernel/sched.c

asmlinkage void __sched preempt_schedule(void) {

struct thread_info *ti = current_thread_info();
/*
*  If there is a non-zero preempt_count or interrupts are disabled,
*  we do not want to preempt the current task. Just return.. */

if (unlikely(ti->preempt_count || irqs_disabled()))
  return;


```

If **the preemption counter** is greater than 0, then preemption is still disabled, and consequently the kernel may not be interrupted — the function terminates immediately. Neither is preemption possible if the kernel has disabled hardware IRQs at important points where processing must be completed in a single operation. irqs_disabled checks whether interrupts are disabled or not, and if they are disabled, the kernel must not be preempted.

The following steps are required if preemption is possible:

```c
kernel/sched.c

do {

	add_preempt_count(PREEMPT_ACTIVE);
  schedule();

  sub_preempt_count(PREEMPT_ACTIVE);
  /*
  * Check again in case we missed a preemption opportunity * between schedule and now.
  */

} while(unlikely(test_thread_flag(TIF_NEED_RESCHED)));
```

Before the scheduler is invoked, the value of the preemption counter is set to **PREEMPT_ACTIVE**.

This sets a flag bit in the preemption counter that has such a large value that it is never affected by the regular preemption counter increments as illustrated by Figure 2-30. It indicates to the `schedule` function that scheduling was not invoked in the normal way but as a result of a kernel preemption. 

After the kernel has rescheduled, code flow returns to the current task — possibly after some time has elapsed, because the preempting task will have run in between — the flag bit is removed again.

![](../img/linux-kernel-per-process-preemption-counter.png)



```c
kernel/sched.c
asmlinkage void __sched schedule(void) {
  ...
  if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
    if (unlikely((prev->state & TASK_INTERRUPTIBLE) && unlikely(signal_pending(prev)))) {
      prev->state = TASK_RUNNING;
    } else {
      deactivate_task(rq, prev, 1);
    }
  }
  ...
}
```



This ensures that the next task is selected as quickly as possible without the hassle of deactivating the current one. If a high-priority task is waiting to be scheduled, it will be picked by the scheduler class and will be allowed to run.

This method is only one way of triggering kernel preemption. Another possibility to activate preemption is after a hardware IRQ has been serviced. If the processor returns to kernel mode after handling the IRQ (return to user mode is not affected), the architecture-specific assembler routine checks whether the value of the preemption counter is 0 — that is, if preemption is allowed — and whether the reschedule flag

is set — exactly as in preempt_schedule. If both conditions are satisfied, the scheduler is invoked, this time via preempt_schedule_irq to indicate that the preemption request originated from IRQ context. The essential difference between this function and preempt_schedule is that preempt_schedule_irq is called with IRQs disabled to prevent recursive calls for simultaneous IRQs.

As a result of the methods described in this section, a kernel with enabled preemption is able to replace processes with more urgent ones faster than a normal kernel could.



#### 低延迟Low Latency

Naturally, the kernel is interested in providing good latency times even if kernel preemption is not enabled. This can, for instance, be important in network servers. While the overhead introduced by kernel preemption is not desired in such an environment, the kernel should nevertheless respond to important events with reasonable speed. If, for example, a network request comes in that needs to be serviced by a daemon, then this should not be overly long delayed by some database doing heavy I/O operations. I have already discussed a number of measures offered by the kernel to reduce this problem: scheduling latency in CFS and kernel preemption. Real-time mutexes as discussed in Chapter 5 also aid in solving the problem, but there is one more scheduling-related action that can help.

**Basically, long operations in the kernel should not occupy the system completely. Instead, they should check from time to time if another process has become ready to run, and thus call the scheduler to select the process. This mechanism is independent of kernel preemption and will reduce latency also if the kernel is built without explicit preemption support.**

The function to initiate conditional rescheduling is `cond_resched`. It is implemented as follows:

```c
kernel/sched.c
int __sched cond_resched(void) {
  if (need_resched() && !(preempt_count() & PREEMPT_ACTIVE))
    __cond_resched();
    return 1;
  }
  return 0;
}
```



need_resched checks if the TIF_NEED_RESCHED flag is set, and the code additionally ensures that the ker- nel is not currently being preempted already34 and rescheduling is thus allowed. Should both conditions be fulfilled, then __cond_resched takes care of the necessary details to invoke the scheduler.

How can cond_resched be used? As an example, consider the case in which the kernel reads in memory pages associated with a given memory mapping. This could be done in an endless loop that terminates after all required data have been read:

```c
for (;;)
 /* Read in data */

if (exit_condition)
	continue;
```



**If a large number of read operations is required**, this can consume a sizeable amount of time. Since the process runs in kernel space, it will not be deselected by the scheduler as in the userspace case, taken that kernel preemption is not enabled. This can be improved by calling `cond_resched` in every loop iteration:

```c
for (;;)
  cond_resched();

  /* Read in data */
  if (exit_condition)
    continue;
```

The kernel has been carefully audited to find the longest-running functions, and calls to cond_resched have been put in the appropriate places. This ensures higher responsiveness even without explicit kernel preemption.

Following a long-time tradition for Unix kernels, Linux has supported task states for both interruptible and uninterruptible sleeps. During the 2.6.25 development cycle, however, another state was added: **TASK_KILLABLE**. 

Tasks in this state are sleeping and do not react to non-fatal signals, but can — incontrast to **TASK_UNINTERRUPTIBLE** — be killed by fatal signals. At the time of writing, almost all places in the kernel that would provide apt possibilities for killable sleeps are still waiting to be converted to the new form.

The scheduler has seen a comparatively large number of cleanups during the development of kernels 2.6.25 and 2.6.26. A new feature added during this period is real-time group scheduling. This means that real-time tasks can now also be handled by the group scheduling framework introduced in this chapter.

Additionally, the scheduler documentation was moved into the dedicated directory Documentation/ scheduler/, and obsolete files documenting the old O(1) scheduler have been removed. Documentation on real-time group scheduling can be found in Documentation/scheduler/sched-rt-group.txt.

### 总结

Linux is a multiuser and multitasking operating system, and thus has to manage multiple processes from multiple users. In this chapter, you have learned that processes are a very important and fundamental abstraction of Linux. The data structure used to represent individual processes has connections with nearly every subsystem of the kernel.

You have seen how Linux implements the traditional fork/exec model inherited from Unix to create new processes that are hierarchically related to their parent, and have also been introduced to Linux- specific extensions to the traditional Unix model in the form of namespaces and the clone system call. Both allow for fine-tuning how a process perceives the system, and which resources are shared between parent and child processes. Explicit methods that enable otherwise separated processes to communicate are discussed in Chapter 5.

Additionally, you have seen how the available computational resources are distributed between pro- cesses by the scheduler. Linux supports pluggable scheduling modules, and these are used to implement completely fair and POSIX soft real-time scheduling policies. The scheduler decides when to switch between which tasks, and is augmented by architecture-specific routines to implement the context switch- ing proper.

Finally, I have discussed how the scheduler must be augmented to service systems with multiple CPUs, and how kernel preemption and low-latency modifications make Linux handle time-constrained situa- tions better.

















































