how to build ffmpeg ?
---------------------

1. winxp/win7 mingw64 environment  

2. download ffmpeg and libmfx source code  
> git clone https://github.com/lu-zero/mfx_dispatch  
> git clone git://source.ffmpeg.org/ffmpeg.git  

3. build libmfx  
> autoreconf -i  
> ./configure --host=i686-w64-mingw32  
> make -j8 && make install  

4. build ffmpeg  
> ./configure \  
> --pkg-config=pkg-config \  
> --arch=x86 \  
> --target-os=mingw32 \  
> --enable-cross-compile \  
> --cross-prefix=i686-w64-mingw32- \  
> --prefix=$PWD/../ffmpeg-win-sdk \  
> --enable-static \  
> --enable-shared \  
> --enable-small \  
> --enable-memalign-hack \  
> --disable-swscale-alpha \  
> --disable-symver \  
> --disable-debug \  
> --disable-programs \  
> --disable-doc \  
> --disable-avdevice \  
> --disable-avfilter \  
> --disable-postproc \  
> --disable-encoders \  
> --disable-muxers   \  
> --disable-devices  \  
> --disable-filters  \  
> --enable-asm \  
> --enable-gpl \  
> --enable-version3 \  
> --enable-nonfree \  
> --enable-dxva2 \  
> --enable-d3d11va \  
> --enable-libmfx  
> make -j8 && make install  


why using winxp/win7 mingw64 ?
------------------------------

--enable-dxva2 and --enable-d3d11va these options will enable dxva2 and d3d11 hardware acceleration for win32 platform, which only could take effect by mingw64 toolchain.

--enable-libmfx this option will enable intel media sdk hardware acceleration for win32, the best way to build libmfx for win32 platform, is using mingw64 on win32.

this configure enables all hardware accelerations ffmpeg currently supported on win32, and disables all encoders and any other useless which not used by a player.


meet "ERROR libmfx not found"
-----------------------------

please check pkg-config installed or not, try command:
pkg-config --list-all

if you can see libmfx, it means pkg-config installed ok, and libmfx complied and installed ok. if not, you could fix this problem first.


make install failed after ffmpeg make ?
---------------------------------------

I also meet this problem, failed to find .lib file. I modified the Makefile of ffmpeg:

> -install: install-libs install-headers  
> +install: install-headers install-libs  

to make it install headers first. for the libs file, we only need .dlls & .def, so find it in ffmpeg source code dir. strip dlls to reduce file size, use vs2005 lib.exe tool to create .lib from .def：

lib /def:avformat-57.def  /MACHINE:IX86 /out:avformat.lib  

after this you will get headers, dlls & libs.


meet _wfopen_s of msvcrt.dll issue ?
------------------------------------

this is dll of msvcrt.dll issue, for winxp the msvcrt.dll does not contains the _wfopen_s api entry.
using dependency walker tool to open dlls of ffmpeg to check will dll has problem.
using UltraEdit (or other) to open and edit the dll, change _wfopen_s to _wfopen.


can't find libstdc++-6.dll and libgcc_s_sjlj-1.dll ?
----------------------------------------------------

they are under the mingw64 toolchain directory. find them and release them with ffmpeg dll and your apps.




rockcarry  
2016-9-1  
