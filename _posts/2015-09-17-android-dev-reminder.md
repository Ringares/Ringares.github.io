---
layout: post
title: "Android之不为人知小角落"
description: "Android Dev Reminder"
category: Develop
tags: [android]
---

记录一些Android开发相关的，不太关注到，但又很使用的小零碎。

##PathMeasure进行Path取点
通过PathMeasure可以计算出path上的各个点的坐标以及这个点的切线

>**public boolean getPosTan (float distance, float[] pos, float[] tan)**

**Description:**

Pins distance to 0 <= distance <= getLength(), and then computes the corresponding position and tangent. Returns false if there is no path, or a zero-length path was specified, in which case position and tangent are unchanged.

**Parameters:**

- distance: The distance along the current contour to sample
- pos: If not null, eturns the sampled position (x==[0], y==[1])
- tan: If not null, returns the sampled tangent (x==[0], y==[1])

**Returns:**

- false if there was no path associated with this measure object

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
	
![example](/images/2015-09-17-android-dev-reminder/pathmeasure.png)

##PathMeasure进行Path截取
To be continue