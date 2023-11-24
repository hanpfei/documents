---
title: ASoC 插孔检测
date: 2023-04-29 19:43:29
categories: Linux 内核
tags:
- Linux 内核
---

ALSA 有一个标准的 API，用来向用户空间表示物理插孔，其内核端可以在 `include/sound/jack.h` 中看到。ASoC 提供了该 API 的一个版本，并增加了两个附加功能：

 - 它允许多个插孔探测方法一起工作于一个用户可见的插孔上。在嵌入式系统中，在单个插孔上表示多个接口，但由硬件的单独位处理是很常见的。

 - 与 DAPM 集成，允许基于探测到的插孔状态自动更新 DAPM 端点 (比如，如果没有耳机则关闭耳机输出)。

这通过把插孔处理分成三件一起工作的事情来完成：插孔本身由一个 `struct snd_soc_jack` 表示，`snd_soc_jack_pins` 集合表示要更新的 DAPM 端点，以及代码块提供插孔报告机制。

比如，系统可以有一个立体声耳机插孔，该插孔具有两种报告机制，一个用于耳机，一个用于麦克风。有些系统在连接耳机时无法使用它们的扬声器输出，因此需要确保在耳机插孔状态变化时更新扬声器和耳机。

## 插孔 - struct snd_soc_jack

它表示系统上的一个物理插孔，即是用户空间可见的东西。插孔本身完全是被动的，它由机器驱动程序设置，并由插孔检测方法更新。

插孔由机器驱动程序调用 `snd_soc_jack_new()` 创建。

## snd_soc_jack_pin

这些表示要根据插孔支持的某些状态位更新的 DAPM 针脚。每个 `snd_soc_jack` 有 0 个或多个将被自动更新的这种东西。它们由机器驱动程序创建，并使用 `snd_soc_jack_add_pins()` 和插孔关联起来。如果需要，可以将端点的状态配置为与插孔的状态相反 (比如，如果没有通过插孔连接麦克风，则启用内置麦克风)。

## 插孔探测方法

实际的插孔检测由能够监视某些系统输入，并通过调用 `snd_soc_jack_report()`，指定要更新的位的子集，更新插孔的代码完成。插孔检测代码应该由机器驱动程序设置，获取插孔更新的配置，以及当插孔连接时报告的事情的集合。

这常常基于 GPIO 的状态完成 - 它的处理程序由 `snd_soc_jack_add_gpio()` 函数提供。也可以使用其它方法，比如集成进 CODEC。CODEC 集成插孔探测的一个例子是 WM8350 驱动程序。

每个插孔可以具有多个报告机制，但至少需要一个有用。

## 机器驱动程序

这些都由机器驱动程序根据系统硬件连接在一起。机器驱动程序将设置 snd_soc_jack 和要更新的针脚的列表，然后设置一个或多个插孔探测机制来根据它们的当前状态更新该针脚。

[原文](linux-kernel/Documentation/sound/soc/jack.rst)

Done.