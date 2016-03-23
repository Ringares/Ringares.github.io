---
layout: post
title: "RxJava Error处理"
description: "RxJava Error Handling"
category: Develop
tags: [RxJava]
---

- [参考资料](http://blog.danlew.net/2015/12/08/error-handling-in-rxjava/)

通常情况下,没有经过处理/捕获的异常会直接自动传递个onError

	Observable.just("Hello!")  
	  .map(input -> { throw new RuntimeException(); })
	  .subscribe(
	    System.out::println,
	    error -> System.out.println("Error!")
	  );

而在某个变换操作中先捕获的异常,则通常有两种处理方式

	//假如有方法如下抛出 IOException
	String transform(String input) throws IOException;

	//第一种,返回值的,仍是抛出RuntimeException
	Observable.just("Hello!")  
	  .map(input -> {
	    try {
	      return transform(input);
	    } catch (Throwable t) {
	      throw Exceptions.propagate(t);
	    }
	  })

	//第二种,返回Observable的,直接返回 Observable.error(t)
	Observable.just("Hello!")  
	  .flatMap(input -> {
	    try {
	      return Observable.just(transform(input));
	    } catch (Throwable t) {
	      return Observable.error(t);
	    }
	  })

原则上, `onError` 是用来处理发生十分严重的问题,而导致序列终断的情况.而如果需要在错误发生的时候进行下一步操作 `onNext` (假如有多个数据源,依次被请求,对某个数据源的请求失败了,还需要继续请求下一个的情况),这种情况该如何处理.

### onErrorReturn & onErrorResumeNext
可能有一下几种途径:

- [onErrorReturn()](http://reactivex.io/RxJava/javadoc/rx/Observable.html#onErrorReturn(rx.functions.Func1)
- [onErrorResumeNext()](http://reactivex.io/RxJava/javadoc/rx/Observable.html#onErrorResumeNext(rx.functions.Func1)

![onErrorReturn](/images/2016-03-18-rxjava-error-handling/onErrorReturn.png)
![onErrorResumeNext](/images/2016-03-18-rxjava-error-handling/onErrorResumeNext.png)

### retry
需要注意的是,`retry` 虽然可以方便的在遇到错误时重试,但最终返回的结果序列会重复的包括发生错误之前的部分.

	Observable.interval(1, TimeUnit.SECONDS)  
	  .map(input -> {
	    if (Math.random() < .5) {
	      throw new RuntimeException();
	    }
	    return "Success " + input;
	  })
	  .retry()
	  .subscribe(System.out::println);

![retry](/images/2016-03-18-rxjava-error-handling/retry.png)
