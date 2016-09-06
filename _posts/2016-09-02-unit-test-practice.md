---
layout: post
title: "Android 单元测试, MVP, 依赖注入 与 MOCK"
description: "Unit Test Practice"
category: Develop
tags: [android]
---

## 什么样的测试适合 Android

Android 测试概念的范围不小, 各种测试框架也有很多选择:

- 单元测试
	- JUnit
	- Robolectric 等
- 集成测试
	- Instrumentation
	- Espresso 等
	
到底什么样的测试适合 Android应用, 先说结论如下, 随后详细说明
	
- 使用 `Junit` 在基于 `Roboliatric` 提供的在 JVM 可运行 Android 测试的环境下
- 对符合 `MVP` 原则的项目结构的模块, 进行小颗粒的单元测试
- 适当的使用 `注入` 和 `Mock` 处理模块间的相互关联, 以及被测单元正确性的验证

这里每一项涉及到的知识都可以展开讲很多, 这一篇不展开细节, 仅先建立其中的联系, 以及解释选择这些工具的原因.
	

## 为什么 Android 的单元测试这么难

可以从测试环境及项目结构两个角度来看待这个问题, 进而从两个角度找寻解决办法.

### 测试环境

Android 项目的编译依赖本地下载的不同版本的 SDK, 而程序的运行必须依赖真实的设备或虚拟机. 这是因为我们的 IDE 和 SDK 只提供了开发和编译一个项目的环境, 并没有提供运行这个项目的环境, 原因是因为 android.jar 里面的实现是不完整的, 它们只是一些 stub, 如果你打开 android.jar 下面的代码去看看, 你会发现方法都只有一行实现:

`throw RuntimeException("stub!!”); `

![stub](/images/2016-09-02-unit-test-practice/stub.png)

而程序编译打包的时候, 编译器只会检查调用方法接口的定义以及语法. 而当在真实设备上运行是, 调用的是系统真实的实现.

在这种情况下, 当我们对项目中的代码进行测试的时候, 不可避免的要运行在真实的设备或模拟器上, 就导致了耗时大大增加, 一次测试需要几分钟, 完全不是单元测试的耗时级别.

同样的, 官方提供的 instrumentation 以及 集成/UI测试的神器 Espresso 也都是要运行在真实环境中的. 但集成测试只是测试体系中占据金字塔顶端的一小部分位置, 并不在单元测试的考虑范围内.

![unit_test](/images/2016-09-02-unit-test-practice/unit_test.png)

Robolectric 就此应运而生, 简单的来说, Robolectric 在本地提供了一套能够只在 JVM 执行的虚假实现(shadow), 我们直接运行/测试的代码, 会去调用 Robolectric 提供的 shadow 实现, 而因为只在本地 JVM 运行, 速度得到了很好的保障, 例如:

![shadow](/images/2016-09-02-unit-test-practice/shadow.png)

除了基本实现了所有的公共接口外, 还提供了很多额外的很多接口，可以读取对应的Android类的一些状态. 例如, 我们知道 ImageView 有一个方法叫 setImageResource(resourceId), 然而并没有一个对应的 getter 方法叫 getImageResourceId(), 这样你是没有办法测试这个ImageView是不是显示了你想要的 image. 而在 Robolectric 实现的对应的 ShadowImageView 里面, 则提供了 getImageResourceId() 这个接口. 你可以用来测试它是不是正确的显示了你想要的 Image. 

![shadow](/images/2016-09-02-unit-test-practice/shadowImageView.png)

**所以当 Robolectric 加入后, 我们就可以用 JUnit 测试纯 Java 代码; 而用 Robolectric 来测试 Android 相关代码等, 并且保证极快的测试速度.**

### 项目结构

MVP 的模型或者叫结构, 作为一个比较热门的话题也有一段时间了. 具体 MVP 的概念就不在此赘述, 仅提供一些比较好的资料.

- [!!! googlesamples 官方的 MVP 使用的例子, 里面还包括很多分支, 分别是配合 dagger2 以及 rxjava 使用的例子等](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)
- [腾讯 Bugly: 一步一步实现Android的MVP框架](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653577546&idx=1&sn=e10be159645a3aa8f6d6f209420fb412&scene=1&srcid=0728U4135JRuGMFkOWkOtQBm&key=8dcebf9e179c9f3a86d9048f80442fd7cadb9f4b0d76a7e184cc8aa4eaeb0c79d39a92070e8e17bdfe87e4fda296cd62&ascene=0&uin=MTYzMjY2MTE1&devicetype=iMac+MacBookPro10%2C1+OSX+OSX+10.11.6+build(15G31)&version=11020201&pass_ticket=V8QaEJ%2BmvU8bCCIOxfM%2F0xMiG4Kpfz5HF%2BfSb%2FYk0MY%3D)
- [译 Android开发中的MVP架构](http://www.jianshu.com/p/7567ed0d1853)
- [另一种与官方推荐不同的 MVP 实践, 可借鉴](http://kymjs.com/code/2015/11/09/01/)

![mvp](/images/2016-09-02-unit-test-practice/mvp.png)

对于经典的 Android MVC 结构来说, 如果只是简单的应用, 业务逻辑写到 Activity 下面并无太多问题, 但一旦业务逐渐变得复杂起来, 每个页面之间有不同的数据交互和业务交流时, activity 的代码就会急剧膨胀, 代码就会变得可读性, 维护性很差.

当实际使用 MVP 的时候, 会发现代码量会增加, 相应带来的是代码清晰度, 维护性和可测试性的提升. 之所以能使代码变的可测试是因为 MVP 将底层的逻辑, 业务逻辑和数据模型等与界面显示和跳转进行了分离, 所以被分离出来的逻辑就基本上是 java 实现, 进而非常便于 JUnit 的测试.

另一方面, 这部分逻辑非常重要,如果产生 BUG 不一定会显式的体现在 QA 的阶段, 并且相较于产品的UI显示及交互相对较少需要对这部分逻辑进行变动和修改, 所以对这部分的测试价值比较高.

**当进行较为庞大复杂的项目时候, 考虑 MVP 的结构, 不仅利于项目的维护,扩展, 还对进行单元测试有非常大的帮助.**

## 依赖注入 (Dependency Injection)

依赖注入之所以是单元测试不可缺少的一环, 可以从两个场景来解释:

- 被测试的模块依赖其它模块
- 测试一个没有返回值的方法

在实践中就发现, 想要用 MVP 分离底层和业务逻辑, 并不容易. 因为每个模块并不会单独运作, 多个模块之间必然是相互依赖的. 例如一个显示数据的模块依赖了用户登录状态的模块, 只有用户登录后才能显示相关的数据. 当我们测试数据模块的某个**方法A**时, 调用了用户状态模块的**方法B**. 这时如何只对 方法A 进行单元测试呢, 那结论就是对这个模块从外部注入一个模拟(mock)的被依赖的模块的对象.

相较于在模块内部创建对象, 一般的 DI 有以下几种实现:

**Setter Injection**

	public class DataPresenter {
		 private UserManager mUserManager;
		 
		 //这种方法肯能是对原有项目影响最小的一种改造方法, 能够临时的满足测试的依赖分离的需要
		 public void setUserManager(UserManager userManager){
		 		this.mUserManager = userManager;
		 }

	    public void fetch(String keyWord) {
	        //... some other code
	        if(userManager.isLogin()){
	        		//... some other code
	        }
	    }
	}

**Argument Injection**

	public class DataPresenter {
	    //这里，DataPresenter不再持有UserManager的一个引用，而是作为方法参数直接传进去
	    public void login(UserManager userManager, String keyWord) {
	        if(userManager.isLogin()){
	        		//... some other code
	        }
	    }
	}
	
**Constructor Injection**

	public class DataPresenter {
	    private final UserManager mUserManager;
	    private final DbManager mDbManager;
	
	    //将UserManager作为构造方法参数传进来, 这种构造注入的方法是最为常用的
	    public DataPresenter(UserManager userManager, DbManager dbManager) {
	        this.mUserManager = userManager;
	        this.mDbManager = dbManager;
	    }
	
	    public boolean update() {
	        //... some other code
	        mDbManager.update(String data);
	    }
	    
	    public boolean fetch(String keyWord) {
	        if(userManager.isLogin()){
	        		//... some other code
	        		return true;
	        }
	        return false;
	    }
	}
	
在测试的时候, 通过注入一个模拟的被依赖对象 `userManager` , 就可以轻松的控制其行为, 例如  `userManager.isLogin()` 的返回值, 从而只对被测方法的逻辑进行测试. 一个被注入的 Mock 对象就可以做到!

另一方面, 如上文所示的 `update(String)` 方法并没有返回值, 那么要如何测试方法的正确性呢. 答案就是, 只需要证明 `mDbManager.update()` 被调用了, 而且参数是预期的. 那么就可以证明在此单元内的正确性. `update` 具体的执行就交给它的单元测试去负责了.

要证明方法的调用, 接收的参数, 甚至某个方法被调用了多少次, 每次调用的参数是什么, 哪些方法没有被调用过, 同样的一个被注入的 Mock 对象就可以做到! 果如被测方法最终调用了自己类的其它方法怎么办...把自己也mock了, 

### Mock

上文所讲的神奇的 Mock 到底是什么? Mock 是一种模拟的概念, 通过模拟一个类, 拦截所有或选择性的拦截其内部的方法, 从而监控/控制其所有行为.

在 Java 单元测试的领域中, 有两个 Mock 框架运用的最为广泛

- Mockito 通常情况下就可以满足需要了
- PowerMock

以下代码展示了一小部分 Mockito 的用法, 具体的 `mock`, `spy`, `verify` 和 `matcher`相关的细节就太多, 不赘述了.

	@RunWith(MockitoJUnitRunner.class)
	public class WeatherMainPresenterTest {
	
	    private ApplicationManager applicationManager;
	    private WeatherMainContract.View view;
	    private MyPreferences myPreferences;
	    private DBManager dbManager;
	    private WeatherMainPresenter presenter;
	
	    @Before
	    public void setup(){
	        applicationManager = mock(ApplicationManager.class);
	        view = mock(WeatherMainContract.View.class);
	        myPreferences = mock(MyPreferences.class);
	        dbManager = mock(DBManager.class);
	
	        //将被 Mock 的对象注入
	        presenter = new WeatherMainPresenter(
	                view,
	                applicationManager,
	                myPreferences,
	                dbManager);
	    }
	
	    @Test
	    public void testStart_city_data_exit() throws Exception {
	        //arrange
	        //控制 mock 对象的行为
	        doReturn("朝阳").when(myPreferences).getCityName();
	        doReturn("101010300").when(myPreferences).getCityKey();
	
	        WeatherMainPresenter presenter_spy = spy(presenter);
	        doNothing().when(presenter_spy).getCityList();
	        doNothing().when(presenter_spy).addWeatherView();
	
	        //act
	        presenter_spy.start();
	
	        //assert
	        //验证setPagerAdapter方法是否被调用, 参数是否是 PagerAdapter 的对象
	        verify(view).setPagerAdapter(Mockito.any(PagerAdapter.class));
	
	        //有当前城市信息的, 校验是否获取城市列表+添加 WeatherView 到界面
	        //验证 自己的两个方法有没有被调用
	        verify(presenter_spy).getCityList();
	        verify(presenter_spy).addWeatherView();
	    }
	}

### Dagger2

值得一提的一个问题是, 当面对项目比较复杂是, 你会发现需要注入的模块对象非常多, 是不是构造参数已经突破天际了. 一些强大的依赖注入框架可以帮助解决类似问题, 并且带来一些其它好处.

Android 中最有名的注入框架应该就是 google 的 Dagger2 了, 通过在编译时就完成依赖注入的框架, 对程序性能几乎没有影响, 并且能够通过注解动态且简单的切换被注入的是真实的对象还是 Mock 对象, 所以和单元测试也很搭.

Dagger2 的学习运用也是很大一部分内容, 留些链接以供学习

- [Android Architecture Blueprints - MVP + Dagger2](https://github.com/googlesamples/android-architecture/tree/todo-mvp-dagger/)
- [Google官方MVP+Dagger2架构详解](http://www.jianshu.com/p/01d3c014b0b1)
- [Dependency Injection with Dagger 2](https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2)
- [Dagger2从入门到放弃再到恍然大悟](http://www.jianshu.com/p/39d1df6c877d)


### 注入时遇到 static 怎么办

当实际使用中, 特别是对现在项目代码测试时, 就会发现以前存在的一些全局的静态方法很难简单的处理. 解决方案是:

- 在使用依赖注入后, 少用静态工具类, 静态是单元测试的死敌
- 没办法, 不得不用的时候, 使用 PowerMock

## 总结
一切实践测试的工具及方法都是为了做好测试, 进而提高项目的质量和效率; 如果应为过于追求细节和工具, 反而导致项目的问题, 那就未免舍本逐末. 所以面对不同的项目情况, 一切都以合适为前提把. 总结对 Android 项目测试的途径就是: 

- 使用 `Junit` 在基于 `Roboliatric` 提供的在 JVM 可运行 Android 测试的环境下
- 对符合 `MVP` 原则的项目结构的模块, 进行小颗粒的单元测试
- 适当的使用 `注入` 和 `Mock` 处理模块间的相互关联, 以及被测单元正确性的验证

## 其它相关资料

- [Against Android Unit Tests](http://www.philosophicalhacker.com/2015/04/10/against-android-unit-tests/) 强烈推荐, 从代码剖析了传统结构难于测试的原因, 并且提出了使代码可测的修改案例, 有理有据, 令人信服! 
- [美团:如何与业务功能结合](http://tech.meituan.com/Android_unit_test.html)

