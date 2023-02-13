# Android源码-Activity启动流程

在ActivityThread中

```
performLaunchActivity 调用onCreate方法
handleResumeActivity->preformResumeActivity（） 调用onResume方法
wm.addView(decor, l) 才开始把我们的decorView加载到WindowManager，View的绘制流程在这个时候才开始， measure，layout draw，这也是为什么我们在onCreate、onResume中无法拿到view的宽高。

```