---
title: ALSA Compress-Offload API
date: 2023-10-15 20:23:29
categories: Linux 内核
tags:
- Linux 内核
---

## 概述

从 ALSA API 的早期开始，它就被定义为支持 PCM，或考虑到了 IEC61937 等固定比特率的载荷。参数和返回值以帧计算是常态，这使得扩展已有的 API 以支持压缩数据流充满挑战。

最近这些年，音频数字信号处理器 (DSP) 常常被集成进片上系统 (SoC) 设计中，且 DSPs 也常被集成进音频编解码器 (这里的音频编解码器与 AAC 之类的音频数据压缩方案不同，它是指主要用于完成模拟信号和数字信号转换的器件) 中。与基于主机的处理相比，在 DSP 这样的处理器上处理压缩数据可以显著降低功耗。Linux 对这类硬件的支持不是很好，主要是因为主线内核中缺乏可用的通用 API。

不是要求更改 ALSA PCM 接口的 API 而破坏兼容性，而是引入了一个新的 “压缩数据” API，为音频 DSP 提供控制和数据流接口。

这个 API 的设计灵感来自于 Intel Moorestown SOC 的 2 年经验，通过许多必须的更正把 API 上传到主线内核而不是 staging 树中，使其可供其他人使用。

## 需求

主要的需求包括如下这些：

 * 字节计数和时间之间的分离。压缩的格式可能每个文件都有一个文件头，或者完全没有文件头。帧和帧之间的载荷大小可能会变化。因此，当处理压缩数据时，可靠地估计音频缓冲区的时长是不可能的。需要专门的机制来实现可靠的音频-视频同步，这需要精确地报告给定时间点，已经渲染的采样数。

 * 处理多种格式。PCM 数据只需要采样率、通道数和位宽的规范。相反地，压缩数据可能是各种各样的格式。音频 DSP 也可以在固件中嵌入对有限数量的音频编码器和解码器的支持，或者可以通过库的动态下载支持更多选择。

 * 聚焦于主要的格式。这个 API 可以为用于音频和视频采集和播放的最流行的格式提供支持。它可能随着音频压缩技术的进步，而添加新的格式。

 * 处理多种配置。即使对于像 AAC 这样给定的格式，一些实现可能支持 AAC 多通道而不是 HE-AAC 立体声。同样，WMA10 M3 级别可能需要许多内存和 CPU 周期。新的 API 需要提供一个通用的方式来列出这些格式。

 * 仅渲染/获取。这个 API 不提供任何硬件加速方法，其中 PCM 采样被返回给用户空间，以做更多处理。这个 API 聚焦于给 DSP 提供压缩数据流，并假设解码之后的数据被路由给一个物理输出或逻辑后端。

 * 复杂性隐藏。对于每个压缩格式，现有的用户空间多媒体框架都具有现成的枚举/结构体。这个新 API 假设有一个平台特有的兼容性层，转换并利用音频 DSP 的能力，如 Android HAL 或 PulseAudio sinks。根据构造，常规的应用程序不应该使用此 API。

## 设计

新 API 在流控制方面与 PCM API 有许多相同的概念。无论内容是什么，启动 (start)，暂停 (pause)，恢复 (resume)，排空 (drain)，和停止 (stop) 命令具有相同的语义。

内存环形缓冲区被分割为一系列片段的概念借鉴自 ALSA PCM API。然而，只能指定字节大小。

拖动/trick 模式假定由主机处理。

不支持快退/快进的概念。提交到环形缓冲区的数据不能失效，除非删除所有缓冲区。

压缩数据 API 对于数据如何被提交给音频 DSP 不做任何假设。从主存储器传输到嵌入式音频集群或外部 DSP 的 SPI 接口的 DMA 传输都是可能的。与在 ALSA PCM 的情况中一样，暴露了一组核心例程；每个驱动程序实现者都必须编写对一组强制例程的支持，并可能使用可选例程。

主要补充内容是

**get_caps**

这个例程返回支持的音频格式列表。在采集流上查询 codecs 将返回编码器，对播放流则将列出解码器。

**get_codec_caps**

对于每个 codec，这个例程返回能力 (capabilities) 的列表。这个例程的意图是确保所有的能力都对应于有效的设置，并最小化配置失败的风险。例如，对于诸如 AAC 之类的复杂编解码器，支持的通道数可能取决于特定的配置 (profile)。如果通过单个描述符来暴露能力 (capabilities)，可能发生配置 (profile)/通道数/格式的特定结合无法支持的情况。同样，嵌入式 DSP 的内存和 CPU 周期有限，某些实现可能会使能力 (capabilities) 列表变得动态并依赖于现有工作负载。除了编解码器设置之外，此例程还返回实现处理的最小缓冲区大小。该信息可以是 DMA 缓冲区大小、同步所需的字节数等的函数，并且可以由用户空间使用来定义在开始播放之前需要在环形缓冲区中写入多少内容。

**set_params**

这个例程设置为特定编解码器选择的配置。参数中最重要的字段是编解码器类型；在大多数情况下解码器将忽略其它参数，而编码器将与设置保持严格一致。

**get_params**

这个例程返回 DSP 使用的实际设置。对设置的更改应该仍然是例外。

**get_timestamp**

时间戳变成多字段结构。它列出了传输的字节数、处理的采样数以及渲染/采集的采样数。所有这些值均可用于确定平均比特率、确定环形缓冲区是否需要重新填充或由于 DSP 上的解码/编码/IO 导致的延迟。

请注意，编解码器/配置 (profile)/模式列表源自 OpenMAX AL 规范，而不是重新发明轮子。修改包括：

 - 添加 FLAC 和 IEC 格式
 - 编码器/解码器能力 (capabilities) 的合并
 - 配置 (profile)/模式列表为位掩码，使描述符更加紧凑
 - 为解码器添加 **set_params** (在 OpenMAX AL 中缺失)
 - 添加 AMR/AMR-WB 编码模式 (在 OpenMAX AL 中缺失)
 - 为 WMA 添加格式信息
 - 需要时添加编码选项 (源自 OpenMAX AL)
 - 添加 rateControlSupported (在 OpenMAX AL 中缺失)

## 状态机

压缩音频流状态机描述如下
```
                                      +----------+
                                      |          |
                                      |   OPEN   |
                                      |          |
                                      +----------+
                                           |
                                           |
                                           | compr_set_params()
                                           |
                                           v
       compr_free()                  +----------+
+------------------------------------|          |
|                                    |   SETUP  |
|          +-------------------------|          |<-------------------------+
|          |       compr_write()     +----------+                          |
|          |                              ^                                |
|          |                              | compr_drain_notify()           |
|          |                              |        or                      |
|          |                              |     compr_stop()               |
|          |                              |                                |
|          |                         +----------+                          |
|          |                         |          |                          |
|          |                         |   DRAIN  |                          |
|          |                         |          |                          |
|          |                         +----------+                          |
|          |                              ^                                |
|          |                              |                                |
|          |                              | compr_drain()                  |
|          |                              |                                |
|          v                              |                                |
|    +----------+                    +----------+                          |
|    |          |    compr_start()   |          |        compr_stop()      |
|    | PREPARE  |------------------->|  RUNNING |--------------------------+
|    |          |                    |          |                          |
|    +----------+                    +----------+                          |
|          |                            |    ^                             |
|          |compr_free()                |    |                             |
|          |              compr_pause() |    | compr_resume()              |
|          |                            |    |                             |
|          v                            v    |                             |
|    +----------+                   +----------+                           |
|    |          |                   |          |         compr_stop()      |
+--->|   FREE   |                   |  PAUSE   |---------------------------+
     |          |                   |          |
     +----------+                   +----------+
```

## 无缝播放

当播放唱片时，解码器能够跳过编码器延迟和填充，并直接从一个曲目内容移动到另一个曲目内容。最终用户可以将其视为无缝播放，因为我们在从一个曲目切换到另一个曲目时没有静音。

此外，由于编码可能会产生低强度噪声。所有类型的压缩数据都很难达到完美的无缝效果，但对于大多数音乐内容来说效果很好。解码器需要知道编码器延迟和编码器填充。所以我们需要将其传递给 DSP。该元数据是从 ID3/MP4 头中提取的，默认情况下不存在于比特流中，因此需要一个新的接口来将此信息传递给 DSP。此外，DSP 和用户空间需要从一个曲目切换到另一个曲目，并开始使用第二个曲目的数据。

主要补充内容是：

**set_metadata**

该例程设置编码器延迟和编码器填充。解码器可以使用它来去除静音。这需要在写入曲目中的数据之前进行设置。

**set_next_track**

该例程告诉 DSP，在此之后发送的元数据和写入操作将对应于后续曲目。

**partial_drain**

当到达文件末尾时调用此函数。用户空间可以通知 DSP 已达到 EOF，现在 DSP 可以开始跳过填充延迟。下一次写入数据也将属于下一个曲目。

无缝播放的顺序流程为：

 - 打开
 - 获得能力 (caps)/编解码器能力 (caps)
 - 设置参数
 - 设置第一首曲目的元数据
 - 填充第一首曲目的数据
 - 触发启动
 - 用户空间结束所有的发送，
 - 通过发送 **set_next_track** 指示下一首曲目的数据，
 - 设置下一首曲目的元数据
 - 然后调用 **partial_drain** 刷新 DSP 中缓冲区的大部分
 - 填充下一首曲目的数据
 - DSP 切换到第二首曲目

(注意：**partial_drain** 和写入下一首曲目的数据的顺序也可以反过来)

## 无缝播放状态机

对于无缝播放，我们从运行状态转移到部分耗尽状态并返回，同时设置元数据和下一首曲目的信号

```
                          +----------+
  compr_drain_notify()    |          |
+------------------------>|  RUNNING |
|                         |          |
|                         +----------+
|                              |
|                              |
|                              | compr_next_track()
|                              |
|                              V
|                         +----------+
|    compr_set_params()   |          |
|             +-----------|NEXT_TRACK|
|             |           |          |
|             |           +--+-------+
|             |              | |
|             +--------------+ |
|                              |
|                              | compr_partial_drain()
|                              |
|                              V
|                         +----------+
|                         |          |
+------------------------ | PARTIAL_ |
                          |  DRAIN   |
                          +----------+
```

## 不支持

 * 支持 VoIP/电路交换呼叫不是此 API 的目标。支持动态比特率变化需要 DSP 和主机堆栈之间的紧密耦合，从而限制了节能。

 * 不支持丢包隐藏。这将需要一个额外的接口，以便解码器在传输过程中丢失帧时合成数据。将来可能会添加此功能。

 * 这个 API 不处理音量控制/路由。公开压缩数据接口的设备将被视为常规 ALSA 设备；改变音量和路由信息将通过常规 ALSA kcontrol 提供。

 * 嵌入式音效。无论输入是 PCM 还是压缩的，都应以相同的方式启用此类音效。

 * 多通道 IEC 编码。不清楚是否需要这样做。

 * 如上所述，不支持编码/解码加速。可以将解码器的输出路由到采集流，甚至实现转码功能。此路由将通过 ALSA kcontrol 启用。

 * 音频策略/资源管理。该 API 不提供任何挂钩来查询音频 DSP 的利用率，也不提供任何抢占机制。

 * 没有 underrun/overrun 的概念。由于写入的字节本质上是压缩的，并且写入/读取的数据不会及时直接转换为渲染输出，因此这不会处理 underrun/overrun 问题，可能会在用户库中处理

## 作者

 * Mark Brown 和 Liam Girdwood 讨论了此 API 的需求
 * Harsha Priya 在 intel_sst 压缩 API 方面的工作
 * Rakesh Ughreja 提供了宝贵的反馈
 * Sing Nallasellan、Sikkandar Madar 和 Prasanna Samaga 在真实平台上演示并量化了音频卸载的优势。

[原文](https://docs.kernel.org/sound/designs/compress-offload.html)

Done.