# Android源码-Touch事件机制

# View的事件机制

> 思考：如果一个View设置了setOnTouchListener 和 setOnClickListener，还重写了onTouchEvent方法，那么一次点击，执行顺序是什么

分情况：

1. 如果setOnTouchListener接口的onTouch方法返回false。onTouchEvent返回也是false。
   1. setOnTouchListener -> onTouchEvent （UP事件之后）-> OnClickListener
2. 如果setOnTouchListener接口的onTouch方法返回true。
   1. 只有setOnTouchListener的onTouch方法会执行。

在View中有两个非常重要的方法

1. dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent event) {
  ……
     boolean result = false; // 一个标记量
  ……
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;  // 关键，存放了关于View的所有Listener信息，如OnTouchListener，onClickListener等
    if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) { // 这里注意onTouch方法的返回值，如果是false，那么result继续是false。
        result = true;
    }

    if (!result && onTouchEvent(event)) { // 关键，如果result是false才会走，这时候走onTouchEvent方法
        result = true;
    }
    
}
```

- 结论：
  - 1 如果setOnTouchListener的onTouch方法返回了true，那么onTouchEvent方法则不会执行。即OnTouchListener的onTouch方法的返回值决定是否分发到onTouchEvent中。
  - 如果重写了dispatchTouchEvent，要注意super的调用，否着setOnTouchListener，onTouchEvent，OnClickListener都没有效果。

2. onTouchEvent，一般会被我们复写

```java
public boolean onTouchEvent(MotionEvent event) {
  ……
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:  // 注意是UP事件
                 ……
                   if (!post(mPerformClick)) {
                       performClickInternal();   // 这里调用了OnClickListener
                    }
     }
}
```

- 结论：
  - onClickListener在onTouchEvent中，处于事件消费的最末端。
  - 复写onTouchEvent方法，需要考虑返回值，**如果不调用super的话，会导致点击事件不会执行（比较重要，要记住）**。

# ViewGroup的事件机制

三个重要的方法

1. dispatchTouchEvent
2. onTouchEvent
3. onInterceptTouchEvent

布局场景

```xml
<com.jiajia.androidview.TouchViewGroup xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".TouchActivity">

    <com.jiajia.androidview.TouchView
        android:id="@+id/touch_view"
        android:background="@color/design_default_color_error"
        android:layout_width="200dp"
        android:layout_height="200dp"/>

</com.jiajia.androidview.TouchViewGroup>
```

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	……
    boolean handled = false;
  if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
  // 默认情况下 down是会走到拦截方法里面的
  if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            }
  
  
  if (!canceled && !intercepted) {
    if (newTouchTarget == null && childrenCount != 0) {
      for (int i = childrenCount - 1; i >= 0; i--) { // 注意反序列的遍历
        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) { 
          // dispatchTransformedTouchEvent中会调用child的dispatcTouchEvent方法
           newTouchTarget = addTouchTarget(child, idBitsToAssign); // 会添加一个新的target，并且会赋值mFirstTouchTarget（一个链表头）
          
          
        }
      }
    }
  }
  return handled;
}
```

如果说子view没有消费点击事件，那么Down事件会走完全程，但是后面的事件则不会再来了。

如果对于ViewGroup来讲，你想拦截子View的touch事件，可以复写onInterceptTouchEvent返回true即可。此时会执行自己的onTouchEvent







