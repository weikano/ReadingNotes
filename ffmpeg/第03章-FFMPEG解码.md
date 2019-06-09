### 第03章-FFMPEG解码

#### 一、AVPacket

> AVPacket对应压缩后的数据。
>
> 典型的使用场景是作为对URL进行demux之后的输出，然后作为decoder的输入；或者是encoder的输出，然后作为muxer的输入。
>
> 对视频来说，这就是一帧压缩后的数据；对音频来说，可能包含很多压缩后的帧。

#### 二、AVFrame

> 用来表示解码后的raw音视频数据
>
> AVFrame必须通过av_frame_alloc()来构造，通过av_frame_free来释放。
>
> AVFrame是典型的一次构造，对不同的数据可以重复。比如单个AVFrame可以用来持有从decoder中收到的不同Frame数据。这种情况下,av_frame_unref会将数据还原至最初最干净的状态。

#### 三、解码过程

