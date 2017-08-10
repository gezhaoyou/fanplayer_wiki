for win32 platform
------------------

there is a vs project file, under player-win32, named ffplayer.sln.

please install vs2005 (or higher), and open ffplayer.sln to build ffplayer for win32 platform.



for android platform
--------------------

you need android source code building enviroment or NDK enviroment.

I am using the android source code building enviroment, the build steps:

1. cd android source code dir (example android-src).

   cd android-src

2. copy ffplayer to android-src dir

   cp -r ffplayer .


3. source build/envsetup.sh


4. lunch & pick a combo

   lunch aosp_arm-eng


5. make android source code

   make -j8


6. cd ffplayer/player-android

   mm -B -j8

then you will get the testplayer.apk in android out dir.


