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

**FRED（Flexible Return and Event Delivery）** 是 Intel 引入的新型特权级切换与事件处理架构，用于替代传统的 IDT 事件投递（IDT event delivery）和 IRET 返回机制，同时 AMD 也宣布在即将到来的 Zen6 中采用，而微软已经在高版本的 Windows 内核中实现了部分 FRED 的代码，所以我认为有必要借此来简单的介绍一下。

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

> 这是两个相关但彼此独立的功能，任何支持 **FRED** 的处理器都将支持 **LKGS**。
{: .prompt-tip }
<!-- markdownlint-restore -->

在高版本 Windows 中，系统会在初始化内核之前调用 `RtlDetectProcessorFeatures` 枚举支持的功能，调用路径如下：

- KiSystemStartup
  - KiInitializeBootStructures
    - KiSetProcessorSignature
      - RtlDetectProcessorFeatures

`RtlDetectProcessorFeatures` 内部会根据 `KiCpuFeatureTable` 来枚举支持的功能，然后将结果 `FeatureBits` 和 `FeatureBits2` 分别存放到 `Prcb->FeatureBits` 和全局变量 `KeFeatureBits2`，我们可以在表项中找到以下两项：

```
KI_CPU_FEATURE_ENTRY <7, 1, 20000h, 0, 14h, 0, 4000000000h, 0> [FRED]
KI_CPU_FEATURE_ENTRY <7, 1, 40000h, 0, 14h, 0, 8000000000h, 0> [LKGS]
```

其中 **FRED** 在 `KeFeatureBits2` 中对应的掩码是 `4000000000h`，**LKGS** 对应的是 `8000000000h`，当系统检测到处理器同时支持这两个功能的时候，会将全局变量 `KiTrapFeatures` 位或 `2`，同时会将 `KiFredEnabled` 设置为 `1`。

## 启用

操作系统可以通过设置 **CR4.FRED[bit 32]** 来启用这个 **FRED**，它的值并不会影响 **LKGS** 指令，以及 **RDMSR** 和 **WRMSR** 对于 **FRED MSR** 的访问。

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> FRED 仅适用于 64 位操作系统（IA32_EFER.LMA =1）。
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

