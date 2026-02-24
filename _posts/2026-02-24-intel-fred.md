---
title: 浅谈 | Intel FRED
date: 2024-02-02
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

**FRED（Flexible Return and Event Delivery）** 是 Intel 引入的新型特权级切换与事件处理架构，用于替代传统的 IDT 事件投递（IDT event delivery）和 IRET 返回机制，同时 AMD 也宣布在即将到来的 Zen6 中采用，并且微软已经在高版本的 Windows 内核中实现了 FRED 的代码，所以我认为有必要借此来简单的介绍一下。

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> 在阅读本篇内容前，需要读者对 x64 架构有一定的了解。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 配置

| 功能     | 支持                                      | 描述                   |
| -------- | ----------------------------------------- | ---------------------- |
| **FRED** | CPUID.(EAX=07H, ECX=1H):EAX.FRED [Bit 17] | 支持 FRED 指令和寄存器 |
| **LKGS** | CPUID.(EAX=07H, ECX=1H):EAX.LKGS [Bit 18] | 支持 LKGS 指令         |

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> FRED 仅适用于 64 位 OS（IA-32e 模式）。
{: .prompt-tip }
<!-- markdownlint-restore -->