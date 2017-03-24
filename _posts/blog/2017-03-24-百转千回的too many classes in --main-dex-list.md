---
layout: post
title: 百转千回的too many classes in --main-dex-list
description: 不坚持，怎么能知道绝望呢~
category: blog
---

# 百转千回的too many classes in --main-dex-list

## 背景：

随着业务越来越庞大，早在两年前，项目已经遭遇了方法是超过65535的问题。
当时的解决方法是：采用google multidex方案解决；（那时项目还小，还未遭遇黑屏，启动速度optdex时间过长的问题），
一年后，黑屏，启动速度，ANR问题趋于明显；于是开始对业务层进行
[优化]("http://iceanson.github.io/Android%E5%90%AF%E5%8A%A8%E9%80%9F%E5%BA%A6-%E6%80%BB%E4%BC%9A%E9%81%87%E5%88%B0%E7%9A%84%E4%B8%8D%E7%97%9B%E4%B8%8D%E7%97%92%E7%9A%84%E5%9D%8E")，有一点点的效果，精简业务入口是优化启动速度第一步；然而好景不长，随着近两个版本大量SDK的接入，在接入multidex的情况下，成功的将主dex再次撑爆：编译时出现too manyclasses in --main-dex-list;于是开始了一翻摸索和尝试。。

## 主dex生成逻辑

too manyclasses in --main-dex-list 这个报错是因为主dex方法数超过65535的情况下，在编译时进行的报错
[具体查看]("http://ct2wj.com/2015/12/22/the-way-to-solve-main-dex-capacity-exceeded-in-Android-gradle-build/").

也就是说，主dex放的类太多了，我们来看一下主dex在gradle里是如何生成的：
	
Multidex的实现原理是将class编译进不同的classes.dex文件中，一般情况下，一个APK文件中只包含了一个classes.dex文件。分包之后就存在一个主的classes.dex，多个副的classes2.dex，classes3.dex，以此类推。

在要启动程序时，Android会先去加载主的classes.dex（这是系统自己加载的），然后在程序启动后再去加载其它副的dex（比如MultiDex.install）。

那哪些class应该被编译到主的classes.dex中呢？

先来看下Multidex的编译过程，它由三个不同的gradle task组成:

### 1、collect{variant}MultiDexComponents task
这个task会读取项目的AndroidManifest.xml文件中注册的application、Activity、service、receiver、provider、instrumentation相关类，并将其class文件路径写到文件buidl/intermediates/multi-dex/${variant.dirName}/manifest_keep.txt中

### 2、shrink{variant}MultiDexComponents task

这个task会调用ProGuard并根据上一步生成的manifest_keep.txt文件内容去压缩class，剔除没有用到的class，生成一个精简的jar包buidl/intermediates/multi-dex/${variant.dirName}/componentClasses.jar

### 3、create{variant}MainDexClassList task
这个task会根据上一步生成的componentClasses.jar去寻找这里面的各个class文字中依赖的class，比如一个class中有一成员变量X，那么X就是依赖的class，componentClasses.jar中所有的class和依赖的class路径都会被写入到文件buidl/intermediates/multi-dex/${variant.dirName}/maindexlist.txt中，这个文件中的类都会被编译进主的classes.dex中去。

[详情可以查看ClassReferenceListBuilder的实现源码]("https://android.googlesource.com/platform/dalvik/+/master/dx/src/com/android/multidex/ClassReferenceListBuilder.java")

也就是说：
对于一个使用了MultiDex的Android工程，编译后在/build/intermediates/multi-dex/{variant_path}/路径下面，可以看到如下几个文件。

```java
componentClasses.jar
components.flags
manifest_keep.txt
maindexlist.txt
```
### 也就是说：精简主dex的大小在于精简manifest_keep.txt的大小,进而减小jar的大小，进而减小maindexlist.txt的大小即可；

## 精简过程

### 首先是manifest_keep.txt大小的控制

这里有两个方面，一方面是从业务下手，减小application、Activity、service、receiver、provider、instrumentation相关类的依赖，特别对第三方SDK的依赖，但是这里对业务要求比较高，比较容易因为业务人员的改动造成分包失败，所以放到下一阶段再做； 
那么就只能从gradle task 任务入手，对collect task任务进行拦截和替换，代码如下(在app/build.gradle最尾部加入如下代码)

```java
afterEvaluate {
        android.applicationVariants.each {
            variant ->
                def collectTask = tasks.findByName("collect${variant.name.capitalize()}MultiDexComponents")//collectZroTestDebugMultiDexComponents
                if (collectTask != null) {

                    List<Action<? super Task>> list = new ArrayList<>()
                    list.add(new Action<Task>() {
                        @Override
                        void execute(Task task) {
                            println "collect${variant.name.capitalize()}MultiDexComponents action execute!---------XXXXXXX mini main dex生效了!!!!$projectDir"
                            def dir = new File("$projectDir/build/intermediates/multi-dex/${variant.dirName}");
                            if (!dir.exists()) {
                                println "$dir 不存在,进行创建"
                                dir.mkdirs()
                            }
                            def manifestkeep = new File(dir.getAbsolutePath() + "/manifest_keep.txt")
                            manifestkeep.delete()
                            manifestkeep.createNewFile()
                            println "先删除,后创建manifest_keep"
                            def backManifestListFile = new File("$projectDir/manifest_keep.txt")
                            backManifestListFile.eachLine {
                                line ->
                                    manifestkeep << line << '\n'
                            }
                        }
                    })
                    collectTask.setActions(list)
                }

        }
    }
```
然后在本地的projectDir/manifest_keep.txt配置最精简的主dex所需要依赖的类：

```java
-keep class com.meiyou.framework.biz.ui.LoadResActivity { <init>(); }
-keep class com.lingan.seeyou.messagein.NotificationTranslucentActivity{ <init>(); }
-keep class com.j256.ormlite.field.**
-keep class com.lingan.seeyou.ui.application.AppShell{
    <init>();
}
-keep class com.tencent.tinker.loader.** {
    *;
}

-keep class com.lingan.seeyou.ui.application.TinkerApp {
    *;
}
-keep class com.lingan.seeyou.ui.application.SeeyouApplication {
    *;
}
-keep public class * implements com.tencent.tinker.loader.app.ApplicationLifeCycle {
    *;
}

-keep public class * extends com.tencent.tinker.loader.TinkerLoader {
    *;
}

-keep public class * extends com.tencent.tinker.loader.app.TinkerApplication {
    *;
}
```
可以看到在这里我们只保留了application和启动页之类的入口；

使用这种优化确实可以减小一部分数量，我们也因此继续开发了半年多的时间，然而好景不长，近两个版本加入的SDK过多，导致application和lanucher页面直接依赖的方法是成功超越65535，再次暴出了too many classes in --main-dex-list；因此我们需要再进一步优化（业务裁剪是在难上加难）；


### 进一步优化

我们知道最终决定主dex大小的，是最后一个maindexlist.txt文件大小，这个文件列出了所有在主dex需要的类，如果我们能裁剪这个文件的大小，就可以将主dex精简下来；
但是gradle task只能拦截collect 这样的任务，无法替换maindexlist.txt内容，我们也暂时没有办法写依赖分析脚本来分析哪些是要放在主dex，哪些是不需要的；于是找到了一个可以干预maindexlist.txt文件内容的项目[DexKnifePlugin]("https://github.com/ceabie/DexKnifePlugin")，但是配置内容需要对整个过程有比较深入的理解才可使用，我们来看下我们是如何精简，使用方法相当简单，具体看github 说明即可，我们重点讲下我们接下来配置文件内容：

```java
# 全局过滤, 如果没设置 -filter-suggest 并不会应用到 建议的maindexlist.
# 如果你想要某个已被排除的包路径在maindex中，则使用 -keep 选项，即使他已经在分包的路径中.
# 注意，没有split只用keep时，miandexlist将仅包含keep指定的类。
#-keep android.support.v4.view.**

# 这条配置可以指定这个包下类在第二dex中.（注意，未指定的类会在被认为在maindexlist中）
#android.support.v?.**

# 使用.class后缀，代表单个类.
#-keep android.support.v7.app.AppCompatDialogFragment.class

# 不包含Android gradle 插件自动生成的miandex列表.
#-donot-use-suggest

# 将 全局过滤配置应用到 建议的maindexlist中, 但 -donot-use-suggest 要关闭.
-filter-suggest

# 不进行dex分包， 直到 dex 的id数量超过 65536.
-auto-maindex

# dex 扩展参数, 例如 --set-max-idx-number=50000
# 如果出现 DexException: Too many classes in --main-dex-list, main dex capacity exceeded，则需要调大数值
#-dex-param --set-max-idx-number=48000

# 显示miandex的日志.
-log-mainlist

#过滤日志。Recommend：在maindexlist中（由推荐列表确定）；Global：在maindexlist中，由全局过滤确定；true，前两者都成立的；false，不在maindexlist中
-log-filter

# 如果你只想过滤 建议的maindexlist, 使用 -suggest-split 和 -suggest-keep.
# 如果同时启用 -filter-suggest, 全局过滤会合并到它们中.
-suggest-split com.google.android.gms.ads.**.**
-suggest-split com.google.android.gms.**.**
-suggest-keep com.facebook.**
-suggest-keep android.support.multidex.**
-suggest-keep com.meetyou.frescopainter.**
```


配置完成之后，就看到了久违的installing apk啦！！！

## 接下来的优化

虽然解决其启动的问题，但是启动速度依然还是很慢，根本原因在于MultiDex install的时间太久，主dex太大，所以接下来的优化方向会是：
### 精简业务入口（application，service,receiver等系统组件） 
 对业务改动要求比较高；它是系统自己加载的主dex需要的数据，这一步需要一个依赖分析的脚本，来确保第一个主dex加载完成之后，我们启动到主页的情况下，能够索引到所有被引用的类，否则很容易出现ClassNotFound；

### 需要编写自定义MultiDex，来加载其余的dex，需要处理各个安卓版本MultiDex的差异;

### 处理第N个Dex未加载完成之前，用户点击了非主dex的页面的时候或者引用到非主dex的类的时候，如何防止因ClassNotFound导致的闪退；



