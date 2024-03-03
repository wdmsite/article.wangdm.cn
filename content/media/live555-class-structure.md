---
title: "Live555类结构"
date: 2024-03-02T21:39:00+08:00
categories:
- Media
tags:
- live555
- RTSP
draft: false
---




Medium

live555几乎所有的处理单元都继承自Medium类；该类抽象了基本的接口，包括环境，task和媒体名和媒体查找函数（lookupByName）以及一些辅助函数。也包括返回当前的环境类UsageEnvironment，以及环境指向下一个TaskToken的指针nextTask等。

ServerMediaSession  

对象的创建函数在文件DynamicRTSPServer.cpp中。DynamicRTSPServer的继承关系是

DynamicRTSPServer::RTSPServerSupportingHTTPStreaming::RTSPServer::Medium

DynamicRTSPServer从RTSPServer继承过来，仅仅添加了构造器和查找函数，没有添加其他成员。构造器是创建socket然后传给 RTSPServer，查找是如果没有已经打开的流服务，那么根据参数创建流服务。RTSPServer的属性有socket，端口号，Session点号，认证机制和ServerMediaSession表。  

当接收到带有URI的请求后，会首先创建SMS，调用 ServerMediaSession的构造函数，除了创建时间戳，拷贝文件名外都是使用的缺省值，并且将初始化子会话链表。当处理describe命令时，ServerMediaSession通过调用generateSDPDescription函数生成。  在NEW_SMS()中创建ServerMediaSession对象，然后创建相应的ServerMediaSubsession并将这个子会话对象添加到添加到会话对象中。  

如子回话是MPEG4，创建MPEG4VideoFileServerMediaSubsession对象，对象的继承关系：  

MPEG4VideoFileServerMediaSubsession::FileServerMediaSubsession  ::OnDemandServerMediaSubsession::ServerMediaSubsession ::Medium  

ServerMediaSession::Medium   

相关类介绍：

ServerMediaSession：添加了子会话链表，SDP描述以及一些媒体相关处理函数。  

ServerMediaSubsession：定义了指向ServerMediaSession的父指针，指向下个一个对象的指针。该媒体的SDP信息，该媒体的读取定位函数等。 ServerMediaSubsession类和具体的流播放相关，是个纯虚类。其中startStream和getStreamParameter是纯虚函数。 

OnDemandServerMediaSubsession：添加了流source处理和RTPSink处理函数以及经典命名属性等。封装seek,pause等处理，把这些接口中clientSessionid号到这里转换成了FramedSource。  该类的成员函数大部分和ServerMediaSubsession相似，在流媒体完成定位等处理。createNewStreamSource和createNewRTPSink是两个纯虚函数，在子类中必须实现。类中getStreamParameters方法会创建streamState。这个方法在处理RTP的Setup命令时被调用。  

FileServerMediaSubsession：增加了文件名和文件大小属性。

MPEG4VideoFileServerMediaSubsession：添加了RTPSink属性，并且实现了OnDemandServerMediaSubsession中定义的两个纯虚函数，即创建了source和sink对象。这个source是MPEG4VideoStreamFramer。该类中还定义了StreamState的内部类。  

StreamState：包含了指向OnDemandServerMediaSubsession的引用，RTPSink指针，BasicUDPSink指针，RTCPInstance指针FramedSource指针，fRTPgs和fRTCPgs(groupsock).

StreamState类可以用OnDemandServerMediaSubsession的fLastStreamToken属性指向。  

类streamState的属性：  

OnDemandServerMediaSubsession& fMaster;      Boolean fAreCurrentlyPlaying;  

unsigned fReferenceCount;                                Port fServerRTPPort, fServerRTCPPort; 

RTPSink* fRTPSink;                                                  BasicUDPSink* fUDPSink; 

float fStreamDuration;                        unsigned fTotalBW; RTCPInstance* fRTCPInstance;

FramedSource* fMediaSource;                  Groupsock* fRTPgs; Groupsock* fRTCPgs;  

 

Sink  

Sink类提供了总的媒体播放接口。sink有两种，一个是BasicUDPSink，一个是RTPSink，如果协商时没有RTP信息，那么创建BasicUDPSink。Source和 Sink通过函数createNewRTPSink和createNewStreamSource。这两个函数在类 OnDemandServerMeidaSubsession中定义为纯虚函数，如果媒体类型是mpeg4videofileserver，那么对应的函数定义在类MPEG4VideoFileServerMediaSubsession中。  

 

 MPEG4ESVideoRTPSink::VideoRTPSink::MultiFramedRTPSink::RTPSink::MediaSink::Medium 

MediaSink定义中有一个媒体源指针，主要处理函数有startplaying(),stopplaying()和 afterPlayingFunc函数指针。

RTPSink类定义了RTP相关的处理和属性。包含Socket组对象，时间处理系列，统计计数处理等相关属性。  

MultiFramedRTPSink是RTPSink的子类，处理buffer中的多个RTP包。类中添加了辅助SDP处理和VOPIsPresent属性和一个判断性处理函数。

MultiFramedRTPSink类完成多帧组包处理主要函数有buildAndSendPacket，packFrame，

sendNext， afterGettingFrame，这几个函数之间有相互调用。内部有OutPacketBuffer属性，在创建时设定为（1000(希望)，1448(最大)）大小，其他是统计或者标识属性。这个发送数据包是通过 fRTPInterface.sendPacket(fOutBuf->packet(), fOutBuf->curPacketSize());实现。这个fRTPInterface是父类RTPSink的属性。  

VideoRTPSink仅仅添加了sdpMediaType处理函数, 返回SDP类型是“video”

MPEG4ESVideoRTPSink中的处理函数doSpecialFrameHandling：首先检测开头的四个字节看是否是 VOP_START_CODE，该函数处理RTP的起始/中止标识和添加时间戳。其他处理包括是否允许分片，是否是起始包判断以及辅助SDP处理。

Source  

createNewStreamSource调用的是MPEG4VideoFileServerMediaSubsession中的定义。在类 OnDemandServerMediaSubsession中的createNewStreamSource定义是一个纯虚函数。  

创建的source是：  

MPEG4VideoStreamFramer:MPEGVideoStreamFramer:FramedFilter:FramedSource:MediaSource:Medium  

MediaSource在Medium类的基础上添加了更多媒体类型判断，比如是H264，mpeg还是jpeg。此外还有一个MIME类型。  

FramedSource类处理成帧类型的媒体，比如 mpeg,mjpeg,h264,amr等音频类型的媒体。函数分帧处理媒体流，主要处理是getNextFrame，afterGetting以及关闭等媒体处理，此外定义了doGetNextFrame纯虚函数，这个函数由getNextFrame调用，处理具体的媒体流。该类还定义了两个函数指针，afterGettingFunc* fAfterGettingFunc; onCloseFunc* fOnCloseFunc;处理。  该类的属性包括数据拷贝的指针，帧的大小，展示时间，和播放间隔，是否当前等待播放标志。

FramedFilter是FramedSource的子类，这是个中间类，主要在类中添加了指向输入源的指针 FramedSource* fInputSource;  

MPEGVideoStreamFramer：是FramedFilter的子类，因为mpeg是时间相关的媒体流，所以在父类的基础上添加了时间处理函数，此外还有 continueReadProcessing函数。主要的属性有：帧率，结束标志，图片计数，展示时间，GOP时间相关内容，图片时间相关属性。此外还有一个重要的类属性： MPEGVideoStreamParser用来分析媒体流。  

MPEG4VideoStreamFramer：在父类的基础上添加了config信息，类如profile等级信息。在该类的定义文件中还实现了MPEG4文件分析类，继承自MPEGVideoStreamParser。处理mepg4相关信息。  

ByteStreamFileSource::FramedFileSource::FramedSource  

source里面有一个非常重要的StreamParser（流分析）对象，用来分析，读取流数据。其中afterGettingBytes和不同媒体流处理相关，内部有一个函数fClientContinueFunc为不同媒体注册的函数。而getNextFrame会调用afterGettingBytes。  

MPEG4ESVideoRTPSource:MultiFramedRTPSource:RTPSource:FramedSource:MediaSource:Medium

RTPSource:添加RTP相关处理，主要属性有RTPInterface，时间标签，处理数据帧拆分packetMarkerbit时间戳频率和统计信息。RTP统计信息：收到的总包数，从reset以后收到的总包数，收到的字节数，初始化序列标识，前面一个RTP包的时间戳，接受到的发送者报告的NTP时间，接受到的发送者报告时间。RTPSource仅仅处理RTP协议相关的问题。MultiFramedRTPsource中的 networkHandler中会调用到这些处理。  

playing  

在服务端的操作中是围绕着StreamState展开的，OnDemandServerMediaSubsession类中的StartStream通过调用StreamState的startplaying开始进行，并且初始化rtpSeqNum和rtpTimestamp (= rtpSink()->presetNextTimestamp()) 两个变量。在MediaSink定义了startPlaying,这里通过调用BasicUDPSink类中的continuePlaying最终调用到了 buildAndSendPacket，函数buildAndSendPacket根据传入的参数是否第一帧分别进行处理。如果是第一帧，那么取当前时间标签，作为发送时间，

在函数packFrame中，首先调用afterGettingFrame1，然后调用 

fSource->getNextFrame(fOutBuf->curPtr(), fOutBuf->totalBytesAvailable(),afterGettingFrame, this, ourHandleClosure, this);处理。

在播放过程中，通过nextTask() = envir().taskScheduler().scheduleDelayedTask

(uSecondsToGo,(TaskFunc*)sendNext, this)；每次都会计算一个下次发送时间和包添加到调度中进行。  

读取媒体数据在MPEG4VideoStreamParser类定义中处理。成员函数parse分析出读取数据的大小。然后读取一帧数据，交给打包代码处理。 


<br/>

**原文地址：**

[Live555类结构](https://www.cnblogs.com/lidabo/p/4388686.html)