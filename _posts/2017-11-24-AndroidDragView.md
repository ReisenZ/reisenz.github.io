---
layout: post
category: customView
title: Android拖动布局
tags: Android, ViewDragHelper
keywords: Android, ViewDragHelper
excerpt: 这里是Reisen, 种族月兔, 现居永远亭, 今天依然是满月呢~
redirect_from:
  - /2017/11/customView/
---

​    Android布局中要对布局中的控件进行自由拖动, 一种方法是重写父类点击事件的方法, 对触摸事件进行处理, 这种方法代码量过大暂不讨论, 另一种方法是利用`ViewDragHelper`来处理触摸事件.

 `ViewDragHelper` 是官方SupportV4包中提供的工具类, 通过调用`ViewDragHelper.create(ViewGroup, float, Callback)`方法来创建ViewDragHelper对象, 并通过Callback回调函数来判断控件是否需要拖动并控制控件移动状态.

#### 创建布局

​    首先要创建布局, 这里我们直接继承`RelativeLayout`并初始化ViewDragHelper

````java
private void initViewDragHelper() {
        mHelper = ViewDragHelper.create(this, 1.0f, new ViewDragHelper.Callback() {
            @Override
            public boolean tryCaptureView(View child, int pointerId) {
                return false;
            }
    }
````

​    该工厂方法可以创建一个VIewDragHelper对象, 第一个参数为自定义ViewGroup, 不能为空, 第二个参数为触摸灵敏度, 值越大越敏感, 第三个参数为回调接口, 而我们对于事件的处理就在这个CallBack接口中.

​    同时需要重写`onInterceptTouchEvent()` 和`onTouchEvent()`两个方法, 使ViewDragHelper接管触摸事件的处理

````java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
	return isDrag ? mHelper.shouldInterceptTouchEvent(ev) : super.onInterceptTouchEvent(ev);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
	if (isDrag) {
		mHelper.processTouchEvent(event);
	}
	return super.onTouchEvent(event);
}
````



#### 处理CallBack逻辑

​    CallBack中判断控件是否可以拖动有以下几个关键方法:

 `tryCaptureView()` : 判断View是否是可拖动, 返回true表示可该view可拖动

 `clampViewPositionHorizontal()` / `clampViewPositionVertical()`   : 决定子view在水平/垂直方向上应该移动到的位置, 返回0表示不允许该方向上的运动

`getViewHorizontalDragRange()` / `getViewVerticalDragRange()` : 以像素为单位返回子view在水平/垂直方向上可移动的距离, 返回0表示不能在该方向上进行移动

````java
@Override
public boolean tryCaptureView(View child, int pointerId) {
  for (Integer childId : mSet) {//可以拖动的控件id集合
    if (child.getId() == childId) {
      child.bringToFront();//将该控件提至顶层
      return true;
    }
  }
  return false;
}

@Override
public int clampViewPositionHorizontal(View child, int left, int dx) {//限定控件移动不能超出屏幕
  if (left + child.getMeasuredWidth() >= getMeasuredWidth()) {
    return getMeasuredWidth() - child.getMeasuredWidth();
  }
  return Math.max(left, 0);
}

@Override
public int clampViewPositionVertical(View child, int top, int dy) {
  if (child.getMeasuredHeight() + top > getMeasuredHeight()) {
    return getMeasuredHeight() - child.getMeasuredHeight();
  }
  return Math.max(top, 0);
}

@Override
public int getViewHorizontalDragRange(View child) {
  for (Integer childId : mSet) {
    if (child.getId() == childId) {
      return getMeasuredWidth() - child.getMeasuredWidth();
    }
  }
  return 0;
}

@Override
public int getViewVerticalDragRange(View child) {
  for (Integer childId : mSet) {
    if (child.getId() == childId) {
      return getMeasuredHeight() - child.getMeasuredHeight();
    }
  }
  return 0;
}
````

​    之后就是两个回调方法`onViewPositionChanged()` 和  `onViewReleased()`, `onViewPositionChanged()`会在控件位置变化时不断被回调, `onViewReleased()`则是在手指松开时进行回调.

````java
@Override
public void onViewReleased(View releasedChild, float xvel, float yvel) {
  resetPoint(releasedChild.getId(), releasedChild.getLeft(), releasedChild.getTop());
}

@Override
public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
  resetPoint(changedView.getId(), left, top);
}

private void resetPoint(int viewId, int x, int y) {
  Point point = mViewsPoint.get(viewId);
  if (point == null) {
    mViewsPoint.put(viewId, new Point(x, y));
  } else {
    point.set(x, y);
  }
}
````

#### 其他问题处理

​    实际上, `onViewPositionChanged()` 和  `onViewReleased()`并非必须重写, 但是一般都会重写该方法并对自定义布局进行一些处理. 首先我们不处理这两个方法看看是什么效果.

![](https://raw.githubusercontent.com/ReisenZ/update/master/vd1.gif)

​    效果是不是很奇怪? 当松手后重新触摸可拖动的控件时, 控件都重新回到一开始的位置上去了. 原因是当触摸控件时, `tryCaptureView()`方法调用`bringToFront()`将这个控件移动到顶层了, 该方法会调用我们父控件的`onLayout()`方法重新布局, 导致控件复位. 

​    原因找到了, 问题也就好解决了. 我们在`onViewPositionChanged()` 和  `onViewReleased()`方法中记录控件所在的位置, 然后重写自定义ViewGroup的`onLayout()`方法并指定其位置就可以了.

````java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
  super.onLayout(changed, l, t, r, b);
  a:
  for (Integer id : mSet) {
    int count = getChildCount();
    for (int i = 0; i < count; i++) {
      View view = getChildAt(i);
      if (view.getId() != id) {
        continue;
      }
      Point point = mViewsPoint.get(id);
      if (point == null) {
        continue a;
      }
      view.layout(point.x, point.y, point.x + view.getMeasuredWidth(), point.y + view.getMeasuredHeight());
    }
  }
}
````

​    我们还可以给拖动的控件设置吸附窗体边缘, 当控件靠近窗体边缘时, 自动吸附上去. 只需要在手指离开屏幕时调用ViewDragHelper的`settleCapturedViewAt()`指定要移动到的位置然后`invalidate()`就可以了.

````java
@Override
public void onViewReleased(View releasedChild, float xvel, float yvel) {
  resetPoint(releasedChild.getId(), releasedChild.getLeft(), releasedChild.getTop());
  //吸附边缘
  if (!isAdsorb) {
    return;
  }
  Point end = adsorb(null, releasedChild);
  if (end == null) {
    return;
  }
  mHelper.settleCapturedViewAt(end.x, end.y);
  invalidate();
}
````

#### 源码分析



[详细代码](https://github.com/ReisenZ/WidgetDemo/blob/master/lib/src/main/java/work/reisen/lib/widget/VDRelativeLayout.java)

#### 参考

[Android DragViewHelper](http://dongchuan.github.io/android/2016/05/29/Android-DragViewHelper.html)

