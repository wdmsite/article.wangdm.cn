---
title: "Live555中RTSP连接建立以及请求消息处理过程"
date: 2024-03-02T22:05:00+08:00
categories:
- Media
tags:
- live555
- RTSP
draft: false
---


## 1，RTSP连接的建立过程

RTSPServer类用于构建一个RTSP服务器，该类同时在其内部定义了一个RTSPClientSession类，用于处理单独的客户会话。

首先创建RTSP服务器(具体实现类是DynamicRTSPServer)，在创建过程中，先建立 Socket(ourSocket)在TCP的554端口进行监听，然后把连接处理函数句柄(RTSPServer:: incomingConnectionHandler)和socket句柄传给任务调度器(taskScheduler)。

任务调度器把socket句柄放入后面select调用中用到的socket句柄集(fReadSet)中，同时将socket句柄和incomingConnectionHandler句柄关联起来。接着，主程序开始进入任务调度器的主循环（doEventLoop），在主循环中调用系统函数select阻塞，等待网络连接。

当RTSP客户端输入(rtsp://192.168.1.109/1.mpg)连接服务器时，select返回对应的scoket，进而根据前面保存的对应关系，可找到对应处理函数句柄，这里就是前面提到的incomingConnectionHandler了。在incomingConnectionHandler中创建了RTSPClientSession，开始对这个客户端的会话进行处理。

具体分析如下：

DynamicRTSPServer::creatnew()：

### 1.调用继承自RTPSever::setUpOurSocket：

1. 调用GroupsockHelper 的setupStreamSocket创建一个socket连接，并绑定，
2. 设置socket的发送缓存大小，
3. 调用listen开始监听端口，设置同时最大能处理连接数LISTEN_BACKLOG_SIZE=20，如果达到这个上限则client端将收到ECONNERREFUSED的错误
4. 测试绑定端口是否为0，为0的话重新绑定断口，并返回系统自己选择的新的端口。
5. 返回建立的socket文件描述符

### 2.调用自己和RTPSever的构造函数：
   
RTPSever构造函数:
1. 用一个UsageEnvironment对象的引用构造其父类Medium
2. 设置最大等待回收连接时间reclamationTestSeconds，超过这个时间从客户端没有RTSP命令或者RTSP的RR包则收回其RTSPClientSession
3. 建立一个HashTable（实际上是一个BasicHashTable）,fServerMediaSessions指向这个表。
4. 调用参数UsageEnvironment对象env的成员,一个TaskScheduler指针所指对象（实际就是一个BasicTaskScheduler对象）的成员函数turnOnBackgroundReadHandling()：
1.调用一个HandlerSet::assignHandler()创建一个Handler，把socketNum【此处为服务器监听的socket描述符】和处理函数RTSPServer::incomingConnectionHandler()，还有指向RTSPSever的指针绑定在一起。
incomingConnectionHandler作用：
    1. 调用accept返回服务器与客户端连接的socket描述符
    2. 设置客户端描述符为非阻塞
    3. 增加客户端socket描述符的发送缓存为50*1024
    4. 为此客户端随机分配一个sessionId
    5. 用客户端socket描述符clientSocket,sessionId,和客户端地址clientAddr调用creatNewClientSession创建一个clientSession。


## 2，请求消息处理过程
上节我们谈到RTSP服务器收到客户端的连接请求，建立了RTSPClientSession类，处理单独的客户会话。在建立 RTSPClientSession的过程中，将新建立的socket句柄（clientSocket）和RTSP请求处理函数句柄RTSPClientSession::incomingRequestHandler传给任务调度器，由任务调度器对两者进行一对一关联。当客户端发出 RTSP请求后，服务器主循环中的select调用返回，根据socket句柄找到对应的incomingRequestHandler，开始消息处理。先进行消息的解析。

RTSPClientSession::RTSPClientSession()构造函数：
1. 重置请求缓存
2. 调用envir().taskScheduler().turnOnBackgroundReadHandling()，这次socketnumber为客户端socket描述符这次的处理函数是RTSPServer::RTSPClientSession::incomingRequestHandler()

RTSPServer::RTSPClientSession::incomingRequestHandler():
调用handleAlternativeRequestByte1(uint8_t requestByte)：
1.fRequestBuffer[fRequestBytesAlreadySeen] =requestByte;把请求字符放入请求缓存fRequestBuffer
2.调用handleRequestBytes(1) 处理请求缓存
handleRequestBytes(int newBytesRead):
1.调用noteLiveness()查看请求是否到期,如果服务器的reclamationTestSeconds> 0，调用taskScheduler对象的rescheduleDelayedTask函数: 参数为(fLivenessCheckTask,fOurServer.fReclamationTestSeconds*1000000,(TaskFunc*)livenessTimeoutTask, this)

其中livenessTimeoutTask()函数作用是删除new出来的clientSession.
1. 调用unscheduleDelayedTask(TaskToken&prevTask)：从DelayQueue中删除prevTask项， prevTask置空.
2. 调用scheduleDelayedTask(int64_t microseconds, TaskFunc* proc, void*clientData)： 

    1.创建一个DelayInterval对象timeToDelay，用microseconds初始化。
    2.创建一个AlarmHandler对象，用proc, clientData, timeToDelay初始化
    3.调用fDelayQueue.addEntry(),把这个AlarmHandler对象加入到延迟队列中
    4.返回AlarmHandler对象的token[long类型]的指针

2. 如果请求的的长度超过请求缓存可读长度fRequestBufferBytesLeft，结束这个连接。
3. 找到请求消息的结尾：。
4. 如果找到消息结尾，调用RTSPServer::RTSPClientSession::handleRequestBytes()[值得关注此函数]把请求字符串转换成命令各部分包括：cmdName[方法]，urlPreSuffix[url地址]，urlSuffix[要读取的文件名]，sceq[消息的Cseq]，否则函数返回需要继续从连接中读取请求。分别存入对应的数组。
5. 如果转换成功，调用handleCmd_xxx()处理对应的cmdName: xxx[此处实现了：OPTIONS，DESCRIBE，SETUP，TEARDOWN，PLAY，PAUSE，GET_PARAMETER，SET_PARAMETER]，其中PLAY，PAUSE，GET_PARAMETER，SET_PARAMETER调用handleCmd_withinSession (cmdName,urlPreSuffix, urlSuffix,cseq,(char const*)fRequestBuffer);
6. 清空 RequestBuffer

比如：消息解析后，如果发现客户端的请求是DESCRIBE则进入handleCmd_DESCRIBE函数。RTSP服务器收到客户端的DESCRIBE请求后，根据请求URL(rtsp://192.168.1.109/1.mpg)，找到对应的流媒体资源，返回响应消息。live555中的ServerMediaSession类用来处理会话中描述，它包含多个（音频或视频）的子会话描述(ServerMediaSubsession)。根据客户端请求URL的后缀(例如是1.mpg), 调用成员函数                  DynamicRTSPServer::lookupServerMediaSession查找对应的流媒体信息 ServerMediaSession。（根据urlSuffix查找）。

如果ServerMediaSession不存在，查找文件是否存在，若文件不存在，则判断ServerMediaSession（即smsExists）是否存在，如果存在则将其remove（调用removeServerMediaSession方法）。但是如果本地存在1.mpg文件，则根据文件名创建一个新的 ServerMediaSession（调用createNewSMS方法，若在该方法中找不到对应的文件扩展名，则将返回NULL）。

如果通过lookupServerMediaSession返回的是NULL，则向客户端发送响应消息并将fSessionIsActive置为FALSE；否则，为该session组装一个SDP描述信息（调用generateSDPDescription方法，该方法在ServerMediaSession类中），组装完成后，生成一个RTSP URL（调用rtspURL方法，该方法在RTSPServer类中）。

在创建ServerMediaSession过程中，根据文件后缀.mpg，创建媒体MPEG-1or2的解复用器(MPEG1or2FileServerDemux)。再由MPEG1or2FileServerDemux创建一个子会话描述 MPEG1or2DemuxedServerMediaSubsession。最后由ServerMediaSession完成组装响应消息中的SDP信息（SDP组装过程见下面的描述），然后将响应消息发给客户端，完成一次消息交互。


<br/>

**原文地址：**

[live555学习之RTSP连接建立以及请求消息处理过程](https://www.cnblogs.com/lidabo/p/4388662.html)