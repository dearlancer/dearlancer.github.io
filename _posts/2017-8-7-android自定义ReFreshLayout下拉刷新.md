---
layout: post
title: 自定义RefreshLayout实现下拉刷新动画
excerpt: “实现下拉刷新控件总的来说就实现几个方面,与自身布局相关的函数onMeasure,onLayout,与事件相关的函数onInterceptTouchEvent,onTouchEvent”
categories: [android]
comments: true
---
## 基本思路
继承自ViewGroup,ViewGroup内部包含两个控件,需要展示的动画效果控件(这里使用ImageView),你的列表控件(ListView,RecycleView等)，通过onMeasure和onLayout对两个控件进行动态布局，通过与事件相关的函数onInterceptTouchEvent，onTouchEvent”对事件进行拦截和处理。

## 基本实现

### 绘制相关
#### onMeasure
```ruby
  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      super.onMeasure(widthMeasureSpec, heightMeasureSpec);
      ensureTarget();
      if (mTarget == null)
          return;
      widthMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredWidth() - getPaddingRight() - getPaddingLeft(), MeasureSpec.EXACTLY);
      heightMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredHeight() - getPaddingTop() - getPaddingBottom(), MeasureSpec.EXACTLY);
      mTarget.measure(widthMeasureSpec, heightMeasureSpec);
      mRefreshView.measure(widthMeasureSpec, heightMeasureSpec);
```
#### onLayout
```ruby
 @Override
 protected void onLayout(boolean changed, int l, int t, int r, int b) {
      ensureTarget();
      if (mTarget == null)
          return;
      int height = getMeasuredHeight();
      int width = getMeasuredWidth();
      int paddingLeft = getPaddingLeft();
      int paddingTop = getPaddingTop();
      int paddingRight = getPaddingRight();
      int paddingBottom = getPaddingBottom();
      mTarget.layout(paddingLeft, paddingTop + mTarget.getTop(), paddingTop + width - paddingRight, paddingTop + height - paddingBottom + mTarget.getTop());
      mRefreshView.layout(paddingLeft, paddingTop, paddingLeft + width - paddingRight, paddingTop + height - paddingBottom);
    }
```

### 事件拦截
#### onInterceptTouchEvent
```ruby
//关键点,什么时候对实践进行拦截(向下拖动的时候),设置一个标志mIsBeingDragged标明控件是否被拖动
 @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (!isEnabled()|| mReturningToStart|| mNestedScrollInProgress || (canChildScrollUp() && !mRefreshing)) {
            return false;
        }
        final int action = ev.getActionMasked();
        if (mReturningToStart && action == MotionEvent.ACTION_DOWN) {
            mReturningToStart = false;
        }
        switch (action) {
            case MotionEvent.ACTION_DOWN:
            //标记按下的的点坐标，mIsBeingDragged=false
            case MotionEvent.ACTION_MOVE:
            //根据之前标记的点来进行计算，是否为向下拖动，是则设置mIsBeingDragged=true
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
            //系列事件处理完毕，重置所有的点
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                break;
        }
        return mIsBeingDragged;
    }    
```
#### onTouchEvent
```ruby
 @Override
 public boolean onTouchEvent(MotionEvent ev) {
      if (!mIsBeingDragged) {
          return super.onTouchEvent(ev);
        }
      final int action = ev.getActionMasked();
      if (mReturningToStart && action == MotionEvent.ACTION_DOWN) {
          mReturningToStart = false;
      }
      switch (action) {
          case MotionEvent.ACTION_MOVE:
             //正在刷新  
                  //向上，如果子控件可以向上偏移，子控件向上移动，不能向上便宜，recycleView滚动
                  //向下，计算偏移距离
             //没有刷新 根据当前手指滑动计算偏移距离,给drawable设置偏移百分比
             //根据计算出的偏移距离重新布局
          case MotionEvent.ACTION_UP:
          case MotionEvent.ACTION_CANCEL:
              //如果下拉距离超过最大下拉距离，回调刷新接口
              //否则，开启动画，让子控件回到顶部
              //重置所有记录
```

## 扩展
### NestedScroll事件相关
实现NestedScrollingChild,NestedScrollingParent接口
NestedScrollingChild负责子View的NestedScroll事件，NestedScrollingParent负责父View的NestedScroll事件
主要的事件交由NestedScrollingChildHelper和NestedScrollingParentHelper处理

### Drawable动画
定义一个上层类实现Drawable.Callback，Animatable接口
public abstract class RefreshDrawable extends Drawable implements Drawable.Callback, Animatable
Drawable.Callback接口主要实现drawable的绘制事件,这里只是作为一个中间件，把事件都交由父类处理
Animatable主要实现动画相关start(),stop(),boolean isRunning()这三个函数
根据isRunning()来判断是否不断重绘自身,start(),stop()由外界调用
