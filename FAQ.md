### 1. 降低 rtsp 播放延时
在 ffplayer.cpp 的 player_prepare 函数中，有段被注释掉的代码：

    av_dict_set(&opts, "rtsp_transport", "tcp", 0);
    av_dict_set(&opts, "buffer_size", "1048576", 0);
    av_dict_set(&opts, "analyzeduration", "500000", 0);
    player->avformat_context->flags |= AVFMT_FLAG_NOBUFFER;

打开试试，我这边实测 rtsp 的延时可以在 200ms 以内。

### 2. 流媒体播放自动重连功能如何实现
可以通过 player_getparam(m_ffPlayer, PARAM_MEDIA_POSITION, &pos); 这样的接口获取当前播放位置，如果 pos 为 -1 说明无法及时获取到媒体数据，很大可能性就是由于网络质量差，导致缓冲或者连接断开。这种情况下，可以关闭播放器再重新打开 player_close -> player_open。


