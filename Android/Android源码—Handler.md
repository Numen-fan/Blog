# Android源码—Handler

## 前言

Handler是Android开发中使用常用的机制，自然也是面试中的高频考点，大家都清楚，在问道Handler时，都会供出他的好伙伴Message、MessageQuene、Looper。

在Handler中会延生出一些问题，这些问题包括：

1. postDelay是如何做到延迟的？
2. Handler是如何做到线程切换的？
3. 主线程`Looper#loop()`死循环会卡顿吗？
4. Handler内存泄露如何处理？

接下来我们来看看Handler相关的一些源码吧。

> Base on Android12

## 基本使用

先来复习一下Handler的基础用法。

```java

// 直接在主线程中创建Handler
@SuppressLint("HandlerLeak")
final Handler handler = new Handler() {
    @Override
    public void handleMessage(@NonNull Message msg) {
        LogUtils.INSTANCE.error(TAG, "来消息了");
    }
};

// 在一个子线程中使用
new Thread(() -> handler.sendEmptyMessage(10086)).start();
```

示例中的Handler在主线程初始化，如果是子线程中的Handler，记得调用`Looper#prepare`和`Looper.loop()`, 后面说为什么。

## Handler的创建相关

首先来看一下Handler构造的方法

```java
    // 【无参构造方法，高版本已经废弃，不建议使用】
		@Deprecated
    public Handler() {
        this(null, false);
    }
		// 【带Looper参数的构造方法】
    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }
```

我们知道Handler需要和Looper、MessageQueue关联。直观的看，Handler对象中需要持有对应的对象引用

```java
public class Handler {
  ……
    final Looper mLooper;
    final MessageQueue mQueue;
}
```

所以带Looper参数的的构造方法意图就很明显了——你的Handler需要和哪个Looper关联。

下面主要看一下无参数的构造方法是如何和Looper关联上的。

```java
    public Handler(@Nullable Callback callback, boolean async) {
        ……
        //【关键点1】调用myLooper获取Looper对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue; // MessageQueue从Looper中取
        mCallback = callback;
        mAsynchronous = async;
    }

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    public static @Nullable Looper myLooper() {
      // 【关键点2】从ThreadLocal中获取Looper
        return sThreadLocal.get();
    }
```

可见当构建Handler没有指定Looper时，**会从ThreadLocal中去获取当前线程的Looper对象**，如果为空，会抛异常，相信不少小伙伴初学Android时都遇到过。那么当前线程的Looper是何时存储到ThreadLocal中的呢？当然就是调用`Looper#prepare`的时候。

```java
    public static void prepare() {
        prepare(true);
    }
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
      // 【关键点3】存储当前线程的Looper对象，且只能初始化一次。
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

这里回答了为什么在子线程中创建Looper时，需要先调用prepare。那么为什么在主线程中创建的Handler不需要呢？

废话不多说，因为主线程在`ActivityThread#main`方法中已经调用了

```java
   public static void main(String[] args) {
       ……
         // 【关键点4】
        Looper.prepareMainLooper();

        ……
        
        // 【关键点5】
        Looper.loop();
    }
```

所以在主线程中创建Handler不再需要初始化Looper等。

## 消息入队

我们用得比较多的方法就是post、sendMessage、sendMessageDelay方法，先挨个看看

### post方法

```java
    public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
		
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

- 可以看到，post接受一个Runnable类型的参数，并将其封装为一个Message对象，并且将Runnable参数赋值给msg的callback字段，**这里要记住，后面有用——Runnable什么时候执行的呢？**

- 最后调用的就是`sendMessageDelayed`

### sendMessage方法

```java
    public final boolean sendMessage(@NonNull Message msg) {
        return sendMessageDelayed(msg, 0);
    }
```

- 最后调用的也是`sendMessageDelay`，第二个参数是0。

### sendMessageDelay方法

可见无论是post还是sendMessage方法，最后都走到了`sendMessageDelayed`

```java
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

 	// 这里注意第二个参数 @param updateMillis 是一个具体的时间点。
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        ……
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
      // 【关键点6】这里要注意target指向了当前Handler
        msg.target = this;
        

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
      	// 【关键点7】调用到了queue#enqueueMessage方法
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

最终会走到handler中的``enqueueMessage`方法，然后走到`quene#enqueneMessage(msg, uptimeMillis)`，从这个命名来看，就是将message放入了消息队列，那么我们来看看，是如何放入队列的吧。

```java
    boolean enqueueMessage(Message msg, long when) {
        ……
        // 【关键点8】对queue对象上锁
        synchronized (this) {
            ……
            msg.markInUse();
            msg.when = when; // msg的when时刻赋值
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
              	// 翻译：新的头结点，如果queue阻塞，则wakeup唤醒
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
               // 翻译：将消息插入到消息队列中，通常我不需要进行唤醒操作，除非........
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) { // for循环，break结束时when < p.when，说明按照when进行排序插入，或者尾节点
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;   // 【关键点9】 找到插入位置，条件尾部或者when从小到大的位置
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
              // 执行插入
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr); // native方法，唤醒
            }
        }
        return true;
    }
```

这里面将msg放到消息队列中，**可以看到这个队列是一个简单的单链表结构，按照msg的when进行的排序，并且进行了synchronized加锁，确保添加数据的线程安全**。之所以采用链表的数据结构，原因是链表方便插入。

初看源码的时候，应该忽略掉wakeup这些处理，关注msg是如何加入队列即可。

到这里，我们了解了message是如何加入消息队列MesssageQueue。但是消息什么时候执行，即重新`Handler#handleMessage`方法，以及`post(Runnable)`中的Runnable什么时候才执行。

## 消息出队执行

我们看完了入队，那么先看消息出队的位置。自然在`MessageQueue`中查找，很容易就定位到`next`方法，调用它的的方法是`Loope#oopOnce`，再往上找就是`Looper#loop`(注意我是Android12)。这个方法自然就很

