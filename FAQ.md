### 1. 流媒体播放自动重连功能如何实现
需要配置初始化参数：

    init_timeout   = 1000;
    auto_reconnect = 1000;

即可实现自动重连功能。

### 2. 开启硬件加速
win32 平台，需要配置初始化参数：

    vdev_render_type = 1;
    video_hwaccel    = 1;


android 平台，需要配置初始化参数：
    video_hwaccel    = 1;




