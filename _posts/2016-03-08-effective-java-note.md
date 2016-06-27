---
layout: post
title: "Effective Java 的一点记录"
description: "effective java note"
category: Develop
tags: [java]
---

对 Effective Java 这本书中的一些思想做一点记录, 持续更新....

## 对象的构建

## 接口和类的封装
对于成员(域,方法,嵌套类,嵌套接口)有四中可能的访问级别,按访问性由低到高依次是:

- **私有的 private** 只有在声明该成员的顶层类内部才可以访问该成员
- **包级私有的 package-private** 默认,或没有指定修饰符时,声明该成员的包内部的任何类都可以访问该成员
- **受保护的 protected** 声明该成员的子类可以访问,并且包内部任何类也可以访问
- **公有的 public** 任何地方都可以访问

一条重要的建议就是:**使类和成员的可访问性最小化**

### 可能存在的安全漏洞

看以下一个安全漏洞,长度为0的数组总是可变的,所以类具有的共有静态final数组域或者返回返回这种域的访问方法,这几乎总是错误的.如果类具有这样的域或者访问方法,客户端就能修改数组中的内容.

	public static final Thing[] VALUES= {...};

修正这个问题有两种方法,可以是数组变成私有的,并增加一个公有的不可变列表

	private static final Thing[] PRIVATE_VALUES= {...};
	public static final List<Thing> VALUES =
		Collection.unmodifiableList(Arrays.asList(PRIVATE_VALUES));

另一个方法是,可以使数组变成私有的,并添加一个共有方法,返回私有数组的一个备份

	private static final Thing[] PRIVATE_VALUES= {...};
	public static final Thing[] values(){
		return PRIVATE_VALUES.clone();
	}

### 和Android中相悖的原则
书中第14条:在公有类中使用访问方法而非公有域,也就是私有化域,而只提供需要的get/set方法.

而在Android开发中频繁的get/set调用会消耗Android有限的性能,所以普遍在android开发中使用了公有域. 从而也要求我们在使用的时候更加小心.

### 复合优先于继承
