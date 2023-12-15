# RTSP推流实现



![image-20231108113434591](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231108113434591.png)

![image-20231108114654721](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231108114654721.png)

## 线程父类：

由AudioCapturer和VideoCapturer线程继承。

![image-20231108210153762](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231108210153762.png)

```c++
RET_CODE Start(){
    worker = new thread(trampoline , this);
}

void * trampoline (void *p){
     ((CommonLooper*)p)->Loop();//多态，子线程(子类重写Loop（）).
}

void CommonLooper::Stop()
{
    request_abort_ = true;
    if(worker_) {
        worker_->join();
        delete worker_;
        worker_ = NULL;
    }
}
```



### 捕获音频AudioCapturer线程：

重点：实现Loop()：捕获数据。

属性：PCM缓冲区、音频帧属性、回调函数、

```c++
RET_CODE AudioCapturer::Init(const Properties properties){
     input_pcm_name_ = properties.GetProperty("input_pcm_name", "buweishui_48000_2_s16le.pcm");
     sample_rate_ = properties.GetProperty("sample_rate", 48000);
     ......
     pcm_buf_size_ = byte_per_sample_ *channels_ *  nb_samples_;
     pcm_buf_ = new uint8_t[pcm_buf_size_];
     frame_duration_ = 1.0 * nb_samples_ / sample_rate_ * 1000;  // 得到一帧的毫秒时间
}
//添加回调函数  std::function<void(uint8_t *, int32_t)> callback_get_pcm_
void AudioCapturer::AddCallback(function<void (uint8_t *, int32_t)> callback)
{
    callback_get_pcm_ = callback;
}
    /* 如何按照播放速率读取：也就是什么时候正常从文件中读取PCM数据到缓冲区,播放完一帧在读取
          按照时间轴顺序播放，当前运行时间-first_frame_time（第一帧的开始时间） = diff
          当diff大于total_frame_duration时，说明该读取下一帧数据了 */
 int AudioCapturer::readPcmFile(uint8_t *pcm_buf, int32_t pcm_buf_size){
     int64_t cur_time  = TimesUtil::GetTimeMillisecond();     // 单位毫秒
     int64_t diff = cur_time - pcm_start_time;
     if(diff > (int64_t)pcm_total_duration_ ){
         size_t ret = fread(pcm_buf_, 1, pcm_buf_size, pcm_fp_);
         pcm_total_duration_ += frame_duration_;
         return 0 ;
     }else{
         retun -1 ;
     }
 }         

//循环读取数据
void AudioCapturer::Loop(){
     pcm_total_duration_ = 0;
     pcm_start_time_ = TimesUtil::GetTimeMillisecond();
     while(true){
        if(request_abort_) {
            break;          // 请求退出
        }
         if((readPcmFile(pcm_buf_, pcm_buf_size_)==0){//0 说明成功读取数据到缓冲区
             
             if(callback_get_pcm_) {//调用回调函数进行下一步处理：在PushWork中实现：对PCM_YUV进行编码函数
                callback_get_pcm_(pcm_buf_, pcm_buf_size_);
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(2)); //等待2 time 
         }
     }
     closePcmFile();
}

```



### 捕获视频VedioCapturer线程：

```c++
//实现逻辑和捕获音频线程实现类似。
```



## 总管PushWork：

对外提供接口，所有的属性都先由PW接收，然后再转接到其他接口。

管理AudioCapturer *audio_capturer_ 和 VideoCapturer *video_capturer_两个线程的操作。

两个捕获线程又分别控制对应编码操作。



通常情况下，通过FFmpeg提取的PCM音频数据是以交错（interleaved）的方式存储，而不是平面（planar）模式。

所以要注意编码器和frame的format 是否和原始PCM的format一致，不一致需要重采样。



### InIt():

```c++
class PushWork {
  public:
    PushWork();
    RET_CODE Init(const Properties &properties);
    RET_CODE DeInit();
  private:
    void PcmCallback(uint8_t *pcm, int32_t size);
    void YuvCallback(uint8_t* yuv, int32_t size);
  private: 
    //属性信息：捕获线程类所需要的信息由PushWork属性转交：由InIt()初始化并且转交
    AudioCapturer *audio_capturer_ = NULL;
    VideoCapturer *video_capturer_ = NULL;
    AACEncoder  *audio_encoder_ = NULL;
    H264Encoder *video_encoder_ = NULL;
    ......
        // 麦克风采样属性
         int mic_sample_rate_ = 48000;
         ......
        // 桌面录制属性
          int desktop_x_ = 0;
          int desktop_width_  = 1920;
         ......
        // 音频编码参数
          int audio_sample_rate_ = AV_SAMPLE_FMT_S16;
         ......
        // 视频编码属性     
          int video_width_ = 1920; 
         ......
             
             
             
}


RET_CODE PushWork::Init(const Properties &properties){
    //设置麦克风采样属性
     mic_sample_rate_ = properties.GetProperty("mic_sample_rate", 48000);
     ......
    // 音频编码参数
     audio_sample_rate_ = properties.GetProperty("audio_sample_rate", mic_sample_rate_);
     ......
    // 视频编码属性
     video_gop_ = properties.GetProperty("video_gop", video_fps_);
     ......
         
    --------------------------------------音频编码器初始化start---------------------------------------  
    //音频编码器初始化 And Frame属性分配
        audio_encoder_ = new AACEncoder(); 
        Properties  aud_codec_properties;
        aud_codec_properties.SetProperty("sample_rate", audio_sample_rate_);
          //...SetProperty...//
        audio_encoder_->Init(aud_codec_properties)；
          //分配帧内存、创建帧属性
        fltp_buf_size_ = av_samples_get_buffer_size(XXXXX);
        fltp_buf_ = (uint8_t *)av_malloc(fltp_buf_size_);
    
        audio_frame_ = av_frame_alloc();    
          audio_frame_->format = audio_encoder_->GetFormat();
          audio_frame_->XXXX = YYYY；
        av_frame_get_buffer(audio_frame_, 0);    
    
    //视频编码器初始化
         video_encoder_ = new H264Encoder();
         Properties  vid_codec_properties;
         vid_codec_properties.SetProperty("width", video_width_);
           //...SetProperty...//
         video_encoder_->Init(vid_codec_properties);
    ----------------------------------音频编码器初始化end-------------------------------------------
        
    ------------------------------------RTSP推流线程start-----------------------------------------
    // 在音视频编码器初始化完， 音视频捕获前
    rtsp_pusher_ =new RtspPusher(msg_queue_);
    rtsp_properties.SetProperty("url", rtsp_url_);
    rtsp_properties.SetProperty("timeout", rtsp_timeout_);
    rtsp_properties.SetProperty("rtsp_transport", rtsp_transport_);
        //...SetProperty...//
    // 创建音频流、音视频流
    rtsp_pusher_->ConfigVideoStream(video_encoder_->GetCodecContext())；
    rtsp_pusher_->ConfigAudioStream(audio_encoder_->GetCodecContext())；  
        
    //启动RTSP推流线程，并且开始读取队列中的数据进行send推流
    rtsp_pusher_->Connect();
    ------------------------------------RTSP推流线程end-----------------------------------------
    
        
    ------------------------------------音视频捕获线程start-----------------------------------------
    //设置音频和视频捕获的属性，属性转接
    audio_capturer_ = new AudioCapturer();
    Properties aud_cap_properties;
    aud_cap_properties.SetProperty("input_pcm_name", input_pcm_name_);
        //...SetProperty...//
          
    //音频捕获线程启动！ 视频同理
    audio_capturer_->Init(aud_cap_properties);
    audio_capturer_->AddCallback(std::bind(&PushWork::PcmCallback, this, std::placeholders::_1,
                                           std::placeholders::_2));
    audio_capturer_->Start();
    ------------------------------------音视频捕获线程end-----------------------------------------    
    return RET_OK;
}
```



### PcmCallback：

重采样：交错->平面、PCM编码操作、

frame->pts = pts ;和后经过编码器得到的packet->pts未必一致。

```c++
void s16le_convert_to_fltp(short *s16le, float *fltp, int nb_samples) {
    float *fltp_l = fltp;   // -1~1
    float *fltp_r = fltp + nb_samples;
    for(int i = 0; i < nb_samples; i++) {
        //转化成float：因为short 范围：-32768 ~ 32767。所以除以。 
        fltp_l[i] = s16le[i*2]/32768.0;     // 0 2 4  
        fltp_r[i] = s16le[i*2+1]/32768.0;   // 1 3 5
    }
}

void PushWork::PcmCallback(uint8_t *pcm, int32_t size){
    //保存重采样之前的数据
    pcm_s16le_fp_ = fopen("push_dump_s16le.pcm", "wb");    
    fwrite(pcm, 1, size, pcm_s16le_fp_);
    /*重采样：这里假设只有fmt格式：2通道s16交错-> float planner需要变换，其余参数都一致，
             所以不必用到重采样器，直接手动数据格式转化*/
    s16le_convert_to_fltp((short *)pcm, (float *)fltp_buf_, audio_frame_->nb_samples);
    av_frame_make_writable(audio_frame_);
    // 将fltp_buf_写入frame
    ret = av_samples_fill_arrays(audio_frame_->data,
                                 audio_frame_->linesize,
                                 fltp_buf_,
                                 audio_frame_->channels,
                                 audio_frame_->nb_samples,
                                 (AVSampleFormat)audio_frame_->format,
                                 0);
    //frame进行编码
    AVPacket *packet = audio_encoder_->Encode(audio_frame_, pts, 0, &pkt_frame, &encode_ret);
    aac_fp_ = fopen("push_dump.aac", "wb");
    audio_encoder_->GetAdtsHeader(adts_header, packet->size)；
    fwrite(adts_header, 1, 7, aac_fp_);
    fwrite(packet->data, 1, packet->size, aac_fp_);
    //编码成功后，应该放入队列中。
    rtsp_pusher_->Push(packet, E_AUDIO_TYPE);
             // Push(packet, E_AUDIO_TYPE)    ： queue_->Push(pkt, media_type);
}
```



### YuvCallback：

对packet写入start_code、pps、sps。

```c++
void PushWork::YuvCallback(uint8_t *yuv, int32_t size){
    int64_t pts = (int64_t)AVPublishTime::GetInstance()->get_audio_pts();
    AVPacket *packet = video_encoder_->Encode(yuv, size, pts,  &pkt_frame, &encode_ret);
    if(packet){
        h264_fp_ = fopen("push_dump.h264", "wb");
        // 写入sps 和pps
          uint8_t start_code[] = {0, 0, 0, 1};
          fwrite(start_code, 1, 4, h264_fp_);
          fwrite(video_encoder_->get_sps_data(), 1, video_encoder_->get_sps_size(), h264_fp_);
          fwrite(start_code, 1, 4, h264_fp_);
          fwrite(video_encoder_->get_pps_data(), 1, video_encoder_->get_pps_size(), h264_fp_);
    }
    fwrite(packet->data,1,  packet->size,h264_fp_);
    
    rtsp_pusher_->Push(packet, E_VIDEO_TYPE);
}
```

### ~PushWork()：

```c++
PushWork::~PushWork(){
    // 从源头开始释放资源
    // 先释放音频、视频捕获
    if(audio_capturer_) {
        delete audio_capturer_;
    }
    if(video_capturer_) {
        delete video_capturer_;
    }
    if(audio_encoder_) {
        delete audio_encoder_;
    }
    if(fltp_buf_) {
        av_free(fltp_buf_);
    }
    if(audio_frame_) {
        av_frame_free(&audio_frame_);
    }
    if(video_encoder_) {
        delete video_encoder_;
    }
    if(rtsp_pusher_) {
        delete rtsp_pusher_;
    }
    LogInfo("~PushWork()");
}
```



## 音频编码器AACEncoder封装：



```c++
//属性：
    int sample_rate_ = 48000;
    int channels_    = 2;
    int bitrate_     = 128*1024;
    int channel_layout_ = AV_CH_LAYOUT_STEREO;
    AVCodec *codec_         = NULL;
    AVCodecContext  *ctx_   = NULL;

AACEncoder::~AACEncoder()
{
    if(ctx_) {
        avcodec_free_context(&ctx_);//内部实现了关闭编码器操作
    }
}
```



### Init():

```c++
RET_CODE AACEncoder::Init(const Properties &properties){
        // 获取参数
    sample_rate_ = properties.GetProperty("sample_rate", 48000);
    ......
         // 查找编码器
    codec_ = avcodec_find_encoder(AV_CODEC_ID_AAC);
         // 分配编码器上下文
    ctx_ = avcodec_alloc_context3(codec_);
         //设置上下文参数
    ctx_->channels      = channels_;
    ctx_->sample_fmt    = AV_SAMPLE_FMT_FLTP;      // 默认aac编码需要planar格式PCM
    ......
        //关联
    avcodec_open2(ctx_, codec_, NULL)；
    return RET_OK;
}
```



### Encode:

```c++
//对输入的frame进行编码操作
AVPacket *AACEncoder::Encode(AVFrame *frame, const int64_t pts, int flush, int *pkt_frame, RET_CODE *ret){
    if(frame){
        frame->pts = pts;
        ret1 = avcodec_send_frame(ctx_, frame);
        if(ret1<0){
            when : ret1 == AVERROR(EAGAIN); //赶紧读取packet，我frame send不进去了
                   ret1 == AVERROR_EOF
                   ......    
        }// <0 不能正常处理该frame
    }
    
    AVPacket *packet = av_packet_alloc();
    ret1 = avcodec_receive_packet(ctx_, packet);
    //条件判断 ret1 ......
    
    return packet;    
}
```



## 视频编码器H264Encoder封装：

`ctx_->extradata` 是 FFmpeg 中的 AVCodecContext 结构体中的一个成员，用于存储额外的编码器或解码器相关数据。

通常，它用于存储一些编码器的配置信息，例如 H.264 编码器可能会在这里存储 SPS（Sequence Parameter Set）和 PPS（Picture Parameter Set）等信息。

![image-20231110162504510](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231110162504510.png)

属性设置：

```c++
private:
    int width_ = 0;
    int height_ = 0;
    int fps_ = 0;       // 帧率
    int b_frames_ = 0;   // 连续B帧的数量
    int bitrate_ = 0;   // 码率
    int gop_ = 0;
    bool annexb_  = false;
    int threads_ = 1;
    int pix_fmt_ = 0;
    //    std::string profile_;
    //    std::string level_id_;

    std::string sps_;
    std::string pps_;
    std::string codec_name_;
    AVCodec *codec_         = NULL;
    AVCodecContext  *ctx_   = NULL;
    AVDictionary *dict_ = NULL;
```



### InIt():

```c++
int H264Encoder::Init(const Properties &properties){
    //设置属性
    width_ = properties.GetProperty("width", 0);
    //.height.fps. b_frames......
    
    //find 编码器：Or codec_ = avcodec_find_encoder_by_name(codec_name_.c_str());
    codec_ = avcodec_find_encoder(AV_CODEC_ID_H264);
    ctx_ = avcodec_alloc_context3(codec_);
    //编码器上下文CTX设置参数
    ctx_->width = width_;
    ctx_->gop_size = gop_;
    ctx->time_base = (AVRational){1, fps};
    ctx->framerate = (AVRational){fps, 1};
    .......//ctx_->pix_fmt\ctx_->codec_type\ctx_->max_b_frames\
    av_opt_set(codec_ctx->priv_data, "preset", "medium", 0);
    //ctx_->flags |= AV_CODEC_FLAG_GLOBAL_HEADER;//存储头部信息：存本地文件时不要去设置
    
    //关联编码器
    avcodec_open2(ctx_, codec_, &dict_);
    
    // 从extradata读取sps pps
    
        if(ctx_->extradata) {
        LogInfo("extradata_size:%d", ctx_->extradata_size);
        // 第一个为sps 7
        // 第二个为pps 8

        uint8_t *sps = ctx_->extradata + 4;    // 直接跳到数据
        int sps_len = 0;
        uint8_t *pps = NULL;
        int pps_len = 0;
        uint8_t *data = ctx_->extradata + 4;
        for (int i = 0; i < ctx_->extradata_size - 4; ++i)
        {
            if (0 == data[i] && 0 == data[i + 1] && 0 == data[i + 2] && 1 == data[i + 3])
            {
                pps = &data[i+4];
                break;
            }
        }
        sps_len = int(pps - sps) - 4;   // 4是00 00 00 01占用的字节
        pps_len = ctx_->extradata_size - 4*2 - sps_len;
        sps_.append(sps, sps + sps_len);
        pps_.append(pps, pps + pps_len);
    }
    
    //frame 属性分配
    frame_ = av_frame_alloc();
    frame_->width = width_;
    frame_->height = height_;
    frame_->format = ctx_->pix_fmt;
    ret = av_frame_get_buffer(frame_, 0);
    return RET_OK;
}
```



### Encode:

```c++
AVPacket *H264Encoder::Encode(uint8_t *yuv, int size, int64_t pts, int *pkt_frame,RET_CODE *ret){
    if(yuv){
        av_image_fill_arrays(frame_->data, frame_->linesize, yuv,
                                         (AVPixelFormat)frame_->format,
                                         frame_->width, frame_->height, 1);
        frame_->pts = pts;
        avcodec_send_frame(ctx_, frame_);        
    }
    AVPacket *packet = av_packet_alloc();
    ret1 = avcodec_receive_packet(ctx_, packet);
    return packet;
}
```



## 队列设计：

![image-20231110203649085](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231110203649085.png)



因为有视频和音频两种类型的包，所以Push到同一个队列时要做区分，故创建一个MyAVPacket结构体。

另外需要记录并且统计队列中的数据信息，故创建一个stats结构体。

```c++
typedef enum media_type {
    E_MEDIA_UNKNOWN = -1,
    E_AUDIO_TYPE,
    E_VIDEO_TYPE
}MediaType;

typedef struct my_avpacket{
    AVPacket *pkt;
    MediaType media_type;
}MyAVPacket;

typedef struct packet_queue_stats
{
    int audio_nb_packets;   // 音频包数量
    int video_nb_packets;   // 视频包数量
    int audio_size;         // 音频总大小 字节
    int video_size;         // 视频总大小 字节
    int64_t audio_duration; //音频持续时长
    int64_t video_duration; //视频持续时长
}PacketQueueStats;
```

### 类属性：

![image-20231111103813221](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231111103813221.png)

```c++
class PacketQueue{
    public:
      PacketQueue(double audio_frame_duration, double video_frame_duration){
          memset(&stats_, 0, sizeof(PacketQueueStats));
      };
     int Push(AVPacket *pkt, MediaType media_type);
     int pushPrivate(AVPacket *pkt, MediaType media_type);
     int Pop(AVPacket **pkt, MediaType &media_type);
     int Drop(bool all, int64_t remain_max_duration)；
         
         
    //下面函数都需要加锁进行获取
    bool Empty();
    void Abort()
    {
        std::lock_guard<std::mutex> lock(mutex_);
        abort_request_ = true;
        cond_.notify_all();
    }
    //...Get_XXXX()...
  int64_t GetAudioDuration()
    {
        std::lock_guard<std::mutex> lock(mutex_);
        int64_t duration = audio_back_pts_ - audio_front_pts_;  //以pts为准
        // 也参考帧（包）持续 *帧(包)数
        if(duration < 0     // pts回绕
                || duration > audio_frame_duration_ * stats_.audio_nb_packets * 2) {
            duration =  audio_frame_duration_ * stats_.audio_nb_packets;
        } else {
            duration += audio_frame_duration_;//
        }
       //下面应该有一个条件判断：当queue.empty()时，duration = 0；
        return duration;
    }
    
 private:
    td::mutex mutex_;
    std::condition_variable cond_;
    std::queue<MyAVPacket *> queue_;
    bool abort_request_ = false;
    // 统计相关
    PacketQueueStats stats_;
    
    double audio_frame_duration_ = 23.21995649; // 默认23.2ms 44.1khz  1024*1000ms/44100=23.21995649ms
    double video_frame_duration_ = 40;  // 40ms 视频帧率为25的  ， 1000ms/25=40ms
    // pts记录
    int64_t audio_front_pts_ = 0;
    int64_t audio_back_pts_ = 0;
    int     audio_first_packet = 1;
    int64_t video_front_pts_ = 0;
    int64_t video_back_pts_ = 0;
    int     video_first_packet = 1;
};
```



### Push()：

互斥锁操作，判断包类型，进而pushPrivate(pkt, media_type)。

并且实时更新back_pts_ = pkt->pts的值。

```c++
int Push(AVPacket *pkt, MediaType media_type){
    //先进行条件判断：pkt存在，类型是否正确
    ......
    //加锁
    lock_guard<std::mutex> lock (m_mutex);
    //Push
    pushPrivate(pkt, media_type);
    m_cond.notify_one();
    return 0;
}

int pushPrivate(AVPacket *pkt, MediaType media_type){
    //条件判断：
    /*1.m_abort_request 是否退出程序
      2.包类型判断 */
     MyAVPacket *mypkt = (MyAVPacket *)malloc(sizeof(MyAVPacket));
     mypkt->pkt = pkt;
     mypkt->media_type = media_type;
    //实时更新队列状态结构体：m_stats
    stats_.audio_nb_packets++; 
    stats_.audio_size += pkt->size;
    audio_back_pts_ = pkt->pts;
     if(audio_first_packet) {
       audio_first_packet  = 0;
       audio_front_pts_ = pkt->pts;
     }
    queue_.push(mypkt); 
    return 0;
}
```



### Pop():

互斥锁操作，判断包类型，进而pushPrivate(pkt, media_type)。

并且实时更新front_pts_ = pkt->pts的值。

```c++
int Pop(AVPacket **pkt, MediaType &media_type){
    //一系列条件判断：pkt、abort_request、
    ......
    //加锁操作
    std::unique_lock<std::mutex> lock(mutex_);
    
    //POP等待队列wait :为空则等待数据 或者 超时等待 或者 程序退出
            if(queue_.empty()) {        // 等待唤醒
            // return如果返回false，继续wait, 如果返回true退出wait
            cond_.wait(lock, [this] {
                return !queue_.empty() | abort_request_;
            });
                //超时时间判断
            /*cond_.wait_for(lock, std::chrono::milliseconds(timeout), [this] {
                return !queue_.empty() | abort_request_; */
        }
             if(abort_request_) {
            LogWarn("abort request");
            return -1;
        }   
     //POP操作
        MyAVPacket *mypkt = queue_.front();
        *pkt        = mypkt->pkt;
        media_type  = mypkt->media_type;
     //实时更新队列状态stats：并且更新pts
            if(E_AUDIO_TYPE == media_type) {
            stats_.audio_nb_packets--;      // 包数量
            stats_.audio_size -= mypkt->pkt->size;
            // 持续时长怎么统计，不是用pkt->duration
                当pop后队列为空，back和front的pts应该都为0，且持续时间为0
            audio_front_pts_ = mypkt->pkt->pts;
        }
    queue_.pop();
    free(mypkt);
    return 1;
}
```



### Drop():

这个函数的目的是删除队列中符合条件的第一个数据包，并根据数据包的类型更新相关的统计信息。如果队列中有多个符合条件的数据包，只会删除第一个遇到的。

![image-20231111103835799](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231111103835799.png)

```c++
int Drop(bool all, int64_t remain_max_duration)
{
    std::lock_guard<std::mutex> lock(mutex_);  // 使用互斥锁确保线程安全访问共享资源

    while (!queue_.empty()) {  // 只要队列不为空就循环处理队列中的元素
        MyAVPacket *mypkt = queue_.front();  // 获取队列的头部元素

        // 如果不是全部删除（all=false），且是视频类型（E_VIDEO_TYPE），且是关键帧
        if (!all && mypkt->media_type == E_VIDEO_TYPE && (mypkt->pkt->flags & AV_PKT_FLAG_KEY)) {
            int64_t duration = video_back_pts_ - video_front_pts_;  // 以pts为准的持续时间

            // 如果持续时间小于0或者超过了阈值，使用阈值作为持续时间
            if (duration < 0 || duration > video_frame_duration_ * stats_.video_nb_packets * 2) {
                duration = video_frame_duration_ * stats_.video_nb_packets;
            }
            duration +=audio_frame_duration_;
            LogInfo("video duration:%lld", duration);
            
            // 如果视频持续时间小于等于给定的最大保留时长，就退出循环
            if (duration <= remain_max_duration)
                break;
        }

        // 处理音频类型
        if (E_AUDIO_TYPE == mypkt->media_type) {
            stats_.audio_nb_packets--;      // 减少音频包数量
            stats_.audio_size -= mypkt->pkt->size;
            audio_front_pts_ = mypkt->pkt->pts;  // 更新音频的前一个pts
        }

        // 处理视频类型
        if (E_VIDEO_TYPE == mypkt->media_type) {
            stats_.video_nb_packets--;      // 减少视频包数量
            stats_.video_size -= mypkt->pkt->size;
            video_front_pts_ = mypkt->pkt->pts;  // 更新视频的前一个pts
        }

        av_packet_free(&mypkt->pkt);  // 释放AVPacket资源
        queue_.pop();  // 从队列中移除元素
        free(mypkt);  // 释放MyAVPacket资源
    }

    return 0;
}

```



## RTSP推流模块：

继承线程父类，当音视频线程捕获模块成功捕获并且编码成pkt后，要push到队列中(在pushwork::callback()实现)，然后在loop()中进行pop、send推流。

和封装aac、h264到mp4方法类似，创建音视频流、输出上下文、然后进行send操作。



### 超时回调：

  具体的调用时机取决于 FFmpeg 执行的操作，比如读取数据、打开网络连接等。当 FFmpeg 执行这些操作时，会定期调用中断回调函数来检查是否需要中断当前操作。

```c++
static int  decode_interrupt_cb(void *ctx)
{
    RtspPusher *rtsp_puser = (RtspPusher *)ctx;
    if(rtsp_puser->IsTimeout()) {
        LogWarn("timeout:%dms", rtsp_puser->GetTimeout());
        return 1;
    }
    //    LogInfo("block time:%lld", rtsp_puser->GetBlockTime());
    return 0;
}
//比如要读取数据了，现在read代码前调用RestTiemout()充值pre，然后系统会自动调用 中断回调函数判断超时。
void RtspPusher::RestTiemout()
{
    pre_time_ = TimesUtil::GetTimeMillisecond();        // 重置为当前时间
}

bool RtspPusher::IsTimeout()
{
    if(TimesUtil::GetTimeMillisecond() - pre_time_ > timeout_) {
        return true;    // 超时
    }
    return false;
}


```



### InIt:

```c++
RET_CODE RtspPusher::Init(const Properties &properties){
    //首先对类属性进行设置
    url_ = properties.GetProperty("url", "");
    rtsp_transport_ = properties.GetProperty("rtsp_transport", "");
    ......
        
    // 初始化网络库
    ret = avformat_network_init();
    char str_error[512] = {0};
    if(ret < 0) {
        av_strerror(ret, str_error, sizeof(str_error) -1);
        LogError("avformat_network_init failed:%s", str_error);
        return RET_FAIL;
    }  
    // 分配AVFormatContext
    avformat_alloc_output_context2(&fmt_ctx_, NULL, "rtsp", url_.c_str());
    av_opt_set(fmt_ctx_->priv_data, "rtsp_transport", rtsp_transport_.c_str(), 0);
    
    fmt_ctx_->interrupt_callback.callback = decode_interrupt_cb;    // 设置超时回调
    fmt_ctx_->interrupt_callback.opaque = this;
    // 创建队列
    queue_ = new PacketQueue(audio_frame_duration_, video_frame_duration_);
    return RET_OK;
}
```



### 创建音视频流：

```c++
RET_CODE RtspPusher::ConfigVideoStream(const AVCodecContext *ctx){
    AVStream *vs = avformat_new_stream(fmt_ctx_, NULL);
    avcodec_parameters_from_context(vs->codecpar, ctx);
    
    video_ctx_ = (AVCodecContext *) ctx;
    video_stream_ = vs;
    video_index_ = vs->index;       // 索引非常重要 fmt_ctx_根据index判别 音视频包
    return RET_OK 
}
```



### Connect:

```c++
RET_CODE RtspPusher::Connect()
{
    if(!audio_stream_ && !video_stream_) {
        return RET_FAIL;
    }
    RestTiemout();//重置时间，调用中断回调函数
    // 连接服务器
    int ret = avformat_write_header(fmt_ctx_, NULL);
   //进行LOOP（）操作
    return Start();     // 启动线程 
}
```



### 消息队列机制：

![image-20231113205914478](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231113205914478.png)

有各种类型的错误消息统一存入到消息队列中进行处理。

设置struct AVMessage ，要有适用性，可以接受不同类型的消息。

需要互斥锁操作

```c++
msg_queue_->notify_msg3(MSG_RTSP_QUEUE_DURATION, stats.audio_duration, stats.video_duration);
msg_queue_->notify_msg2(MSG_RTSP_ERROR, ret);
......
```

```c++
#define MSG_FLUSH                   1
#define MSG_RTSP_ERROR              100
#define MSG_RTSP_QUEUE_DURATION     101
对应what
typedef struct AVMessage
{
    int what;           // 消息类型
    int arg1;
    int arg2;
    void *obj;          //如果2个参数不够用，则传入结构体
    void (*free_l)(void *obj);
}AVMessage;
    inline void msg_init_msg(AVMessage *msg)
    {
        memset(msg, 0, sizeof(AVMessage));
    }
//存入消息
    int msg_queue_put(AVMessage *msg)
    {
        std::lock_guard<std::mutex> lock(mutex_);
        int ret = msg_queue_put_private(msg);
        if(0 == ret) {
            cond_.notify_one();     // 正常插入队列了才会notify
        }
        return ret;
    }

 // 把队列里面what类型的消息全部删除
    void msg_queue_remove(int what)
    {
        std::lock_guard<std::mutex> lock(mutex_);
        while(!abort_request_ && !queue_.empty()) {
            std::list<AVMessage *>::iterator it;
            AVMessage *msg = NULL;
            for(it = queue_.begin(); it != queue_.end(); it++) {
                if((*it)->what == what) {
                    msg = *it;
                    break;
                }
            }
            if(msg) {
                if(msg->obj && msg->free_l) {
                    msg->free_l(msg->obj);
                }
                av_free(msg);
                queue_.remove(msg);
            } else {
                break;
            }
        }
//消息类型存入：
    void notify_msg4(int what, int arg1, int arg2, void *obj, int obj_len)
    {
        AVMessage msg;
        msg_init_msg(&msg);
        msg.what = what;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        msg.obj = av_malloc(obj_len);
        msg.free_l = msg_obj_free_l;
        memcpy(msg.obj, obj, obj_len);
        msg_queue_put(&msg);
    }    
        
    void notify_msg3(int what, int arg1, int arg2)
    {
        AVMessage msg;
        msg_init_msg(&msg);
        msg.what = what;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        msg_queue_put(&msg);
    }

```

### Get():

get取出消息、

```c++

    // 返回值：-1代表abort; 0 代表没有消息;  1代表读取到了消息
    // timeout: -1代表阻塞等待; 0; 代表非阻塞等待; >0 代表有超时的等待; -2 代表参数异常
    int msg_queue_get(AVMessage *msg, int timeout)
    {
        if(!msg) {
            return -2;
        }
        std::unique_lock<std::mutex> lock(mutex_);
        AVMessage *msg1;
        int ret;
        for(;;) {
            if(abort_request_) {
                ret = -1;
                break;
            }
            if(!queue_.empty()) {
                msg1= queue_.front();
                *msg = *msg1;
                queue_.pop_front();
                av_free(msg1);      // 释放msg1
                ret = 1;
                break;
            } else if(0 == timeout) {
                ret = 0;        // 没有消息
                break;
            } else if(timeout < 0){
                cond_.wait(lock, [this] {
                    return !queue_.empty() | abort_request_;    // 队列不为空或者abort请求才退出wait
                });
            } else if(timeout > 0) {
//                LogInfo("wait_for into");
                cond_.wait_for(lock, std::chrono::milliseconds(timeout), [this] {
//                    LogInfo("wait_for leave");
                    return !queue_.empty() | abort_request_;        // 直接写return true;是错误的
                });
                if(queue_.empty()) {
                    ret = 0;
                    break;
                }
            } else {
                ret = -2;
                break;
            }
        }
        return ret;
    }
```



### SendPacket():

把pkt赋值对应流index，并且av_rescale_q ：TimeBase转化成流的单位

```c++
int RtspPusher::sendPacket(AVPacket *pkt, MediaType media_type){
    AVRational dst_time_base;
    AVRational src_time_base = {1, 1000};      // 我们采集、编码 时间戳单位都是ms
    
    if(E_VIDEO_TYPE == media_type) {
        pkt->stream_index = video_index_;
        dst_time_base = video_stream_->time_base;
    } else if(E_AUDIO_TYPE == media_type) {}
    
    pkt->pts = av_rescale_q(pkt->pts, src_time_base, dst_time_base);
    pkt->duration = 0;
    RestTiemout();//重置时间后自动调用中断回调函数
    int ret = av_write_frame(fmt_ctx_, pkt);
    
    //消息队列机制
     if(ret < 0) {
        msg_queue_->notify_msg2(MSG_RTSP_ERROR, ret);//发送消息
        char str_error[512] = {0};
        av_strerror(ret, str_error, sizeof(str_error) -1);
        LogError("av_write_frame failed:%s", str_error);        // 出错没有回调给PushWork
        return -1;
    }
    
    return 0;
}
```



### Loop():

推流线程循环Pop操作取出队列中pkt和所属类型 、然后进行send 发送数据包推流。

```c++
void RtspPusher::Loop(){
    AVPacket *pkt = NULL;
    MediaType media_type;
    while (true) {
        if(request_abort_) {
            LogInfo("abort request");
            break;
        }
        //多久打印一次debug信息
        debugQueue(debug_interval_);
        //检查队列中总数据包持续时长是否过大，否则drop直到符合条件。
        checkPacketQueueDuration(); // 可以每隔一秒check一次
        
        ret = queue_->PopWithTimeout(&pkt, media_type, 1000);
        if(1 == ret) {  // 1代表 读取到消息
            if(request_abort_) {
                LogInfo("abort request");
                av_packet_free(&pkt);
                break;
            }
            switch (media_type) {
            case E_VIDEO_TYPE:
                ret = sendPacket(pkt, media_type);
                av_packet_free(&pkt);
                break;
            case E_AUDIO_TYPE:
                ret = sendPacket(pkt, media_type);
                av_packet_free(&pkt);
                break;
            default:
                break;
            }
        }
}
    //检查队列中总持续时长是否过大，否则drop直到符合条件。
void RtspPusher::checkPacketQueueDuration()
{
    PacketQueueStats stats;
    queue_->GetStats(&stats);
    if(stats.audio_duration > max_queue_duration_ || stats.video_duration > max_queue_duration_) {
        msg_queue_->notify_msg3(MSG_RTSP_QUEUE_DURATION, stats.audio_duration, stats.video_duration);
        LogWarn("drop packet -> a:%lld, v:%lld, th:%d", stats.audio_duration, stats.video_duration,
                max_queue_duration_);
        queue_->Drop(false, max_queue_duration_);
    }
}
```





## Main.cpp:

```c++
#define RTSP_URL "rtsp://192.168.0.148/live/livestream"
int main(){
    MessageQueue *msg_queue_ = new MessageQueue();
    {
    PushWork push_work;
    Properties properties;
    
    ......
    properties.SetProperty("audio_test", 1);    // 设置属性
    // 音频test模式
    // 麦克风采样属性
    // 音频编码属性
    
    //视频test模式
    // 桌面录制属性
    // 视频编码属性
    ......
        // 配置rtsp
        //1.url
        //2.udp
        properties.SetProperty("rtsp_url", RTSP_URL);
        properties.SetProperty("rtsp_transport", "tcp");    // udp or tcp
        properties.SetProperty("rtsp_timeout", 15000);
        properties.SetProperty("rtsp_max_queue_duration", 1000);
               
    push_work.Init(properties);   
    
    
    //循环从消息队列中获取消息并且打印
        AVMessage msg;
        int ret = 0;
        while (true) {
//            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
            ret = msg_queue_->msg_queue_get(&msg, 1000);
            if(1 == ret) {
                switch (msg.what) {
                case MSG_RTSP_ERROR:
                    LogError("MSG_RTSP_ERROR error:%d", msg.arg1);
                    break;
                case MSG_RTSP_QUEUE_DURATION:
                    LogError("MSG_RTSP_QUEUE_DURATION a:%d, v:%d", msg.arg1, msg.arg2);
                    break;
                default:
                    break;
                }
            }
            
        }
        delete msg_queue_;
        
        return 0;
}
    
```

























































