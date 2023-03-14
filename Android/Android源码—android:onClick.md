# Android源码追踪—android:onClick

## 前言

之前对源码的阅读，总是用时一通乱七八糟的跳转，以学会使用为目的；过了一段时间，就忘记了，因此打算将一些源码的阅读经历记录下来，也通过敲一遍的过程，加深理解。

最开始，用一个比较简单的例子来小试牛刀吧

## 问题描述

对于View（Button、TextView等）的点击事件，常用的写法是通过`findViewById`获取View的实例，然后通过`setOnClickListener`设置监听事件，比如我们有如下Button控件。

```xml
<Button
        android:id="@+id/btn_submit"
        android:text="Submit"/>
```

设置点击事件（假设在Activity中）

```java
findViewById(R.id.btn_submit).setOnClickListener(v -> {
  			// 提交
  			doSubmit(v);
		}
  )
  
/**
* 提交处理
*/
public void doSubmit(View v) {
  	……
}
```



但是还有一种写法是在xml布局中通过android:onClick属性直接指定点击执行的函数。

```xml
<Button
        android:id="@+id/btn_submit"
        android:text="Submit"
        android:onClick="doSubmit"/>
```

【思考】

- xml中设置onClick属性，如何映射到doSubmit方法？



## 源码分析过程

首先我们知道诸如`android:xxx`之类的属性是会在某个attrs文件中定义的，此处的`android:onClick`是View的属性，定义在如下文件中。

> Android>sdk>platforms>android-xx>data>res>values>attrs.xml

````xml
<declare-styleable name="View">
  	……
		<attr name="onClick" format="string" />  <!--是一个string类型的值。-->
  	……
</declare-styleable name="View">
````



在View的构造函数中，会解析出此属性的值。

```java
public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
	this(context);
  // 1 获取属性
	// attrs就是xml中的属性，com.android.internal.R.styleable.View就是上述 <declare-styleable name="View">
	inal TypedArray a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);
  
  // 2 处理属性值
   for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
            switch (attr) {
                case R.styleable.View_onClick:
                    ……
                    final String handlerName = a.getString(attr);
                    if (handlerName != null) {
                        setOnClickListener(new DeclaredOnClickListener(this, handlerName));
                    }
                		break;
            }
  
}
```

看这里，**如果变量handlerName不为空，就会为此View设置点击事件了**，这个handlerName就是onClick属性的值doSubmit，但这个点击事件，并不是我们所熟悉的OnClickListener。

进一步看看这个`DeclaredOnClickListener`类

```java
private static class DeclaredOnClickListener implements OnClickListener {
        private final View mHostView;
        private final String mMethodName;

        private Method mResolvedMethod; // 【Method属性】
        private Context mResolvedContext; // 【Context】

        public DeclaredOnClickListener(@NonNull View hostView, @NonNull String methodName) {
            mHostView = hostView;
            mMethodName = methodName;
        }

        @Override
        public void onClick(@NonNull View v) {
            if (mResolvedMethod == null) { // 初始化mResolvedMethod 和 mResolvedContext变量
                resolveMethod(mHostView.getContext(), mMethodName);
            }
            try {
              	// 【重点】通过反射
                mResolvedMethod.invoke(mResolvedContext, v);
            } catch (IllegalAccessException e) {
                ……
            }
        }

        @NonNull
        private void resolveMethod(@Nullable Context context, @NonNull String name) {
            while (context != null) {
                try {
                    if (!context.isRestricted()) {
                      // 【重点】通过Class获取方法。
                        final Method method = context.getClass().getMethod(mMethodName, View.class);
                        if (method != null) {
                            mResolvedMethod = method;
                            mResolvedContext = context;
                            return;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    // Failed to find method, keep searching up the hierarchy.
                }

                if (context instanceof ContextWrapper) { 
                    context = ((ContextWrapper) context).getBaseContext();
                } else {
                    // Can't search up the hierarchy, null out and fail.
                    context = null;
                }
            }
          ……
        }
    }
```



`DeclaredOnClickListener`实现了`OnClickListener`，其中重点是参数`mResolvedMethod`和`mResolvedContext`。

- `mResolvedMethod`是Method类型，通过`context.getClass().getMethod(mMethodName, View.class)`获取，这个mMethodName就是上面获取的**handlerName**（doSubmit）。
- `mResolvedContext`这里就可以理解为doSubmit所在的类，这里是Activity。

## 结论

在onClick事件中最终通过反射`mResolvedMethod.invoke(mResolvedContext, v);`执行了doSubmit方法。

## 思考

doSubmit的访问权限是否可以设置为private呢？

答案：不可以，因为源码中没有调用`mMethod.setAccessible(true);`注入所有修饰符。

其实在onClick属性的注释中就已经说明了。

```xml
<!-- Name of the method in this View's context to invoke when the view is
             clicked. This name must correspond to a public method that takes
             exactly one parameter of type View. For instance, if you specify
             <code>android:onClick="sayHello"</code>, you must declare a
             <code>public void sayHello(View v)</code> method of your context
             (typically, your Activity). -->
```



## 最后

1. 对于android:onClick属性，是不建议这样用的，官方也进行了@deprecated注解，这样的代码不利于维护。
2. 通过追踪它的实现方式，慢慢开始源码阅读之旅。
3. 其实，很多注解框架，都是通过反射实现的，比如XUtils等。









