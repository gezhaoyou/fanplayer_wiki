how to build ffmpeg ?
---------------------

1. winxp/win7 msys2 environment  
download and install msys2

2. install required packages

>     pacman -S git
>     pacman -S make
>     pacman -S mingw-w64-i686-gcc
>     pacman -S mingw-w64-i686-yasm
>     pacman -S mingw-w64-i686-nasm
>     pacman -S mingw-w64-i686-pkg-config
>     pacman -S mingw-w64-i686-diffutils


3. execute build_ffmpeg_for_win32.sh script  
   (you can find it in ffplayer project source code.)

> ./build_ffmpeg_for_win32.sh



can't find xxxxx.dll ?
----------------------

they are under the msys2 directory. find them and release them with ffmpeg dlls and your apps.


