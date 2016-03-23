---
layout: post
title: "RxJava 常用 Operators"
description: "RxJava Operators"
category: Develop
tags: [RxJava]
---

记录用到的变换操作符和使用场景,以及一些遇到的Tips

##功能性的Operator

##过滤作用的Operator

###filter
![filter](/images/2016-03-22-rxjava-operators/filter.png)

>Filters items emitted by an Observable by only emitting those that satisfy a specified predicate.  
>返回一个Observable,只发出从源头Observable发出的,满足特定条件的事件.

**场景**

- 获取数据后,剔除无效或不合适的数据

###distinct
![distinct](/images/2016-03-22-rxjava-operators/distinct.png)

>Returns an Observable that emits all items emitted by the source Observable that are distinct.  
>返回一个Observable,只发出从源头Observable发出的,不重复的事件.

**场景**

- 用于对一个请求多次尝试后,或者多个数据源有重复数据时,筛选出不重复的数据.

###take
![take](/images/2016-03-22-rxjava-operators/take.png)

>Returns an Observable that emits only the first count items emitted by the source Observable. If the source emits fewer than count items then all of its items are emitted.  
>返回一个Observable,只发出从源头Observable发出的,取前几个,或开始后一段时间内的事件.

**场景**

- ...

**相关**

`first` `last` `takeFirst` `takeLast`
first在源头Observable没有数据发出的时候,会报NoSuchElementException;而takeFirst则会直接onComplete

###debounce(throttleWithTimeout)
![debounce](/images/2016-03-22-rxjava-operators/debounce.png)

>Returns an Observable that only emits those items emitted by the source Observable that are not followed by another emitted item within a specified time window.  
>返回一个Observable,只发出从源头Observable发出的,在一定时间内没有其他事件跟随的事件.

**场景**

- 输入框搜索: 在停止连续输入后一段时间,触发查询的请求

###throttleFirst
![throttleFirst](/images/2016-03-22-rxjava-operators/throttleFirst.png)

>Returns an Observable that emits only the first item emitted by the source Observable during sequential time windows of a specified duration.  
>返回一个Observable,只发出从源头Observable在一系列确定时间区间发出的第一个事件.

**场景**

- 防止界面的多次点击:在一定时间内的多次点击,只响应第一次

**相关**

`throttleLast` `sample`

##组合性的Operator

###concat
![concat](/images/2016-03-22-rxjava-operators/concat.png)

>Returns an Observable that emits the items emitted by each of the Observables emitted by the source Observable, one after the other, without interleaving them.  
>

**相关**

`merge` `concatMap`
concat在意顺序,而merge不保证顺序

###mergeDelayError


###switchMap


###combineLatest

>Combines a collection of source Observables by emitting an item that aggregates the latest values of each of the source Observables each time an item is received from any of the source Observables, where this aggregation is defined by a specified function.  
>将多个Observable组合起来,每当任何一个Observable发出事件时,将所有Observable的最后发出的数据总合起来,通过一个特点的function发出一个事件.

**场景**

- 多条件校验: 输入信息时,在每个输入框编辑后进行校验,所有条件通过后,提交按钮才可用

##compose & Transformers
用`transformers`来包装变换操作,用`compose`来进行复用.