---
layout: post
title: "RxJava 多数据源加载数据"
description: "RxJava Obtain Data From Multiple Sources"
category: Develop
tags: [RxJava]
---

- [参考资料](http://blog.danlew.net/2015/06/22/loading-data-from-multiple-sources-with-rxjava/)

考虑一个问题,如何用RxJava完成数据加载的三级缓存.这里主要用到的是 `concat` 和 `first`

- `concat` 会一次重多个Observable发出data,并且不会交叉
- `first` 只从Observable发出第一个满足条件的data

![concat](/images/2016-01-06-rxjava-beginner/concat.png)
![first](/images/2016-01-06-rxjava-beginner/first.png)

	// Our sources (left as an exercise for the reader)
	Observable<Data> memory = ...;  
	Observable<Data> disk = ...;  
	Observable<Data> network = ...;

	// Retrieve the first source with data
	Observable<Data> source = Observable  
	  .concat(memory, disk, network)
	  .first();

需要注意的是 `concat` 中Observable 的顺序,因为是依次被执行的.
下一步要完善的是缓存的功能,在从disk或network加载数据后,要对数据进行缓存.

	Observable<Data> networkWithSave = network.doOnNext(data -> {  
	  saveToDisk(data);
	  cacheInMemory(data);
	});

	Observable<Data> diskWithCache = disk.doOnNext(data -> {  
	  cacheInMemory(data);
	});

再然后,现在的设计导致一旦数据缓存之后就不会过期,无法获取最新的数据.因此我们需要在 `first` 中加上判断数据过期条件.

	Observable<Data> source = Observable  
	  .concat(memory, diskWithCache, networkWithSave)
	  .first(data -> data.isUpToDate());

Last point, 于 `first` 操作符类似的一个是 `takeFirst`,它们的区别在于, `first` 在没有Observable能发出有效的数据的时候会抛出 `NoSuchElementException`; 而 `takeFirst` 会简单的走到 `onComplete`.
