---
title: Minix IPC
date: 2019-12-09 14:16:22
tags: minix
---

# Minix kernel, services and user tasks
Minix services:
* rs: Reincarnation Server
* ds: Data Store
* ipc: shared memory and semaphore
* sched: scheduler
* vfs: Virtual File Systems
* vm: Virtual Memory
* is: System Information Service
* mib: Manage Information Base
* pm: Process Manager
* drivers, filesystems, ...

![minix-procs](/img/minix_procs.svg)

# kernel IPC
Minix kernel provides interface for IPC and kernel call. System services can do kernel IPC and kernel calls, but user process can only do kernel IPC.
Kernel IPC delivers 64 bytes message between processes.
* send: if receiver is not receiving, it will block
* receive: if no messages is sending, it will block
* sendrec: send and receive
* sendnb: non-blocking send
* senda: async send
* notify: no message

# kernel safecopy
Copying data between processes is done by `safecopy` kernel call.
* Process A grants some memory in its own address space to process B
* Process B calls `safecopy` to read/write that memory

Or,
* Process C grants some memory in process A's address space to process B
* Process B calls `safecopy` to read/write that memory

`safecopy` first verifies the grants, and then creates temporary mapping and copy the data.

# example: `sys_read`
- user process calls `sys_read`
  `sys_read` constructs read request message, and send that message to `vfs` by kernel IPC (`ipc_sendrec`).

- `vfs` sends read request to `mfs`
  `vfs` receives the message from user, grants user buffer to `mfs`, and sends read request to `mfs`

- `mfs` reads data from block device driver and writes data to user buffer using `safecopy`

![minix-procs](/img/minix_sys_read.svg)

# shared memory and semaphore
Service `ipc` manages shared mamory and semaphore.
