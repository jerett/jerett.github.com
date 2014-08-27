---
layout: post
title: "基于SDL、FFMpeg简易播放器"
date: 2014-08-27 17:02:48 +0800
comments: true
categories: 
---
#吐槽
博客在今年一月份的时候就搭建好了，但是一直没写。最近花了接近两个月的时间完成了IOS客户端的rtmp推流，用到了ffmpeg，并看了一堆音视频编码技术。趁这两天霸道总裁去北京，开始准备好管理一下博客。
#简介
这篇文章主要介绍使用SDL和ffmpeg，c语音编写一个简单播放器的大概实现原理，其中最难理解的就是如何同步音视频。	
实现主要参考老外的ffmpeg tutorial(http://dranger.com/ffmpeg/)。这系列的教程很赞，整体难度给人感觉也很大，主要是教程ffmpeg的sdk似乎是0.6的，而ffmpeg的API接口几乎是每次版本都更新，所以需要更改的地方很多。网上翻译这些列文章也很多，但能找到最新的代码也已经out of date了，所以几乎都没法在ffmepg 2.1上编译。	
最近在研究ffmpeg，顺便基于ffmpeg最新的sdk实现了这个播放器，并且将代码进行了模块化，可以直接```git clone https://github.com/jerett/sdl_ffmpeg_player.git```。
#SDL
SDL是一个跨平台的多媒体开发的函数库，包括win，unix，ios，android。在这个播放器里主要用来显示视频和播放声音。
#ffmpeg
ffmepg是一个音视频解码、编码库，在这个播放器里主要用来对文件进行解码，再将编码的数据编码成SDL要求的数据格式。
#原理
先贴一张工作的流程图。	
{%img /images/sdl_ffmpeg_player.png%}
播放器有4个线程，主线程创建其他线程。	
线程之间有几乎都存在生产者消费者的情况，包括:解码线程生产未编码数据，音视频编码线程消费数据；视频编码线程生产picture图片数据，主线程消费并显示图片；	
###主线程
主线程负责初始化ffmpeg和SDL，创建解码线程，并且循环读取事件，分配内存，显示图片。
###解码线程
解码线程用ffmpeg打开文件，创建context，分析音频流和视频流，并创建音频和视频编码线程。并通过av_read_frame循环读取文件，将AVPacket添加到缓存队列中。为了防止内存过大，缓存队列设置最大BUFFER_SIZE，如果大于BUFFER_SIZE,则sleep等待消费者消费。	
###音频编码线程
音频编码线程是SDL创建，我们只需提供声音数据的回调函数，将音频数据填入回调函数即可。SDL会将填入的数据进行播放。
###视频编码线程
视频编码线解码数据，并将解码的AVFrame转化为SDL可以显示的SDL_YUVOverlay，加入图片缓存队列，等待主线程显示。
###音视频同步
最蛋疼的问题就是如何实现音视频同步了。	
首先我们知道ffmpeg readPacket的时候每个packet都有pts和dts，既然如此我们按照pts显示就行了，为什么会不同步？首先可能因为视频解码编码时间过长，编码完成后可能已经过了应该显示的pts时间，这时候应该尽快显示此帧。有时候解码过快，还未到显示的pts，我们需要延迟显示此帧。		
但是音频的解码效率很高，而且音频是恒定频率，所以pts稳定增加，所以我们选择视频同步到音频，及以音频为参考时间，调整视频的播放时间进行音视频同步。		
下面是同步的核心代码
```c		
void video_refresh_timer(void *userdata) {

    VideoState *is = (VideoState *)userdata;
    VideoPicture *vp;
    double actual_delay, delay, sync_threshold, ref_clock, diff;

    if(is->video_st) {
        if(is->pictq_size == 0) {
            schedule_refresh(is, 1);
        } else {

            /* printf("vidoe clock %f  audio clock %f \n",is->video_clock,is->audio_clock); */
            /* printf("audio clock %f",is->audio_clock); */

            vp = &is->pictq[is->pictq_rindex];

            delay = vp->pts - is->frame_last_pts; /* the pts from last time */
            /* printf("delay %f ",delay); */
            if(delay <= 0 || delay >= 1.0) {
                /* if incorrect delay, use previous one */
                delay = is->frame_last_delay;
            }
            /* save for next time */
            is->frame_last_delay = delay;
            is->frame_last_pts = vp->pts;

            /* update delay to sync to audio */
            ref_clock = get_audio_clock(is);
            diff = vp->pts - ref_clock;
            /* printf("diff %f \n",diff); */

            /* Skip or repeat the frame. Take delay into account
               FFPlay still doesn't "know if this is the best guess." */
            sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;
            if(fabs(diff) < AV_NOSYNC_THRESHOLD) {
                if(diff <= -sync_threshold) {
                    delay = 0;
                } else if(diff >= sync_threshold) {
                    delay = 2 * delay;
                }
            }

            is->frame_timer += delay;
            /* computer the REAL delay */
            actual_delay = is->frame_timer - (av_gettime() / 1000000.0);
            /* printf("diff %f actual delay %f \n",diff,actual_delay); */
            if(actual_delay < 0.010) {
                /* Really it should skip the picture instead */
                actual_delay = 0.010;
            }
            schedule_refresh(is, (int)(actual_delay * 1000 + 0.5));
            /* show the picture! */
            video_display(is);

            /* update queue for next picture! */
            if(++is->pictq_rindex == VIDEO_PICTURE_QUEUE_SIZE) {
                is->pictq_rindex = 0;
            }
            SDL_LockMutex(is->pictq_mutex);
            is->pictq_size--;
            SDL_CondSignal(is->pictq_cond);
            SDL_UnlockMutex(is->pictq_mutex);
        }
    } else {
        schedule_refresh(is, 100);
    }
}
```		
同步代码在else里面，首先我们注意到delay这个值，delay是这一帧图像与上一帧的时间差，我们计算它，如果合理就使用，不合理则使用记录的上次的delay。然后我们计算pts与audio_clock的时间差，如果diff过大，说明现在的pts过大，我们需要延后这个picture显示，否则pts会越来越大。如果diff过小，我们需要抓紧显示这张图片，负责video_clock会与audio_clock更小。frame_timer初始值为播放的开始时间，每次增加delay，从而计算出实际的delay。为什么不简单的使用delay这个值呢？我觉得长期累加的值才能产生更稳定的播放效果，这是一个数学论证，作为程序员的我们了解这种思想就好@V@。当然，这只是我的推测。对了，我们还要注意sync_threshold变量的赋值，我们选取max(delay,AV_SYNC_THRESHOLD)赋值，其中AV_SYNC_THRESHOLD是一个宏，大小定义为0.01。因为AV_SYNC_THRESHOLD作为阈值只是一个猜测，我们将delay加入作为猜测才能更好适应更多的输入文件，这样设计音频与视频显示最多也就差一个视频的时间戳。		
#结束
自己的理解大概就是这些，如果有错误，希望能指出，有什么问题想讨论可以发邮件到jered@gmail.com。


  
  