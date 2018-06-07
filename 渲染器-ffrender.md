## ffrender 的接口定义
渲染器主要负责音频和视频的渲染呈现，ffplayer 中渲染器在 ffrender.cpp 中实现，其接口定义如下：

    void* render_open(int adevtype, int srate, AVSampleFormat sndfmt, int64_t ch_layout,
                      int vdevtype, void *surface, AVRational frate, AVPixelFormat pixfmt, int w, int h);
    void render_close   (void *hrender);
    void render_audio   (void *hrender, AVFrame *audio);
    void render_video   (void *hrender, AVFrame *video);
    void render_setrect (void *hrender, int type, int x, int y, int w, int h);
    void render_start   (void *hrender);
    void render_pause   (void *hrender);
    void render_reset   (void *hrender);
    int  render_snapshot(void *hplayer, char *file, int w, int h, int waitt);
    void render_setparam(void *hrender, int id, void *param);
    void render_getparam(void *hrender, int id, void *param);

## render_open
render_open 函数，用于创建一个 render 对象，函数参数 adevtype 和 vdevtype 分别用于指定 adev 和 vdev 的类型。

    void* render_open(int adevtype, int srate, AVSampleFormat sndfmt, int64_t ch_layout,
                      int vdevtype, void *surface, AVRational frate, AVPixelFormat pixfmt, int w, int h);

在 ffplayer 中 adev 和 vdev 都是抽象的音视频设备。adev 是指音频回放设备，vdev 是指视频回放设备。在 win32 平台上 adev 目前实现了 WAVEOUT 的 adev，vdev 实现了 GDI 和 D3D 两种类型的 vdev。除了设备类型之外，我们还需要指明音频采样率，即 srate 参数；音频采样格式，sndfmt 参数，音频通道类型，ch_layout 参数。采样率很容易理解，常见的 8KHz, 44100Hz, 48000Hz。采样格式，就是每一个采样点用什么样的数据类型来表示，常见的是 16bit 有符号整数，这个是 win32 WAVEOUT api 接口用的，还有 float 类型的。通道类型指的是有多少个通道，每个通道是什么含义或用途，比如常见的单声道、立体声，还有 5.1 声道等等。采样格式和通道类型，在 ffmpeg 中都有专门的类型和宏定义。

视频设备需要指定的还有 surface 参数，这个参数用于传入 win32 的窗口句柄，或者 android 平台的 Handler 对象。win32 平台上，传入窗口句柄，主要用于 gdi 或者 d3d 的绘图，以及播放器消息的发送。android 平台上主要用于发送播放器消息，而用于视频渲染的 surface 则是通过 vdev_android_setwindow 这个接口来传递，是 android 平台专用的接口。frate 用于指定视频帧率，这个是用的 AVRational 类型，是一个 num/den 的分数形式，比整数形式的帧率更加精确。pixfmt 是像素格式，常见的有 RGB24、YUV420P 等，也是用的 ffmpeg 定义的类型即 AVPixelFormat。w 和 h 用于指定视频的宽高。

可以看到 render_open 的函数参数，基本就包含了音视频回放所需要的全部信息。在 render 的前一阶段，demux + decode 这个阶段，我们是可以从多媒体文件里面得到的信息就有，音频采样率、音频格式、音频声道类型，以及视频宽高、帧率、像素格式。这些是音视频回放的必要的信息。有了这些信息，我们就可以创建 render 对象，来进行音视频回放。另一方面，因为不同平台，有不同的音视频渲染方式或者设备，而且一些设备的创建又依赖于类似于窗口句柄这样的东西，同时 render 还要负责发送播放器消息，消息包括播放进度、播放完成、等等。因此我们的 render_open 接口中就多出了 adevtype, vdevtype, surface 这是三个参数。

## render_close
void render_close(void *hrender);  
用于关闭一个 render 对象，释放所有相关资源

## render_audio 和 render_video
void render_audio(void *hrender, AVFrame *audio);  
void render_video(void *hrender, AVFrame *video);

render_audio 用于渲染一个音频帧，而 render_video 用于渲染一个视频帧。

这里我们可以看到，音视频的相关接口，是一个对称的接口，包括函数命名和参数，都是对称的，这样看起来就很舒服，很整齐。
这两个接口接收的参数，一个是 render 对象的指针或者叫句柄，一个就是 AVFrame。而这个 AVFrame 又是什么？就是音视频解码线程解码出来的 AVFame，解码出来直接就扔给这两个接口。扔过去后其他就不用管了，剩下的包括视频的像素格式转换、缩放、渲染，音频的格式转换、变采样、渲染，音视频同步，...... 等等，全部都由 render 内部去处理了。这样我们的解码线程就可以看起来很简单很清晰（请参考阅读 ffplayer.cpp 中的相关代码）。

## render_setrect
void render_setrect(void *hrender, int type, int x, int y, int w, int h);

这个接口主要用于 win32 平台的视频显示区域的设置。win32 平台的渲染都是在窗口句柄上做的，我们这个接口在窗口的显示区域内，还可以进行一个区域设定。这里面有一个 type 用于指定类型，因为我们的播放器可以渲染视频，同时还可以渲染音频的波形或者频率图（win32 平台上）。这两个可以在同一个 hwnd 上渲染，可以分别指定区域，来避免干涉（如果重叠了，效果就是闪烁啦）。在 android 平台上，视频显示区域由 app 的 layout 来指定，不用通过播放器的接口函数来指定，因此这个函数是用不上的（反而省事了是不）。

## start & pause & reset
void render_start(void *hrender);  
void render_pause(void *hrender);  
void render_reset(void *hrender);  

start, pause, reset 分别对应开始渲染，停止渲染，复位 render 对象。start 和 pause 的应用场景是，视频播放的暂停和恢复。reset 的应用场景是 seek 操作。

## render_snapshot
int render_snapshot(void *hplayer, char *file, int w, int h, int waitt);

这个用于视频截图，视频播放过程中，可以将当前视频帧保存为 .jpg 或者 .png 的图片。

## render_setparam & render_getparam
void render_setparam(void *hrender, int id, void *param);  
void render_getparam(void *hrender, int id, void *param);

这两个接口，是扩展的参数设置和获取接口。


## ffrender 要做的基本工作
从接口分析我们可以看到，ffrender 的内部实现大致要做哪些：
1. 创建/维护/销毁 adev vdev 设备
2. 视频的像素格式转换，缩放，回放，帧率控制
3. 音频的格式转换，变采样，回放，帧率控制
4. 音视频同步和播放器消息发送
5. 其他...

adev 和 vdev 这个很容易理解，就是抽象的音频设备和视频设备。为什么要这么分隔开来做抽象，而不是全部做到 ffrender 模块中？主要还是为了代码复用。不同平台上能支持的 vdev 和 adev 都是不同的。比如 gdi 的 vdev 设备，其像素格式是 rgb 的，绘图是采用的 gdi 的 api。而 d3d 的 vdev 又有可能能支持 yuv 格式，绘图采用的又是 Direct3D 的 api。

另一方面，多媒体文件解码出来的音频数据、视频数据，其格式又可能是多种多样的，那么就会需要进行像素格式转换和音频格式转换。这部分是用的 swscale 和 swresample 库，是纯软件的算法，可移植性极高。因此通过 adev + vdev 的抽象接口，来隔离了平台差异，使得 ffrender 内部的音视频格式转换的相关代码，是可以实现复用的。

而帧率控制和音视频同步，则是在 adev 和 vdev 之间的相互配合和实时通信来实现的。通信机制也很简单，就是实时的读取当前的 apts 和 vpts。同步的目标，就是是 apts 和 vpts 之间的差值最小。播放器消息的发送，主要是播放完成消息的生成和发送，这个也是 render 去做的。播放进度消息，我更倾向于查询，而不是通知。播放完成消息，在 demux 线程中是做不了的，以前我也认为 demux 到了文件末尾就播放完成了。实际上不是这样的，因为你还有一个 packet 队列啊，demux 线程就算检查到了文件末尾，但是 packet 队列里面还有数据。等于是播放提前完成了，packet 队列里面还有一大段音视频还没有播放完...

## ffrender 中实现的更多功能
除了上面提到的这些基本功能，ffrender 还实现了更多的东西，比如软件音量的控制算法，变速播放的实现，音频的波形渲染和频谱渲染。软件音量控制算法，说简单易懂一点，就是对每个音频采样点做了一个乘法运算和饱和运算，具体的可以参考 wiki 上的文章说明。变速播放说白了，也就是变帧率变采样率，就这么简单，具体也可看看 wiki 上的文章说明。音频的波形绘制代码也挺简单，几十行就搞定了，使用 gdi 绘图的，频率渲染，需要用 FFT 算法，为此我还自己专门写了一个 FFT。

另外我们看代码可以发现，除开 render_veffect_thread 这个用于渲染音频波形和频谱的线程（无关紧要），ffrender 内部是没有开其他线程的，也就是说像素格式转换和音频格式转换，是在 render_video 和 render_audio 接口中实现的，而这两个接口是被 ffplayer.cpp 中的视频解码线程和音频解码线程所调用的。就是说解码和格式转换，都是在解码线程中去做的。这样做实际效果就已经可以了，按照 KISS 原则，就没有必要再去开线程了。

最后再提下硬解码和变速不变调。硬解码也是需要在 ffrender 中提供了相关接口的。硬解码的原则基本上就是要 0-copy 的渲染，简单说就是 packet 扔给解码器解码后，得到的 AVFame 数据，是需要直接送给硬件渲染器进行渲染的，这中间是不需要做（也很难做）任何像素格式转换的。硬解码一般都是针对高清视频而已，音频基本上都是软解码就足够了。硬解出来的高清视频，比如 4K，放到内存中，数据量都是极大的，哪怕再去做一次拷贝动作，都是及其消耗 cpu 时间的，更不要说像素格式转换。以 dxva2 的硬解码为例，解码出来直接就是一个 d3d 的 surface，这个 surface 直接扔给 d3d 设备渲染就可以了。我在 ffrender 中定义了 PARAM_VDEV_D3D_ROTATE 参数控制字，通过 render_setparam 把 surface 传给了 d3d vdev 进行直接渲染。要研究 dxva2 硬解码的，可以此为线索看代码。

变速不变调，是用的 soundtouch 这个开源库，没什么好说的，接口和代码都挺简单的。根据 CONFIG_ENABLE_SOUNDTOUCH 这个宏定义去查看相关代码就可以了，其实真的很简单。





