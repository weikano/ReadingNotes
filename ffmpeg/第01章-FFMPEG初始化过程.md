### 第01章-FFMPEG初始化过程

> **基于FFMPEG 4.1**

#### 1. av_register_all

> av_register_all、av_register_input_format、av_register_output_format都被标记为deprecated，但是在当前脚本编译下，他们的实现都是一样的。下面以av_register_all来说明

```c++
//allformats.c
void av_register_all() {
  //ff_thread_once是基于pthread_once定义的宏，实际上是调用av_format_init_next方法
  ff_thread_once(&av_format_next_init, av_format_init_next);
}
//这个方法其实没什么用处
//muxter用于将音视频编码后的数据（可能还有字幕）封装成一个文件，比如MP4类型的封装格式
//demuxer则用于解封装。
static void av_format_init_next(void) {
  //muxter
  AVOuputFormat* prevout = nullptr, *out;
 	//demuxer
  AVInputFormat* previn = nullptr, *in;
  for(int i=0;(out = const_cast<AVOuputFormat*>muxer_list[i]);i++) {
    if(prevout) {
      prevout->next = out;
    }
    prevout = out;
  }
  //muxter处理类似
}
```

#### 2. avcodec_register_all

> 用于注册所有的编解码器，也可以使用av_register_codec_parser和av_register_bitstream_filter来指定注册

```c++
//allcodecs.h
void avcodec_register_all() {
  ff_thread_once(&av_codec_next_init, av_codec_init_next);
}
static void av_codec_init_next() {
  AVCodec* prev = nullptr, *p;
  void* i= 0;
  //av_codec_iterate遍历并初始化AVCodec
  //最终会调用AVCodec.init_static_data方法
  while((p=const_cast<AVCodec*>av_codec_iterate(&i))) {
    if(prev) {
      prev->next = p;
    }
    prev = p;
  }
}

const AVCodec* av_codec_iterate(void** opaque) {
  uintptr_t i = (uintptr_t)(opaque);
  const AVCodec* c = codec_list[i];
  ff_thread_once(&av_codec_static_init, av_codec_init_static);
  if(c) {
    *opaque=(void*)(i+1);
  }
  return c;
}

static void av_codec_init_static(void){
  for (int i = 0; codec_list[i]; i++) {
    if (codec_list[i]->init_static_data){
      codec_list[i]->init_static_data((AVCodec*)codec_list[i]);
    }    
  }
}

```

#### 3. avformat_network_init

> 在调用avformat_open_input时，如果传入的URL为网络链接，那么需要调用该方法来初始化网络协议

```c++
//utils.c
int avformat_network_init() {
#if CONFIG_NETWORK
  int ret;
  //处理winsock，在Linux平台上不进行任何处理
  if((ret = ff_network_init())<0) {
    return ret;
  }
  //根据编译FFMPEG中configure时的配置：--enable-openssl或者--enable-gnutls来决定是哪种tls方式，暂时分析openssl，那么会调用ff_openssl_init()
  if((ret == ff_tls_init())<0) {
    return ret;
  }
#endif  
  return 0;
}

int ff_openssl_init() {
  //使用mutex来对线程加锁
  ff_lock_avformat();
  if(!openssl_init) {
    //实际上调用了ssl.h中的OPEN_init_ssl
    SSL_library_init();
    SSL_load_error_strings();    
  }
  openssl_init++;
  ff_unlock_avformat();
}

```