---
layout: post
title: "单元测试异步任务及Robolectric框架"
description: "UnitTest for asynctask"
category: Develop
tags: [android,test]
---

##InstrumentationTestCase的一些tips

这里尝试的例子都是在Android Studio 中，基于 InstrumentationTestCase, AndroidTestCase 而进行的。

> extends InstrumentationTestCase
> 
> extends AndroidTestCase

###初始化及回收

    @Override
    protected void setUp() throws Exception {
        super.setUp();
    }

    @Override
    protected void tearDown() throws Exception {
        super.tearDown();
    }


##异步任务的单元测试

对异步任务的测试需要暂停当前的线程，等异步任务执行完成之后，继续暂停的线程，完成测试

	// 异步test
	public void testMyAsyncTask() throws Throwable {
	    // 指示任务何时结束
	    final CountDownLatch signal = new CountDownLatch(1);
	    // 异步的任务
	    AsyncTask<String, Void, String> myTask = new AsyncTask<String, Void, String>() {
	        @Override
	        protected String doInBackground(String... arg0) {
	            //Do something meaningful.
	            try {
	                Thread.sleep(2000);
	            } catch (InterruptedException e) {
	                // TODO Auto-generated catch block
	                e.printStackTrace();
	            }
	            return "something happened!";
	        }
	
	        @Override
	        protected void onPostExecute(String result) {
	            super.onPostExecute(result);
	            System.out.println("-->" + result);
	            signal.countDown();
	        }
	    };
	    // 开始异步任务
	    myTask.execute("Do something");
	
	    // 暂停当前的线程,等待异步任务完成
	    try {
	        signal.await();
	    } catch (InterruptedException e) {
	        fail();
	        e.printStackTrace();
	    }
	}

##测试框架Robolectric的坑
尝试使用Robolectric进行测试，环境是Android Studio 1.4， sdk=21
这是基于Junit，脱离了Android sdk，只需要jvm就可以快速的进行测试。

	dependencies {
	    testCompile 'junit:junit:4.12'
	    testCompile 'org.robolectric:robolectric:3.0-rc3'
	}
	
将BuildVariants修改为`Unit Test`，就会在`-src`下生成`test`文件夹

![BuildVariants](/images/2015-10-20-unittest-for-asynctask/BuildVariants.png)

![结构](/images/2015-10-20-unittest-for-asynctask/struc.png)

运行测试，并自动生成测试报告

![runtest](/images/2015-10-20-unittest-for-asynctask/runtest.png)

![result](/images/2015-10-20-unittest-for-asynctask/result.png)

运行测试的方法还有

- 在Terminal中 `./gradlew test`
- 右键测试类名, run

![runtest2](/images/2015-10-20-unittest-for-asynctask/runtest2.png)


	
###测试代码Example
	package com.ring.unittest;
	
	import android.widget.Button;
	import android.widget.TextView;
	
	import org.junit.Before;
	import org.junit.Test;
	import org.junit.runner.RunWith;
	import org.robolectric.Robolectric;
	import org.robolectric.annotation.Config;
	
	import static org.junit.Assert.assertEquals;
	import static org.junit.Assert.assertNotNull;
	import static org.junit.Assert.fail;
	
	/**
	 * To work on unit tests, switch the Test Artifact in the Build Variants view.
	 */
	@RunWith(RobolectricDataBindingTestRunner.class)
	@Config(constants = BuildConfig.class, sdk = 21)
	public class ExampleUnitTest {
	    @Test
	    public void addition_isCorrect() throws Exception {
	        assertEquals(4, 2 + 2);
	    }
	
	    // 引用待测Activity
	    private MainActivity mActivity;
	
	    // 引用待测Activity中的TextView和Button
	    private TextView textView;
	    private Button button;
	
	    @Before
	    public void setUp() throws Exception {
	        // 获取待测Activity
	        mActivity = Robolectric.setupActivity(MainActivity.class);
	
	        // 初始化textView和button
	        textView = (TextView) mActivity.findViewById(R.id.textView);
	        button = (Button) mActivity.findViewById(R.id.button);
	    }
	
	    // 测试界面初始化结果
	    @Test
	    public void testInit() throws Exception {
	        assertNotNull(mActivity);
	        assertNotNull(textView);
	        assertNotNull(button);
	
	        // 判断包名
	        assertEquals("com.xuxu.roboletricdemo", mActivity.getPackageName());
	
	        // 判断textView默认显示的内容
	        assertEquals("Hello world!", textView.getText().toString());
	    }
	
	    // 测试点击button，textView显示的内容
	    @Test
	    public void testButton() throws Exception {
	        // 点击button
	        button.performClick();
	
	        // 判断点击后textView的内容
	        assertEquals("Hello xuxu!", textView.getText().toString());
	    }
	
	    // 一个失败的用例
	    @Test
	    public void testFail() throws Exception {
	        fail("This case failed!");
	    }
	}

###问题（坑）
Q: 开始测试，执行的时候卡在`:app:testDebugUnitTest`

A: 不知道原因，可能是测试框架缺少某些关联的包，执行`grandle projects`中的`test`任务会在`studio`的`Run`界面开始下载一些东西，自动下载完成后问题解决。

![run](/images/2015-10-20-unittest-for-asynctask/run.png)

![run2](/images/2015-10-20-unittest-for-asynctask/run2.png)

Q: 报错 `build/intermediates/res/debug/values is not a directory`

A: 这是因为项目结构的问题导致的，将测试代码的`@RunWith`改为以下代码类就可以解决问题 

[感谢这篇资料提出的解决方案](https://philio.me/android-data-binding-with-robolectric-3/)

![Q1](/images/2015-10-20-unittest-for-asynctask/Q1.png)

	package com.ring.unittest;
	
	import org.robolectric.RobolectricTestRunner;
	import org.robolectric.annotation.Config;
	import org.robolectric.manifest.AndroidManifest;
	import org.robolectric.res.FileFsFile;
	import org.robolectric.util.Logger;
	import org.robolectric.util.ReflectionHelpers;
	
	/**
	 * Created by ring
	 * on 15/10/22.
	 */
	public class RobolectricDataBindingTestRunner extends RobolectricTestRunner {
	
	    private static final String BUILD_OUTPUT = "build/intermediates";
	
	    public RobolectricDataBindingTestRunner(Class<?> klass) throws org.junit.runners.model.InitializationError {
	        super(klass);
	    }
	
	    @Override
	    protected AndroidManifest getAppManifest(Config config) {
	        if (config.constants() == Void.class) {
	            Logger.error("Field 'constants' not specified in @Config annotation");
	            Logger.error("This is required when using RobolectricGradleTestRunner!");
	            throw new RuntimeException("No 'constants' field in @Config annotation!");
	        }
	
	        final String type = getType(config);
	        final String flavor = getFlavor(config);
	        final String applicationId = getApplicationId(config);
	
	        final FileFsFile res;
	        if (FileFsFile.from(BUILD_OUTPUT, "res", flavor, type).exists()) {
	            res = FileFsFile.from(BUILD_OUTPUT, "res", flavor, type);
	        } else {
	            // Use res/merged if the output directory doesn't exist for Data Binding compatibility
	            res = FileFsFile.from(BUILD_OUTPUT, "res/merged", flavor, type);
	        }
	        final FileFsFile assets = FileFsFile.from(BUILD_OUTPUT, "assets", flavor, type);
	
	        final FileFsFile manifest;
	        if (FileFsFile.from(BUILD_OUTPUT, "manifests").exists()) {
	            manifest = FileFsFile.from(BUILD_OUTPUT, "manifests", "full", flavor, type, "AndroidManifest.xml");
	        } else {
	            // Fallback to the location for library manifests
	            manifest = FileFsFile.from(BUILD_OUTPUT, "bundles", flavor, type, "AndroidManifest.xml");
	        }
	
	        Logger.debug("Robolectric assets directory: " + assets.getPath());
	        Logger.debug("   Robolectric res directory: " + res.getPath());
	        Logger.debug("   Robolectric manifest path: " + manifest.getPath());
	        Logger.debug("    Robolectric package name: " + applicationId);
	        return new AndroidManifest(manifest, res, assets, applicationId);
	    }
	
	    private String getType(Config config) {
	        try {
	            return ReflectionHelpers.getStaticField(config.constants(), "BUILD_TYPE");
	        } catch (Throwable e) {
	            return null;
	        }
	    }
	
	    private String getFlavor(Config config) {
	        try {
	            return ReflectionHelpers.getStaticField(config.constants(), "FLAVOR");
	        } catch (Throwable e) {
	            return null;
	        }
	    }
	
	    private String getApplicationId(Config config) {
	        try {
	            return ReflectionHelpers.getStaticField(config.constants(), "APPLICATION_ID");
	        } catch (Throwable e) {
	            return null;
	        }
	    }
	}

