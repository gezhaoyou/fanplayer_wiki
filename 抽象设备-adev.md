fanplayer 中的 adev 是一个抽象的，用于播放音频的设备，其接口定义如下：

    void* adev_create  (int type, int bufnum, int buflen);
    void  adev_destroy (void *ctxt);
    void  adev_lock    (void *ctxt, AUDIOBUF **ppab);
    void  adev_unlock  (void *ctxt, int64_t pts);
    void  adev_pause   (void *ctxt, int pause);
    void  adev_reset   (void *ctxt);
    void  adev_syncapts(void *ctxt, int64_t *apts);
    void  adev_curdata (void *ctxt, void **buf, int *len);
    void  adev_setparam(void *ctxt, int id, void *param);
    void  adev_getparam(void *ctxt, int id, void *param);

在 fanplayer 中，我们假定了 adev 支持的音频格式固定为 44100Hz 采样率，16bit signed，立体声。这样的假定在目前看来是比较合理的，并且可以简化我们的问题。我们在创建 adev 时，就只需要知道音频回放所需要的缓冲区个数和缓冲区大小，因此我们的 adev_create 函数就定义为了：
    void* adev_create(int type, int bufnum, int buflen);
其中 type 用于指定 adev 类型，这个是为支持多种类型的 adev 设备而设计的。目前在 win32 平台上仅支持 WAVEOUT 这样的类型，在 android 平台仅支持 AudioTrack 这样的类型。因此 type 参数在目前的实现中，实际上是没有用到的。

adev_destroy, adev_pause, adev_reset, adev_setparam, adev_getparam 这几个接口都没什么好讲的，分别对应了 adev 对象的销毁，暂停/播放，复位操作，设置/获取参数。复位操作一般用于 seek 操作的过程中，在 seek 时如果播放器处于播放状态，并且已经有了一部分音频数据解码被送到了 adev，那么我们需要对 adev 执行 reset 操作，使得 adev 能丢掉当前正在播放的和处于缓冲状态的音频数据。如果不丢那么 seek 后我们就会听到 seek 之前的声音播放了一小段，造成不好的体验。adev 内部还实现了一个软件的音量算法，可以实现软件音量控制，这个音量的设置和读取，是通过 adev_setparam 和 adev_getparam 来进行的。同时这个 setparam 和 getparam 也是 fanplayer 标准的参数设置和读取接口，可方便的扩展接口和功能。

adev_lock 和 adev_unlock 操作用于访问 adev 的音频缓冲区。在 ffmpeg 解码出音频数据后，会丢给 render 使用 swresample 将音频数据转换为 44100Hz 16bit 立体声的数据，然后这个数据就需要送给 adev 去播放了。如何送呢，adev_lock 这个接口，会锁定住当前 adev 内部空闲可用的可装填音频数据的 buffer，这个 buffer 的定义为：

    typedef struct
    {
        int16_t *data;
        int32_t  size;
    } AUDIOBUF;

adev_lock 调用后，就获得了这个 buffer 的指针，我们就可以将音频数据填入这个 buffer。填充完毕后，我们需要调用 adev_unlock 把当前的 buffer 解锁，解锁的同时也意味着这个 buffer 会加入到 adev 的音频播放队列去播放。

adev_syncapts 这个接口需要说明一下，它是用于实现音视频同步的。我们已经知道要实现音视频同步，需要确切的记录下当前正在播放的音频的 pts 和视频的 pts。而这两个 pts 分别是由 adev 和 vdev 记录并维护的。adev 和 vdev 中，每一个被播放的音频帧或视频帧，都有一个对应的 pts，并且排成一个队列。adev 和 vdev 的内部的回放线程，是可以知道当前被播放器的 pts 的，这个很容易理解，我们看看内部实现代码就明白了。现在的问题是记录下来的 apts 和 vpts 保存在哪里？全局变量？不合理。render 对象中？不方便。adev 中放 apts，vdev 中放 vpts ？也不方便。为啥说不方便，因为音视频同步是在 vdev 内部的绘制线程去做的，是通过 delay 来做的，而 adev 和 vdev 都是抽象设备，我们除了能在创建的时候得到 void* 指针（或者叫句柄，或者叫 context），就只有通过接口函数进行访问，比如定义 adev_get_apts 或者通过 adev_get_param 来做。不管怎样，要么会增加接口，要么就是不方便，甚至影响性能。

因此，我的解决办法，就是将 apts 和 vpts 的值，都保存到了 vdev 中，在 vdev 创建后通过

    void vdev_getavpts(void *ctxt, int64_t **ppapts, int64_t **ppvpts);

可以获取指向 apts 和 vpts 变量的指针，这样通过指针我们就可以高效地随时随地地访问了。而 adev 的 adev_syncapts 接口就是把 apts 的指针保存到 adev 的内部。这个 apts 保存的地方是在 vdev 内部，但是实际却是由 adev 的代码通过指针去更新这个变量的值。理解这个原理是很关键的，理解了才能帮助我们理解 adev 和 vdev 之间的关系，以及如何实现音视频同步。

adev_curdata 这个接口，是为了获取当前正在播放的音频帧，主要是为了实现音频播放的视觉效果，比如绘制波形图，绘制频谱图等。这个接口很简单，使用方法可以看下 ffrender.cpp 中的 render_veffect_thread 线程是如何绘制视觉效果的。

adev 的内部实现，其实也挺简单，写不了几行代码。以 adev-win.cpp 为例，这个是 win32 上 waveout 的实现，总共不超过 200 行代码就实现了 adev 的全部功能。相对于使用 sdl 这类库，也多写不了多少代码，反而我们可以对代码可以做更好的控制和优化。从 waveout 的 api 来讲，我们需要 waveOutOpen 一个设备，设定需要的采样率，位数，和声道类型；然后使用 waveOutPrepareHeader 初始化音频缓冲区；填充缓冲区后使用 waveOutWrite 写出音频数据；而 waveOutRestart 和 waveOutPause 则是播放暂停的开关... 另外，在 adev 内部我们同样使用了 bufsem 这样一个信号量来做同步，是因为这里面需要 waveOutWrite 写出去的音频缓冲区，本质上也就是排成了一个队列。render 对象内部的线程（也就是 adev_unlock 的调用者，实际上是音频解码线程）是音频数据的生成者；waveout api 对应的音频设备或驱动，是音频数据的消费者。bufsem 代表的是队列中的空闲缓冲区的个数，当 waveOutProc 中接收到了 WOM_DONE 消息，意味着当前的音频缓冲区播放结束，当前这个缓冲区就可以回收重复利用，因此通过对 head 变量的操作和 ReleaseSemaphore 来回收缓冲区并增加 bufsem 信号量。增加 bufsem 实际上是通知生产者：“我们这边多了一个空位，可以给你拿去填充新的音频数据了”。因此 adev_lock 这个接口是会阻塞的，当所有的缓冲区都被填满了，就阻塞了。

理解完整个 adev 接口和内部实现，我们可以对播放器的音频回放，有一个很好的理解。再补充说两个问题，一个是有朋友问为什么很多接口是 void 的返回类型？为什么不加个返回值，用于表示成功失败，或其他调用结果？

这个问题很好，很多时候我们都有这样的困惑，申请内存是不是有可能申请不到？各种 api 执行，是否是会失败？这样一搞，tmd 是不是我们写个代码每写一行都要做个错误判断？这样是不是也太繁琐了？代码效率如何能提高呢？我以前写个代码也一样，很多接口函数都留了返回值，代码也写得累啊。按照 KISS 原则，这样的做法真心是不必要的吧。因此我的设计原则是，能用 void 返回类型的，就决不返回错误代码，尽量多用 void 类型的返回值，就是说我做接口设计和实现的时候，就要尽最大努力去保证，这个接口是能执行成功了。这样简单的限定，反而可以有效提升整个系统的稳定性，也使得代码更加简洁高效。如果实在不能保证接口一定能成功执行，我们才加返回值，甚至也可以不加，因为我们只要保证不产出副作用即可，比如死机、崩溃这类的问题。简单点说，adev_lock 这个方法，很多人觉得有没有可能性 lock 失败，得不到 buffer 呢？有可能吧。那么为啥不要返回值？因为调用者它可以处理啊。得不到 buffer 我们就得不到正确的指针，调用者只要不往里面填充数据，就不会 crash 是吧。不会 crash 那么我们的 buffer 就算 lock 失败了，其他部分的代码或逻辑，也可以正常跑啊，顶多是 adev 设备没有出声音，无法播放声音而已。那么没有声音，我们可以很容易判断出问题，很容易知道是 adev 相关代码有问题，实在不行我可以加 log 打印错误信息啊，这样也可以方便调试。

另外一个问题就是 adev 的实现或者扩展。可以看到我们在 fanplayer 中将 adev 假定为了 44100Hz 16bit 立体声，那么要支持 DirectSound 5.1 声道怎么做？答案是，你的重新实现一个新的 adev，内部采用 DirectSound 去做实现，使其可以支持 5.1 声道。同时还要修改 ffrender.cpp 代码，主要是采样转换的代码修改：

    //++ allocate & init swr context
    int samprate = render->speed_type_cur ? ADEV_SAMPLE_RATE : (int)(ADEV_SAMPLE_RATE * 100.0 / render->speed_value_cur);
    if (render->swr_context) {
        swr_free(&render->swr_context);
    }
    render->swr_context = swr_alloc_set_opts(NULL, AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_S16, samprate,
        render->chan_layout, render->sample_fmt, render->sample_rate, 0, NULL);
    swr_init(render->swr_context);
    //-- allocate & init swr context


AV_CH_LAYOUT_STEREO 要改为 AV_CH_LAYOUT_5POINT1 或自己需要的类型。同时如果你要做兼容，比如兼容 waveout 的立体声，那么得增加一个当前 adev 支持的声道类型的判断。根据当前的声道类型，去初始化 ffrender.cpp 中的 swr 的相关代码。

好了这部分能想到的该讲的内容都讲完了。

