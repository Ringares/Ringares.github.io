---
layout: post
title: "checkable控件状态的研究及自定义状态"
description: "checkable container"
category: Develop
tags: [android]
---

##问题一，实现checkable
如何使**TextView**具有能够被**check**的属性，从而通过修改一个**selector**就能完成**checked和uncheked**的背景切换。

事实上系统已经提供了一个`android.widget.CheckedTextView`，就是一个Textview的子类同时实现了`checkable`接口。

这里有一个简化版的实现

	<selector xmlns:android="http://schemas.android.com/apk/res/android">
	    <item android:drawable="@drawable/checked_bg" 
	        android:state_checked="true" />
	    <item android:drawable="@drawable/normal_bg"></item>
	</selector>

---

	public class CheckableTextView extends TextView implements Checkable {
	    public CheckableTextView(Context context) {
	        super(context);
	    }
	
	    public CheckableTextView(Context context, AttributeSet attrs) {
	        super(context, attrs);
	    }
	
	    public CheckableTextView(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	    }
	
	    private boolean mChecked;
	    private static int[] CHECKED_STATE_SET = {android.R.attr.state_checked};
	
	    @Override
	    protected int[] onCreateDrawableState(int extraSpace) {
	        final int[] drawableState = super.onCreateDrawableState(extraSpace + 1);
	        if (isChecked()) {
	            mergeDrawableStates(drawableState, CHECKED_STATE_SET);
	        }
	        return drawableState;
	    }
	
	    @Override
	    public void setChecked(boolean checked) {
	        if (mChecked != checked) {
	            mChecked = checked;
	            refreshDrawableState();
	        }
	    }
	
	    @Override
	    public boolean isChecked() {
	        return mChecked;
	    }
	
	    @Override
	    public void toggle() {
	        setChecked(!mChecked);
	    }
	}
	
关键的代码就是在状态改变的时候，调用`refreshDrawableState()`来刷新状态的**drawable**。

然后在`onCreateDrawableState()`方法中，根据check的状态来选择相应的**drawableState**。

	private static int[] CHECKED_STATE_SET = {android.R.attr.state_checked};
	
`CHECKED_STATE_SET`指定了生成的drawable的状态，据此找到selector中的具体状态的资源。


##问题扩展，使任意View或者ViewGroup能够具有checkable的能力
当然我们还是可以通过自定义view并实现check爱不了接口来实现功能。然而有没有具有更高复用性的方案呢。。。

这个提供一个方案，要点如下：

- 一个实现了checkable接口的ViewGroup/FrameLayout
- 用这个ViewGroup来包裹想要具有checkable功能的自定义的控件
- `childView.setDuplicateParentStateEnabled(true);`将父view的状态传递到子view

需要注意的问题:

- 这个容器的LayoutParam和自定义的控件的测量模式可能造成的冲突

---

	public class CheckableContainer extends FrameLayout implements Checkable {
	    private boolean mChecked;
	    private static int[] CHECKED_STATE_SET = {android.R.attr.state_checked};
	
	    public CheckableContainer(Context context) {
	        super(context);
	    }
	
	    public View getChildView() {
	        return getChildAt(0);
	    }
	
	    @Override
	    protected int[] onCreateDrawableState(int extraSpace) {
	        final int[] drawableState = super.onCreateDrawableState(extraSpace + 1);
	        if (isChecked()) {
	            mergeDrawableStates(drawableState, CHECKED_STATE_SET);
	        }
	        return drawableState;
	    }
	
	    @Override
	    public void setChecked(boolean checked) {
	        if (mChecked != checked) {
	            mChecked = checked;
	            refreshDrawableState();
	        }
	    }
	
	    @Override
	    public boolean isChecked() {
	        return mChecked;
	    }
	
	    @Override
	    public void toggle() {
	        setChecked(!mChecked);
	    }
	
	    @Override
	    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	        View childView = getChildView();
	        childView.layout(0, 0, right - left, bottom - top);
	        childView.measure(MeasureSpec.makeMeasureSpec(right - left, MeasureSpec.EXACTLY), 0);
	
	    }
	}
	
##自定义控件状态

###res/values/新建一个xml文件：drawable_status.xml

	<?xml version="1.0" encoding="utf-8"?>
	<resources>
	    <declare-styleable name="MessageStatus">
	        <attr name="state_selfdefined" format="boolean" />
	    </declare-styleable>
	</resources>

###需要这个状态的容器或view
同样的需要两步`refreshDrawableState();`和`onCreateDrawableState`。这里需要来判断的状态变为自定义的`R.attr.state_selfdefined`

	private boolean flag;
	private static final int[] STATE_FLAG = {R.attr.state_selfdefined};
		
	public void setStatus(boolean isTrue) {
	    if (flag != isTrue) {
	        flag = isTrue;
	        refreshDrawableState();
	    }
	}
	
	@Override
	protected int[] onCreateDrawableState(int extraSpace) {
	    if (flag) {
	        final int[] ints = super.onCreateDrawableState(extraSpace + 1);
	        int[] mergeDrawableStates = mergeDrawableStates(ints, STATE_FLAG);
	        return mergeDrawableStates;
	    }
	    return super.onCreateDrawableState(extraSpace);
	}
	
###backgroud的selector
也是要使用自己定义的状态~

	<?xml version="1.0" encoding="utf-8"?>
	<selector xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tip="http://schemas.android.com/apk/res/com.ring.tabflowlayout">
	    <item android:drawable="@drawable/checked_bg" tip:state_selfdefined="true" />
	    <item android:drawable="@drawable/normal_bg"></item>
	</selector>
	
###大功告成，别忘了设置背景就是了
