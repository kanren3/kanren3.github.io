---
title: 浅谈 | Intel FRED
date: 2026-02-24
categories:
  - "Intel"
tags:
  - "Intel"
  - "AMD"
  - "Windows Internals"
comments: true
image:
  path: ../assets/img/2026-02-24-intel-fred/cover.jpg
  lqip: ../assets/img/2026-02-24-intel-fred/cover-lqip.jpg
---

## 概述

**FRED（Flexible Return and Event Delivery）** 是 Intel 引入的新型特权级切换与事件处理架构，用于替代传统的 IDT 事件投递（IDT event delivery）和 IRET 返回机制，同时 AMD 也宣布在即将到来的 Zen6 中采用此功能，所以我认为有必要借此来简单的介绍一下。

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> 在阅读本篇内容前，需要读者对 x64 架构有一定的了解。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 枚举

| 功能 | 支持                                      | 描述                   |
| ---- | ----------------------------------------- | ---------------------- |
| FRED | CPUID.(EAX=07H, ECX=1H):EAX.FRED [Bit 17] | 支持 FRED 指令和寄存器 |
| LKGS | CPUID.(EAX=07H, ECX=1H):EAX.LKGS [Bit 18] | 支持 LKGS 指令         |

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> 这是两个独立的功能，任何支持 FRED 的处理器都将支持 LKGS。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 启用

操作系统可以通过设置 **CR4.FRED[bit 32]** 来启用 FRED，启用后，传统 **IDT**、**SYSCALL**、**SYSENTER** 事件，都将统一转换成 **FRED** 事件。

> 开启与否，并不会影响 **LKGS** 指令，也不会影响 **RDMSR** 和 **WRMSR** 对于 **FRED MSR** 的访问。

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> FRED 仅适用于 64 位操作系统（IA-32e 模式），而 AMD 在设计之初取消了 IA-32e 模式下的 SYSENTER 指令，所以猜测未来 FRED 事件中也不会存在 SYSENTER。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 配置

| FRED MSR          | 地址        | 功能                         |
| ----------------- | ----------- | ---------------------------- |
| IA32_FRED_CONFIG  | 1D4H        | 配置 FRED 的功能             |
| IA32_FRED_STKLVLS | 1D0H        | 异常向量各自的最低栈级别     |
| IA32_FRED_RSPn    | 1CCH - 1CFH | n= 0 - 3，各栈级别对应的 RSP |
| IA32_FRED_SSPn    | 1D1H - 1D3H | n= 1 - 3，各栈级别对应的 SSP |
| IA32_FRED_SSP0    | 6A4H        | 复用曾经的 IA32_PL0_SSP      |

- **IA32_FRED_CONFIG**：
  - **Bits 1:0**：当前栈级别（CSL）。
  
  - **Bit 3**：设置以后，如果事件传递不更改堆栈，则应将影子堆栈指针 (SSP) 递减 8。
  
  - **Bits 8:6**：是不换栈时 RSP 递减量。
  
  - **Bits 10:9**：是 CPL=0 时可屏蔽中断的栈级别。
  
  - **Bits 63:12**：事件处理入口 RIP（4K对齐）。
  
- **IA32_FRED_STKLVLS**：
  - 代表的是 32 个异常向量在 CPL=0 时使用的栈级别，每个向量占 2 位。
  
- **IA32_FRED_RSPn**：
  - 代表的是每个栈级别对应的 RSP。
  
- **IA32_FRED_SSPn**：
  - 代表的是每个栈级别对应的 SSP。

------

FRED 统一了所有事件的入口，并将它们分为两个，分别来处理用户态事件与内核态事件，并通过不同的指令来返回：

| 入口地址                         | 来源    | 描述                     |
| -------------------------------- | ------- | ------------------------ |
| IA32_FRED_CONFIG & ~FFFH         | CPL = 3 | 使用 ERETU（返回用户态） |
| (IA32_FRED_CONFIG & ~FFFH) + 256 | CPL = 0 | 使用 ERETS（返回内核态） |
