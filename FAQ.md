### 1. 降低 rtsp 播放延时
在 ffplayer.cpp 的 player_prepare 函数中，有段被注释掉的代码：

    av_dict_set(&opts, "rtsp_transport", "tcp", 0);
    av_dict_set(&opts, "buffer_size", "1048576", 0);
    av_dict_set(&opts, "analyzeduration", "500000", 0);
    player->avformat_context->flags |= AVFMT_FLAG_NOBUFFER;

打开试试，我这边实测 rtsp 的延时可以在 200ms 以内。

### 2. 流媒体播放自动重连功能如何实现
需要配置初始参数：

    init_timeout   = 500;
    auto_reconnect = 500;

即可实现自动重连功能。


