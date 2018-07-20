win32 platform
--------------

there is a vs project file, under player-win32, named fanplayer.sln.  
please install vs2005 (or higher), and open fanplayer.sln to build fanplayer for win32 platform.



android platform
----------------

you need install android-sdk and android-ndk.

### 1. build fanplayer jni library for apk.

under fanplayer\player-android\jni dir, execute build.bat (window) or build.sh (ubuntu).


### 2. build fanplayer apk

under fanplayer\player-android\apk dir, execute build.bat (window).  
or using android studio to open the project, to build and debug.


got compile error ?
-------------------
### 1. for vs2015 you may got "timespec" : "struct" redefinition compile error.

please modify the pthread.h file in player-win32\pthreads-win32\include, add
> #define _TIMESPEC_DEFINED

at the begin of this file.

### 2. for vs2015 got "error LINK2026: module unsafe for SAFESEHimage"
please change settings in project configuration:
> Configuration Propertiese -> Linker -> Advanced -> Image Has Safe Exception Handlers

set to No (/SAFESEH:NO)

### 3. for android ndk build got "conditional branch to PLT in THUMB-2 not supported yet."
please change build_ffmpeg_for_android.sh, remove "--enable-thumb" from ffmpeg configure.

