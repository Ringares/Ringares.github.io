---
layout: post
title: "RxJava 初步探索"
description: "RxJava Beginner"
category: Develop
tags: [RxJava]
---

这篇主要介绍 RxJava的一些基本概念.先贴一下这方面一些很好的资料:

- [官方资料](http://reactivex.io/documentation/operators.html)
- [大头鬼Bruce的博客](http://blog.csdn.net/lzyzsd/article/details/41833541)
- [扔物线的给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)
- [错误或异常在RxJava中的处理](http://blog.danlew.net/2015/12/08/error-handling-in-rxjava/)

##主要概念
在官网的介绍中,说到 ReactiveX 是一个使用可观察序列(Observable的序列)来组成的异步的基于事件的程序的库. 而RxJavaze则是其用于JVM上的实现.

RxJava的优势相较于传统的异步回调, AsyncTask或者Handler来说要简洁,而且是能够随着程序越来越复杂而依然保持其简洁. 而且由多个Observable灵活组成的序列也能很好的满足模块间的解耦和测试.

在RxJava中有4个主要的概念

- 观察者/订阅者
- 被观察者
- 订阅这个动作
- 事件

![rx concept](/images/2016-01-06-rxjava-beginner/rx_concept1.png)

![rx concept](/images/2016-01-06-rxjava-beginner/rx_concept2.png)

从观察者模式的角度开始取理解RxJava中的概念是比较容易的,就第一张图所示的按钮点击回调所示:

- Button是被观察者,OnClickListener中是观察者,观察者发出了OnClick这个事件.

- 那么同样的Observable就是被观察者,Observer就是观察者,而在这里能发出的事件或者数据有三类,onNext,OnCompleted和OnError,会被观察者分别处理.

- 最后订阅这个动作表示将被观察者和观察者关联起来.

这几个概念在代码中的实现就是

###观察者/订阅者的创建

	/****************************************
	 * 1. 观察者/订阅者的创建
	 *************************************/
	
	Observer<String> observer = new Observer<String>() {
	    @Override
	    public void onNext(String s) {
	        /**普通事件*/
	    }
	
	    @Override
	    public void onCompleted() {
	        /**事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。
	         * RxJava 规定，当不会再有新的 onNext() 发出时，需要触发 onCompleted() 方法作为标志。*/
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        /**事件队列异常。在事件处理过程中出异常时，onError() 会被触发，同时队列自动终止，不允许再有事件发出。
	         在一个正确运行的事件序列中, onCompleted() 和 onError() 有且只有一个，并且是事件序列中的最后一个。
	         需要注意的是，onCompleted() 和 onError() 二者也是互斥的，即在队列中调用了其中一个，就不应该再调用另一个。*/
	    }
	};
	
	/**
	 * 抽象类 Subscriber 对 Observer 接口进行了一些扩展，但他们的基本使用方式是完全一样的
	 * 不仅基本使用方式一样，实质上，在 RxJava 的 subscribe 过程中，Observer 也总是会先被转换成一个 Subscriber 再使用。
	 * 所以如果你只想使用基本功能，选择 Observer 和 Subscriber 是完全一样的。它们的区别对于使用者来说主要有两点
	 * <p/>
	 * 1.onStart(): 这是 Subscriber 增加的方法。它会在 subscribe 刚开始，而事件还未发送之前被调用，
	 * 可以用于做一些准备工作，例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。
	 * <p/>
	 * 2.unsubscribe(): 这是 Subscriber 所实现的另一个接口 Subscription 的方法，用于取消订阅。在这个方法被调用后，Subscriber 将不再接收事件。
	 * 一般在这个方法调用前，可以使用 isUnsubscribed() 先判断一下状态。
	 * unsubscribe() 这个方法很重要，因为在 subscribe() 之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。
	 */
	Subscriber<String> subscriber = new Subscriber<String>() {
	    @Override
	    public void onNext(String s) {
	
	    }
	
	    @Override
	    public void onCompleted() {
	
	    }
	
	    @Override
	    public void onError(Throwable e) {
	
	    }
	};
	
	
###被观察者 Observable的创建

	/****************************************
	 * 2. 被观察者 Observable的创建
	 * *************************************/
	
	/**
	 * 这里传入了一个 OnSubscribe 对象作为参数。
	 * OnSubscribe 会被存储在返回的 Observable 对象中，它的作用相当于一个计划表，当 Observable 被订阅的时候，OnSubscribe 的 call() 方法会自动被调用
	 */
	Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
	    @Override
	    public void call(Subscriber<? super String> subscriber) {
	        subscriber.onNext("Hello");
	        subscriber.onNext("Hi");
	        subscriber.onNext("Aloha");
	        subscriber.onCompleted();
	    }
	});
	
	/**
	 * create() 方法是 RxJava 最基本的创造事件序列的方法。基于这个方法， RxJava 还提供了一些方法用来快捷创建事件队列
	 */
	Observable observable2 = Observable.just("Hello", "Hi", "Aloha");
	// 将会依次调用：
	// onNext("Hello");
	// onNext("Hi");
	// onNext("Aloha");
	// onCompleted();
	
	String[] words = {"Hello", "Hi", "Aloha"};
	Observable observable3 = Observable.from(words);
	//将传入的数组或 Iterable 拆分成具体对象后，依次发送出来
	// 将会依次调用：
	// onNext("Hello");
	// onNext("Hi");
	// onNext("Aloha");
	// onCompleted();
	
	
###进行订阅
	
	/****************************************
	 * 3. 进行订阅
	 * 创建了 Observable 和 Observer 之后，再用 subscribe() 方法将它们联结起来，整条链子就可以工作了
	 * observable.subscribe(observer);
	 * observable.subscribe(subscriber);
	 *************************************/
	public void testSubscribe() {
	    Subscription subscribe = observable.subscribe(observer);
	
	    if (!subscribe.isUnsubscribed()) {
	        subscribe.unsubscribe();
	    }
	}
	
	/**不完整定义的回调
	 * */
	private String tag;
	Action1<String> onNextAction = new Action1<String>() {
	    // onNext()
	    @Override
	    public void call(String s) {
	        Log.d(tag, s);
	    }
	};
	Action1<Throwable> onErrorAction = new Action1<Throwable>() {
	    // onError()
	    @Override
	    public void call(Throwable throwable) {
	        // Error handling
	    }
	};
	Action0 onCompletedAction = new Action0() {
	    // onCompleted()
	    @Override
	    public void call() {
	        Log.d(tag, "completed");
	    }
	};
	
	public void testSubscribe2() {
	// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
	    observable.subscribe(onNextAction);
	// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
	    observable.subscribe(onNextAction, onErrorAction);
	// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
	    observable.subscribe(onNextAction, onErrorAction, onCompletedAction);
	}


##常用Operator及使用场景
RxJava提供了对事件序列进行变换的支持, 这一点也是大多数人觉得RxJava很好用的原因之一.

###map

![map](/images/2016-01-06-rxjava-beginner/map.png)

###flatMap

![flatMap](/images/2016-01-06-rxjava-beginner/flatmap.png)

###buffer

![buffer](/images/2016-01-06-rxjava-beginner/buffer.png)

###debounce

![debounce](/images/2016-01-06-rxjava-beginner/debounce.png)

###throttleFirst

![throttleFirst](/images/2016-01-06-rxjava-beginner/throttlefirst.png)

###filter

![filter](/images/2016-01-06-rxjava-beginner/filter.png)

##RxJava的线程调度
Scheduler(调度器)在Rx中起到举足轻重的作用,一手包揽了线程切换,异步任务的实现.先看一下默认提供的scheduler

- `Schedulers.immediate()`: 直接在当前线程运行，相当于- - 不指定线程。这是默认的 Scheduler。
- `Schedulers.newThread()`: 总是启用新线程，并在新线程执行操作。
- `Schedulers.io()`: I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
- `Schedulers.computation()`: 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
- 另外， Android 还有一个专用的 `AndroidSchedulers.mainThread()`，它指定的操作将在 Android 主线程运行。

**对线程进行控制**

- `subscribeOn()`: 指定 subscribe() 所发生的线程，即 Observable.OnSubscribe 被激活时所处的线程。或者叫做事件产生的线程。
- `observeOn()`: 指定 Subscriber 所运行在的线程。或者叫做事件消费的线程。


##在Android中的使用
要注意在Activity生命周期变化时取消订阅,防止内存泄露.

`CompositeSubscription`通过`add()`,集中管理subscriptions,在需要的时候可以一起取消订阅.

	@Override
	protected void onResume() {
	    super.onResume();
	    subscriptions = new CompositeSubscription();
	    bindRxObservable();
	}
	
	@Override
	protected void onPause() {
	    super.onPause();
	    RxUtils.unsubscribe(subscriptions);
	}