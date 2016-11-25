---
layout: post
title: Android Jni调试-你需要的从零开始
description: 不要放弃治疗~
category: blog
---

# Android so调试-你需要的从零开始
>本文为 GcsSloop 原创文章，采用 [知识共享 署名-非商业性使用-禁止演绎 4.0 国际 许可协议](https://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh) 发布，转载请注明作者以及原链。
> 本文在 [DiyCode](http://www.diycode.cc/topics/392)  和 [ice_Anson的个人博客](http://iceanson.github.io/) 同时首发，关注作者的 [DiyCode帐号](http://www.diycode.cc/sloop) 或者 [作者微博](http://weibo.com/iceanson?is_all=1) 可第一时间收到新文章推送。

## 前言
	
最新公司项目需要开发一个人脸融合的demo，C++部分已经由另外团队开发完成，需要跟Android进行对接。但是在so库编译完成之后，发现闪退，logcat看不到任何信息，
只能学习一下如何对so库进行调试，还是有点门槛的，再此根据实践经验理清一下思路和结果。

先说一个概念：so库的调试不是像java代码一样使用AS直接在界面里可以调试的，我们使用的gdb调试是需要在终端里进行操作调试，下面会有详细解释。

## 准备工作

1、Mac+NDKr9+AS+SDK，android5.0的设备

2、确保 android-sdk/tools 和 android-sdk/platform-tools在Path路径里；

3、确保设备已经连接-使用adb shell 看看是否能正常执行；

4、确保你的app的application加入了android:debugable="true"

5、编译使用的so库，必须用ndk-build DEBUG=1来生成，才可进行调试；

## 调试步骤

### 一、从设备提取所需要的库文件(不要怕，这里真的是只机械化操作)
	
	1、新建文件夹,取名Log（这个名字任意，自己方便就行）；
	2、cd Log;
	3、新建文件夹system_lib和vendor_lib，名字最好和这个一样了，如果自定义的话，后边的参数自行修改;
	4、cd system_lib,adb pull /system/lib，你会发现下拉了好多so库；
	5、cd ../vendor_lib,adb pull /vendor/lib，这时候也会下来好多库；
	6、cd .. 此时应该在Log目录下了，执行 
	adb pull /system/bin/app_process；
	adb pull /system/bin/app_process32；
	adb pull /system/bin/app_process64；
	adb pull /system/bin/linker 
	adb pull /system/bin/linker64
	
OK，现在所需要的库已经都准备完成了。接下就是安装gdb；

### 二、安装GDB server

1、从ndk里拷贝gdbserver
路径是：/Users/mu/Downloads/android-ndk-r9/prebuilt/android-arm/gdbserver
	
2、拷贝到工程目录下的 app/src/main/jniLibs的armeabi,armeabi-v7a,x86下边；并重命名为gdbserver.so;
	
!["sds"](http://7xnby9.com1.z0.glb.clouddn.com/so1.png)

3、用AS 运行工程，gdbserver就会运行在你的设备里了。

### 三、断点

1、给你需要调试的那行java代码进行断点，然后debug到那一步，然后保持这种状态；

![](http://7xnby9.com1.z0.glb.clouddn.com/SO2.png)

此时你会看到logcat打印处理端口号，它是这样的：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO3.png)


OK，现在app已经处于调试状态了，我们可以启动gdb server进行调试了；

### 四、启动GDB Server

1、打开终端

2、查询PID：

	adb shell "ps | grep com.richard.glestutorial"；
	其中com.richard.glestutorial是app的包名；
如果app重新调试的话，这个PID是会变的，每次进行attach的时候需要重新查询一下。
![](http://7xnby9.com1.z0.glb.clouddn.com/SO4.png)

3、端口映射：

	adb forward tcp:8100 localfilesystem:/data/data/com.richard.glestutorial/debug-pipe
	其中com.richard.glestutorial是app的包名；

4、手机上启动gdbserver作为服务端，然后让它attach到3820这个进程

	adb shell run-as com.richard.glestutorial /data/data/com.richard.glestutorial/lib/gdbserver.so +debug-pipe --attach 3820 
	其中com.richard.glestutorial是app的包名；

执行成功的话你看到的应该是这样,说明Gdbserver已经在监听调试了:
![](http://7xnby9.com1.z0.glb.clouddn.com/SO6.png)

接下来我们要启动GDB进行调试了
### 四、启动GDB 

1、打开另外一个终端；

2、进入NDK目录/Users/mu/Downloads/android-ndk-r9/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin
	
3、执行./arm-linux-androideabi-gdb /Users/mu/IMYFaceSwap-android/Log/app_process

这里如果出现"not in executable format: File format not recognized"的话，请执行

./arm-linux-androideabi-gdb /Users/mu/IMYFaceSwap-android/Log/app_process32

或者

./arm-linux-androideabi-gdb /Users/mu/IMYFaceSwap-android/Log/app_process64

成功的话它应该这样的，这个时候我们就进入gdb了：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO7.png)

其中这个路径/Users/mu/IMYFaceSwap-android/Log，是你刚才新建的Log的路径；	

4、在gdb里执行target remote:8100，关联到app的端口，就是我们刚才端口映射的那个端口

5、设置linker关联；
set solib-search-path /Users/mu/IMYFaceSwap-android/Log/:/Users/mu/IMYFaceSwap-android/Log/system_lib:/Users/mu/IMYFaceSwap-android/Log/vendor_lib:/Users/mu/IMYFaceSwap-android/Log/vendor_lib/egl:/Users/mu/IMYFaceSwap-android/app/src/main/obj/local/armeabi-v7a

这句话是告诉gdb，如果linker需要load其他依赖so的时候，gdb需要在哪个目录去找对应的文件；

这里对几个路径进行解释一下：

/Users/mu/IMYFaceSwap-android/Log/ 刚才新建了Log文件夹位置；

/Users/mu/IMYFaceSwap-android/Log/system_lib 刚才新建了Log文件夹位置；

/Users/mu/IMYFaceSwap-android/Log/vendor_lib 刚才新建了Log文件夹位置；

Users/mu/IMYFaceSwap-android/Log/vendor_lib/egl 刚才新建了Log文件夹位置；

/Users/mu/IMYFaceSwap-android/app/src/main/obj/local/armeabi-v7a；这个是在jni文件下使用ndk-build DEBUG=1编译后在工程里生成的文件路径

6、使用info sharedlibrary 检测一下是否有找到依赖，确保每个so库前边都是Yes;正常的话是这样：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO8.png)
确认完了之后，执行quit退出；

7、接下来个cpp文件下断点；执行

b com_richard_glestutorial_GLRenderer.cpp:88

b com_richard_glestutorial_GLRenderer.cpp:89

b com_richard_glestutorial_GLRenderer.cpp:90

![](http://7xnby9.com1.z0.glb.clouddn.com/SO11.png)
可以看到下断点成功；
对应的cpp源码是这样：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO12.png)


8、最激动人心的时刻，调试:
gdb里执行c，回车，此时会显示continue...
![](http://7xnby9.com1.z0.glb.clouddn.com/SO13.png)

然后回到AS,执行 Run -> Resume Program
你就会在终端里看到执行到了你的断点的位置了：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO14.png)
然后怎么进行下一步？直接执行c回车，c回车，即可，直到定位到闪退的那一行：
![](http://7xnby9.com1.z0.glb.clouddn.com/SO15.png)

可以看到，代码执行过这行： int code =  IMY::FSWrapper::swapFace();之后就闪退了，接下来就叫给C++端吧。搞定！

9、注意事项：

gdb调试忽略SIGPIPE：在gdb里执行handle SIGPIPE nostop noprint即可；

关于so库调试，最好开三个终端比较不容易乱；查询PID一个；开启gdbserver一个；gdb调试一个；

[https://github.com/mapbox/mapbox-gl-native/wiki/Android-debugging-with-remote-GDB](https://github.com/mapbox/mapbox-gl-native/wiki/Android-debugging-with-remote-GDB)

----------
QQ:452825089

mail:452825089@qq.com

wechat:ice3897315

blog:http://iceAnson.github.io