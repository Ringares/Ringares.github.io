---
layout: post
title: "单元测试相关"
description: "UnitTest for asynctask"
category: 
tags: [android,test]
---

这里尝试的例子都是在Android Studio 中，基于 InstrumentationTestCase 而进行的。

> extends InstrumentationTestCase

##单元测试的一些tips

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

