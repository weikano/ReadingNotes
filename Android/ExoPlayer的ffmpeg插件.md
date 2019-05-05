### ExoPlayer的ffmpeg插件

#### 代码结构

- ffmpeg_jni.cc：JNI方法关联

- FfmpegAudioRenderer：使用ffmpeg进行decode和render audio

- FfmpegDecoder：通过decode方法进行解码

- FfmpegLibrary：通过LibraryLoader加载ffmpeg相关的so库

  ```java
  public synchronized boolean isAvaiable() {
    try {
      //nativeLibs包括编译后生成的avutil, avresample, avcodec, ffmpeg
      for(String lib:nativeLibs) {
        System.loadLibrary(lib)
      }
    }catch(Exception e) {
      
    }
  	return isAvailable  
  }
  ```

#### JNI相关

```c++
#include "jni.h"
//JNI相关的方法关联通过两个预处理命令LIBRARY_FUN和DECODER_FUNC
//首先声明函数，然后实现。
//RETURN_TYPE返回值类型，NAME方法名，VARARGS方法参数
#define DECODER_FUNC(RETURN_TYPE, NAME, ...) \
  extern "C" { \
  JNIEXPORT RETURN_TYPE \
    Java_com_google_android_exoplayer2_ext_ffmpeg_FfmpegDecoder_ ## NAME \
      (JNIEnv* env, jobject thiz, ##__VA_ARGS__);\
  } \
  JNIEXPORT RETURN_TYPE \
    Java_com_google_android_exoplayer2_ext_ffmpeg_FfmpegDecoder_ ## NAME \
      (JNIEnv* env, jobject thiz, ##__VA_ARGS__)\

#define LIBRARY_FUNC(RETURN_TYPE, NAME, ...) \
  extern "C" { \
  JNIEXPORT RETURN_TYPE \
    Java_com_google_android_exoplayer2_ext_ffmpeg_FfmpegLibrary_ ## NAME \
      (JNIEnv* env, jobject thiz, ##__VA_ARGS__);\
  } \
  JNIEXPORT RETURN_TYPE \
    Java_com_google_android_exoplayer2_ext_ffmpeg_FfmpegLibrary_ ## NAME \
      (JNIEnv* env, jobject thiz, ##__VA_ARGS__)\

jint JNI_Onload(JavaVM* vm, void* reserved) {
  avcodec_register_all();
  return JNI_VERSION_1_6;
}

//FfmpegLibrary#ffmpegGetVersion
//获取ffmpeg当前的版本
LIBRARY_FUNC(jstring, ffmpegGetVersion) {
  return env->NewStringUTF(LIBAVCODEC_IDENT);
}
//FfmpegLibrary#ffmpegHasDecoder
//返回是否支持当前的codec的decoder
LIBRARY_FUNC(jboolean, ffmpegHasDecoder, jstring codecName) {
  return getCodecByName(env, codecName) != NULL;
}
//FfmpegDecoder#ffmpegInitialize
//初始化AVCodecContext并返回AVCodecContext的指针，createContext方法在会调用libavcodec/options.c中的avcode_alloc_context3方法构造一个AVCodecContext指针，然后avcodec_open2(libavcodec/utils.c)
DECODER_FUNC(jlong, ffmpegInitialize, jstring codecName, jbyteArray extraData,
             jboolean outputFloat, jint rawSampleRate, jint rawChannelCount) {
  AVCodec *codec = getCodecByName(env, codecName);
  if (!codec) {
    LOGE("Codec not found.");
    return 0L;
  }
  return (jlong)createContext(env, codec, extraData, outputFloat, rawSampleRate,
                              rawChannelCount);
}
//FfmpegDecoder#ffmpegDecode
DECODER_FUNC(jint, ffmpegDecode, jlong context, jobject inputData,
    jint inputSize, jobject outputData, jint outputSize) {
  if (!context) {
    LOGE("Context must be non-NULL.");
    return -1;
  }
  if (!inputData || !outputData) {
    LOGE("Input and output buffers must be non-NULL.");
    return -1;
  }
  if (inputSize < 0) {
    LOGE("Invalid input buffer size: %d.", inputSize);
    return -1;
  }
  if (outputSize < 0) {
    LOGE("Invalid output buffer length: %d", outputSize);
    return -1;
  }
  uint8_t *inputBuffer = (uint8_t *) env->GetDirectBufferAddress(inputData);
  uint8_t *outputBuffer = (uint8_t *) env->GetDirectBufferAddress(outputData);
  AVPacket packet;
  av_init_packet(&packet);
  packet.data = inputBuffer;
  packet.size = inputSize;
  return decodePacket((AVCodecContext *) context, &packet, outputBuffer,
                      outputSize);
}
//FfmpegDecoder#ffmpegGetChannelCount
DECODER_FUNC(jint, ffmpegGetChannelCount, jlong context) {
  if (!context) {
    LOGE("Context must be non-NULL.");
    return -1;
  }
  return ((AVCodecContext *) context)->channels;
}
//FfmpegDecoder#ffmpegGetSampleRate
DECODER_FUNC(jint, ffmpegGetSampleRate, jlong context) {
  if (!context) {
    LOGE("Context must be non-NULL.");
    return -1;
  }
  return ((AVCodecContext *) context)->sample_rate;
}
//FfmpegDecoder#ffmpegReset
DECODER_FUNC(jlong, ffmpegReset, jlong jContext, jbyteArray extraData) {
  AVCodecContext *context = (AVCodecContext *) jContext;
  if (!context) {
    LOGE("Tried to reset without a context.");
    return 0L;
  }

  AVCodecID codecId = context->codec_id;
  if (codecId == AV_CODEC_ID_TRUEHD) {
    // Release and recreate the context if the codec is TrueHD.
    // TODO: Figure out why flushing doesn't work for this codec.
    releaseContext(context);
    AVCodec *codec = avcodec_find_decoder(codecId);
    if (!codec) {
      LOGE("Unexpected error finding codec %d.", codecId);
      return 0L;
    }
    jboolean outputFloat =
        (jboolean)(context->request_sample_fmt == OUTPUT_FORMAT_PCM_FLOAT);
    return (jlong)createContext(env, codec, extraData, outputFloat,
                                /* rawSampleRate= */ -1,
                                /* rawChannelCount= */ -1);
  }

  avcodec_flush_buffers(context);
  return (jlong) context;
}

DECODER_FUNC(void, ffmpegRelease, jlong context) {
  if (context) {
    releaseContext((AVCodecContext *) context);
  }
}
```

