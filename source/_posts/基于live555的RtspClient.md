---
title: 基于live555的RtspClient
date: 2018-01-24 18:29:20
categories:
 术业专攻
tags:
 [Live555,Visual Studio]
---
<Excerpt in index | 首页摘要>
<!-- more -->
<The rest of contents | 余下全文>

# 基于live555的RtspClient
## 前言
    很早之前做的一个监控客户端使用的一个庞大的SDK，现在把RtspClient这端独立出来。后面可能还会写一些列关于音视频相关的Blog，包括视频解码、视频显示、在YUV数据中叠加图片等等。
### 音视频系列
>* [基于live555的RtspClient]()

## 正文
### 开发环境
>* Visual Studio 2013
>* Windows 10

### 开发工具
>* Live555源码

### 开发过程
#### 一、编译Live555库
使用vs2013编译live555库网上应该有一大把资料。
参考这个[Live555编译](http://blog.csdn.net/bluecheney1990/article/details/42086585),
使用方法二，编译出来4个静态库：
>* BasicUsageEnvironmentlib.lib
>* groupsocklib.lib
>* liveMedialib.lib
>* UsageEnvironmentlib.lib

#### 二、开发RtspClient项目
在编译好了live555的库后，用vs创建项目引入静态库，这里就不详细的介绍了，主要来讲讲RtspClient的代码。
live555的源码里面，testProgs\testRTSPClient.cpp 和 testProgs\openRTSP.cpp 详细的介绍了具体流程。
那我们慢慢根据代码来慢慢探索live555 RtspClient 整个连接过程。

> 创建RTSPClient对象。

```c++
m_pScheduler = BasicTaskScheduler::createNew();
m_pEnv = BasicUsageEnvironment::createNew(*m_pScheduler);
m_pRtspClient = RTSPClientEx::createNew(*m_pEnv, m_szUrl.c_str(), 0, "RTSP Client");
```
首先是创建TaskScheduler对象和UsageEnvironment对象,然后根据RTSP地址m_szUrl 创建一个RTSPClient对象，一个RTSPClient对象代表一个RTSP客户端。

> 发送DESCRIBE命令

```c++
Authenticator auth;
auth.setUsernameAndPassword(m_szAccount.c_str(), m_szPassword.c_str());
m_pRtspClient->sendDescribeCommand(continueAfterDESCRIBE, &auth);
if (!wait_Live555_response(wait_live555_time))
{
    continue;
}
```
调用sendDescribeCommand函数发送DESCRIBE命令，回调函数是continueAfterDESCRIBE函数，在收到RTSP服务器端对DESCRIBE命令的回复时调用。大家看到这里可能发现下面的wait_Live555_response怎么和testRTSPClient.cpp里面的不同了，这里我是参考了vlc代码里面的实现。这在里调用wait_Live555_response堵塞线程等待continueAfterDESCRIBE返回结果才执行下一步。wait_Live555_response这个函数会在接下来的代码中发挥同样的作用 wait_live555_time 是超时时间。

```c++
static void TaskInterruptRTSP(void *p_private)
{
	CRtspPullClass *pPullClass = (CRtspPullClass*)p_private;
	pPullClass->m_event_rtsp = 0xff;
}

bool CRtspPullClass::wait_Live555_response(int i_timeout)
{
	TaskToken task;
	m_event_rtsp = 0;
	if (i_timeout > 0)
	{
		task = m_pScheduler->scheduleDelayedTask(i_timeout * 1000,
			TaskInterruptRTSP,
			this);
	}
	m_event_rtsp = 0;
	m_b_error = true;
	m_ilive555_ret = 0;
	m_pScheduler->doEventLoop(&m_event_rtsp);
	if (i_timeout > 0)
		m_pScheduler->unscheduleDelayedTask(task);
	return !m_b_error;
}
```
上面是wait_Live555_response()的具体实现,先通过scheduleDelayedTask设置一个超时回调，然后doEventLoop堵塞Live555的线程，这里m_event_rtsp = 0的时候调用doEventLoop就是堵塞，如果当m_event_rtsp = 1的时候这里就继续往下执行。

然后让我们来看看continueAfterDESCRIBE()回调里的实现

```c++
static void continueAfterDESCRIBE(RTSPClient* rtspClient, int resultCode, char* resultString)
{
	CRtspPullClass* pPullClass = ((RTSPClientEx*)rtspClient)->GetPullClass();
	UsageEnvironment& env = rtspClient->envir();
	pPullClass->m_ilive555_ret = resultCode;
	if (resultCode == 0)
	{
		char* sdpDescription = resultString;
		free(pPullClass->m_p_sdp);
		pPullClass->m_p_sdp = NULL;
		if (sdpDescription)
		{
			pPullClass->m_p_sdp = strdup(sdpDescription);
			pPullClass->m_b_error = false;
		}
		else
			pPullClass->m_b_error = true;
	}
	else
		pPullClass->m_b_error = true;
	delete[] resultString;
	pPullClass->m_event_rtsp = 1;
}
```
如果回调返回成功的话就会获取到sdp信息。然后我们就会更具sdp信息创建一个MediaSession对象，MediaSession表示客户端请求服务器端某个媒体资源的会话。MediaSubsession，表示MediaSession的子会话，创建MediaSession的同时也创建了包含的MediaSubsession对象。然后客户端对服务器端的每个ServerMediaSubsession发送SETUP命令请求建立连接。让我们看接下来创建MediaSession的代码
```c++
typedef struct _VIDEO_PARAM
{
	char codec[256];
	int width;
	int height;
	int colorbits;
	int framerate;
	int bitrate;
	char vol_data[256];
	int vol_length;
}VIDEO_PARAM;



typedef struct  _AUDIO_PARAM
{
	char codec[256];
	int samplerate;
	int bitspersample;
	int channels;
	int framerate;
	int bitrate;
}AUDIO_PARAM;


typedef struct  __STREAM_AV_PARAM
{
	unsigned char	ProtocolName[32];
	short  bHaveVideo;//0 表示没有视频参数
	short  bHaveAudio;//0 表示没有音频参数
	VIDEO_PARAM videoParam;//视频参数
	AUDIO_PARAM audioParam;//音频参数
	char		szUrlInfo[512];//注意长度
}STREAM_AV_PARAM;

if (!(m_pSession = MediaSession::createNew(*m_pEnv, m_p_sdp)))
{
	continue;
}
if (m_pSession->hasSubsessions() ==false)
{
	continue;
}
m_dwLastRecvDataTick = GetTickCount();
STREAM_AV_PARAM * pAvParam = &m_avParam;
memset(pAvParam, 0, sizeof(STREAM_AV_PARAM));
memset(&m_AudioParam, 0, sizeof(AUDIO_PARAM));
memset(&m_VideoParam, 0, sizeof(VIDEO_PARAM));
sprintf((char *)pAvParam->ProtocolName, "%s", "AjVisionHD");
MediaSubsession* subsession = NULL;
MediaSubsessionIterator iter(*m_pSession);
```
STREAM_AV_PARAM 是一个包含媒体流参数的结构体，因为在解码的时候需要知道音视频的类型是H.265 还是H.264 是AAC还是G.711。音频的采样率、码率、通道数。视频的长宽、码率、和帧率。
取得MediaSubsession，然后客户端对服务器端的每个MediaSubsessionn发送SETUP命令请求建立连接。
```c++
while ((subsession = iter.next()) != NULL)
{
	if (subsession->initiate() == false)
	{
		continue;
	}
	if (subsession->rtpSource() != NULL)
	{
		desiredPortNum = subsession->clientPortNum();
		if (desiredPortNum != 0)
		{
			subsession->setClientPortNum(desiredPortNum);
			desiredPortNum += 2;
		}
		subsession->rtpSource()->setPacketReorderingThresholdTime(200000);
		if (strcmp(subsession->mediumName(), "video") == 0)
		{
			int nIsH264 = -1;
			if ((strcmp(subsession->codecName(), "h264") == 0)
				|| (strcmp(subsession->codecName(), "H264") == 0))
				nIsH264 = 1;
			else if ((strcmp(subsession->codecName(), "h265") == 0)
				|| (strcmp(subsession->codecName(), "H265") == 0))
				nIsH264 = 0;
			if (nIsH264 == -1)
				continue;
			VIDEO_PARAM *pVideoParam = &m_VideoParam;
			memset(pVideoParam, 0, sizeof(VIDEO_PARAM));
			strcpy(pVideoParam->codec, subsession->codecName());
			pVideoParam->width = subsession->videoWidth();
			pVideoParam->height = subsession->videoHeight();
			pVideoParam->framerate = subsession->videoFPS();
			pVideoParam->bitrate = 0;
			if (subsession->fmtp_config() != NULL)
			{
				unsigned configLen;
				unsigned char* configData = parseGeneralConfigStr(subsession->fmtp_config(), configLen);
				if (configData != NULL)
				{
					memcpy(pVideoParam->vol_data, configData, configLen);
					pVideoParam->vol_length = configLen;
					delete[] configData; configData = NULL;
				}
			}
			pAvParam->bHaveVideo = 1;
			memcpy(&pAvParam->videoParam, pVideoParam, sizeof(VIDEO_PARAM));
			if (m_pVideoSink)
			{
				CVideoStreamSink::close(m_pVideoSink);
				m_pVideoSink = NULL;
			}
			try
			{
				m_pVideoSink = new CVideoStreamSink(*m_pEnv, nIsH264, this, (LONG)this, 4 * 1024 * 1024);
			}
			catch (...)
			{
				m_pVideoSink = NULL;
				continue;
			}
			if (m_pVideoSink)
			{
				m_pVideoSink->SetStreamName("");
				if ( m_nLinkMode== 2)
					m_pRtspClient->sendSetupCommand(*subsession, default_live555_callback, false, false, true);
				else
					m_pRtspClient->sendSetupCommand(*subsession, default_live555_callback, false, m_nLinkMode, false);
				m_dwLastRecvDataTick = GetTickCount();
				if (!IsRunning())
					break;
				if (!wait_Live555_response(wait_live555_time))
				{
					continue;
				}
				m_dwLastRecvDataTick = GetTickCount();
				if (m_nLinkMode == 2)
				{//组播
					subsession->setDestinations(subsession->connectionEndpointAddress());
				}
				subsession->miscPtr = m_pRtspClient;
				m_pVideoSink->startPlaying(*(subsession->readSource()), subsessionAfterPlaying, subsession);
				if (subsession->rtcpInstance() != NULL)
			    	subsession->rtcpInstance()->setByeHandler(subsessionByeHandler, subsession);
				subsession->sink = m_pVideoSink;
			}

		}
		else if (strcmp(subsession->mediumName(), "audio") == 0)
		{
			AUDIO_PARAM *pAudioParam = &m_AudioParam;
			memset(pAudioParam, 0, sizeof(AUDIO_PARAM));
			strcpy(pAudioParam->codec, subsession->codecName());
			DebugLog("CRtspPullClass::RtspLinkThread() %s recv audio type = %s", m_szUrl.c_str(), subsession->codecName());
			if (strstr(pAudioParam->codec, "PCMU") || strstr(pAudioParam->codec, "PCMA")) // for G711
			{
				pAudioParam->bitrate = 64000;
				pAudioParam->bitspersample = 16;
				pAudioParam->channels = 1;//subsession->numChannels();
				pAudioParam->framerate = 8;
				pAudioParam->samplerate = subsession->rtpTimestampFrequency();

			}
			else //for AAC
			{
				unsigned configLen;
				unsigned char* configData = parseGeneralConfigStr(subsession->fmtp_config(), configLen);
				if (configData != NULL)
				{
					unsigned char samplingFrequencyIndex = (configData[0] & 0x07) << 1;
					samplingFrequencyIndex |= (configData[1] & 0x80) >> 7;
					if (samplingFrequencyIndex > 15)
					{
						samplingFrequencyIndex = 0;
					}

					pAudioParam->bitspersample = 16;
					pAudioParam->samplerate = samplingFrequencyTable[samplingFrequencyIndex];
					pAudioParam->channels = 2;
					pAudioParam->bitrate = pAudioParam->samplerate * pAudioParam->bitspersample / 8;
					pAudioParam->framerate = 8;
					delete[] configData; configData = NULL;
				}
				else
				{
					pAudioParam->bitrate = 16000;
					pAudioParam->bitspersample = 16;
					pAudioParam->channels = 2;
					pAudioParam->framerate = 16;
					pAudioParam->samplerate = 16000;
				}
			}
			pAvParam->bHaveAudio = 1;
			memcpy(&pAvParam->audioParam, pAudioParam, sizeof(AUDIO_PARAM));
			if (m_pAudioSink)
			{
				CAudioStreamSink::close(m_pAudioSink);
				m_pAudioSink = NULL;
			}
			try
			{
				m_pAudioSink = new CAudioStreamSink(*m_pEnv, this, (LONG)this, 64 * 1024);
			}
			catch (...)
			{
				CAudioStreamSink::close(m_pAudioSink);
				m_pAudioSink = NULL;
				continue;
			}
			if (m_pAudioSink)
			{
				if (m_nLinkMode == 2)
				{
					m_pRtspClient->sendSetupCommand(*subsession, default_live555_callback, false, false, true);//,TRUE);
				}
				else
					m_pRtspClient->sendSetupCommand(*subsession, default_live555_callback, false, m_nLinkMode);//,TRUE);			
				m_dwLastRecvDataTick = GetTickCount();
				if (!IsRunning())
					break;
				if (!wait_Live555_response(wait_live555_time))
				{
					if (m_pAudioSink)
					{
						CAudioStreamSink::close(m_pAudioSink);
						m_pAudioSink = NULL;
					}
					subsession->sink = NULL;
					continue;
				}
				if (m_nLinkMode == 2)
				{//组播
					subsession->setDestinations(subsession->connectionEndpointAddress());
				}
				subsession->miscPtr = m_pRtspClient;
				m_pAudioSink->startPlaying(*(subsession->readSource()), subsessionAfterPlaying, subsession);
				if (subsession->rtcpInstance() != NULL)
					subsession->rtcpInstance()->setByeHandler(subsessionByeHandler, subsession);
				subsession->sink = m_pAudioSink;
				m_dwLastRecvDataTick = GetTickCount();
			}
		}
	}
}
if (m_pVideoSink || m_pAudioSink)
{
	m_pRtspClient->sendPlayCommand(*m_pSession, default_live555_callback, 0, -1, 1);
	if (!wait_Live555_response(wait_live555_time))
	{
		continue;
	}
	m_isStoping = 0;
	m_pScheduler->doEventLoop(&m_isStoping);
}

```
在上面的代码中，首先调用MediaSubsession的initiate函数初始化MediaSubsession，然后通过MediaSubsession获取音视频的参数填充结构体。最后为MediaSubsession创建MediaSink对象来请求和保存服务器端发送的数据。然后对MediaSubsession发送SETUP命令。同样调用wait_Live555_response等待回调返回数据，然后调用MediaSink::startPlaying函数开始准备播放对应的MediaSubsession。如果建立连接成功了最后会发送Play命令请求开始传送数据同样等到回复后调用m_pScheduler->doEventLoop(&m_isStoping)堵塞线程;

>下面的代码是MediaSink里面的实现了

```c++


void CMediaSinkSink::afterGettingFrame(void* clientData, unsigned frameSize,
 unsigned /*numTruncatedBytes*/,
 struct timeval presentationTime,
 unsigned /*durationInMicroseconds*/) 
{
	CVideoStreamSink* sink = (CVideoStreamSink*)clientData;	
	fSource->getNextFrame(fBuffer, fBufferSize,afterGettingFrame, this,onSourceClosure, this);
} 
```
在DummySink::afterGettingFrame函数中只是简单地打印出了某个MediaSubsession接收到了多少字节的数据，然后接着利用FramedSource去读取数据。可以看出，在RTSP客户端，Live555也是在MediaSink和FramedSource之间形成了一个循环，不停地从服务器端读取数据。

### 参考文献
>* [VLC](http://www.videolan.org/vlc/download-sources.html)

### 最后的话
>* 如果遇到问题可以给我发邮件[chenyiyu@gmail.com](mailto:chenyiyu@gmail.com)
>* 转载请注明原地址，[Alvin的博客](http://alvinCyy.github.io) 谢谢！