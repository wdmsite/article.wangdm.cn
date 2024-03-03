---
title: "Live555学习之基本类介绍及计划任务深度探讨"
date: 2024-03-02T22:05:00+08:00
categories:
- Media
tags:
- live555
- RTSP
draft: false
---


liveMedia项目的源代码包括四个基本的库，各种测试代码以及Media Server。四个基本的库分别是:

UsageEnvironment&TaskScheduler, groupsock, liveMedia和BasicUsageEnvironment。

1，基础类介绍：

BasicUsageEnvironment和UsageEnvironment中的类都是用于整个系统的基础功能类．用于事件的调度，实现异步读取事件的句柄的设置以及错误信息的输出。比如UsageEnvironment代表了整个系统运行的环境，它提供了错误记录和错误报告的功能，无论哪一个类要输出错误，就需要保存UsageEnvironment的指针．而TaskScheduler则提供了任务调度功能．整个程序的运行发动机就是它，它调度任务，执行任务（任务就是一个函数）．TaskScheduler由于在全局中只有一个，所以保存在了UsageEnvironment中．而所有的类又都保存了UsageEnvironment的指针，所以谁想把自己的任务加入调度中，那是很容易的．在此还看到一个结论：整个live555（服务端）只有一个线程．

类HashTable：实现了哈稀表．定义了一个通用的hash表，其它代码要用到这个表。

liveMedia库中有一系列类，基类是Medium，这些类针对不同的流媒体类型和编码。

基于liveMedia的程序，需要通过继承UsageEnvironment抽象类和TaskScheduler抽象类，定义相应的类来处理事件调度，数据读写以及错误处理。live项目的源代码里有这些类的一个基本实现，这就是“BasicUsageEnvironment”库。BasicUsageEnvironment主要是针对简单的控制台应用程序，利用select实现事件获取和处理。这个库利用Unix或者Windows的控制台作为输入输出，出于应用程序原形或者调试的目的，用户可以用这个库开发传统的运行与控制台的应用。

类DelayQueue：译为＂延迟队列＂，它是一个队列，每一项代表了一个要调度的任务（在它的fToken变量中保存）．同时保存了这个任务离执行时间点的剩余时间．可以预见，它就是在TaskScheduler中用于管理调度任务的东西．注意，此队列中的任务只被执行一次！执行完后这一项即被无情抛弃！

类HandlerSet：Handler集合．Handler是什么呢？它是一种专门用于执行socket操作的任务（函数），HandlerSet被TaskScheduler用来管理所有的socket任务（增删改查）．所以TaskScheduler中现在已调度两种任务了：socket任务（handlerSet）和延迟任务(DelayQueue)．其实TaskScheduler还调度第三种任务：Event，这个后面再说．

类Groupsock：这个是放在单独的库Groupsock中。它封装了socket操作，增加了多播支持和一对多单播的功能．但好像不支持TCP。它管理着一个本地socket和多个目的地址，因为是UDP，所以只需知道对方地址和端口即可发送数据。Groupsock的构造函数有一个参数是struct in_addr const& groupAddr，在构造函数中首先会调用父类构造函数创建socket对象，然后判断这个地址，若是多播地址，则加入多播组。Groupsock的两个成员变量destRecord* fDests和DirectedNetInterfaceSet fMembers都表示目的地址集和，但貌似这个变量DirectedNetInterfaceSet fMembers没有用到，且DirectedNetInterfaceSet是一个没有被继承的虚类，看起来fMembers没有什么用。仅fDesk也够用了，在addDestination()和removeDestination()函数中就是操作fDesk，添加或删除目的地址。

2，基本概念
    先来熟悉在liveMedia库中Source，Sink以及Filter等概念。Sink就是消费数据的对象，比如把接收到的数据存储到文件，这个文件就是一个Sink。Source就是生产数据的对象，比如通过RTP读取数据。数据流经过多个'source'和'sinks'，下面是一个示例：
          source1' -> 'source2' (a filter) -> 'source3' (a filter) -> 'sink'
    从其它Source接收数据的source也叫做"filters"。Module是一个sink或者一个filter。数据接收的终点是Sink类，MediaSink是所有Sink类的基类。Sink类实现对数据的处理是通过实现纯虚函数continuePlaying()，通常情况continuePlaying调用fSource -> getNextFrame来为Source设置数据缓冲区，处理数据的回调函数等，fSource是MediaSink的类型为FramedSource*的类成员。

3，计划任务(TaskScheduler)深入探讨

我们且把三种任务命名为：socket handler,event handler,delay task。

这三种任务的特点是，前两个加入执行队列后会一直存在，而delay task在执行完一次后会立即弃掉。

socket handler保存在队列BasicTaskScheduler0::HandlerSet* fHandlers中;

event handler保存在数组BasicTaskScheduler0::TaskFunc *

 fTriggeredEventHandlers[MAX_NUM_EVENT_TRIGGERS]中;

delay task保存在队列BasicTaskScheduler0::DelayQueue fDelayQueue中。


   下面看一下三种任务的执行函数的定义：
socket handler为
typedef void BackgroundHandlerProc(void* clientData, int mask);
event handler为
typedef void TaskFunc(void* clientData);
delay task 为
typedef void TaskFunc(void* clientData);//跟event handler一样。

    再看一下向任务调度对象添加三种任务的函数的样子：
socket handler为：
void setBackgroundHandling(int socketNum, int conditionSet　,BackgroundHandlerProc* handlerProc, void* clientData)
event handler为:
EventTriggerId createEventTrigger(TaskFunc* eventHandlerProc)
delay task为：
TaskToken scheduleDelayedTask(int64_t microseconds, TaskFunc* proc,void* clientData)

socket handler添加时为什么需要那些参数呢？socketNum是需要的，因为要select socket（socketNum即是socket()返回的那个socket对象）。conditionSet也是需要的，它用于表明socket在select时查看哪种装态，是可读？可写？还是出错？再看BackgroundHandlerProc的参数，socketNum不必解释，mask是什么呢？它正是对应着conditionSet，但它表明的是select之后的结果，比如一个socket可能需要检查其读/写状态，而当前只能读，不能写，那么mask中就只有表明读的位被设置。

event handler是被存在数组中。数组大小固定，是32项，用EventTriggerId来表示数组中的项，EventTriggerId是一个32位整数，因为数组是32项，所以用EventTriggerId中的第n位置１表明对应数组中的第n项。成员变量fTriggersAwaitingHandling也是EventTriggerId类型，它里面置1的那些位对应了数组中所有需要处理的项。这样做节省了内存和计算，但降低了可读性，而且也不够灵活，只能支持32项或64项，其它数量不被支持。


<br/>

**原文地址：**

[Live555学习之基本类介绍及计划任务深度探讨](https://www.cnblogs.com/lidabo/p/4388669.html)