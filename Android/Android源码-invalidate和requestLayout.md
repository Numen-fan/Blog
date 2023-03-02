# Android源码-invalidate和requestLayout

```
p.invalidateChild(this, damage);
```

父类 

https://www.youtube.com/watch?v=vYyRW6n_U6k

```
parent = parent.invalidateChildInParent(location, dirty);
if (view != null) {
    // Account for transform on current parent
    Matrix m = view.getMatrix();
    if (!m.isIdentity()) {
        RectF boundingRect = attachInfo.mTmpTransformRect;
        boundingRect.set(dirty);
        m.mapRect(boundingRect);
        dirty.set((int) Math.floor(boundingRect.left),
                (int) Math.floor(boundingRect.top),
                (int) Math.ceil(boundingRect.right),
                (int) Math.ceil(boundingRect.bottom));
    }
} while (parent != null);
```

一直找到最外层的ViewRootImpl中

ViewRootImpl

```
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread(); // 这里解释了为什么不能在子线程更新UI，其实是可以的，只是需要在原始线程
    if (DEBUG_DRAW) Log.v(mTag, "Invalidate child: " + dirty);

    if (dirty == null) {
        invalidate();
        return null;
    } else if (dirty.isEmpty() && !mIsAnimating) {
        return null;
    }

    if (mCurScrollY != 0 || mTranslator != null) {
        mTempRect.set(dirty);
        dirty = mTempRect;
        if (mCurScrollY != 0) {
            dirty.offset(0, -mCurScrollY);
        }
        if (mTranslator != null) {
            mTranslator.translateRectInAppWindowToScreen(dirty);
        }
        if (mAttachInfo.mScalingRequired) {
            dirty.inset(-1, -1);
        }
    }

    invalidateRectOnScreen(dirty);

    return null;
}
```



// 找到这个非常重要的方法

```
private void performTraversals() {}
```



```
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
        	mView.draw(canvas); // 不是我们的布局view，只是最外层的View
        }
```



invalidate流程：一路往上调用，到最外层ViewRootImpl， draw（）->dispatchDraw()，一路往下画，最终画到我们的View。所以需要明确，invalidate()是牵连着我们的整个布局



1. **requestLayout重新绘制视图** 子View调用requestLayout方法，会标记当前View及父容器，同时逐层向上提交，直到ViewRootImpl处理该事件，ViewRootImpl会调用三大流程，从measure开始，对于每一个含有标记位的view及其子View都会进行测量、布局、绘制。

2. **invalidate在UI线程中重新绘制视图**，当子View调用了invalidate方法后，会为该View添加一个标记位，同时不断向父容器请求刷新，父容器通过计算得出自身需要重绘的区域，直到传递到ViewRootImpl中，最终触发performTraversals方法，进行开始View树重绘流程(只绘制需要重绘的视图)。
2. **postInvalidate在非UI线程中重新绘制视图**
2. 