---
layout: post
title: "Fresco的一点研究"
description: "Study of Fresco"
category: Develop
tags: [android]
---

这一篇主要对Fresco个模块进行分别的说明(源码的分析有一部分会在代码中以注释的形式写出),过程中涉及到一些各模块的联系,东一句西一句的可能不太连贯.Fresco框架确实包含比较多的内容,对各模块有了解后,再走一遍初始化及请求流程可能就会比较清楚了.

东西比较多,如有疏漏请多包涵啦~XD

没有涉及如何使用和配置,具体如何使用,可参考:

[http://fresco-cn.org/docs/getting-started.html](http://fresco-cn.org/docs/getting-started.html#_)

[http://blog.happyhls.me/2015/08/03/fresco学习笔记/](http://blog.happyhls.me/2015/08/03/fresco学习笔记/)

## 常用图片加载框架比较

| 框架 | 出品 | 特点 |
|    ---    |    :---:    |    ---    |
|Universal ImageLoader|[nostra13](https://github.com/nostra13/Android-Universal-Image-Loader)|1.支持下载进度监听<br>2.可以在 View 滚动中暂停图片加载<br>3.默认实现多种内存缓存算法<br>4.支持本地缓存文件名规则定义|
|Picasso|[square](https://github.com/square/picasso)|1.自带统计监控功能,包括缓存命中率,已使用内存大小等<br>2.支持优先级处理<br>3.支持飞行模式、并发线程数根据网络类型而变<br>4.自己没有实现本地缓存,交给okhttp实现,这样的好处是可以通过请求 Response Header 中的 Cache-Control 及 Expired 控制图片的过期时间|
|Glide|[bumptech](https://github.com/bumptech/glide)|1.Glide 不仅是一个图片缓存，它支持 Gif、WebP、缩略图。甚至是 Video，所以更该当做一个媒体缓存<br>2.支持优先级处理<br>3.与 Activity/Fragment 生命周期一致，支持 trimMemory<br>4.支持 okhttp、Volley<br>5.内存缓存中有引用记数的设计<br>6.与 Activity/Fragment 生命周期一致，支持 trimMemory|
|Fresco|[facebook](https://github.com/facebook/fresco)|1.Native 缓存,自动回收<br>2.支持JPEG的渐进式展现<br>3.支持Gif, WebP<br>4.自带圆角,自定义焦点裁切<br>5.多层绘制图片,支持例如点击显示图层及加载进度条等<br>5.官方没有说,但是看源码也有缓存命中的回调|

[详情及具体分析可见](http://www.trinea.cn/android/android-image-cache-compare)

## Drawee模块
Drawee模块用于展示图片,是一个类mvc架构.

- M -> DraweeHierarchy
- V -> DraweeView
- C -> DraweeController

三者的互动关系很简单，DraweeView 把获得的 Event 转发给 Controller，然后 Controller 根据 Event 来决定是否需要显示和隐藏 （包括动画）图像，而这些图像都存储在 Hierarchy 中，最后 DraweeView 绘制时直接通过 DraweeHierarchy.getTopLevelDrawable 就可以获取需要显示的图像。

![runtest](/images/2015-12-02-study-of-fresco/drawee-mvc.png)

**注意点和基本关系**

- 现在DraweeView还是继承自ImageView,官方声明以后将会直接继承自View,所以尽量不要使用ImageView自己的方法
- 出于解耦的考量,DraweeView中包含成员DraweeHolder,所有`get/setHierarchy`及`get/setController`等等的方法都是DraweeHolder实现的,便于自定义view替换DraweeView
- DraweeHierarchy提供获取顶层drawable的方法
- DraweeController接收并处理DraweeView通过DraweeHolder传递过来的Attach/Detach/TouchEvent事件

### DraweeView的继承体系

![draweehierarchy](/images/2015-12-02-study-of-fresco/DraweeView.png)

**GenericDraweeView**继承自DraweeView,实现了两种构造方式

- 传入GenericDraweeHierarchy,
- 从xml构造,并通过`inflateHierarchy()`从`GenericDraweeHierarchyBuilder`创建一个GenericDraweeHierarchy,**在这个过程中提取了xml设置的各种属性**

然后将设置给ViewHolder,并直接显示DraweeHierarchy的TopDrawable

	//DraweeView
	/** Sets the hierarchy. */
	public void setHierarchy(DH hierarchy) {
	  mDraweeHolder.setHierarchy(hierarchy);
	  super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
	}

**SimpleDraweeView**的构造与GenericDraweeView相似,其中在`init()`方法中用到的`sDraweeControllerBuilderSupplier`,其实是在`Fresco.init()`最初的初始化时创建好的.

### DraweeHierarchy的继承体系

![draweehierarchy](/images/2015-12-02-study-of-fresco/DraweeHierarchy.png)

**DraweeHierarchy**接口被设计来代表一个drawable树,为了可以动态可改变的显示图片.相较于传统Android系统中的View的层叠要轻量的多.在这个drawable树中只有顶层的TopDrawable才对外暴露,因此这个接口只有一个`getTopLevelDrawable`的方法.**通过这种多层的设计,可以在一个展示View中设置占位图,进度条,错误图,重试图,覆盖图(水印)等等,还是比较强大的**


	/** 一个hierarchy的例子
	*   o FadeDrawable (top level drawable)
	*   |
	*   +--o ScaleTypeDrawable
	*   |  |
	*   |  +--o BitmapDrawable
	*   |
	*   +--o ScaleTypeDrawable
	*      |
	*      +--o BitmapDrawable
	*/

**SettableDraweeHierarchy**继承自DraweeHierarchy,并添加了在这个drawable树中设置不同状态的方法,例如

	//这些方法只能被Controller调用
	void reset();

	void setImage(Drawable drawable, float progress, boolean immediate);

	void setProgress(float progress, boolean immediate);

	void setFailure(Throwable throwable);

	void setRetry(Throwable throwable);

	void setControllerOverlay(Drawable drawable);


**GenericDraweeHierarchy**是SettableDraweeHierarchy接口的通用实现类, 是DraweeView中真正持有的实体


### DraweeController的继承体系

![DraweeController](/images/2015-12-02-study-of-fresco/DraweeController.png)

**DraweeController**接口定义了两件事

- 获取和设置Hieraychy
- view的各种事件通知过来，controller来控制这些逻辑的操作（onAttach/onDetach/onTouchEvent/getAnimatable）

**AbstractDraweeController**抽象的接口实现类

- 最关键的功能：实现了客户端向服务端的提交请求，即向DataSource中注册观察者
- 在有结果返回的时候，在主线程通知客户端更新即可，即设置Hierarychy的drawable即可

具体来说, (以PipelineDraweeController为例)在通过builder.build()创建Controller的过程中,会调用`obtainDataSourceSupplier`来获取所需的DataSourceSupplier,进而在请求图片(submitRequest)的过程中获取当前的DataSource. 而通过在DataSource中订阅DataSourceSubscriber,使得请求到的数据在改变时能够通过controller将获取到的图片或者中间结果传递到DraweeHierarchy中,最终显示出来(...写得有点绕...x_x),对这个过程有个大概的了解有助于于之后[DataSource](#jump2datasource)模块进行连接

具体代码:

	//图片被attach到界面是,开始请求
	public void onAttach() {
	  if (FLog.isLoggable(FLog.VERBOSE)) {
	  //打印日志
	  ...
	  //记录event
	  mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
	  Preconditions.checkNotNull(mSettableDraweeHierarchy);
	  mDeferredReleaser.cancelDeferredRelease(this);
	  mIsAttached = true;
	  if (!mIsRequestSubmitted) {
	  	//关键:发送request!!往下
	  	submitRequest();
	  }
	}

	//具体请求如何被发送的
	protected void submitRequest() {
	  mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
	  getControllerListener().onSubmit(mId, mCallerContext);
	  mSettableDraweeHierarchy.setProgress(0, true);
	  mIsRequestSubmitted = true;
	  mHasFetchFailed = false;
	  //从DataSourceSupplier中获取当前的DataSource
	  mDataSource = getDataSource();
	  //打印日志
	  ...
	  final String id = mId;
	  final boolean wasImmediate = mDataSource.hasResult();
	  //订阅的回调
	  final DataSubscriber<T> dataSubscriber =
	      new BaseDataSubscriber<T>() {
	        @Override
	        public void onNewResultImpl(DataSource<T> dataSource) {
	          // isFinished must be obtained before image, otherwise we might set intermediate result
	          // as final image.
	          boolean isFinished = dataSource.isFinished();
	          float progress = dataSource.getProgress();
	          T image = dataSource.getResult();
	          if (image != null) {
	            //在成功的回调方法中,并且image不为空,进行成功的处理
	            //这里主要判断是否是正确的datasource
	            //mSettableDraweeHierarchy.setImage()将获取的image传递到DraweeHierarchy中
	            //将之前显示的图片全部回收(这里使用ashmem的概念处理java的内存,在后面的内存管理部分细讲)
	            onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
	          } else if (isFinished) {
	            //image为空,做失败处理
	            onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
	          }
	        }
	        @Override
	        public void onFailureImpl(DataSource<T> dataSource) {
	          //失败的处理
	          onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
	        }
	        @Override
	        public void onProgressUpdate(DataSource<T> dataSource) {
	          boolean isFinished = dataSource.isFinished();
	          float progress = dataSource.getProgress();
	          //进度的更新处理
	          onProgressUpdateInternal(id, dataSource, progress, isFinished);
	        }
	      };
	  //向DataSource订阅!!!这里在DataSource的部分再细讲
	  mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
	}

接下来就是**AbstractDraweeController**的实现类,分别是

- **PipelineDraweeController**: Fresco默认的实现,也就是SimpleDraweeView中使用的, 用来桥接`image pipeline`和 `SettableDraweeHierarchy`
- **VolleyDraweeController**: 用来桥接`volley`和`SettableDraweeHierarchy`,按照注释的意思,也就是说如果我们要使用Volley来做网络请求的话,需要用到这种controller(还没仔细研究这里,只是猜测~)


## DataSource模块
<span id="jump2datasource">DataSource</span>接口类似于Java中的Future(为了返回异步任务的结果),其区别在于Future只能返回最终的结果,**而DataSource的设计使其能返回一系列结果(例如渐进式的显示图片,或者加载图片过程中不同状态的显示)**

这个类图可以回头再看..

![DataSource](/images/2015-12-02-study-of-fresco/DataSource.jpeg)
![DataSource](/images/2015-12-02-study-of-fresco/DataSource.png)

DataSubscriber和DataSource 一起构成观察者模式.DataSource提供了注册DataSubscriber的方法,当DataSource数据发生变化时,在`Executor`中通知所有的观察者.DataSubscriber会响应数据的几种变化,如下,这些变化会通过controller反应在DraweeHierarchy上

	//当有新的结果准备从datasource被接收的时候调用
	//dataSource.getResult()获取结果
	//dataSource.isFinished()判断是否是最终结果
	void onNewResult(DataSource<T> dataSource);

	//pipeline中有错误时呗调用
	void onFailure(DataSource<T> dataSource);

	//request被取消时调用
	void onCancellation(DataSource<T> dataSource);

	//进度更新时被调用
	void onProgressUpdate(DataSource<T> dataSource);

之前在**DraweeController**部分中讲了,在`AbstractDraweeControllerBuider`的
`protected abstract AbstractDraweeController obtainController()`的实现中有获取DataSourceSuppiler的方法

    //以PipelineDraweeControllerBuilder为例,看获取Controller的实现.
	@Override
	protected PipelineDraweeController obtainController() {
	  //获取oldController在有可能的情况下进行复用
	  DraweeController oldController = getOldController();
	  PipelineDraweeController controller;
	  if (oldController instanceof PipelineDraweeController) {
	    controller = (PipelineDraweeController) oldController;
	    controller.initialize(
	        //获取DataSourceSupplier
	        obtainDataSourceSupplier(),
	        generateUniqueControllerId(),
	        getCallerContext());
	  } else {
	    controller = mPipelineDraweeControllerFactory.newController(
	        obtainDataSourceSupplier(),
	        generateUniqueControllerId(),
	        getCallerContext());
	  }
	  return controller;
	}

    //AbstractDraweeControllerBuilder中
	//获取controller可用的DataSourceSuppiler
	protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier() {
	  if (mDataSourceSupplier != null) {
	    return mDataSourceSupplier;
	  }

	  Supplier<DataSource<IMAGE>> supplier = null;

	  // final image supplier;
	  //图片最终显示的的datasource的suppiler,分为两种
	  //1.只有一个请求地址的
	  //2.FirstAvailableDataSource,是有多个请求地址(数组形式),依次请求,只有当当前的请求失败或者返回空时,才请求下一个地址;依次进行,知道获取第一个可用的结果
	  //可见传入参数是ImageRequest,也就是说controller,dataSource和图片的请求是这样联系起来的
	  if (mImageRequest != null) {
	    supplier = getDataSourceSupplierForRequest(mImageRequest);
	  } else if (mMultiImageRequests != null) {
	    supplier = getFirstAvailableDataSourceSupplier(mMultiImageRequests, mTryCacheOnlyFirst);
	  }

	  // increasing-quality supplier; highest-quality supplier goes first
	  //Fresco支持双分辨率的请求,传入高低不同分辨率的两个请求地址,会依次获取数据,提高低网速下的体验.这里就是添加低分辨率的request.
	  if (supplier != null && mLowResImageRequest != null) {
	    List<Supplier<DataSource<IMAGE>>> suppliers = new ArrayList<>(2);
	    suppliers.add(supplier);
	    suppliers.add(getDataSourceSupplierForRequest(mLowResImageRequest));
	    supplier = IncreasingQualityDataSourceSupplier.create(suppliers);
	  }

	  // no image requests; use null data source supplier
	  if (supplier == null) {
	    supplier = DataSources.getFailedDataSourceSupplier(NO_REQUEST_EXCEPTION);
	  }

	  return supplier;
	}

具体这个请求是怎么发送出去的呢?之前有讲到在DraweeController的`onAttach`中获取了dataSource并且订阅了观察者,用于处理datasource返回的结果.来看一下实现中这个dataSource到底是怎么获取的:

	//controller中从DataSourceSupplier中get一个datasource, 这个suppiler就是上一段代码中,为controller获取的适当的dataSourceSuppiler
	//从 supplier = getDataSourceSupplierForRequest(mImageRequest) 进入

	@Override
	protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
	  //打印log
	  ...
	  return mDataSourceSupplier.get();
	}



	/** Creates a data source supplier for the given image request. */
	protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(REQUEST imageRequest) {
	  return getDataSourceSupplierForRequest(imageRequest, /* bitmapCacheOnly */ false);
	}

	/** Creates a data source supplier for the given image request. */
	protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
	    final REQUEST imageRequest,
	    final boolean bitmapCacheOnly) {
	  final Object callerContext = getCallerContext();
	  return new Supplier<DataSource<IMAGE>>() {

	    //所以controller中从DataSourceSuppiler中获取DataSource就是下面这个,再往下看具体的实现
	    @Override
	    public DataSource<IMAGE> get() {
	      return getDataSourceForRequest(imageRequest, callerContext, bitmapCacheOnly);
	    }
	    @Override
	    public String toString() {
	      ...
	    }
	  };
	}

	//PipelineDraweeControllerBuilder中具体的实现
	@Override
	protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
	    ImageRequest imageRequest,
	    Object callerContext,
	    boolean bitmapCacheOnly) {
	  if (bitmapCacheOnly) {
	    return mImagePipeline.fetchImageFromBitmapCache(imageRequest, callerContext);
	  } else {
	    //一般请求传入的bitmapCacheOnly是false,待会会从这儿进去看pipeline的内部
	    return mImagePipeline.fetchDecodedImage(imageRequest, callerContext);
	  }
	}

可见是通过Pipeline来获取的这个datasource.其内部是发起了一个`submitFetchRequest`返回一个DataSource.Pipeline也是Fresco一个重要的组成部分,在Pipline模块中详细说明.

## Pipeline模块
简单来说pipeline就是实现了三级缓存,解码,变形等等,完成了提供可呈现图片的所有工作.

![ImagePipeline](/images/2015-12-02-study-of-fresco/ImagePipeline.png)

Facebook官方中已经说明,ImagePipeline负责完成加载图像,并且将结果反馈(以回调或者说观察者的方式)出来.

这里先解释几个概念

- Producer: 为了实现业务的隔离而设计的接口; 将pipeline中需要进行的每一项任务作为一个producer,通常将前一个producer作为参数传递个下一个producer,从而实现面向接口的业务模块剪的隔离.
- Consumer: producer生产的结果会最终传递到consumer中,再通过实现了DataSource接口的适配器通知外部

具体代码:

	/**
	 * Submits a request for execution and returns a DataSource representing the pending decoded image(s).
	 * <p>The returned DataSource must be closed once the client has finished with it.
	 * @param imageRequest the request to submit
	 * @return a DataSource representing the pending decoded image(s)
	 */
	public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
	    ImageRequest imageRequest,
	    Object callerContext) {
	  try {
	    //有两块,下面的代码会分别进行分析
	    //1.首先获取 producerSequence: 解码图片的请求队列
	    Producer<CloseableReference<CloseableImage>> producerSequence =
	        mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);    
	    //2.
	    return submitFetchRequest(
	        producerSequence,
	        imageRequest,
	        ImageRequest.RequestLevel.FULL_FETCH,
	        callerContext);
	  } catch (Exception exception) {
	    return DataSources.immediateFailedDataSource(exception);
	  }
	}

	//******************************
	//1.第一步 getDecodedImageProducerSequence
	//******************************
	//返回一个用于请求 解码图片的队列,可见这个队列是和imageRequest相关的,源码中,imageRequest是一个不可改变的JavaBean,其中包含了所有Pipeline请求图片所需要的所有信息.
	public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
	    ImageRequest imageRequest) {
	  //继续看getBasicDecodedImageSequence的实现
	  Producer<CloseableReference<CloseableImage>> pipelineSequence =
	      getBasicDecodedImageSequence(imageRequest);
	  if (imageRequest.getPostprocessor() != null) {
	    return getPostprocessorSequence(pipelineSequence);
	  } else {
	    return pipelineSequence;
	  }
	}

	//从这里就可以清楚地看到,针对不同的uri类型生成了不同的FetchSequence也就是Producer
	private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(
	    ImageRequest imageRequest) {
	  Preconditions.checkNotNull(imageRequest);

	  Uri uri = imageRequest.getSourceUri();
	  //判空
	  Preconditions.checkNotNull(uri, "Uri is null.");
	  //是否是网络请求
	  if (UriUtil.isNetworkUri(uri)) {
	    return getNetworkFetchSequence();
	  } else if (UriUtil.isLocalFileUri(uri)) {
	    //是否是本地video
	    if (MediaUtils.isVideo(MediaUtils.extractMime(uri.getPath()))) {
	      return getLocalVideoFileFetchSequence();
	    } else {//本地图片
	      return getLocalImageFileFetchSequence();
	    }
	  } else if (UriUtil.isLocalContentUri(uri)) {
	    return getLocalContentUriFetchSequence();
	  } else if (UriUtil.isLocalAssetUri(uri)) {
	    return getLocalAssetFetchSequence();
	  } else if (UriUtil.isLocalResourceUri(uri)) {
	    return getLocalResourceFetchSequence();
	  } else if (UriUtil.isDataUri(uri)) {
	    return getDataFetchSequence();
	  } else {
	    //throw 异常
	    ...
	  }
	}

拿最复杂的`getNetworkFetchSequence()`举例,追踪进去看代码就是对不同模块producer一层一层的包装(内存中获取->切换Thread->...编码缓存->本地缓存->webP转换->从网络请求),运用典型的装饰设计模式.

    //******************************
	//1.第二步 submitFetchRequest
	//******************************
	private <T> DataSource<CloseableReference<T>> submitFetchRequest(
	    Producer<CloseableReference<T>> producerSequence,
	    ImageRequest imageRequest,
	    ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
	    Object callerContext) {
	  try {
	    //计算出图片请求的最低请求级别
	    ImageRequest.RequestLevel lowestPermittedRequestLevel =
	        ImageRequest.RequestLevel.getMax(
	            imageRequest.getLowestPermittedRequestLevel(),
	            lowestPermittedRequestLevelOnSubmit);
	    //创建出settableProducerContext,请求的上下文,包括了请求的信息,优先级,请求id等
	    SettableProducerContext settableProducerContext = new SettableProducerContext(
	        imageRequest,
	        generateUniqueFutureId(),
	        mRequestListener,
	        callerContext,
	        lowestPermittedRequestLevel,
	      /* isPrefetch */ false,
	        imageRequest.getProgressiveRenderingEnabled() ||
	            !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
	        imageRequest.getPriority());
	    //根据创建的settableProducerContext,再将利用Producer和DataSource中间的适配器,创建了一个DataSource(重要!!!)
	    return CloseableProducerToDataSourceAdapter.create(
	        producerSequence,
	        settableProducerContext,
	        mRequestListener);
	  } catch (Exception exception) {
	    return DataSources.immediateFailedDataSource(exception);
	  }
	}

主要看下`CloseableProducerToDataSourceAdapter.create(...)`,创建了一个可关闭的producer到数据源的适配器,接下来要详细说明下producer和datasource之间处理逻辑.

### Producer与DataSource的关联

    //接着上一段代码
	public static <T> DataSource<CloseableReference<T>> create(
	    Producer<CloseableReference<T>> producer,
	    SettableProducerContext settableProducerContext,
	    RequestListener listener) {
	  return new CloseableProducerToDataSourceAdapter<T>(
	      producer, settableProducerContext, listener);
	}

	private CloseableProducerToDataSourceAdapter(
	    Producer<CloseableReference<T>> producer,
	    SettableProducerContext settableProducerContext,
	    RequestListener listener) {
	  //构造里,只是调用了父类的方法
	  super(producer, settableProducerContext, listener);
	}

	//父类 AbstractProducerToDataSourceAdapter的构造
	//关键的代码:
	protected AbstractProducerToDataSourceAdapter(
	    Producer<T> producer,
	    SettableProducerContext settableProducerContext,
	    RequestListener requestListener) {
	  mSettableProducerContext = settableProducerContext;
	  mRequestListener = requestListener;
	  mRequestListener.onRequestStart(
	      settableProducerContext.getImageRequest(),
	      mSettableProducerContext.getCallerContext(),
	      mSettableProducerContext.getId(),
	      mSettableProducerContext.isPrefetch());
	  //查看produceResults的方法,发现其实就是一个接口,只是通知producer开始生产结果
	  //!!!而我们最后会讲到createConsumer(),最终的结果会传递到这个consumer中,记一下,之后会讲到
	  producer.produceResults(createConsumer(), settableProducerContext);
	}

为了研究`producer.produceResults()`生产的具体结果,我们举例来看一下
**BitmapMemoryCacheProducer**,这是图片内存存取的producer

	//BitmapMemoryCacheProducer
	@Override
	public void produceResults(
	    final Consumer<CloseableReference<CloseableImage>> consumer,
	    final ProducerContext producerContext) {

	  final ProducerListener listener = producerContext.getListener();
	  final String requestId = producerContext.getId();
	  listener.onProducerStart(requestId, getProducerName());
	  final ImageRequest imageRequest = producerContext.getImageRequest();
	  final CacheKey cacheKey = mCacheKeyFactory.getBitmapCacheKey(imageRequest);

	  //根据cacheKey在内存中查找
	  CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);

	  if (cachedReference != null) {
	    //如果存在在内存中,直接通知consumer
	    boolean isFinal = cachedReference.get().getQualityInfo().isOfFullQuality();
	    if (isFinal) {
	      listener.onProducerFinishWithSuccess(
	          requestId,
	          getProducerName(),
	          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
	      consumer.onProgressUpdate(1f);
	    }
	    consumer.onNewResult(cachedReference, isFinal);
	    cachedReference.close();
	    if (isFinal) {
	      return;
	    }
	  }

	  //如果内存中没有,并且请求的最低等级=BITMAP_MEMORY_CACHE,则返回consumer一个空结果
	  if (producerContext.getLowestPermittedRequestLevel().getValue() >=
	      ImageRequest.RequestLevel.BITMAP_MEMORY_CACHE.getValue()) {
	    listener.onProducerFinishWithSuccess(
	        requestId,
	        getProducerName(),
	        listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
	    consumer.onNewResult(null, true);
	    return;
	  }

	  //需要inputProducer来提供,也就是调用前一个producer的produceResults的方法,同样的,如果有结果的话通过consumer(这里是wrappedConsumer来返回),需要的还可以继续要求再之前的producer来提供结果
	  Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(consumer, cacheKey);
	  listener.onProducerFinishWithSuccess(
	      requestId,
	      getProducerName(),
	      listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
	  mInputProducer.produceResults(wrappedConsumer, producerContext);
	}

	//Q:wrappedConsumer是个啥东西?

	//如下: 从名字就可以看出是个代理或者委托的模式,也就是说现在的produser要求它前面的producer提供结果,前面的producer将结果传给wrappedConsumer,这是就可以先做一部分出里,再传递给当前的consumer
	//以下面这个内存存取的producer的consumer委托为例,就是在返回最终结果的时候,在内存中进行了缓存.
	//如果不需要对结果做处理的话,就直接传递原始的consumer到另一个producer就行
	protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(
	    final Consumer<CloseableReference<CloseableImage>> consumer,
	    final CacheKey cacheKey) {
	  return new DelegatingConsumer<
	      CloseableReference<CloseableImage>,
	      CloseableReference<CloseableImage>>(consumer) {
	    @Override
	    public void onNewResultImpl(CloseableReference<CloseableImage> newResult, boolean isLast) {
	      // ignore invalid intermediate results and forward the null result if last
	      if (newResult == null) {
	        if (isLast) {//判断结果是否是空,并且是否是最终结果
	          getConsumer().onNewResult(null, true);
	        }
	        return;
	      }
	      // stateful results cannot be cached and are just forwarded
	      //阶段性结果不需要被缓存,直接发给consumer就行
	      if (newResult.get().isStateful()) {
	        getConsumer().onNewResult(newResult, isLast);
	        return;
	      }
	      // if the intermediate result is not of a better quality than the cached result,
	      // forward the already cached result and don't cache the new result.
	      if (!isLast) {
	        CloseableReference<CloseableImage> currentCachedResult = mMemoryCache.get(cacheKey);
	        if (currentCachedResult != null) {
	          try {
	            QualityInfo newInfo = newResult.get().getQualityInfo();
	            QualityInfo cachedInfo = currentCachedResult.get().getQualityInfo();
	            if (cachedInfo.isOfFullQuality() || cachedInfo.getQuality() >= newInfo.getQuality()) {
	              getConsumer().onNewResult(currentCachedResult, false);
	              return;
	            }
	          } finally {
	            CloseableReference.closeSafely(currentCachedResult);
	          }
	        }
	      }
	      // cache and forward the new result
	      //缓存并传递结果!!!!!!!
	      CloseableReference<CloseableImage> newCachedResult =
	          mMemoryCache.cache(cacheKey, newResult);
	      try {
	        if (isLast) {
	          getConsumer().onProgressUpdate(1f);
	        }
	        getConsumer().onNewResult(
	            (newCachedResult != null) ? newCachedResult : newResult, isLast);
	      } finally {
	        CloseableReference.closeSafely(newCachedResult);
	      }
	    }
	  };
	}

从上面的流程可知,freso运用的有点像职责链的模式,将每一个职责分离并串联起来.并且在要对结果处理时,进行拦截,添加自己的处理(采用代理或包装).

不论这个链有多长,最终还是要传递给consumer,通过适配器来向外界通知最终的结果.这一部分在`createConsumer()`中.

	private Consumer<T> createConsumer() {
	  return new BaseConsumer<T>() {
	    @Override
	    protected void onNewResultImpl(@Nullable T newResult, boolean isLast) {
	      AbstractProducerToDataSourceAdapter.this.onNewResultImpl(newResult, isLast);
	    }

	    @Override
	    protected void onFailureImpl(Throwable throwable) {
	      AbstractProducerToDataSourceAdapter.this.onFailureImpl(throwable);
	    }

	    @Override
	    protected void onCancellationImpl() {
	      AbstractProducerToDataSourceAdapter.this.onCancellationImpl();
	    }

	    @Override
	    protected void onProgressUpdateImpl(float progress) {
	      AbstractProducerToDataSourceAdapter.this.setProgress(progress);
	    }
	  };
	}

可见consumer中最终调用的是适配器的几个接口实现方法,而在适配器中直接调用了父类`AbstractDataSource`的方法,也就是通知了所有的订阅者,**这样就将所有的模块全部串联起来了**. **Well Done!!**


## Fresco初始化过程

### 初始化ImagePipelineFactory
需要在使用Drawee之前进行初始化,一般就在`Application.onCreate()`中进行

	/** Initializes Fresco with the default config. */
	public static void initialize(Context context) {
	  ImagePipelineFactory.initialize(context);
	  initializeDrawee(context);
	}

	/** Initializes Fresco with the specified config. */
	public static void initialize(Context context, ImagePipelineConfig imagePipelineConfig) {
	  //1.初始化ImagePipelineFactory
	  ImagePipelineFactory.initialize(imagePipelineConfig);
	  //2.初始化Drawee
	  initializeDrawee(context);
	}

区别在于是否用默认的`ImagePipelineConfig`,而`ImagePipelineConfig`是通过一个builder来构造,从而确定所有的属性

	private ImagePipelineConfig(Builder builder) {
	  mAnimatedImageFactory = builder.mAnimatedImageFactory;

	  //以mBitmapMemoryCacheParamsSupplier的设置为例:
	  mBitmapMemoryCacheParamsSupplier =
	      builder.mBitmapMemoryCacheParamsSupplier == null ?
	          new DefaultBitmapMemoryCacheParamsSupplier(
	              (ActivityManager) builder.mContext.getSystemService(Context.ACTIVITY_SERVICE)) :
	          builder.mBitmapMemoryCacheParamsSupplier;
	  ...
	  ...
	  //所有配置都类似,如果builder中的属性不为空,则赋值到ImagePipelineConfig中;如果为空,则再创建默认的config
	}
通过这样的设计,使用者可以只指定builder中的某几个配置.

而后通过这个`ImagePipelineConfig`生成静态的`ImagePipelineFactory`对象,这个对象会再随后的`initializeDrawee(context)`初始化Drawee时用到

	public static void initialize(ImagePipelineConfig imagePipelineConfig) {
	  sInstance = new ImagePipelineFactory(imagePipelineConfig);
	}

### 初始化Drawee

    //com.facebook.drawee.backends.pipeline.Fresco
	private static void initializeDrawee(Context context) {
	  sDraweeControllerBuilderSupplier = new PipelineDraweeControllerBuilderSupplier(context);
	  SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
	}

	//com.facebook.drawee.view.SimpleDraweeView
	//fresco初始化的时候将创建一个static的对象sDraweeControllerBuilderSupplier,用来在随后生成Drawee实例的时候获取DraweeControllerBuilder
	private static Supplier<? extends SimpleDraweeControllerBuilder> sDraweeControllerBuilderSupplier;

	/** Initializes {@link SimpleDraweeView} with supplier of Drawee controller builders. */
	public static void initialize(
	    Supplier<? extends SimpleDraweeControllerBuilder> draweeControllerBuilderSupplier) {
	  sDraweeControllerBuilderSupplier = draweeControllerBuilderSupplier;
	}

	//com.facebook.drawee.view.SimpleDraweeView
	//在SimpleDraweeView实例初始化的时候从sDraweeControllerBuilderSupplier获取SimpleDraweeControllerBuilder,这个builder会在给Drawee创建DraweeController是使用,详情可见下面的DraweeView实例化过程
	private void init() {
	  if (isInEditMode()) {
	    return;
	  }
	  Preconditions.checkNotNull(
	      sDraweeControllerBuilderSupplier,
	      "SimpleDraweeView was not initialized!");
	  mSimpleDraweeControllerBuilder = sDraweeControllerBuilderSupplier.get();
	}

## Drawee的实例化过程
以SimpleDraweeView为例,有两种

	//在初始化完成后, 直接设置uri,这是会自动从sDraweeControllerBuilderSupplier中获取builder来构建一个controller并赋值到DraweeView中
	public void setImageURI(Uri uri, @Nullable Object callerContext) {
	  DraweeController controller = mSimpleDraweeControllerBuilder
	          .setCallerContext(callerContext)
	          .setUri(uri)
	          .setOldController(getController())
	          .build();
	  setController(controller);
	}

	//第二种是通过builder创建一个区别于Fresco初始化配置的controller,便于更细节画的需求
	Uri uri = Uri.parse("http://pooyak.com/p/progjpeg/jpegload.cgi?o=1");
	ImageRequest request = ImageRequestBuilder.newBuilderWithSource(uri)
	    .setProgressiveRenderingEnabled(true)
	    .build();
	DraweeController controller = Fresco.newDraweeControllerBuilder()
	    .setImageRequest(request)
	    .build();
	mProgressiveJpegView.setController(controller);

## 内存管理
Fresco最强大的地方在于它对图片资源的内存优化.在Android中由java代码创建的对象存活在虚拟机中分配的内存里,这块内存受到严格的限制,在APP使用图片较多的情况下,会有泄露的风险,并且长期占用内存,导致可用内存不足,频繁GC,大大影响性能和流畅性.

[facebook的post, 简单描述了fresco的内存是如何管理的(需要翻墙)](https://code.facebook.com/posts/366199913563917/introducing-fresco-a-new-image-library-for-android/)

Fresco则是利用了**ashmem**, ashmem的操作类似于Native堆的申请释放操作, 但是有系统提供的调用方法.Android可以"解锁"(unpin)内存而不是释放(free)它,这部分unping的内存处于一个懒释放(lazy free)类似软引用的状态,也就是说只有当系统真的需要更多的内存时,才回去真正释放这部分内存.而当android去锁定(pin)这部分内存时,如果没有被释放,那么数据就任然存在.

更多ashmem的知识可参见

[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6651971)

[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）驱动程序源代码分析](http://blog.csdn.net/luoshengyang/article/details/6664554)

[Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析](http://blog.csdn.net/luoshengyang/article/details/6666491)

### Bitmap在Ashmem中的使用
Ashmem内存区域是不能被Java应用直接使用的,但这其中有一些例外,而Bitmap是其中一个.

	BitmapFactory.Osptions = new BitmapFactory.Options();
	options.inPurgeable = true;
	Bitmap bitmap = BitmapFactory.decodeByteArray(jpeg, 0, jpeg.length, options);

Purgeable被设置成true以后,这个Bigmap就是保存在Ashmem内存区域中的,Java的垃圾回收是不能回收这篇区域的内存的.当Android系统需要渲染这个Bitmap的时候,会调用pin,渲染完成后会调用unpin.而unpin后的内存空间表示能被其他人所使用.

如果被unpin的Bitmap需要重新渲染,系统会再次Decode这个Bitmap.**而这个Decode的过程是在UI线程上完成的,这个操作也大大增加了CPU的负担**.所以Google后来废弃了这个pureable的参数.

后来Google提供了另外一个Flag,叫inBitmap.很遗憾的是,知道Android4.4后,这个新的Flag才得到完善.而Fresco致力于实现一个包括Android2.3以及以上的Android系统都能完美工作的图片加载管理开源库,因此Fresco放弃了使用inBitmap的解决方案.

### Fresco是如何利用Ashmem去给Bitmap分配和管理内存?
上面说到的pin和unpin两个操作,对应的NDK调用是AndroidBitmap_lockPixels和unlockPixels.按照我们一惯认知,为了避免内存泄漏,这两者必须成对出现.而Fresco为了避免Bitmap再次渲染而导致的在UI线程Decode的过程,偏偏不在渲染完成后调用unlockPixels.

这样做后,Fresco需要自己去管理这块内存区域,保证当这个Bitmap不再使用时,Ashmem的内存空间能被unpin.而Fresco选择在Bitmap离开屏幕可视范围时候(onDetachWindow等时候),去做unpin.

这些都在SImpleDraweeView中封装好了,也就是说一般情况下我们使用SimpleDraweeView就行了,不太需要担心内部的处理.


### 不同版本不同对策
由Fresco文档和`com.facebook.imagepipeline.platform.PlatformDecoder`源码可知,Fresco对不同的系统版本使用了不用的内存处理策略

- 在5.0 SDK21 Lolopop以下的系统,使用ashmem
- 在5.0 及以上系统使用了ART虚拟机,相比之下,内存管理有了很大改进,所以Bitmap缓存直接位于Java的heap上

## 总结
如果不算ashmem的purgable,fresco还用到了三个缓存,首先是DiskCache,然后还有两个MemoryCache,分别是保存DecodedBitmap的和保存EncodedImage的缓存,除此之外,为了避免频繁的申请内存,回收内存造成内存抖动,fresco还用到了大量的对象池(基于`com.facebook.imagepipeline.memory.BasePool`),除此以外,还有`com.facebook.common.references.OOMSoftReference`使用软引用的地方.

这些细节的东西暂时先留个坑吧,以后再看@@
