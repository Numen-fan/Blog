# Android源码-invalidate

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