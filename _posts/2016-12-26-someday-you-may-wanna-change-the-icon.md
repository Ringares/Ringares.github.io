---
layout: post
title: "动态地给应用换个图标"
description: "Someday you may wanna change the icon"
category: Develop
tags: [android]
---

快过节了, 怎么给应用换个图标搞搞气氛呢, 就像这样; 虽然即刻应该只是更新了版本换了图标而已.
![](/images/2016-12-26-someday-you-may-wanna-change-the-icon/14827436496084.jpg)
这篇简单讲下实现图标变换的一种方式, 和它的局限性, 特定需求下可能会有机会发挥作用.

## 简单原理
通过 `Manifest` 注册另一个 Launcher入口 `activity-alias`, 将其指向对应的实际 activity. 但是将其设置为 `android:enabled="false"`. 此时应用有两个入口, 一个默认开启, 一个默认不可用, 安装后, 显示的只有一个开启的入口. 而特定条件满足时,可以通过 `PackageManager` 来重新控制哪一个入口得到启用, 从用户的角度来看就像是应用换了一个图标一样. 其实不光图标, `android:label` 也就是入口名也一样可以不同.

## 实现效果

![查看gif](/images/2016-12-26-someday-you-may-wanna-change-the-icon/iconchanger.gif)



## 代码实现
代码很少就可以实现基础功能~

1.在 `Manifest` 多注册一个 `acticity-alias`, 但是先设 `enabled=false`
![](/images/2016-12-26-someday-you-may-wanna-change-the-icon/14827441806189.jpg)

2.通过PackageManager 控制入口的显示
![](/images/2016-12-26-someday-you-may-wanna-change-the-icon/14827444019500.jpg)

3.简单代码逻辑
![](/images/2016-12-26-someday-you-may-wanna-change-the-icon/14827444597651.jpg)


## 问题及局限

1.不能实时改变, 调用 `PackageManager` 处理 `Component` 后, 根据 `ROM` 的不同, 经过不同的时间(MI4上大概是10s), `Launcher` 就会刷新, 从而应用的图标就会改变了

2.当程序中, 调用 `PackageManager` 处理 `Component`, 而当`Launcher` 刷新时, 应用会被自动关闭, 所以需要仔细考虑调用的位置(可能在特定时间点, 应用退出前走一下这个逻辑)

3.当入口从 `.MainActivity` 变为 `.Xmas` 后, 如果从 AndroidStudio 直接 run 的话, 你会发现报错了, 因为安装后编译器去默认打开的入口还是 `.MainActivity`, 然而这个入口已经换成了 `.Xmas` 在系统中不可用了. 但其它安装方式并没有问题, 而且保持被覆盖前的入口.
![](/images/2016-12-26-someday-you-may-wanna-change-the-icon/14828249602578.jpg)

## 参考原文
[参考链接地址](http://www.jianshu.com/p/6c1c9d3a6d1a)





