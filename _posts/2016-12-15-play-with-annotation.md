---
layout: post
title: "如何把注释玩出花"
description: "Play with Annotation"
category: Develop
tags: [android]
---

简而言之 Android&Java 的 doc 注释, 可以使用一些固定的 `@xxx` 标识, 以及大部分的 html 标签. 良好的 doc 注释习惯还是对团队的合作有比较大的帮助.
对我而言, 可能比较实用的

- @deprecated 标记废弃
- {@link CLASS#METHOD(...)} 标记废弃后,链接到新的替代类或方法

## 最常用的注释
比较常见的注释如下, 涉及到了几个关键字

- @param
- @return
- @see
- @deprecated
- {@link 需要在大括号中}
- {@link CLASS#METHOD(...)}

---

	/**
     * 方法说明
     *
     * @param agr1 参数1
     * @param arg2 参数2
     * @return true
     * @see android.widget.Toast
     * @deprecated 已废弃，建议使用{@link #demoMethod2(int, int)} (int)} (int)}
     * <p>
     * {@link ReferenceDemo2#demoMethod(int, int)}
     */
    public boolean demoMethod(int agr1, int arg2) {
        return true;
    }

![table](/images/2016-12-15-play-with-annotation/normal.jpg)


## 列表
	/**
	 * Created by ring
	 * on 16/12/14.
	 * <p>
	 * <p>
	 * Once you have created a tree of views, there are typically a few types of
	 * common operations you may wish to perform:
	 * <ul>
	 * <li><strong>Set properties:</strong> for example setting the text of a
	 * {@link android.widget.TextView}. </li>
	 * <li><strong>Set focus:</strong> Call {@link #demoMethod2(int, int)}.</li>
	 * <li><strong>Set up listeners:</strong>You can register such a listener using
	 * {@link #demoMethod2(int, int)}. </li>
	 * <li><strong>Set visibility:</strong> You can hide or show views using
	 * {@link #demoMethod2(int, int)} .</li>
	 * </ul>
	 * </p>
	 */
	
	public class ReferenceDemo2 {}
	
![table](/images/2016-12-15-play-with-annotation/list.jpg)


## 列个Table
	/**
	 * <p>
	 * In fact, you can start by just
	 * overriding {@link #onDraw(android.graphics.Canvas)}.
	 * <table border="2" width="85%" align="center" cellpadding="5">
	 * <thead>
	 * <tr><th>Category</th> <th>Methods</th> <th>Description</th></tr>
	 * </thead>
	 * <p>
	 * <tbody>
	 * <tr>
	 * <td rowspan="2">Creation</td>
	 * <td>Constructors</td>
	 * <td>The second form should parse and apply
	 * any attributes defined in the layout file.
	 * </td>
	 * </tr>
	 * <tr>
	 * <td><code>{@link #onFinishInflate()}</code></td>
	 * <td>Called after a view and all of its children has been inflated
	 * from XML.</td>
	 * </tr>
	 * <p>
	 * <tr>
	 * <td rowspan="1">Layout</td>
	 * <td><code>{@link #onMeasure(int, int)}</code></td>
	 * <td>Called to determine the size requirements for this view and all
	 * of its children.
	 * </td>
	 * </tr>
	 * <p>
	 * </table>
	 * </p>
	 *
	 * @param agr1 arg1
	 * @param arg2 arg2
	 */I
	public void demoMethod2(int agr1, int arg2) {
	}

![table](/images/2016-12-15-play-with-annotation/table.jpg)

## Bonus Tip
多语言适配的应用会有多个 `values-xx` 及 `strings.xml` 文件, 其实 Android Studio 不仅帮我们准备了一个多语言编辑器, 而且在 strings 文件里 `F1` 查看注释文档时就会清楚地显示所有语言的值.

![](/images/2016-12-15-play-with-annotation/tips.jpg)


