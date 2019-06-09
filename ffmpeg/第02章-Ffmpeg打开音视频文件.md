### 第02章-Ffmpeg打开音视频文件

Ffmpeg打开音视频文件通过avformat_open_input方法，打开后通过avformat_find_stream_info方法来获取音视频信息。下面以http协议为例：http://localhost/myapp/mystream/myfile.mp4

#### 一、avformat_open_input

```c++
//utils.c
//ps为通过avformat_alloc_context的AVFormatContext指针的指针
//filename可以是url也可以是本地,
//fmt如果设置为nullptr，那么会自动检测，如果非nullptr，那么会指定
//有时候打开filename时，会需要特定的参数，那么就放在options中
//比如ffplay rstp://mms.cnr.cn/cnr003?MzE5MTg0IzEjIzI5NjgwOQ==会出现错误，需要指定传输方式为TCP
//ffplay -rtsp_transport tcp rstp://mms.cnr.cn/cnr003?MzE5MTg0IzEjIzI5NjgwOQ==
//这时候就会用到options
void set_options() {
  AVDictionary
 *avdic=NULL;
  char option_key[]="rtsp_transport";
  char option_value[]="tcp";
  av_dict_set(&avdic,option_key,option_value,0);
}
int avformat_open_input(AVFormatContext **ps, const char *filename,ff_const59 AVInputFormat *fmt, AVDictionary **options) {
  //...
  //打开filename，并且尝试获取AVFormatContext
  if((ret = init_input(s, filename, &tmp)) < 0) {
    goto fail;
  }
}

static int init_input(AVFormatContext *s, const char* filename, AVDictionary** options) {
  //s->pb是否为nullptr。s->pb为AVIOContext，一般是Nullptr，这里跳过
  //s->iformat，这里的iformat是在avformat_open_input中fmt参数传入，一般也为nullptr，跳过
  
  //s->io_open函数是在avformat_alloc_context中通过avformat_get_context_defaults初始化，指向io_open_default方法。如果AVFormatContext的open_cb不为Nullptr，调用open_cb。否则调用ffio_open_whitelist,通过ffurl_open_whitelist和ffio_fdopen来获取对应的URLContext、AVIOContext等信息
  s->io_open(s, &s->pb, filename, AVIO_FLAG_READ | s->avio_flags, options);
  return av_probe_input_buffer2(xxx,xxx);
}

```

##### 1、ffurl_open_whitelist

ffio_fdopen主要用来初始化AVIOContext，没什么好说的

```c++
//aviobuf.c
int ffio_open_whitelist(AVIOContext **s,xxx) {
  URLContext* h;
  ffurl_open_whitelist(&h, filename, xxx);
  ffio_fdopen(s, h);
  return 0;
}
//avio.c
int ffurl_open_whitelist(URLContext ** pub,xxx, URLContext* parent) {
  //通过url_find_protocol方法，根据filename查找对应的URLProtocol。
  //在protocol_list.c中有所有URLProtocol，这里的http协议对应ff_http_protocol
  ffurl_alloc(puc, filename,xxx);
  //通过URLContext进行连接
  ffurl_connect(*puc,options);
}
int ffulr_connect(URLContext* uc,xxx options) {
 	//通过uc中的URLProtocol的url_open或者url_open2函数打开对应的url, ff_http_protocol有url_open2，那么就是http_open方法
}
//http.c
static int http_open(xxx) {
  //调用了http_open_cnx_internal
  http_open_cnx();
}

static int http_open_cnx_internal(xxx) {
  //这里注意lower_proto。
  //默认的lower_proto是tcp，如果是https，那么会使用tls
  //然后首先通过lower_proto对应的URLProtocol进行连接
  //这里的url变成lower_proto://xxxx.xx/xx.mp4，接下来就会交给ff_tcp_protocol来处理建立连接
  ffurl_open_whitelist();
  //tcp处理完后交给http协议
	http_connect();
}
static int http_connect(xxx) {
  //初始化http头信息
  //通过ffurl_write() 方法写入头信息和body
  //等待获取返回信息
  http_read_header();
}

```

##### 2、av_probe_input_buffer2

通过字节流来判断input类型。每次探索都会返回一个score值，如果值太低，那么就增大buffer size继续读。如果达到最大probe size了，返回最高的score

#### 二、avformat_find_stream_info

用于给每个媒体流（AVStream）结构体赋值，关键流程如下：

> 1. 获取AVCodecParserContext：av_parser_init
> 2. 查找解码器：find_probe_decoder
> 3. 打开解码器：avcodec_open2()
> 4. 读取完整的一帧压缩编码的数据：read_frame_internal，av_read_frame内部就是调用read_frame_internal
> 5. 解码一些压缩编码数据：try_decode_frame

##### 1. avcodec_open2

> 1. 为各种结构体分配内存
> 2. 将输入的AVDictionary选项输入到AVCodecContext
> 3. 检查
> 4. 如果是编码器，检查输入参数是否符合编码器要求
> 5. 调用AVCodec的init方法初始化编码器

##### 2. read_frame_internal

> 读取一帧压缩过的码流数据，可以认为av_read_frame和read_frame_internal功能相同

##### 3. try_deocode_frame

> 先判断解码器是否已经打开。没有打开就调用avcodec_open2方法打开。接下来根据不同的音视频流类型，调用不同的解码函数进行解码：如果是VIDEO或者AUDIO，那么就调用avcodec_send_packet，然后avcodec_receive_frame；如果是SUBTITLE，那么就avcodec_decode_subtitle2。while语句中的had_codec_parameters方法，用来判断AVStream中的编解码相关的属性是否已经完全解析