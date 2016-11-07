---
layout: post
title: Roboletric探索之路，从抗拒到依赖
description: Roboletric Android Unit Testing
category: blog
---

##我为什么以前抗拒Android Unit Testing

- 1、懒，人类最大的天敌；
- 2、不是不知道什么是单元测试，只是需求太多了，哪有时间~；
- 3、需要学习单元测试的语言或者框架，不熟悉，所以从没尝试过；
- 4、没见到单元测试的好处，一想到要花时间就望而却步；
- 5、至少只是我个人之前的感受，我相信有很多的程序猿同胞们都跟我有类似的感受；
	
##既然抗拒，为什么现在要尝试Android Unit Testing呢

大势所趋，bug量的增多不得不让我们提高代码的质量，不是我们完不成功能，只是我们验证功能的成本实在太高，随着工程的复杂度的增加，run一次模拟器或者真机，在window上的花费至少是一分钟以上，甚至三四分钟，所以有些人偷懒，包括我，有时候把那些看上去“没有问题的代码”提交到了主干上，随之产生了bug，然后进入修复bug-》run-》修复bug->run；花费了更多的时间和资源；
>我们的燃眉之急是要尽快改善这个问题，从根源着手，就是【增强自测】

##测试手段

现在是个讲究效率的时代，我们希望能够快速高效的验证我们的代码逻辑是否有问题，我们不希望验证一个简单的逻辑或者一个方法是否有效，是通过run一次模拟器或者整个工程实现的，这样花费的时间太长了，降低了开发效率；

- 1、所以我们要解决的第一个痛点是，如何快速验证；

我们选择了Robolectric单元测试框架，原因有好几个，最大的原因是：

>Robolectric
Test-Drive Your Android Code
Running tests on an Android emulator or device is slow! Building, deploying, and launching the app often takes a minute or more. That's no way to do TDD. There must be a better way.

>Wouldn't it be nice to run your Android tests directly from inside your IDE? Perhaps you've tried, and been thwarted by the dreaded java.lang.RuntimeException: Stub!?
>

它不需要Run你的模拟器，直接在jvm上运行你的测试代码，能在几秒钟之内快速验证，通过体验之后，它确实非常高效，编写测试代码反而加速了开发效率。
具体的原理描述可参见：

[Robolectic官网](http://robolectric.org/)

[Robolectic介绍](https://hkliya.gitbooks.io/unit-test-android-with-robolectric/content/0-introduction.html)

##Talk is cheap ,show me the code
###环境配置
Android Stuido 1.5.1

junit:junit:4.12

Robolectric 3.0(不要用3.0-r3，有很多bug，踩了很多坑)

具体配置：

1、在app的build.gradle依赖添加如下：

>testCompile 'junit:junit:4.12'  
testCompile ’org.robolectric:robolectric:3.0’ 

2、在android studio左下角的Build Variants->TestArtifact，选择为Unit Tests；
>![如图](http://7xnby9.com1.z0.glb.clouddn.com/0D878ECA-B0EF-4FE2-A64A-0B42A0A2B254.png)

3、编译；

编译通过之后就已经集成了Robolectic单元测试框架了。

###我的第一个单元测试

####先描述以下踩过的坑：

>1.使用3.0-r3版本，原因是google搜到的例子是3.0-r3,傻傻的掉进了坑里；
>找不到合并后的mainifest;ContextWraper为空，webview初始化异常，无法加载so库

>2.使用了3.0版本后，依然无法加载so库，现象是启动application的时候如果调用so库会出现闪退，目前robolectic还不支持这个东西，作者在github的issue已经说明；
>
>3.这一点导致无法robolectic无法直接集成到我们的主工程；为了不影响正常项目开发，我们建了一个AppTest的空项目，集成了Robolectic框架；将所有用例分类写在里边，各个模块要测试的时候，把依赖写入build.gradle即可；

####第一单元测试

>创建测试类
>![日历分析页面测试demo](http://7xnby9.com1.z0.glb.clouddn.com/388902D3-EE30-499A-B9AF-CB401DBFD73B.png)
参数解释：

@RunWith(RobolectricGradleTestRunner.class)；声明使用哪个Runner,使用GradleTestRunner会自动帮我们加载所需要的插件，一般我们配这个就可以；

@Config(constants = BuildConfig.class,sdk=18)；配置测试项目的BuildConfig和sdk版本，BuildConfig是编译器自动生成；@Config还可以配置很多其他的熟悉，比如Manifest,Application,assert资源等等，具体了解可参见
[http://robolectric.org/configuring/](http://robolectric.org/configuring/)；

extend TestCase:这个必须继承的类；	
	
@Before是前置条件，也就是在执行@Test之前会执行的方法，这个很好理解，@After同理；

@Test具体的单元测试方法；

####例子解释：	
@Before

	@Before
    public void setUp() {
    	 //获取当前运行环境的Context;
        Context context =  RuntimeEnvironment.application.getApplicationContext();
        //初始化BeanManager
        BeanManager.getUtilSaver().setContext(context);
        //初始化日历模块
        CalendarController.getInstance().init(context, new OnCalendarListener(){});
    }
	
	每次执行test的时候，Robolectic执行顺序是：
	模拟启动执行你的application，
	执行@Before
	执行@Test,
	执行@After
	所以如果没有没有执行初始化逻辑，@test很有可能会失败；
	或者application里调用了so或者初始化了webview，也会失败；
	一般在@Before我们做的是初始化的工作和构造一些模拟数据的操作；
@Test

 	@Test
    public void doTestMainUI(){
    	 //启动AnalysisMainActivity，并获取activity对象
        Activity activity = Robolectric.setupActivity(AnalysisMainActivity.class);
        //获取里边的控件
        RelativeLayout linearLayout = (RelativeLayout)activity.findViewById(R.id.calendarNodataLayout);
        int visible = linearLayout.getVisibility();
        //判断是否可见
        assertEquals(visible, View.VISIBLE);
    }
	
	@Test
    public void doTestCurrentIdentify(){
    	  //获取当前身份，可以在setup设置身份
        int value = CalendarController.getInstance().getIdentifyManager().getIdentifyModelValue();
        //验证身份
        assertEquals(value, IdentifyModel. NORMAL);
    }

 	@Test
    public void testDomain(){
                List<HttpDnsModel> list = mHttpDnsCacheManager.getHttpDnsModels();
                assertEquals(list.size(), 2);
                //验证解析格式
                String domain = mHttpDnsCacheManager.getDomain("http://api.myms.meiyou.com/configs");
                assertEquals(domain,"api.myms.meiyou.com");
                //验证解析格式
                domain = mHttpDnsCacheManager.getDomain("https://api.myms.meiyou.com/configs");
                assertEquals(domain, "api.myms.meiyou.com");
                //验证命中缓存
                HttpDnsModel model =  mHttpDnsCacheManager.getFromCache("api.myms.meiyou.com/configs");
                assertEquals(model!=null,true);
                assertEquals(model.getIp(), "211.151.209.71");
                //验证替换
                String result = mHttpDnsCacheManager.replaceDomainToIp("http://api.myms.meiyou.com/configs", domain, model.getIp());
                assertEquals("http://211.151.209.71/configs",result);

        }
              
  	@Test
   	public void testGetPhoto(){
   					//记录器
                final Transcript transcript = new Transcript();
                //创建一个activity
                Activity activity = new Activity() {
                        @Override protected void onActivityResult(int requestCode, int resultCode, Intent data) {
                                //记录收到的结果
                                transcript.add("onActivityResult called with requestCode " + requestCode + ", resultCode " + resultCode + ", intent data " + data.getData());
                        }
                };
                //启动去相册选择相册
                activity.startActivity(new Intent().setType("image/*"));
				   //获取影子类，模拟设置ActivityResult结果
                Shadows.shadowOf(activity).receiveResult(new Intent().setType("image/*"), Activity.RESULT_OK,
                        new Intent().setData(Uri.parse("content:foo")));
                //验证收到的结果       
                transcript.assertEventsSoFar("onActivityResult called with requestCode -1, resultCode -1, intent data content:foo");
        }



####强大的影子类：
[影子类的详细了解](http://robolectric.org/extending/)；
 影子类是对安卓原生api类的一种拓展，通俗的解释是增加了一些用于测试的很方便的方法；3.0有很多影子类，获取方法是：Shadows.shadowOf();
 上边的 Shadows.shadowOf(activity).receiveResult的方法；
 影子类还可以自定义，遇到比较复杂的功能或者需要的功能可能会很有用；

####单元测试的例子

大量的测试例子都在github的源码里边，可以详细参照；
[Robolectric Github 源码地址](https://github.com/robolectric/robolectric)；

Done
----------
QQ:452825089

mail:452825089@qq.com

wechat:ice3897315

blog:http://iceAnson.github.io
