# Android源码—setContentView

## 前言

今天我们来看另一个老生常谈的问题——setContentView。相信不少做Android开发的同学在面试过程中都遇到过这个问题。

先来看一个图片

![](https://raw.githubusercontent.com/Numen-fan/BlogPicRepo/main/img/20230222191724.png)

大家都很熟悉吧，这张图片描述了Activity的结构，简单总结一句话就是：一个Activity，内部有一个PhoneWindow，PhoneWindow内部有一个DecorView，我们通常调用的方法`setContentView(R.layout.xxx)`其实是将自己写写的布局放在上图的ContentView部分。

看了很多博客，背了很多遍，或许早就烂熟于心了，那么接下来，我们一起来看源码，理清源码中这个图的结构，看看setContentView都做了那些工作。

## 源码

Activity中setContentView方法很简单。

```java
    private Window mWindow; // 有一个Window变量

		public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
		……
    public Window getWindow() {
        return mWindow;
    }
```

可以看到Activity中持有一个Window对象，接着看看`mWindow`的初始化；

```java
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        attachBaseContext(context);
        ……
        mWindow = new PhoneWindow(this, window, activityConfigCallback); // 是PhoneWindow实例
       ……
    }

```

这里已经交代了Activity中的`mWindow`是PhoneWindow对象。这里扩展一下，你知道attach方法在什么时机调用的吗，这涉及到Activity的启动流程了，不再这里多说。

接下来看看PhoneWindow中setContentView。

```java
    private DecorView mDecor; // 一个DecorView对象
    ViewGroup mContentParent; // 一个ViewGroup

@Override
public void setContentView(int layoutResID) {
   ……
    if (mContentParent == null) {
      // 【关键点1】
        installDecor();  // 初始化mDecor变量
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ……
    } else {
      // 【关键点2】
        mLayoutInflater.inflate(layoutResID, mContentParent); // 将我们自己写得布局加载到一个Android:R.id.content的布局之中。
    }
   ……
}
```

这段代码中，有两个关键的地方需要注意

【关键点1】`installDecor();`

```java
    private DecorView mDecor; // 一个DecorView对象
    ViewGroup mContentParent; // 一个ViewGroup

		private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
          // 【关键点3】
            mDecor = generateDecor(-1);
            ……
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
          // 【关键点4】
            mContentParent = generateLayout(mDecor);   
          
            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
             	……

            } else {
                ……
            }

            ……
    }
```

在【关键点3】`mDecor = generateDecor(-1);`初始化了`mDecor`,

```java
    protected DecorView generateDecor(int featureId) {
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```

其实就是new了一个`DecorView`。这里要记住，**`DecorView`就是一个FrameLayout**

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
  			……
        ViewGroup mContentRoot; // 这个后面会讲的
  		  ……
}
```

接下来看【关键点4】`mContentParent = generateLayout(mDecor);   `

```java
    protected ViewGroup generateLayout(DecorView decor) {
        …… // 这里省略了老多代码了
        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss; // 看这里
            setCloseOnSwipeEnabled(true);
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;   // 这里
            } else {
                layoutResource = R.layout.screen_title_icons; // 看这里
            }
            
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            ……
            layoutResource = R.layout.screen_progress; // 看这里
           
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
           
            ……
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            
           layoutResource = xxx // 看这里
            
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode; // 看这里
        } else {
            
            layoutResource = R.layout.screen_simple; // 看这里
        }

       // 【关键点5】
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
      	//【关键点6】
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        ……
      	// 【关键点7】
        return contentParent;
    }

```

这个方法很长，我们抓住我们关注的点





