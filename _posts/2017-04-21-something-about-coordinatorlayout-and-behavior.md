---
layout: post
title: "CoordinatorLayout 与 Behavior 若干笔记"
description: "Something about CoordinatorLayout and Behavior"
category: Develop
tags: [android]
---

本篇将会涉及到如下内容:

- **CoordinatorLayout**
- **Behavior**
- AppBarLayout
- CollapsingToolbarLayout
- NestedScrollView, RecyclerView, SwipRefreshLayout

由于国内的特殊应用生态, 可能不能照搬 Material Design. 但完全不用担心没法使用原生的控件, 其实 CoordinatorLayout 配合自定义的 Behavior 就完全可以比较方便的实现很多交互的效果了.


## 使用 Behavior 的三种方式

一般来说前两种会比较常用.

**1. 设置 LayoutParameter**
代码设置 View 的 CoordinatorLayout.LayoutParameter
android.support.design.widget.CoordinatorLayout.LayoutParams#setBehavior

**2. 设置 xml 布局中属性**
app:layout_behavior="@string/appbar_scrolling_view_behavior"
或 app:layout_behavior = "自定义 Behavior 完整类名"

**3. 注解方式, 对自定义 View 使用 Behavior**
@CoordinatorLayout.DefaultBehavior(SelfDefinedBehavior.class) 

	//以系统控件为例: android.support.design.widget.FloatingActionButton
	@CoordinatorLayout.DefaultBehavior(FloatingActionButton.Behavior.class)
	public class FloatingActionButton extends VisibilityAwareImageButton {
		...
	}

源码调用链: 

	CoordinatorLayout#onMeasure ->
	CoordinatorLayout#prepareChildren ->
	CoordinatorLayout#getResolvedLayoutParams ->
	childView.getClass().getAnnotation(DefaultBehavior.class)

## 系统控件使用的例子

**AppBarLayout & Scroll**
![查看norma gif](/images/2017-04-21-something-about-coordinatorlayout-and-behavior/normal.gif)
	
	<?xml version="1.0" encoding="utf-8"?>
	<android.support.design.widget.CoordinatorLayout 
	    xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    android:id="@+id/main_content"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent">
	
	    <android.support.design.widget.AppBarLayout
	        android:id="@+id/appbar"
	        android:layout_width="match_parent"
	        android:layout_height="52dp">
	
	        <RelativeLayout
	            android:id="@+id/toolbar"
	            android:layout_width="match_parent"
	            android:layout_height="52dp"
	            app:layout_scrollFlags="scroll">
	
	            <TextView
	                android:layout_width="wrap_content"
	                android:layout_height="wrap_content"
	                android:layout_centerInParent="true"
	                android:text="Title"
	                android:textColor="@android:color/white"
	                android:textSize="20dp" />
	
	        </RelativeLayout>
	
	    </android.support.design.widget.AppBarLayout>
	
	    <android.support.v7.widget.RecyclerView
	        android:id="@+id/rv"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
	
	</android.support.design.widget.CoordinatorLayout>

<br>

RecyclerView 的属性 `app:layout_behavior="@string/appbar_scrolling_view_behavior"` 在这的作用是告诉 CoordinatorLayout 我是被 AppBarLayout 依赖的, 这个 Behavior 决定了 layout 的位置. (CoordinatorLayout 在 onLayout()方法中会根据 Behavior 调整子 View 的位置)

<br>

AppBarLayout 就是通过注解使用了 AppBarLayout.Behavior. 这个 Behavior 有依赖 `NestedScrollingChild` 也就是 RecyclerView 的滚动行为 (下文自定义部分会提到).

<br>

AppBarLayout 的子 View 需要声明一个属性`app:layout_scrollFlags` 这个属性决定了滚动时 AppBarLayout 的规则.

- **scroll**: 所有想滚动出屏幕的view都需要设置这个flag- 没有设置这个flag的view将被固定在屏幕顶部.
- **enterAlways**: 这个flag让任意向下的滚动都会导致该view变为可见, 启用快速"返回模式"
- **snap**: 这个flag 使 View在滚动结束的时候一定会靠向摸一个边界
- **enterAlwaysCollapsed**: 当你的视图已经设置minHeight属性又使用此标志时, 你的视图只能以最小高度进入, 只有当滚动视图到达顶部时才扩大到完整高度
- **exitUntilCollapsed**: 当视图会在滚动时, 它一直滚动到设置的minHeight时完全隐藏


<br>

**CollapsingToolbarLayout**
![查看collapsing gif](/images/2017-04-21-something-about-coordinatorlayout-and-behavior/collapsing.gif)

	<?xml version="1.0" encoding="utf-8"?>
	<android.support.design.widget.CoordinatorLayout 
	    xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    android:id="@+id/main_content"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent">
	
	    <android.support.design.widget.AppBarLayout
	        android:id="@+id/main.appbar"
	        android:layout_width="match_parent"
	        android:layout_height="300dp">
	
	        <android.support.design.widget.CollapsingToolbarLayout
	            android:id="@+id/collapsingToolBar"
	            android:layout_width="match_parent"
	            android:layout_height="match_parent"
	            android:minHeight="52dp"
	            app:contentScrim="#ff4444"
	            app:expandedTitleMarginEnd="64dp"
	            app:expandedTitleMarginStart="48dp"
	            app:layout_scrollFlags="scroll|snap|exitUntilCollapsed">	
	            <ImageView
	                android:id="@+id/iv_image"
	                android:layout_width="match_parent"
	                android:layout_height="match_parent"
	                app:layout_collapseMode="parallax" />
	
	            <android.support.v7.widget.Toolbar
	                android:id="@+id/toolbar"
	                android:layout_width="match_parent"
	                android:layout_height="?attr/actionBarSize"
	                app:layout_collapseMode="pin"
	                app:popupTheme="@style/ThemeOverlay.AppCompat.Light">
	            </android.support.v7.widget.Toolbar>
	        </android.support.design.widget.CollapsingToolbarLayout>
	    </android.support.design.widget.AppBarLayout>
	
	    <android.support.v4.widget.NestedScrollView
	        android:id="@+id/rv"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        app:layout_behavior="@string/appbar_scrolling_view_behavior">
	        
	    </android.support.v4.widget.NestedScrollView>
	</android.support.design.widget.CoordinatorLayout>

## 自定义 Behavior 的注意点及方法

Behavior 是用来控制, 在 CoordinatorLayout 下的子 View(child) 依赖另一个同级 View(dependency) 时的行为; 当 dependency 的 行为改变时, child 可以作出相应的改变. 被依赖 View(dependency) 可以分为两类:

1. **任意 View, 一般参照其位置及本身状态信息, 如所在位置, 大小, alpha 等等** (例如 FloatingActionButton.Behavior)
2. **实现了 NestedScrollingChild 接口的类, 主要参照滚动相关的信息** (例如 AppBarLayout.Behavior)

<br>

具体来说, 满足以下几点, Behavior 才会生效:

- 拥有 Behavior 的 View 必须是 CoordinatorLayout 的直接子 View
- 被依赖的 View 必须是相同层级, 也就是说, 一定也是 CoordinatorLayout 下的直接子 View
- 如果依赖滚动行为, 必须是实现了 NestedScrollingChild 的子类, **NestedScrollView, RecyclerView, SwipRefreshLayout**. (所以 ScrollView 和 ListView 都是无效的) 

**下面正式开始自定义 Behavior**
**1. 依赖一个 View 的位置及本身状态信息**

![查看dependvie gif](/images/2017-04-21-something-about-coordinatorlayout-and-behavior/dependview.gif)

在这个例子中 AppBarLayout 和之前的例子一样, 是随列表滚动而隐藏的. 自定义了一个 Behavior, 使得底部的 tab 随着顶部的 AppBarLayout 隐藏.

其中主要代码如下, 先自定义属性用于标记被依赖的 View

	<!--自定义属性 设定被依赖 view 的 id-->
	<resources>
	    <declare-styleable name="CustomBehaviorStyle">
	        <attr name="anchor_id" format="integer|reference" />
	    </declare-styleable>
	</resources>

	<!--app:anchor_id: 依赖 AppBarLayout-->
	<!--app:layout_behavior: Behavior 的完整类名-->
	<RelativeLayout
	        android:id="@+id/btn_tabs"
	        android:layout_width="match_parent"
	        android:layout_height="52dp"
	        android:layout_gravity="bottom"
	        android:background="#f5f5f5"
	        app:anchor_id="@id/appbar"
	        app:layout_behavior="ringares.com.coordinatorlayoutdemo.behavior.TopBtmScrollBehavior">
	       
而依赖一个 View 的位置及本身状态信息的 Behavior, 主要需要重写两个方法:

<br>

>**CoordinatorLayout.Behavior#layoutDependsOn
>(CoordinatorLayout parent, V child, View dependency)**

这个方法在 Layout 阶段至少被调用一次, 来决定是否有依赖的 dependency. 如果依赖关系成立, 那么在 dependency 的大小和位置改变时, 下面这个方法 `onDependentViewChanged` 就会被调用.

另外需要注意的是:当确定依赖关系后, 当 dependency 被布局(或测量)后 child 会紧接着被布局(或测量), CoordinatorLayout 会无视子 view 的顺序(原因是 CoordinatorLayout 内有个 ComparatormLayoutDependencyComparator 会按照依赖关系对所有的子 View 进行排序), 这会影响它们的测量以及布局顺序).

<br>
		
>**CoordinatorLayout.Behavior#onDependentViewChanged
>(CoordinatorLayout parent, V child, View dependency)**

使 child 响应 dependency 的改变 

<br>

	public class TopBtmScrollBehavior extends CoordinatorLayout.Behavior {
	
	    private int id;
	    public TopBtmScrollBehavior() {
	    }
	
	    public TopBtmScrollBehavior(Context context, AttributeSet attrs) {
	        super(context, attrs);
	        TypedArray typedArray = context.getResources().obtainAttributes(attrs, R.styleable.CustomBehaviorStyle);
	        id = typedArray.getResourceId(R.styleable.CustomBehaviorStyle_anchor_id, -1);
	        typedArray.recycle();
	    }
	
	    /**
	     * @param parent
	     * @param child
	     * @param dependency 被依赖的 view
	     * @return true 如果 child 依赖的 view 是 dependency
	     */
	    @Override
	    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
	    	//确定依赖 class 的方式
	    	//return dependency.getClass() == AppBarLayout.class;

			//灵活的 xml id 的方式
			return dependency.getId() == id;
	    }
	
	    /**
	     * @param parent
	     * @param child
	     * @param dependency
	     * @return true 如果 child 的大小和位置被 behavior 改变了
	     */
	    @Override
	    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
	        child.setTranslationY(-dependency.getTop());
	        return true;
	    }
	}

<br>

**2. 依赖滚动**
上面的例子也可以用依赖滚动的方式来实现, 这种方式需要关注的主要是这几个方法:

	@Override
	public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
		//nestedScrollAxes 是滑动方向, 可用于判断
		return true;//这里返回true, 才会接受到后续滑动事件。
	}
	
	@Override
	public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dx, int dy, int[] consumed) {
			//可以告诉系统, 这个 behavior 需要消耗多少滚动的距离
		}       
		 
	@Override
	public void onNestedScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
		//进行滑动事件处理
	}
	
	public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target, float velocityX, float velocityY) {
		//类似 onNestedPreScroll
		return false; //Behavior 是否消耗掉了 filing
	}
	
	@Override
	public boolean onNestedFling(CoordinatorLayout coordinatorLayout, View child, View target, float velocityX, float velocityY, boolean consumed) {
		//当进行快速滑动
		return super.onNestedFling(coordinatorLayout, child, target, velocityX, velocityY, consumed);
	}
	
	@Override
	public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) {
		//滚动停止时
	}

只要 CoordinatorLayout 有 NestedScrollingChild, 他滑动就会触发这几个回调. 无论你是否依赖了那个View. 而且 NestedScrollingChild 不需要是直接子 View.

这几个方法的调用流程如下:

1. 如果你对滚动事件感兴趣, 可以重写 onStartNestedScroll() 函数. 在该函数中可以知道滚动的方向（水平滚动或者垂直滚动）, 如果你想继续收到该方向的滚动事件, 则必须返回 true.

2. onStartNestedScroll() 返回 true 以后, 在滚动的 View 开始滚动之前调用 onNestedPreScroll() 函数, 在该函数内你的 Behavior 可以吃掉（消耗）部分或者全部滚动的距离, 最后的 int[] 参数为返回值参数, 告诉系统你处理了多少滚动距离.（比如 用户在屏幕上滑动了100像素，滚动的 View 本来应该滚动 100 像素, 但是你的 Behavior 完全吃掉了这100个像素的滚动距离, 则 滚动的 View 就没有滚动了.）

3. 滚动的 View 滚动的时候将会调用 onNestedScroll() .这个函数可以知道滚动的 View 滚动的多少距离, 还有多少没有消耗的距离（滚动到头了）.

4. 对于 fling 操作是同样的处理流程.

5. 当嵌套滚动停止的时候, 会调用 onStopNestedScroll().

<br>

	public class ScrollBasedBehavior extends CoordinatorLayout.Behavior {
	    private boolean isAnimate;
	
	    public ScrollBasedBehavior() {
	    }
	
	    public ScrollBasedBehavior(Context context, AttributeSet attrs) {
	        super(context, attrs);
	    }
	
	    @Override
	    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
	        System.out.println("-------onStartNestedScroll");
	        return true;
	    }
	
	    @Override
	    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dx, int dy, int[] consumed) {
	        System.out.println("-------onNestedPreScroll");
	        if (dy > 0 && !isAnimate && child.getTranslationY() < child.getHeight()) {
	            child.setTranslationY(child.getTranslationY() + dy);
	        } else if (dy < 0 && !isAnimate && child.getTranslationY() > 0) {
	            child.setVisibility(View.VISIBLE);
	            if (child.getTranslationY() + dy < 0) {
	                child.setTranslationY(0);
	            } else {
	                child.setTranslationY(child.getTranslationY() + dy);
	            }
	        }
	    }
	
	    @Override
	    public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, View child, View target) {
	        System.out.println("-------onStopNestedScroll");
	        if (child.getTranslationY() < child.getHeight() / 2) {
	            changeState(child, 0);
	        } else {
	            changeState(child, child.getHeight());
	        }
	    }
	
	
	    private void changeState(final View view, final int scrollY) {
	        ViewPropertyAnimator animator = view.animate().translationY(scrollY).setInterpolator(new FastOutSlowInInterpolator()).setDuration(200 * scrollY / view.getHeight());
	        animator.setListener(new Animator.AnimatorListener() {
	            @Override
	            public void onAnimationStart(Animator animator) {
	                isAnimate = true;
	            }
	
	            @Override
	            public void onAnimationEnd(Animator animator) {
	                if (view.getTranslationY() == view.getHeight()) {
	                    view.setVisibility(View.GONE);
	                }
	                isAnimate = false;
	            }
	
	            @Override
	            public void onAnimationCancel(Animator animator) {
	                view.setTranslationY(scrollY);
	            }
	
	            @Override
	            public void onAnimationRepeat(Animator animator) {
	            }
	        });
	        animator.start();
	    }
	}

## 总结

**使用 Behavior 的三种方式**

- 代码设置 LayoutParameter
- xml app:layout_behavior
- 注解

<br>

**Behavior 依赖的两种类型和重写方法**

- 任意 View, 一般参照其位置及本身状态信息, 如所在位置, 大小, alpha 等等 (例如 FloatingActionButton.Behavior)
- 实现了 NestedScrollingChild 接口的类, 主要参照滚动相关的信息 (例如 AppBarLayout.Behavior)

<br>

**Behavior 生效的注意点**

- 拥有 Behavior 的 View 和被依赖的  dependency 必须都是 CoordinatorLayout 下的直接子 View
- 如果依赖滚动行为, 必须是实现了 NestedScrollingChild 的子类

