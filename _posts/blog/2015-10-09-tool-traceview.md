---
layout: post
title: 工具篇-TraceView 
description: 让我们远离卡顿和黑屏 2015-10-09
category: blog
---

## 让我们远离卡顿和黑屏##

### 一、与Traceview的不得已de 相逢 ###
当我们被山一样的业务需求压倒喘不过气的时候，经常就有人跑到你身边说
	    为什么某个页面的启动速度那么慢？
		为什么这台手机app的启动速度那么慢？
		为什么滑动在有些手机上会出现卡顿？
		为什么*****那么慢？
早期我们使用了疑问句的回复：
        1. 内存不够？
        2. 一定是手机不好吧？
        3. 我的手机明明不会啊！！
        4. 可是别的应用不会卡，UI也没我们复杂
其实我们内心很清楚，一定是哪里出bug了
刚开始的时候使用了很土的打点的办法来查看每个方法执行的时间，天真的以为可以轻松解决这个问题，恩，果然很天真，写了N多的打点，效果不明显，也就是-并没有什么卵用了，猿类生存环境进一步恶化。
接着开始了google，性能分析的工具traceview渐渐浮出水面。

### 二、 ^-^ Traceview 毫无保留的付出和帮助 ^-^ ###
1. **开始使用traceview**
	
	**(1) 场景一**

	我们需要查看某段代码的执行时间消耗在什么地方,那么我们就可以这么写：
	
	代码开始时执行：
	```java
	android.os.Debug.startMethodTrace(String tracename);```
	>这是trace文件名称,如果你写trace_home，生成的文件名就是trace_home.trace
	
	代码结束时执行：
	```java
	andoird.os.Debug.stopMethodTrace();	```

	完了之后启动app,让这段代码执行完毕，此时，trace文件已经在你的sdcard目录下，然后执行
	```java
	adb pull /sdcard/trace_home.trace;```
	会将此文件pull到你的用户目录下，也就是Administrator文件夹下。	
	
	打开方式：可以使用
	```java 
	traceview trace_home.trace```或者打开TraceView工具来打开这个文件
	
	**(2) 场景二**

	我们并不知道耗时是在哪段代码，我们只知道这个页面滑动比较卡，那么我们可以这么做：

	打开Android Device Monitor,选择进程，点击start Method Profiling

	![](http://i.imgur.com/WNy4HXN.jpg)

	当其变为黑色时候说明已经开始采集数据，此时滑动之前认为卡顿的页面，停止后，再点击一次进行停止,此时会自动跳转到traceview页面，页面如下：
	![](http://7xnby9.com1.z0.glb.clouddn.com/trace_view_overview.jpg)
	
	至此，我们已经揭开了造成卡顿的幕后黑色的神秘的面纱。

2. **traceview参数分析-找到我们想要的**

	抛一个问题：我们想要的是什么？是我们的具体业务代码执行了多少时间；
	然而从上图里，我们几乎看不出业务代码在哪里，所以，我们开始找，在这之前解释一下下面这几个参数：
	
	(1)上图的上左边区域：是指线程的名称和数量，这里可清楚的看到运行着哪些线程，如main指主线程，Picasso-Idea是Picasso线程；
	
	(2)上图的上右边区域：是指对应线程所分到的cpu执行时间和具体的执行方法，所有黑色区域的底部边缘都有对应的颜色对应这上图底部区域总具体的某个执行方法；
	
	(3)上图的底部区域：罗列了此次统计执行时间和对应的执行方法，这是我们要重点分析的对象；先熟悉几个重要的指标：
	
		a：(0)toplevel是顶级节点
			可以从这里一级一级看到有哪些子节点执行了了什么;
			以及每个子节点执行的时间;
			此节点的之后每个节点都有Parent和Children两个分支;
			是字面上的意思，可以追溯到父节点和子节点
		b: Incl Cpu Time %是指总时间的百分比
		c: Incl Cpu Time 是指占用了多少时间（毫秒）； 
		d: Calls+RecurCalls/Total指调用次数+递归次数/总数 
		e: Cpu Time/Call 每执行 一次该方法消费的时间（毫秒）
		
	**具体分析**
	上图中底部区域图如下：（MDP抽风，上传不了图了- -，一定是代理坏了，口头描述吧）
		
		（1）展开0（toplevel）如下图
![](http://7xnby9.com1.z0.glb.clouddn.com/top_level.jpg)
上图检测到的5718ms主要消耗在Thread.run和NativieStart.main这两个上面，我们主要来看一下main里边做了什么东西，一直点击main的childeren直到这里：
![](http://7xnby9.com1.z0.glb.clouddn.com/main_fenbu.jpg)
这里开始出现了时间分布，TraversalRunable跟踪下去是计算绘制和实际绘制的时间，无法跟踪到业务代码,这部分的优化需要对布局优化，检查过度绘制；
我们我们主要看一下FlingRunnable做了什么，继续狂戳children直到这里：
![](http://7xnby9.com1.z0.glb.clouddn.com/fling_final.jpg)
好了，业务代码全部都出现了，哪个方法调用了多少次，消耗了多少时间，一目了然；
在这里，我们发现：
getview调用了22次，消耗时间690ms
getview 每次执行时间 31ms;时间越短性能越好
fillResource(）调用了22次,消耗了192ms
handleOtherType()调用了19次，消耗190ms
这里调试的手机是魅蓝note,感觉稍微有点卡，不是很明显；
从上边参数有两个优化点：fillResource（）改为只执行一次，handleOtherType的时间主要是消耗在：
SkineEngine每次都执行了getResource()重新拿资源转为Drawable或者bitmap的缘故，需要对SkinEngine的资源ID和Drawable做缓存，可大量减少IO时间；
![](http://7xnby9.com1.z0.glb.clouddn.com/handleSetOther.jpg)

Done
----------
blog:http://iceAnson.github.io