---
title: 浅谈 | Windows VBS 无法禁用的原因
date: 2025-11-21
categories:
  - "Windows Internals"
tags:
  - "Windows Internals"
comments: true
image:
  path: /assets/img/2025-11-21-vbs-disable-failure-reason/cover.jpg
---

## 引言

在某些追求极致游戏性能的用户眼里，微软的 **Virtualization-based Security (VBS)** 是一个讨厌的东西，它依赖于 **Microsoft Hypervisor (MSHV)**，所以以前禁用它的方式就是修改引导配置 `bcdedit /set hypervisorlaunchtype off`，但在某次更新以后，大家发现这个方法无效了。

## 原理

在探寻为何会禁用失败之前，我们需要先了解禁用的原理，也就是从 `bcdedit /set hypervisorlaunchtype off` 下手，所以我们首先需要使用 **IDA** 打开 低版本 Windows 中的 `bcdedit.exe`，搜索到关键字符串 `hypervisorlaunchtype`，然后查看引用：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/1.png)

------

然后我们查看交叉引用，发现处于 `BcdElements` 范围中，在添加结构体并重新解析以后，得到了以下数据：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/2.png)

------

其中 `250000F0h` 代表的就是这个引导项数据的类型标识符，现在我们需要打开 `winload.efi`，去通过搜索立即数的方式找到何时使用到了这个数据：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/3.png)

------

通过分析函数名和函数的使用情况，最后确定是 `OslGetHypervisorLaunchType` 是决定是否加载 **MSHV** 的关键因素：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/4.png)

------

如下所示，当 `OslGetHypervisorLaunchType` 调用成功，返回的 `HypervisorLaunchType` 为 1 的时候，便会去调用 `HvlpLoadHypervisor`：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/5.png)

------

由此，我们便可以知晓为何在低版本 Windows 中，使用 `bcdedit /set hypervisorlaunchtype off` 可以成功的禁用掉 **VBS**。

## 差异

那么是什么原因导致的在新版本中 `bcdedit /set hypervisorlaunchtype off` 不生效呢？我们打开新版本的 `winload.efi`，然后转到 `OslGetHypervisorLaunchType` 函数：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/7.png)

------

如图所示，在 `BlVsmpSystemPolicy[bit 48]` 处于置位状态下时，将无视 `hypervisorlaunchtype` 的值，强制加载 **MSHV**，`BlVsmpSystemPolicy` 是从 `EFI Variable` 同步而来，名为 `VbsPolicy`，是系统默认的引导策略，无法修改：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/8.png)

------

但在微软提供的文档里，有一些方法可以对某些 `EFI Variable` 进行控制，例如：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/9.png)

------

如图所示，将 `SecConfig.efi` 作为引导项，即可通过一些选项来控制某些 `EFI Variable`，我们可以打开它，来查看支持哪些选项：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/10.png)

------

表中存放的是选项名字，以及它们所对应的 `Flags`，后续会根据 `Flags` 去进行功能的调度：

![](/assets/img/2025-11-21-vbs-disable-failure-reason/11.png)

------

假如我添加选项 `DISABLE-VBS`，那么对应的 `Flag` 是 `20h`，进而会去执行 `DisableConfig`，传参为 `2`，执行以后， `EFI Variable` 中的 `VbsPolicy` 便会被修改为禁用状态，此时我们就可以继续使用 `hypervisorlaunchtype` 来控制是否启用 **MSHV** 了。

## 结语

微软一直在想方设法推广 **MSHV**，并且种种迹象表明正在收紧用户对于操作系统的控制权，这种现象有利有弊，从个人的角度来看，我还是希望这种东西是可以由用户控制的。

