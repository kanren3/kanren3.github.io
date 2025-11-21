---
title: 浅谈 | Intel LASS
date: 2024-11-18
categories:
  - "Intel"
tags:
  - "Intel"
comments: true
image:
  path: ../assets/img/2024-11-18-intel-lass/cover.jpg
---

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> 在阅读本篇内容前，需要读者对 x64 架构有基本的了解。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 引言

**Intel LASS** 是在 **Arrow Lake** 引入的内存隔离防护机制，它的作用有些类似 **SMAP** 和 **SMEP**，可以用来控制内存访问权限，本篇将简单说明一下它的用途与基本用法。

## 访问权限

在早期的 x64 分页模式下，内存的访问权限由页表项中的 **U/S** 位控制：
- 复位状态下，代表当前页面属于 **User Mode**。
- 置位状态下，代表当前页面属于 **Supervisor Mode**。

当 **CR4.SMAP** 置位的时候，**Supervisor Mode** 是否可以访问 **User Mode** 页面，取决于 **EFLAGS.AC** 标志位：

- 复位状态下，完全不可以访问。
- 置位状态下，可以显式访问，**但是依旧不可以隐式访问**。

当 **CR4.SMEP** 置位的时候，**Supervisor Mode** 永远不可以对 **User Mode** 页面进行取指操作。

## 侧信道攻击

 **SMAP** 和 **SMEP** 虽然保证了内存的访问权限，但是仍然无法逃避掉 **Page Walk** ，而这个过程中，攻击者可以通过使用大量的内存访问操作，通过统计时间信息，来间接的分析获得页表的分页结构等信息，于是便诞生了 **Intel LASS**。

## LASS

**LASS** 全称为 **Linear Address Space Separation**，它的作用便是在 **Page Walk** 之前，就对虚拟地址预先进行检测，如果检测未通过则直接产生异常，这也就避免了刚刚我们提到的侧信道攻击。

想要使用 **LASS**，我们首先需要通过检测 **CPUID.(EAX=07H,ECX=1):EAX.LASS[bit 6]** 来判断当前处理器是否支持此功能，然后便是通过 **CR4.LASS** 来控制 **LASS** 的开启和关闭了。

当 **LASS** 启用以后，是否可以使用此虚拟地址寻址，是由虚拟地址的**最高的有效位**来决定的，而曾经 **Page Walk** 中越权所产生的 **#PF** 异常会转变为 **#GP** 和 **#SS**，操作系统有义务添加对于这种异常的处理，并且在寻址前 **CR4.SMAP** 与 **CR4.SMEP** 的作用也发生了一些微小的变化：

- **CR4.SMEP** 无论是否置位，一律会视为置位状态，**Supervisor Mode** 永远不可以对**最高的有效位**为 0 的虚拟地址进行取指。
- **CR4.SMAP** 则可以控制 **Supervisor Mode** 是否可以对**最高的有效位**为 0 的虚拟地址进行寻址访问。
  - 复位状态下，完全不可以进行寻址访问。
  - 置位状态下，和曾经一样，通过 **EFLAGS.AC** 标志来决定是否可以进行显性的寻址访问。

而 **User Mode** 则是永远无法对**最高的有效位**为 1 的虚拟地址进行访问，理论上这永远不可能。

## LAM

**LAM** 全称为 **Linear Address Masking**，本篇不重点讲此功能，略微介绍一下。

在启用此功能后，曾经的 **地址规范检查 (Canonicality Checking)** 将不再生效，虚拟地址没有用到的高位可以由操作系统用来存放任意元数据：

- 4 级页表下是高 15 位，由 **CR3.LAM_U48** 控制启用。
- 5 级页表下是高 6 位，由 **CR3.LAM_U57** 控制启用。
- **CR4.LAM_SUP** 控制是否也为 **Supervisor Mode Address** 启用。

这也是我前面会给虚拟地址的**最高有效地位**加粗的主要原因。

## 结尾

本篇内容参考最新的 **Intel SDM**，手头没有合适的处理器进行实验，如有错误请指正。