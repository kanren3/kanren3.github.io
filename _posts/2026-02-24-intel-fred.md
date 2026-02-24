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

**FRED（Flexible Return and Event Delivery）** 是 Intel 引入的新型特权级切换与事件处理架构，用于替代传统的 IDT 事件投递（IDT event delivery）和 IRET 返回机制，同时 AMD 也宣布在即将到来的 Zen6 中采用，并且微软已经在高版本的 Windows 内核中实现了 FRED 的代码，所以我认为有必要借此来简单的介绍一下。

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> FRED 仅适用于 64 位操作系统（IA-32e Mode），在阅读本篇内容前，需要读者对 x64 架构有一定的了解。
{: .prompt-tip }
<!-- markdownlint-restore -->

## 枚举

| 功能     | 支持                                      | 描述                   |
| -------- | ----------------------------------------- | ---------------------- |
| **FRED** | CPUID.(EAX=07H, ECX=1H):EAX.FRED [Bit 17] | 支持 FRED 指令和寄存器 |
| **LKGS** | CPUID.(EAX=07H, ECX=1H):EAX.LKGS [Bit 18] | 支持 LKGS 指令         |

在高版本 Windows 中，初始化内核阶段会调用到 `KiInitializeBootStructures`， 这个函数内部首先会调用 `KiSetProcessorSignature`，内部会调用到`RtlDetectProcessorFeatures`，这个函数会根据 `KiCpuFeatureTable` 来枚举支持的功能，通过 AI 可以轻松的分析出结构体的大致用途：

![](../assets/img/2026-02-24-intel-fred/1.png)

------

```
KI_CPU_FEATURE_ENTRY <7, 1, 20000h, 0, 14h, 0, 4000000000h, 0> [FRED]
KI_CPU_FEATURE_ENTRY <7, 1, 40000h, 0, 14h, 0, 8000000000h, 0> [LKGS]
```

我们可以直接从表中找到这两项，只有当 CPU 同时支持这两个功能的时候，`KiFredEnabled` 才会被设置。
