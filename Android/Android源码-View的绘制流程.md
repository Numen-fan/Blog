# Android源码-View的绘制流程

为什view.post中可以拿到宽高了。  绘制流程是在onResume之后才开始的。

ViewRootImpl加载decorview, 是最外层 ViewParent，在WindManagerGlobal中addView时进行的初始化

```java
root.setView(view, wparams, panelParentView, userId);
```

 // View的绘制流程，应该从这里开始，这里面 requestLayout();

```java
// Schedule the first layout -before- adding to the window
// manager, to make sure we do the relayout before receiving
// any other events from the system.
requestLayout();  // 注释大意，第一次绘制
```

requestLayout（）会走到

```
scheduleTraversals() {} // 一般的网上博客讲解绘制流程都是从这里开始
-> private void performTraversals() {}
-> mView.measure(childWidthMeasureSpec, childHeightMeasureSpec); 

```

以LinearLayout为例

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

然后走到

```
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec); // 交给child两个测量模式。
}
```

next

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {}
```

去拿到子控件的Measurespec

这里面会根据父布局和子布局的width和height进行确定。

- 特别注意如果父布局是wrap_content，就算子布局是match_parent，那么子布局的测量模式仍然是AT_MOST。
- 如果父布局是match_parent，那么子布局match_parent的测量模式就是exactly

```java
child.measure(childWidthMeasureSpec, childHeightMeasureSpec); // 交给child两个测量模式。
-> setMeasuredDimension(), // 这个时候我们的宽高才是真正的拥有了值
```



接下来自己的宽高，根据不同的ViewGroup类型，有自己的测量模式，比如LinearLayout的垂直布局，会不断的累加子view的height。这里还是会依据自己的测量模式。

# Layout

perforeLayout

View的onLayout方法，由具体实现的View的去实现。

如果是GONE的view，则不会进行摆放，这就是为什么INVISIBLE是会占用空间的。

随后再调用具体child.layout方法

以LinearLayout的vertical布局为例，不断的累加top，垂直的layout。



Draw

performDraw（）用于绘制自己和子view，对于ViewGroup，先绘制自己背景，再去for循环绘制子view，对于View，绘制背景，再绘制自己的内容。

drawSoftware()->mView.draw();

-> drawBackground（）   画背景

onDraw()  // 画自己  

dispatchDraw() // 

