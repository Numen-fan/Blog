# Android源码—View.post为什么可以获取View宽高

## 前言

有一个经典的问题，我们在Activity的onCreate中可以获取View的宽高吗？onResume中呢？

对于这类八股问题，只要看过都能很容易得出答案：**不能**。

紧跟着追问一个，那为什么View.post为什么可以获取View宽高？

今天来看看这些问题，到底为何？

今日份问题：

1. 为什么onCreate和onResume中获取不到view的宽高？
2. 为什么View.post为什么可以获取View宽高？

> 基于Android API 29版本。

## 问题1、为什么onCreate和onResume中获取不到view的宽高?

首先我们清楚，要拿到View的宽高，那么View的绘制流程（measure—layout—draw）至少要完成measure，【记住这一点】。

还要弄清楚Activity的生命周期，关于Activity的启动流程，后面单独写一篇，本文会带一部分。

另外布局都是通过`setContentView(int)`方法设置的，所以弄清楚`setContentView`的流程也很重要，后面也补一篇。

首先要知道Activity的生命周期都在`ActivityThread`中, 当我们调用`startActivity`时，最终会走到`ActivityThread`中的`performLaunchActivity`

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
				……
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
          // 【关键点1】通过反射加载一个Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
           ……
        } catch (Exception e) {
            ……
        }

        try {
            ……

            if (activity != null) {
                ……
                // 【关键点2】调用attach方法，内部会初始化Window相关信息
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                ……
                  
                if (r.isPersistable()) {
                  // 【关键点3】调用Activity的onCreate方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ……
            }
            ……
        return activity;
    }
```

`performLaunchActivity`中主要是创建了Activity对象，并且调用了`onCreate`方法。

onCreate流程中的`setContentView`只是解析了xml，初始化了`DecorView`，创建了各个控件的对象；即将xml中的<TextView /> 转化为一个`TextView`对象。**并没有启动View的绘制流程**。

上面走完了onCreate，接下来看onResume生命周期，同样是在`ActivityThread`中的`performResumeActivity`

```java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ……
        // 【关键点1】performResumeActivity 中会调用activity的onResume方法
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
				……
          
        final Activity a = r.activity;

        ……
          
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE); // 设置不可见
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            ……
              
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                  // 【关键点2】在这里，开始做View的add操作
                    wm.addView(decor, l); 
                } else {
                    ……
                    a.onWindowAttributesChanged(l);
                }
            }

            
        } else if (!willBeVisible) {
           ……
        }
       ……
    }
```

`handleResumeActivity`中两个关键点

1. 调用`performResumeActivity`, 该方法中`r.activity.performResume(r.startsNotResumed, reason);`会调用Activity的`onResume`方法。
2. **执行完Activity的`onResume`后调用了`wm.addView(decor, l); `，到这里，开始将此前创建的DecorView添加到视图中，也就是在这之后才开始布局的绘制流程**



> 到这里，我们应该就能理解，为何onCreate和onResume中无法获取View的宽高了，一句话就是：**View的绘制要晚于onResume。**



## 问题2、为什么View.post为什么可以获取View宽高？

那接下来我们开始看第二个问题，先看看`View.post`的实现。

```java
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
      	// 添加到AttachInfo的Handler消息队列中
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // 加入到这个View的消息队列中
        getRunQueue().post(action);
        return true;
    }
```

post方法中，首先判断`attachInfo`成员变量是否为空，如果不为空，则直接加入到对应的Handler消息队列中。否则走`getRunQueue().post(action);`

从Attach字面意思来理解，其实就可以知道，当View还没有attach时，才会拿到`mAttachInfo`, 因此我们在onResume或者onCreate中调用`view.post()`，其实走的是`getRunQueue().post(action)`。

收下我们看一下`mAttachInfo`在什么时机才会赋值。

在`View.java`中

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
}
```

dispatch相信大家都不会陌生，分发；那么一定是从根布局上开始分发的，我们可以全局搜索，可以看到

![](https://raw.githubusercontent.com/Numen-fan/BlogPicRepo/main/img/20230220192708.png)

不要问为什么一定是这个，因为我看过，哈哈哈

其实`ViewRootImpl`就是一个布局管理器，这里面有很多内容，可以多看看。

在`ViewRootImpl`中直接定位到`performTraversals`方法中；这个方法一定要了解，而且特别长，下面我抽取几个关键点。

```java
    private void performTraversals() {
    	……
    	// 【关键点1】分发mAttachInfo
    	host.dispatchAttachedToWindow(mAttachInfo, 0);
    	……
        
      //【关键点2】开始测量
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      ……
      //【关键点3】开始布局
      performLayout(lp, mWidth, mHeight);
      ……
      // 【关键点4】开始绘制
      performDraw();
      ……
    }
```

再强调一遍，这个方法很长，内部很多信息，但其实总结来看，就是View的绘制流程，上面的【关键点2、3、4】。也就是这个方法执行完成之后，我们就能拿到View的宽高了；到这里，我们终于看到和View的宽高相关的东西了。

但还没结束，我们post出去的任务，什么时候执行呢，上面host可以看成是跟布局，一个ViewGroup，通过一层一层的分发，最后我们看看View的`dispatchAttachedToWindow`方法。

```java
 void dispatchAttachedToWindow(AttachInfo info, int visibility) {
     mAttachInfo = info;
     ……
     // Transfer all pending runnables.
     if (mRunQueue != null) {
         mRunQueue.executeActions(info.mHandler);
         mRunQueue = null;
     }
}
```

这里可以看到调用了`mRunQueue.executeActions(info.mHandler);`

```java
public void executeActions(Handler handler) {
    synchronized (this) {
        final HandlerAction[] actions = mActions;
        for (int i = 0, count = mCount; i < count; i++) {
            final HandlerAction handlerAction = actions[i];
            handler.postDelayed(handlerAction.action, handlerAction.delay);
        }

        mActions = null;
        mCount = 0;
    }
}
```

这就很简单了，就是将post中的Runnable，转移到mAttachInfo中的Handler, **等待接下来的调用执行。**

**这里要结合Handler的消息机制，我们post到Handler中的消息，并不是立刻执行，不要认为我们是先`dispatchAttachedToWindow`的，后执行的测量和绘制，就没办法拿到宽高。实则不是，我们只是将Runnable放到了handler的消息队列，然后继续执行后面的内容，也就是绘制流程，结束后，下一个主线程任务才会去取Handler中的消息，并执行。**

## 结论

1. onCreate和onResume中无法获取View的宽高，是因为还没执行View的绘制流程。
2. view.post之所以能够拿到宽高，是因为在绘制之前，会将获取宽高的任务放到Handler的消息队列，等到View的绘制结束之后，便会执行。

## 参考资料

1. https://cloud.tencent.com/developer/article/1870415