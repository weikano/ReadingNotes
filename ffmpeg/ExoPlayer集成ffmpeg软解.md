### ExoPlayer集成ffmpeg软解

#### 一、jni中的初始化：avcodec_register_all

```c++
//ffmpeg_jni.cc
jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  JNIEnv* env;
  if(vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6)!=JNI_OK) {
    return -1;
  }
  //注册编解码器
  avcodec_register_all();
  return JNI_VERSION_1_6;
}
//allcodecs.c
void void avcodec_register_all(void) {
  //ff_thread_once其实是一个调用了pthread_once的宏，用av_codec_next_init变量来确保av_codec_init_next方法只被调用一次
  ff_thread_once(&av_codec_next_init, av_codec_init_next);
}
static void av_codec_init_next(void) {
  //AVCodec结构体在avcodec.h中定义
  AVCodec* prev = NULL, *p;
  void* i = 0;
  while((p=(AVCodec*)av_codec_iterate(&i))) {
    if(prev) {
      prev->next = p;
    }
    prev = p;
  }
}
const AVCodec* av_codec_iterate(void** opaque) {
  uintptr_t i = (uintptr_t)*opaque;
  //codec_list在codec_list.c中定义，是一个指向AVCodec*的数组，包含&ff_flac_decoder(flacdec.c中), &ff_vorbis_decoder, NULL
  //调用av_codec_init_static
  const AVCodec* c = codec_list[i];
  ff_thread_once(&av_codec_static_init, av_codec_init_static);
  if(c) {
    *opaque = (void*)(i+1);
  }
  return c;
}
static void av_codec_init_static(void) {
  for(int i=0;codec_list[i];i++) {
    //init_static_data是一个方法指针，需要一个AVCodec参数
    //即这段代码是调用codec_list中的AVCodec*的init_static_data方法
    //ff_libvpx_vp9_decoder的init_static_data指向ff_vp9_init_static
    //ff_libx265_encoder->init_static_data = libx265_encode_init_csp
    //ff_libvpx_vp9_encoder->init_static_data=ff_vp9_init_static
    //ff_libaom_av1_encoder->init_static_data=av1_init_static
    //ff_libx264_encoder.init_static_data = x264_init_static
    if(codec_list[i]->init_static_data) {
      codec_list[i]->init_static_data((AVCodec*)codec_list[i]);
    }
  }
}
```

#### 二、FfmpegDecoder初始化过程

##### 1. 创建decoder

```java
public FfmpegDecoder(
	int numInputBuffers,
  int numOutputBuffers,
  int initialInputBufferSize,
  Format format,
  boolean outputFloat
) throw FfmpegDecodeException {
	super(new DecoderInputBuffer[numInputBuffers], new SimpleOutputBuffer[numOutputBuffers]);
	//根据mimetype获取对应的codecname
	codecName = FfmpegLibrary.getCodecName(format.simpleMimeType, format.pcmEncoding);
	//...
	//调用native中的ffmpegInitialize方法获取指向AVCodecContext的指针
	nativeContext = ffmpegInitialize(codecName, extraData, xxx);
}
```

##### 2. ffmpegInitialize方法解析
```c++
//ffmpeg_jni.cc中
//DECODER_FUNC是一个预定义宏，用来给FfmpegDecoder中的方法生成对应的jni方法，同样功能的还是有LIBRARY_FUNC，用在FfmpegLibrary.java中
DECODER_FUNC(jlong, ffmpegInitialize, jstring codecName, jbyteArray extraData, ...) {
	//获取对应的编解码器对应的AVCodec结构体
	AVCodec* codec = getCodecByName(env, codecName);
	if(!codec) {
		return 0L;
	}
	//调用createContext方法获取
	return (jlong)createContext(env, codec, extraData, ...);
}
AVCodec* getCodecByName(JNIEnv* env,jstring codecName) {
	const char* nameChars = env->GetStringUTFChars(codecName, NULL);
	AVCodec* codec = avcodec_find_decoder_by_name(nameChars);
	env->ReleaseStringUTFChars(codecName, nameChars);
	return codec;
}
//获取AVCodecContext指针
AVCodecContext *createContext(JNIEnv *env, AVCodec *codec, jbyteArray extraData,jboolean outputFloat, jint rawSampleRate, jint rawChannelCount) {
	//获取AVCodecContext指针, avcodec_alloc_context3方法在/libavcodec/options.c中
  AVCodecContext *context = avcodec_alloc_context3(codec);
  if (!context) {
    LOGE("Failed to allocate context.");
    return NULL;
  }
  //接下来初始化AVCodecContext结构体的一些成员
  context->request_sample_fmt =
      outputFloat ? OUTPUT_FORMAT_PCM_FLOAT : OUTPUT_FORMAT_PCM_16BIT;
  if (extraData) {
    jsize size = env->GetArrayLength(extraData);
    context->extradata_size = size;
    context->extradata =
        (uint8_t *) av_malloc(size + AV_INPUT_BUFFER_PADDING_SIZE);
    if (!context->extradata) {
      LOGE("Failed to allocate extradata.");
      releaseContext(context);
      return NULL;
    }
    env->GetByteArrayRegion(extraData, 0, size, (jbyte *) context->extradata);
  }
  if (context->codec_id == AV_CODEC_ID_PCM_MULAW ||
      context->codec_id == AV_CODEC_ID_PCM_ALAW) {
    context->sample_rate = rawSampleRate;
    context->channels = rawChannelCount;
    context->channel_layout = av_get_default_channel_layout(rawChannelCount);
  }
  context->err_recognition = AV_EF_IGNORE_ERR;
  //调用avcodec_open2方法来初始化一个音视频编解码器AVCodecContext，最终会调用AVCodec的init方法，在/libavcodec/utils.c中实现
  int result = avcodec_open2(context, codec, NULL);
  if (result < 0) {
    logError("avcodec_open2", result);
    releaseContext(context);
    return NULL;
  }
  return context;
}
```

##### 3. avcodec_find_decoder_by_name解析

```c++
//libavcodec/utils.c
int av_codec_is_encoder(const AVCodec *codec){
  return codec && (codec->encode_sub || codec->encode2 ||codec->send_frame);
}
//allcodecs.c
AVCodec* avcodec_find_decoder_by_name(const char* name) {
  return find_codec_by_name(name, av_codec_is_encoder);
}
//两个参数，一个表解码器的名字，一个是方法指针。
static AVCodec* find_codec_by_name(const char* name, int(*x)(const AVCodec*)){
  void *i = 0;
  const AVCodec *p;
  if (!name)
    return NULL;
	//遍历codec_list查找符合条件的AVCodec
  while ((p = av_codec_iterate(&i))) {
    if (!x(p))
      continue;
    if (strcmp(name, p->name) == 0)
      return (AVCodec*)p;
  }
  return NULL;
}

```

##### 4. avcodec_alloc_context3解析

```c++
//libavcodec/options.c
AVCodecContext* avcodec_alloc_context3(const AVCodec* codec){
  //av_malloc位于libavutil/mem.c中
  //去掉一大堆宏定义，可以缩减为通过malloc函数分配内存，然后进行了内存对齐，并做了一些错误检查
  //av_realloc调用realloc函数
  //av_mallocz调用了av_malloc然后通过memset设置为0
  //av_calloc简单封装了av_mallocz
  //av_free调用free
  AVCodecContext *avctx = av_malloc(sizeof(AVCodecCOntext));
  if(!avctx) {
    return NULL;
  }
  //libavcodec/options.c中定义
  //用来给AVCodecContext进行初始化
  if(init_context_defaults(avctx, codec) < 0) {
    av_free(avctx);
    return NULL;
  }
  return avctx;
}
```

#### 三、FfmpegDecoder真正的视频解析工作

##### 1. java层

```java
//FfmpegDecoder.java
protected @Nullable FfmpegDecodeException decode(DecoderInputBuffer inbuf, SimpleOutputBuffer outBuf, boolean reset) {
  if(reset) {
    //是否重新获取AVCodecContext，其实这里只在TrueHD时才会重新获取AVCodec，然后调用createContext方法获取AVCodecContext，其他情况只是清空AVCodecContext的buffer
    nativeContext = ffmpegReset(nativeContext, extraData);
    //...其他
    int result = ffmpegDecode(nativeContext, inputData, inputSize, outputData, outputBufferSize);
    //...
  }
}
```

##### 2. jni层

```c++
//ffmpeg_jni.cc
DECODER_FUNC(jint, ffmpegDecoder, jlong context, ...) {
  //忽略前面的错误检查
  AVPacket packet;
  //初始化AVPacket结构体，没什么其他的函数调用
  av_init_packet(&packet);
  packet.data = inputBuffer;
  packet.size = inputSize;
  return decoderPacket((AVCodecContext*)context, &packet, outputBuffer, outputSize);
}
```

##### 3. decoderPacket解析

```c++
//ffmpeg_jni.cc
int decodePacket(AVCodecContext* context, AVPacket* packet, outbuffer, outputsize) {
  int result = 0;
  //libavcodec/decode.c
  //packet里面包含了inputData，现在把packet加入队列
  //初始化AVCodecInternal的DecodeFilterContext等成员
  //把packet数据传送给AVCodecInternal->filter.bsfs[0]
  //然后调用decode_receive_frame_internal解码
  //如果AVCodec没有定义receive_frame函数指针，那么会调用decode_simple_receive_frame来解码
  //以h264为例，对应的解码器为ff_h264_decoder，并没有设置receive_frame
  //decode_simple_receive_frame会调用decode_simple_internal
  //抛开一长串代码，最后会调用AVCodec的decode函数指针
  result = avcodec_send_packet(context, packet);
  int outSize = 0;
  while(true) {
    //AVFrame：用来存储非压缩的数据(视频对应RGB/YUV像素数据，音频对应PCM采样数据)
    //会尝试从缓存中获取AVFrame，如果没有，则调用decode_receive_frame_internal进行解码
    AVFrame* frame = av_frame_alloc();
    result = avcodec_receive_frame(context, frame);
    //重新采样
    AVSampleFormat sampleFormat = context->sample_fmt;
    int channelCount = context->channels;
    int channelLayout = context->channel_layout;
    int sampleRate = context->sample_rate;
    int sampleCount = frame->nb_samples;
    int dataSize = av_samples_get_buffer_size(NULL, channelCount, sampleCount,
                                              sampleFormat, 1);
    AVAudioResampleContext *resampleContext;
    if (context->opaque) {
      resampleContext = (AVAudioResampleContext *) context->opaque;
    } else {
      resampleContext = avresample_alloc_context();
      av_opt_set_int(resampleContext, "in_channel_layout",  channelLayout, 0);
      av_opt_set_int(resampleContext, "out_channel_layout", channelLayout, 0);
      av_opt_set_int(resampleContext, "in_sample_rate", sampleRate, 0);
      av_opt_set_int(resampleContext, "out_sample_rate", sampleRate, 0);
      av_opt_set_int(resampleContext, "in_sample_fmt", sampleFormat, 0);
      // The output format is always the requested format.
      av_opt_set_int(resampleContext, "out_sample_fmt",
          context->request_sample_fmt, 0);
      result = avresample_open(resampleContext);
      if (result < 0) {
        logError("avresample_open", result);
        av_frame_free(&frame);
        return -1;
      }
      context->opaque = resampleContext;
    }
    int inSampleSize = av_get_bytes_per_sample(sampleFormat);
    int outSampleSize = av_get_bytes_per_sample(context->request_sample_fmt);
    int outSamples = avresample_get_out_samples(resampleContext, sampleCount);
    int bufferOutSize = outSampleSize * channelCount * outSamples;
    if (outSize + bufferOutSize > outputSize) {
      LOGE("Output buffer size (%d) too small for output data (%d).",
           outputSize, outSize + bufferOutSize);
      av_frame_free(&frame);
      return -1;
    }
    result = avresample_convert(resampleContext, &outputBuffer, bufferOutSize,
                                outSamples, frame->data, frame->linesize[0],
                                sampleCount);
    av_frame_free(&frame);
    if (result < 0) {
      logError("avresample_convert", result);
      return result;
    }
    int available = avresample_available(resampleContext);
    if (available != 0) {
      LOGE("Expected no samples remaining after resampling, but found %d.",
           available);
      return -1;
    }
    outputBuffer += bufferOutSize;
    outSize += bufferOutSize;
  }
  return outSize;
}
```

