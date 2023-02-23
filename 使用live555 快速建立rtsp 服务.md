# 使用live555 快速建立rtsp 服务

1、 live555
live555除了可以建立客户端，当然也可以做一个服务端了，快速做一个服务端就把live555的lib文件编译好，放进来进行。

以h264 为例子，实际上，无论是读取文件也好，还是web摄像头编码也好，需要的其实就是sps，pps信息，就是这么简单。

1.1 读取文件
如果是读取文件，读的过程中要把sps，pps记录下来，如果是加密流，分为两种情况，一是解密，二是直接传输。

1.2 摄像头编码发送
摄像头采用编码h264或者h265后可以发送到连接的客户端，建立一个CameraFrameSource从FramedSource继承，声明如下：

```
 class CameraFrameSource : public FramedSource {
   public:
      static CameraFrameSource* createNew(UsageEnvironment&, CameraH264Encoder*);
      CameraFrameSource(UsageEnvironment& env, CameraH264Encoder*);
      ~CameraFrameSource() = default;

   private:
      static void deliverFrameStub(void* clientData) {
         ((CameraFrameSource*)clientData)->deliverFrame();
      };
      void doGetNextFrame() override;
      void deliverFrame();
      void doStopGettingFrames() override;
      void onFrame();

   private:
      CameraH264Encoder   *fEncoder;
      EventTriggerId       fEventTriggerId;
   };
```

主要转发的代码如下

```
void CameraFrameSource::deliverFrame()
   {
      if (!isCurrentlyAwaitingData()) return; // we're not ready for the buff yet

      static uint8_t* newFrameDataStart;
      static unsigned newFrameSize = 0;
    
      /* get the buff frame from the Encoding thread.. */
      if (fEncoder->fetch_packet(&newFrameDataStart, &newFrameSize)) {
         if (newFrameDataStart != nullptr) {
            /* This should never happen, but check anyway.. */
            if (newFrameSize > fMaxSize) {
               fFrameSize = fMaxSize;
               fNumTruncatedBytes = newFrameSize - fMaxSize;
            }
            else {
               fFrameSize = newFrameSize;
            }
    
            gettimeofday(&fPresentationTime, nullptr);
            memcpy(fTo, newFrameDataStart, fFrameSize);
    
            //delete newFrameDataStart;
            //newFrameSize = 0;
    
            fEncoder->release_packet();
         }
         else {
            fFrameSize = 0;
            fTo = nullptr;
            handleClosure(this);
         }
      }
      else {
         fFrameSize = 0;
      }
    
      if (fFrameSize > 0)
         FramedSource::afterGetting(this);
   };
```

主main函数里面实现rtspserver的run，把刚才的framesource加入到里面。如果是h265，同样的流程。

```
CameraRTSPServer::CameraRTSPServer(CameraH264Encoder * enc, int port, int httpPort)
      : fH264Encoder(enc), fPortNum(port), fHttpTunnelingPort(httpPort), fQuit(0), fBitRate(5120) //in kbs
   {
      fQuit = 0;
   };

   CameraRTSPServer::~CameraRTSPServer()
   {
   };

   void CameraRTSPServer::run()
   {
      TaskScheduler    *scheduler;
      UsageEnvironment *env;
      char stream_name[] = "your name";

      scheduler = BasicTaskScheduler::createNew();
      env = BasicUsageEnvironment::createNew(*scheduler);
    
      UserAuthenticationDatabase* authDB = nullptr;
    
      // if (m_Enable_Pass){
      // 	authDB = new UserAuthenticationDatabase;
      // 	authDB->addUserRecord(UserN, PassW);
      // }
    
      OutPacketBuffer::increaseMaxSizeTo(5242880);  // 1M
      RTSPServer* rtspServer = RTSPServer::createNew(*env, fPortNum, authDB);
    
      if (rtspServer == nullptr)
      {
         *env << "LIVE555: Failed to create RTSP server: %s\n", env->getResultMsg();
      }
      else {
         if (fHttpTunnelingPort)
         {
            rtspServer->setUpTunnelingOverHTTP(fHttpTunnelingPort);
         }
    
         char const* descriptionString = "Combined Multiple Cameras Streaming Session";
    
         CameraFrameSource* source = CameraFrameSource::createNew(*env, fH264Encoder);
         StreamReplicator* inputDevice = StreamReplicator::createNew(*env, source, false);
    
         ServerMediaSession* sms = ServerMediaSession::createNew(*env, stream_name, stream_name, descriptionString); 
         CameraMediaSubsession* sub = CameraMediaSubsession::createNew(*env, inputDevice);
    
         sub->set_PPS_NAL(fH264Encoder->PPS_NAL(), fH264Encoder->PPS_NAL_size());
         sub->set_SPS_NAL(fH264Encoder->SPS_NAL(), fH264Encoder->SPS_NAL_size());
         if (fH264Encoder->h264_bit_rate() > 102400) {
            sub->set_bit_rate(fH264Encoder->h264_bit_rate());
         }
         else {
            sub->set_bit_rate(static_cast<uint64_t>(fH264Encoder->pict_width() * fH264Encoder->pict_height() * fH264Encoder->frame_rate() / 20));
         }
         
         sms->addSubsession(sub);
         rtspServer->addServerMediaSession(sms);
    
         char* url = rtspServer->rtspURL(sms);
         *env << "Play this stream using the URL \"" << url << "\"\n";
         delete[] url;
    
         //signal(SIGNIT,sighandler);
         env->taskScheduler().doEventLoop(&fQuit); // does not return
    
         Medium::close(rtspServer);
         Medium::close(inputDevice);
      }
    
      env->reclaim();
      delete scheduler;
   };
}
```

2、推流到其他服务器

```
2.1.OPTIONS会话
    OPTIONS rtsp://x.x.x.x.:445/live/1001 RTSP/1.0
    User-Agent: youragentname
    CSeq: 1
2.2 ANNOUNCE
    ANNOUNCE rtsp://x.x.x.x:445/live/1001 RTSP/1.0
    User-Agent: youranentname
    CSeq: 2
    Content-Type: application/sdp
    Content-Length: number

    v=0
    o=- 1464329709034587 1 IN IP4 127.0.0.1
    s= h264 Transport Stream, streamed by qianbo
    m= video 0 RTP/AVP 96
    c=IN IP4 0.0.0.0
    a=control:control:streamid=0
    m=audio 0 RTP/AVP 97
    a=rtpmap:97 MPEG4-GENERIC/44100/1
    a=fmtp:97 profile-level-id=1;mode=AAC-hbr;sizelength=13;indexlength=3;indexdeltalength=3; 				   					config=1208
    a=control:streamid=1
2.3 SETUP tcp
    SETUP rtsp://x.x.x.x:445/live/trackid1 RTSP/1.0
    Transport: RTP/AVP/TCP;unicast;interleaved=0-1;mode=record
    User-Agent: youragentname
    CSeq: 3
2.4 RECORD
    RECORD rtsp://x.x.x.x:445/live RTSP/1.0
    Session: 0x11111
    User-Agent: youragentname
    CSeq: 4
2.5 TEARDOWN
    TEARDOWN rtsp://x.x.x.x:445/live RTSP/1.0
    Session: 0x11111
    User-Agent: youragentname
    CSeq: 4
```

这个方式并不是使用rtsp server 来接收下级的流，而是使用live555 来push到上级server，主代码如下所示

```
#include "liveMedia.hh"
#include "BasicUsageEnvironment.hh"

UsageEnvironment* env;

// To make the second and subsequent client for each stream reuse the same
// True：The client that is started later always starts playing from the position that the first client has already played
// False：Every client plays video files from the beginning
Boolean reuseFirstSource = False;

// This function prints relevant information
static void announceStream(RTSPServer* rtspServer, ServerMediaSession* sms, char const* streamName);

int main(int argc, char** argv) {

    // 1. Create a task scheduler and initialize the use environment
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    
    // 2. Create an interactive environment
    env = BasicUsageEnvironment::createNew(*scheduler);
    // The following is the code for permission control. After setting, the client without permission cannot connect
    UserAuthenticationDatabase* authDB = NULL;
    
    authDB = new UserAuthenticationDatabase;
    authDB->addUserRecord("admin", "Suntek123"); // replace these with real strings  


    // 3. Create an RTSP server and start monitoring the connection of the module client. 
    // Note that if the port here is not the default port 554, you must specify the port number when accessing the URL:
    RTSPServer* rtspServer = RTSPServer::createNew(*env, 554, authDB);
    if (rtspServer == NULL) {
        *env << "Failed to create RTSP server: " << env->getResultMsg() << "\n";
        exit(1);
    }
    
    char const* descriptionString = "Session streamed by \"h264LiveMediaServer\"";
    
    // A H.264 video elementary stream:
    {
        char const* streamName = NULL; // Stream name, media name
    
    // 4. Create a media session 
        // When the client orders, the stream name streamName must be input to tell the RTSP server which stream is ordering.
        // Media session manages session-related information such as session description, session duration, stream name, etc.
        // When running h264LiveMediaServer through the IDE, live555 pushes the video or audio in the project working directory. The working directory is the directory at the same level as *.vcxproj,
        //第二个参数:媒体名、三:媒体信息、四:媒体描述
        ServerMediaSession* sms = ServerMediaSession::createNew(*env, streamName, streamName, descriptionString);
    
    //5. Add 264 sub-session. The file name here is the name of the file that is actually opened.
        sms->addSubsession(H264LiveVideoServerMediaSubsession::createNew(*env, reuseFirstSource));
    //6. Add session for rtspserver, but streamName = NULL
        rtspServer->addServerMediaSession(sms);
        announceStream(rtspServer, sms, streamName);
    }
    
    // Trying to create an HTTP server for the RTSP-over-HTTP channel.
    if (rtspServer->setUpTunnelingOverHTTP(80) || rtspServer->setUpTunnelingOverHTTP(8000) || rtspServer->setUpTunnelingOverHTTP(8080)) {
        *env << "\n(We use port " << rtspServer->httpServerPortNum() << " for optional RTSP-over-HTTP tunneling.)\n";
    }
    else {
        *env << "\n(RTSP-over-HTTP tunneling is not available.)\n";
    }

   // Entering the event loop, the read event of the socket and the delayed sending of the media file are all completed in this loop.
    env->taskScheduler().doEventLoop(); // does not return

    return 0; // only to prevent compiler warning
}

static void announceStream(RTSPServer* rtspServer, ServerMediaSession* sms, char const* streamName) {
    char* url = rtspServer->rtspURL(sms);
    UsageEnvironment& env = rtspServer->envir();
    env << "\nstreamName: " << streamName << "\n";
    env << "Play using the URL \"" << url << "\"\n";
    delete[] url;
}
```

ok，读者可以详细研究一下。

————————————————

版权声明：本文为CSDN博主「qianbo_insist」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/qianbo042311/article/details/125349800