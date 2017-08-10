I write a script for build ffmpeg for android.

enviroment needed:

1. a ubuntu pc which can connect to internet


2. install the android standalone toolchain

   arm-linux-androideabi-4.7.1

   download link: https://pan.baidu.com/s/1jHO6uCA passcode: k7x9


3. setup the toolchain:

   export PATH=$PATH:$HOME/arm-linux-androideabi-4.7.1/bin


4. execute build_ffmpeg_for_android.sh

   ./build_ffmpeg_for_android.sh



I suggest to use toolchain which version under 4.9.

because some friend using arm-linux-androideabi-4.9.x get the compile error. but 4.8.x and 4.7.x is ok.




