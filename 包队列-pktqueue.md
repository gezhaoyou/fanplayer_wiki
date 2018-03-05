包队列的作用

包队列的作用是用于缓存，解复用 (demux) 阶段出来的音视频的 packet，并将音视频 packet 分发给对应的音视频解码线程。

我们前面已经知道，为了有效利用 cpu 资源，保证视频回放的帧率稳定、连贯、音视频同步等，我们必须要采用多线程的架构，来流水线的方式，来提高系统的处理能力和数据吞吐量。流水线上的工作者，就是不同的工作线程，比如 demux 线程，audio decoding 线程，video decoding 线程，... 而不同工位之间的衔接，就是这个包队列了，英文叫做 packet queue.


在解复用阶段，数据从一个输入文件读出，并分离为“视频数据包”、“音频数据包”、“字幕数据包”等。我们常用的就是音视频数据，因此我们至少需要建立，video packet queue 和 audio packet queue 两个队列。

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


在 ffplayer 的实现中，将音视频 packet 队列，集中在了 pktqueue 这样一个对象里面，统一进行管理。并定义了相应接口：

void* pktqueue_create (int   size); // important!! size must be power of 2
void  pktqueue_destroy(void *ctxt);
void  pktqueue_reset  (void *ctxt);

AVPacket* pktqueue_write_dequeue  (void *ctxt); // dequeue a packet for writing
void      pktqueue_write_enqueue_a(void *ctxt, AVPacket *pkt); // enqueue a packet as audio
void      pktqueue_write_enqueue_v(void *ctxt, AVPacket *pkt); // enqueue a packet as video
void      pktqueue_write_cancel   (void *ctxt, AVPacket *pkt); // cancel current packet writing

AVPacket* pktqueue_read_dequeue_a (void *ctxt); // dequeue a audio packet for reading
void      pktqueue_read_enqueue_a (void *ctxt, AVPacket *pkt); // enqueue a audio packet for reading
AVPacket* pktqueue_read_dequeue_v (void *ctxt); // dequeue a video packet for reading
void      pktqueue_read_enqueue_v (void *ctxt, AVPacket *pkt); // enqueue a video packet for reading

这样做的好处在于，packet 不用每次都 av_packet_alloc 出来，然后用完再 av_packet_free 掉，这样减少动态内存申请的开销。实际上在 pktqueue 的内部是同时维护着三个队列的，分别是 free 队列，audio 队列，和 video 队列。

初始化时 free 队列里面就一次性申请了 DEF_PKT_QUEUE_SIZE 个可用的 packet。pktqueue_write_dequeue 这个接口，实际上是从 free 队列取出一个空闲的 packet，对应了前面代码的 av_packet_alloc。pktqueue_write_enqueue_a 和 pktqueue_write_enqueue_v 分别是把 pkt 放入 audio 队列和 video 队列，对应了前面代码的 audio 和 video 的入队操作。pktqueue_read_dequeue_a 和 pktqueue_read_dequeue_v  被用在了解码线程，分别用于把 audio 和 video packet 出队，对应了前面代码的出队操作
