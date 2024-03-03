---
title: "Live555中RTP包的打包与发送过程分析"
date: 2024-03-02T21:39:00+08:00
categories:
- Media
tags:
- live555
- RTSP
- RTP
- H264
draft: false
---


这里主要分析一下，live555中关于RTP打包发送的部分。在处理完PLAY命令之后，就开始发送RTP数据包了(其实在发送PLAY命令的response包之前，就会发送一个RTP包，这里传输就已经开始了)

先介绍下主要的流程：RTP包的发送是从MediaSink::startPlaying函数调用开始的，在StartPlaying函数的最后会调用函数continuePlaying。

continuePlaying函数是定义在MediaSink类中的纯虚函数，需要到特定媒体的sink子类中实现，对于H264来讲是在H264VideoRTPSink中实现的。H264VideoRTPSink：continuePlaying中会创建一个辅助类H264FUAFragmenter，用于H264的RTP打包。因为H264的RTP包，有些特殊需要进一步处理。接着调用其父类MultiFramedRTPSink::continuePlaying实现。

在函数MultiFramedRTPSink::continuePlaying中主要是调MultiFramedRTPSink::buildAndSendPacket函数。

看名字就知道MultiFramedRTPSink::buildAndSendPacket()函数将完成打包并发送工作。传递了一个True参数，表示这是第一个packet。在该函数的最后调用了MultiFramedRTPSink::packFrame() ，进一步的工作，在packFrame函数中进行,

MultiFramedRTPSink::packFrame() 将为RTP包填充数据。packFrame函数需要处理两种情况：
1).buffer中存在未发送的数据(overflow data)，这时可以将调用afterGettingFrame1函数进行后续处理工作。2).buffer不存在数据，这时需要调用source上的getNextFrame函数获取数据。getNextFrame调用时，参数中有两个回调用函数：afterGettingFrame函数将在获取到数据后调用，其中只是简单的调用了afterGettingFrame1函数而已；ourHandleClosure函数将在数据已经处理完毕时调用，如文件结束。可以看出，两种情况下最后都是要调用afterGettingFrame1函数，

afterGettingFrame1函数的复杂之处在于处理frame的分片，若一个frame大于TCP/UDP有效载荷(程序中定义为1448个字节)，就必需分片了。最简单的情况就是一个packet(RTP包)中最多只充许一个frame，即一个RTP包中存在一个frame或者frame的一个分片，H264就是这样处理的：方法是将剩余的数据记录为buffer的溢出部分。下次调用packFrame函数时，直接从溢出部分复制到packet中。不过应该注意，一个frame的大小不能超过buffer的大小(默认为60000)，否则会真的溢出, 那就应该考虑增加buffer大小了。

处理完相关分片信息，将会调用函数MultiFramedRTPSink::sendPacketIfNecessary()来发送数据包。sendPacketIfNecessary()中函数处理一些发送的细节，如packet中含有Frame则调用RTPInterface::sendPacket函数发送packet。并将下一次RTP的发送操作加入到任务调度器中：nextTask() = envir().taskScheduler().

scheduleDelayedTask。函数参数中传递了sendNext函数指针。sendNext函数又调用了buildAndSendPacket函数，轮回了。。。！

最后来看下RTPInterface::sendPacket函数。若是使用UDP方式发送，将调用Groupsock::output函数，可以实现组播功能。groupsock只实现了UDP发送功能，当用TCP方式传送时调用sendRTPOverTcP函数，这个函数中直接调用socket的send函数。至此关于RTP打包发送的主要流程分析就差不多了，具体细节实现，可参考源代码。


<br/>

**原文地址：**

[Live555中RTP包的打包与发送过程分析](https://www.cnblogs.com/lidabo/p/4388666.html)