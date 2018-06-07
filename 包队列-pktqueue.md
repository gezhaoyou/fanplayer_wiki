## 包队列的作用

包队列的作用是用于缓存解复用 (demux) 阶段出来的音视频的 packet，并将音视频 packet 分发给对应的音视频解码线程。

我们前面已经知道，为了有效利用 cpu 资源，保证视频回放的帧率稳定、连贯和音视频同步等，我们必须要采用多线程的架构，采用流水线的方式，来提高系统的处理能力和数据吞吐量。流水线上的工作者，就是不同的工作线程，比如 demux 线程，adecode 线程，vdecode 线程，... 而不同工位之间的衔接，就是这个包队列了，英文叫做 packet queue.


在解复用阶段，数据从一个输入文件读出，并分离为“视频数据包”、“音频数据包”、“字幕数据包”等。我们常用的就是音视频数据，因此我们至少需要建立，video packet queue 和 audio packet queue 两个队列。


## 设计原理和运行流程

在传统的设计中，通常将 video packet queue 和 audio packet queue 设计为两个独立的队列。每个队列都有各自的入队和出队方法。我们来看看，这个队列需要定义和实现那些接口：

1. 队列要创建，同时指定容量 - void* queue_create(size);
2. 队列要销毁 - void queue_destory(void *queue);
3. 入队列 - void queue_enqueue(void *queue, AVPacket *packet)
4. 出队列 - AVPacket* queue_dequeue(void *queue);

这样基本的流程就应该是：

    
    // player_open 函数中创建队列
    void *aqueue = queue_create(100); // 创建音频 packet 队列
    void *vqueue = queue_create(100); // 创建视频 packet 队列
    ...
    
    
    // demux 线程中读取 packet 并分别入队
    ...
    AVPacket *packet = av_packet_alloc(); // 申请一个 packet
    av_read_frame(player->avformat_context, packet); // 从文件读取数据到 packet
    if (packet->stream_index == player->astream_index) { // 如果是音频 packet
        queue_enqueue(aqueue, packet);
    }
    if (packet->stream_index == player->vstream_index) { // 如果是音频 packet
        queue_enqueue(vqueue, packet);
    }
    ...
    
    
    // 音频解码线程
    while (!(player->player_status & PS_CLOSE))
    {
        ...
        AVPacket *packet = queue_dequeue(aqueue); // 音频 packet 出队列
        avcodec_decode_audio4(player->acodec_context, aframe, &gotaudio, packet); // 解码 packet 得到 aframe
        render_audio(player->render, aframe); // 渲染 aframe
        av_packet_free(packet); // 释放 packet
        ...
    }
    
    
    // 视频解码线程
    while (!(player->player_status & PS_CLOSE))
    {
        ...
        AVPacket *packet = queue_dequeue(vqueue); // 视频 packet 出队列
        avcodec_decode_video2(player->vcodec_context, vframe, &gotvideo, packet); // 解码 packet 得到 vframe
        render_video(player->render, vframe); // 渲染 vframe
        av_packet_free(packet); // 释放 packet
        ...
    }
    
    
    // player_close 函数中销毁队列
    queue_destroy(aqueue); // 销毁音频 packet 队列
    queue_destroy(vqueue); // 销毁视频 packet 队列
    ...



以上就是用伪代码表示出来的，音频视频 packet 队列运行的基本流程。



## fanplayer 中的 pktqueue

在 fanplayer 的实现中，将音视频 packet 队列，集中在了 pktqueue 这样一个对象里面，统一进行管理。并定义了相应接口：

    void* pktqueue_create (int   size); // important!! size must be power of 2
    void  pktqueue_destroy(void *ctxt);
    void  pktqueue_reset  (void *ctxt);

    void      pktqueue_free_enqueue(void *ctxt, AVPacket *pkt); // enqueue a packet to free-queue
    AVPacket* pktqueue_free_dequeue(void *ctxt); // dequeue a packet from free-queue
    void      pktqueue_free_cancel (void *ctxt, AVPacket *pkt); // cancel packet dequeuing from free-queue

    void      pktqueue_audio_enqueue(void *ctxt, AVPacket *pkt); // enqueue a packet from audio-queue
    AVPacket* pktqueue_audio_dequeue(void *ctxt); // dequeue a audio packet to audio-queue

    void      pktqueue_video_enqueue(void *ctxt, AVPacket *pkt);  // enqueue a packet from video-queue
    AVPacket* pktqueue_video_dequeue(void *ctxt); // dequeue a audio packet to video-queue

在 pktqueue 的内部同时维护着三个队列，分别是 free 队列，audio 队列，和 video 队列。这样做的好处在于，packet 不用每次都 av_packet_alloc 出来，然后用完再 av_packet_free 掉。而是从 free 队列取出空闲的 packet，用完后再归还给 free 队列。这样可以减少动态内存申请的开销。

初始化时 free 队列里面就一次性申请了 DEF_PKT_QUEUE_SIZE 个可用的 packet。申请 packet 的操作，就变成了从 free 队列出队。demux 出了音频 packet，就用 pktqueue_audio_enqueue 入队到 audio 队列；demux 出了视频 packet，就用 pktqueue_video_enqueue 入队到 video 队列。音频解码线程中，使用 pktqueue_audio_dequeue 出队列，得到音频 packet 并解码，然后将 packet 归还（入队）给 free 队列（视频解码线程也是有对应操作）。


另外 pktqueue 中还对队列的实现进行了简单的优化，假定 DEF_PKT_QUEUE_SIZE 为 2 的幂次，这样队列 head 和 tail 的增长就只需要做 ++ 的操作。在根据 head 和 tail 去获取队列元素时，需要用 (ppq->vsize - 1) 做一下位与操作。其中的原理很简单，就是取模运算，因为 size 是 2 的幂次，所以可以通过位与来取模，从而实现了循环队列里的回卷操作。

举个例子，如果队列的 size 为 100，head 从 0 开始，增加到 99 就是最大了，再加到 100 就应该回卷到 0，因此需要对 100 取模。但是如果 size 为 128，head 任意增加，我们始终只需要取 head 的低 7 位，就是执行 & 0x7f 的操作。（当然，这个优化其实不是影响性能的关键，也的确是可有可无，哈哈）

在 pktqueue 的实现中还需要注意的是线程安全问题。整个运行过程中，存在 demux, adecode, vdecode, 三个线程，那么我们分别看看那些操作存在线程安全问题。

对于 audio 队列的操作：

1. demux 线程会执行 pktqueue_audio_enqueue
2. adecode 线程会执行 pktqueue_audio_dequeue

pktqueue_audio_enqueue 和 pktqueue_audio_dequeue 两个函数，没有被竞争的共享资源，因此不存在线程安全问题。

同理 video 队列也不存在线程安全问题。


但是对于 free 队列：

1. demux 线程会执行 pktqueue_free_dequeue
2. demux 线程会执行 pktqueue_free_cancel
3. adecode 线程会执行 pktqueue_free_enqueue
4. vdecode 线程会执行 pktqueue_free_enqueue

在 pktqueue_free_enqueue 和 pktqueue_free_cancel 两个函数，存在被竞争的共享资源，就是 ppq->ftail 这个变量，因此我们需要对其进行加锁保护。我们用到了 pthread_mutex_t 这样的锁，同时尽量缩小加锁的粒度，可以看到我们仅仅保护了一行代码：

    void pktqueue_free_enqueue(void *ctxt, AVPacket *pkt)
    {
        PKTQUEUE *ppq = (PKTQUEUE*)ctxt;
        pthread_mutex_lock  (&ppq->lock);
        ppq->fpkts[ppq->ftail++ & (ppq->fsize - 1)] = pkt;
        pthread_mutex_unlock(&ppq->lock);
        sem_post(&ppq->fsem);
    }
    
    void pktqueue_free_cancel(void *ctxt, AVPacket *pkt)
    {
        PKTQUEUE *ppq = (PKTQUEUE*)ctxt;
        pthread_mutex_lock  (&ppq->lock);
        ppq->fpkts[ppq->ftail++ & (ppq->fsize - 1)] = pkt;
        pthread_mutex_unlock(&ppq->lock);
        sem_post(&ppq->fsem);
    }

这个加锁也许看起来没啥作用但却是极为必要的。大家也可以做下测试，如果将这个锁去掉，然后打开播放器，在播放过程中反复执行 seek 操作。就会有较大几率出现播放花屏问题。原因就在于失去了锁的保护，pktqueue 内部维护的状态变量就会因线程竞争，而无法保证正确的值。在 seek 操作过程中，会调用 pktqueue_reset 接口，里面会执行 pktqueue_free_enqueue 操作。就是说执行 seek 操作的线程，和 adecode, vdecode 线程 都可能执行 pktqueue_free_enqueue 操作，这就造成了共享资源的竞争，因此必须进行保护。

## 总结

通过这篇文章的讲解，大家应该基本明白，播放器实现中的 packet 队列的作用，运行原理和实现方法。同时我们分析了 pktqueue 可能存的线程安全问题以及解决办法。我们可以初步掌握多线程编程的线程安全和并发问题的分析和解决方法。多线程问题分析的关键在于线程，而不在于变量和加锁。只有存在了线程对资源的竞争，我们才去考虑保护和加锁，同时要尽量缩小加锁的粒度，提升系统的执行效率和吞吐量。













