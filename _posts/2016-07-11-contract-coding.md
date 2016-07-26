---
layout: post
title: " 契约编程: Annotation"
description: "Contract Coding: Annotation"
category: Develop
tags: [android]
---
我们可以从系统源码及大多是开源项目里看到很多 Annotation 注解的身影. 此篇介绍 Android 为我们提供的 android.support.annotation, 以及在什么场景下如何使用.

Annotations 可以帮助你写出更有意义的契约,它的表现力要大于注释和文档, 而且 Inspection 可以利用这些 Annotations 在 coding 的同时显示警告或错误, 帮你检测出潜伏的bug.

[Improve Code Inspection with Annotations](https://developer.android.com/studio/write/annotations.html#enum-annotations)

	dependencies {
	    compile 'com.android.support:support-annotations:23.3.0'
	}

## Nullness Annotations

![NotNull_0](/images/2016-07-11-contract-coding/NotNull_0.png)

![NotNull_1](/images/2016-07-11-contract-coding/NotNull_1.png)

Nullness Annotations 一般包括 `@Nullable` 和 `@NotNull` 用于标明参数,字段或方法返回值能否为 null. 通常组件对外提供 API 时, 外部调用者并不知道接口方法有没有做 参数 null 的处理, 往往在 调用前以及接口内部都做了重复空处理或者都没做. 这种情况下, 如果非空参数就经过两次重复的验证, 不仅浪费性能而且强迫症玩家会比较痛苦. Nullness Annotations 就能很好的解决这种问题, 接口的调用者能直观看到  API 设计者的意图.


## Resource Annotations

![StringRes](/images/2016-07-11-contract-coding/StringRes.png)

Resource Annotations 一般用于标明参数的具体类别是 res 资源 id. 一个 int 类型的参数可以接收 int 值或者 res 值, 如果没有文档的话, 对调用者来说并不知道应该要传什么, 很容易造成异常.

## Thread Annotations

![thread](/images/2016-07-11-contract-coding/thread.png)

`@MainThread` 和 `@UiThread` 基本上是一样的, 其细微的区别在于: `@MainThread` 用于生命周期相关; `@UiThread` 用于 View Hierarchy.

Thread Annotations 一般就用于标明方法或构造的线程要求.

## Value Constraint Annotations

![IntRange](/images/2016-07-11-contract-coding/IntRange.png)

`@IntRange`, `@FloatRange` 一般用于表明接收参数的取值范围, 例如之前遇到的时间控件的 API, 要求传入的分钟就是 `@IntRange(from = 0, to = 59)` 以及设置透明度的 `@IntRange(from=0,to=255)`等

![Size](/images/2016-07-11-contract-coding/Size.png)

![Size_1](/images/2016-07-11-contract-coding/Size_1.png)

`@Size` 一般用于限制集合和数组的尺寸. `@Size(2)`要求一定有两个元素; `@Size(min=1, max=10)` 要求至少有1个元素又不超过10个.

## Permission Annotations

![RequiresPermission](/images/2016-07-11-contract-coding/RequiresPermission.png)

Permission Annotations 一般用于方法, 构造, 标明需要什么权限, 如果 Manifest 里没有申明的就会以红下划线标出.

## CheckResults Annotations

![CheckResult](/images/2016-07-11-contract-coding/CheckResult.png)

CheckResults Annotations 一般用于当接口的返回值需要被使用时, 若调用者未使用, 则会标识.出来

## CallSuper Annotations

![CallSuper](/images/2016-07-11-contract-coding/CallSuper.png)

CallSuper Annotations !!!很有用处. 当子类重写父类方法时,如果父类中含有一些初始化必须要走的代码,父类就可以用 `@CallSuper` 要求子类必须调用父类实现 `super.XXX()`.

例如: `Activity` 源码中的 `onCreate` 等生命周期方法.

![CallSuper_1](/images/2016-07-11-contract-coding/CallSuper_1.png)

## Enumerated Annotations

![EnumeratedAnnotations](/images/2016-07-11-contract-coding/EnumeratedAnnotations.png)

Enumerated Annotations 用于限定一些预先设定的作为状态或模式的常量, 在 API 的入参及返回时, 只接收预先设定的常量中的一个.

上图是 ActionBar 的源码, 其中就设定了3个 Int 常量作为导航模式. 具体的使用方式如下:

- `@interface NavigationMode` 创建一个注解
- `@IntDef({...})` 设定 type 在哪几个值中
- `@Retention(RetentionPolicy.SOURCE)` 标识给编译器不要将注解保存到 .class 中

还有某些情况下16进制常量是可以通过位运算组合使用的, `IntDef` 的字段 `flag == true` 就可以满足这种情况, 如下图所示

![EnumeratedAnnotations_1](/images/2016-07-11-contract-coding/EnumeratedAnnotations_1.png)

## 总结
android 自带的注解可以有效的帮助多人协作的接口做输入和输出的限制, 对于单人开发的模块也可以比文档更直观更方便的体现一些重要的信息. 个人认为以下几种最为常用, 并且值得立刻开始使用:

- Nullness Annotations
- Resource Annotations
- Value Constraint Annotations
- CallSuper Annotations
- Enumerated Annotations


