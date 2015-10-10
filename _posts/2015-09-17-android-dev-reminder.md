---
layout: post
title: "Android之不为人知小角落"
description: "Android Dev Reminder"
category: Develop
tags: [android]
---

记录一些Android开发相关的，不太关注到，但又很实用的小零碎。

##PathMeasure进行Path取点
通过PathMeasure可以计算出path上的各个点的坐标以及这个点的切线

>**public boolean getPosTan (float distance, float[] pos, float[] tan)**
>
>**Description:**
>
>Pins distance to 0 <= distance <= getLength(), and then computes the corresponding position and tangent. Returns false if there is no path, or a zero-length path was specified, in which case position and tangent are unchanged.
>
>**Parameters:**
>
>- distance: The distance along the current contour to sample
>- pos: If not null, eturns the sampled position (x==[0], y==[1])
>- tan: If not null, returns the sampled tangent (x==[0], y==[1])
>
>**Returns:**
>
>- false if there was no path associated with this measure object

例子代码:

	Path path1 = new Path();
	path1.addCircle(width / 2, height / 2, height / 2, Path.Direction.CW);
	canvas.drawPath(path1, paint);
	
	PathMeasure pathMeasure = new PathMeasure(path1, false);
	
	float length = pathMeasure.getLength();
	float[] points = new float[2];
	float[] tangent = new float[2];
	
	System.out.println("canvas: " + width + " " + height);
	for (int i = 0; i < 8; i++) {
	    pathMeasure.getPosTan(length * i / 8, points, tangent);
	
	    System.out.println("Path的" + i + "/8");
	    System.out.println(points[0] + " " + points[1]);
	    System.out.println(tangent[0] + " " + tangent[1]);
	}

在圆上取点的输出的结果

	I/System.out﹕ canvas: 540 240
	
	I/System.out﹕ Path的0/8
	I/System.out﹕ point: x=390.0 y=120.0
	I/System.out﹕ tangent: 0.0 1.0
	
	I/System.out﹕ Path的1/8
	I/System.out﹕ point: x=354.8528 y=204.85281
	I/System.out﹕ tangent: -0.7071068 0.7071067
	
	I/System.out﹕ Path的2/8
	I/System.out﹕ point: x=270.00006 y=240.0
	I/System.out﹕ tangent: -1.0 3.0698317E-7
	...
	I/System.out﹕ Path的7/8
	I/System.out﹕ point: x=354.85287 y=35.147232
	I/System.out﹕ tangent: 0.7071065 0.70710707
	
![path取点](/images/2015-09-17-android-dev-reminder/pathmeasure.png)

##PathMeasure进行Path截取
先看效果
![path截取](/images/2015-09-17-android-dev-reminder/PathMeasure.gif)

>**public boolean getSegment (float startD, float stopD, Path dst, boolean startWithMoveTo)**
>
>**Description:**
>
>Given a start and stop distance, return in dst the intervening segment(s). If the segment is zero-length, return false, else return true. startD and stopD are pinned to legal values (0..getLength()). If startD <= stopD then return false (and leave dst untouched). Begin the segment with a moveTo if startWithMoveTo is true

具体实现就是一个值动画加上`PathMeasure.getSegment()`方法，从而不停的重绘而达成想要的效果。知道原理后可以YY出很多比较炫的效果。


	@Override
	protected void onDraw(Canvas canvas) {
	    super.onDraw(canvas);
	
	    if (drawPath != null) {
	        canvas.drawPath(drawPath, paint);
	    }
	
	}
	
	public void drawWithStatus() {//可传参绘制不同状态
	
	    Point startPoint = new Point(width / 4, height / 2);
	    Point endPoint = new Point(width * 3 / 4, height / 4);
	
	    Path path = new Path();
	    path.moveTo(startPoint.x, startPoint.y);
	    path.lineTo(width / 2, height * 3 / 4);
	    path.lineTo(endPoint.x, endPoint.y);
	    pathMeasure = new PathMeasure(path, false);
	
	    if (mPhareAnimator == null) {
	        mPhareAnimator = ValueAnimator.ofFloat(0.0F, 1.0F);
	        mPhareAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	            @Override
	            public void onAnimationUpdate(ValueAnimator animation) {
	                float value = (Float) animation.getAnimatedValue();
	                updatePrecess(value);
	            }
	        });
	
	        mPhareAnimator.setDuration(NORMAL_ANIMATION_DURATION);
	        mPhareAnimator.setInterpolator(new LinearInterpolator());
	    }
	
	    //清空canvas,重置值动画
	    drawPath.reset();
	    invalidate();
	    mPhareAnimator.cancel();
	    mPhareAnimator.start();
	}
	
	public void updatePrecess(float process) {
	    this.process = process;
	    if (pathMeasure.getSegment(0, process * pathMeasure.getLength(), drawPath, true)) {
	        drawPath.rLineTo(0, 0);//解决低版本可能出现的绘制问题
	    }
	
	    invalidate();
	}
	
##XML预览
>**xmlns:tools="http://schemas.android.com/tools"**

这个命名空间可以帮助我们更好的进行xml布局时的预览

###tools中的属性不会打包到APK中
它拥有android：中所有的属性，但它标识的属性仅仅在预览中有效，不会影响真正的运行结果。
举个例子：

    <TextView
        android:text="Footer"
        android:layout_width="wrap_content"
        android:layout_height="100dp"
        />
        
这是我们之前的一个写法，把textView的text属性用android：来标识。如果我们希望这个textview的文字在代码中实时控制，默认是没文字怎么办？这就需要tools的帮助了。

	<TextView
        tools:text="Footer"
        android:layout_width="wrap_content"
        android:layout_height="100dp"
        />
        
把第一行的android替换为tools这样既可以能在预览中看到效果，又不会影响代码实际运行的结果。因为在实际运行的时候被tools标记的属性是会被忽略的。你完全可以理解为它是一个测试环境，这个测试环境和真实环境是完全独立的，不会有任何影响。

**问题:** tools标签不支持代码提示，而且自己的属性也不能提示，全是靠自己记忆，或者先用android来代替，然后替换android为tools。

###tools帮助预览listview等
以前全靠脑补其中填充item的样子，借助tools可以进行item的预览


	<?xml version="1.0" encoding="utf-8"?>
	<ListView xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    tools:listheader="@layout/header_list"
	    tools:listitem="@layout/item_list"
	    tools:listfooter="@layout/footer_list"
	    />
	    
![listview预览](/images/2015-09-17-android-dev-reminder/tools.png)

实验下来发现其实还是有问题 (环境: Android Studio 1.4)

- 只有当ListView处于布局最底层时才显示item预览
- footer貌似不能正常显示
