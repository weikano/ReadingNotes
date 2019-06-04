### 第02章-Ffmpeg打开音视频文件

Ffmpeg打开音视频文件通过avformat_open_input方法，打开后通过avformat_find_stream_info方法来获取音视频信息

#### 1. avformat_open_input

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
  
}
```

