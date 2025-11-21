---
title: 技巧 | Windows 可执行内存的写入防护
date: 2024-03-18
categories:
  - "Windows Internals"
tags:
  - "Windows Internals"
comments: true
image:
  path: /assets/img/2024-03-18-executable-memory-write/cover.jpg
---

在内核中免不了要调用ZwAllocateVirtualMemory为进程申请内存，申请的内存属性一般会为 `PAGE_READWRITE` 或 `PAGE_EXECUTE_READWRITE`，而在二月份的某一天里，我的群友发现对某个游戏申请的 `PAGE_EXECUTE_READWRITE` 属性内存，是无法写入的，一旦写入会导致进程直接崩溃，无论是内核模式写入还是用户模式写入。

## ZwAllocateVirtualMemory

在Windows下，使用ZwAllocateVirtualMemory在申请成功以后并不会立即分配物理内存，而是会在首次访问虚拟地址，触发#PF异常的时候，异常处理例程将物理内存地址填充到页表，并根据VAD设置页表属性。

## ManageExecutableMemoryWrites

在1709版本以后，EPROCESS.Flags中增加了一个名为 `ManageExecutableMemoryWrites` 的标志位，这个标志位可用于管理控制对于可执行内存的写入，置位以后，ETHREAD.SameThreadApcFlags中的 `AllowWritesToExecutableMemory` 标志位开始生效。

## AllowWritesToExecutableMemory

当EPROCESS.Flags中的 `ManageExecutableMemoryWrites` 置位以后，ETHREAD.SameThreadApcFlags中的 `AllowWritesToExecutableMemory` 表示当前线程是否允许写入可执行内存。在1903版本以后，该标志位分成了两个 `AllowUserWritesToExecutableMemory` 和 `AllowKernelWritesToExecutableMemory` 。

## KiPageFault

在申请的 `PAGE_EXECUTE_READWRITE` 内存首次访问的时候，系统会判断异常地址为用户地址，然后调用MiUserFault，紧接着到达MiValidFault，在MiValidFault中判断 `ManageExecutableMemoryWrites` 如果置位，则根据CPL判断使用SameThreadApcFlags中的哪个标志位进行判断:

* 如果为 `AllowUserWritesToExecutableMemory` 则直接返回 `STATUS_EXECUTABLE_MEMORY_WRITE` 这个错误代码。
* 如果为 `AllowKernelWritesToExecutableMemory` 则去调用MiForceCrashForInvalidAccess函数。

## MiForceCrashForInvalidAccess

* 首先判断EPROCESS.Flags2的 `FatalAccessTerminationRequested` 是否置位。
* 如果置位，则根据EPROCESS.Flags3的 `SystemProcess`  来创建CrashDump。
* 然后调用ZwCreateThreadEx为进程的**NULL**地址创建一个线程来促使进程崩溃。
* 如果这个线程创建失败，则直接调用PsTerminateProcess来结束进程。