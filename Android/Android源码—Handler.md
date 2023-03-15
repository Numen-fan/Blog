# Android源码—Handler

## 前言

`Handler`是Android开发中使用常用的机制，自然也是面试中的高频考点，大家都清楚，在问到`Handler`时，都会供出他的好伙伴`Message`、`MessageQuene`、`Looper`。

在`Handler`中会延生出一些问题，这些问题包括：

1. `postDelay`是如何做到延迟的？
2. `Handler`是如何做到线程切换的？
3. 主线程`Looper#loop()`死循环会卡顿吗？
4. `Handler`内存泄露如何处理？
5. Handler、Looper、MessageQueue、Thread的对应关系？

接下来我们带着问题来看看Handler的一些源码吧。

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

首先来看一下`Handler`构造的方法。

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

我们知道`Handler`需要和`Looper`、`MessageQueue`关联。直观的看，`Handler`对象中需要持有对应的对象引用

```java
public class Handler {
  ……
    final Looper mLooper;
    final MessageQueue mQueue;
}
```

所以带`Looper`参数的的构造方法意图就很明显了——你的`Handler`需要和哪个`Looper`关联。

主要看一下无参数的构造方法是如何和`Looper`关联上的。

```java
    public Handler(@Nullable Callback callback, boolean async) {
        ……
        //【关键点1】调用myLooper方法获取Looper对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue; // 从Looper中取MessageQueue
        mCallback = callback;
        mAsynchronous = async;
    }

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    public static @Nullable Looper myLooper() {
      // 【关键点2】从ThreadLocal中获取Looper
        return sThreadLocal.get();
    }
```

可见当构建`Handler`没有指定`Loope`r时，**会从ThreadLocal中去获取当前线程的Looper对象**，如果为空，会抛异常，相信不少小伙伴初学Android时都遇到过。那么当前线程的`Looper`是何时存储到`ThreadLocal`中的呢？当然就是调用`Looper#prepare`的时候。

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

这里回答了为什么在子线程中创建`Looper`时，需要先调用`prepare`。那么为什么在主线程中创建的`Handler`不需要呢？

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

所以在主线程中创建`Handler`不再需要初始化`Looper`等。

## 消息入队

我们用得比较多的方法就是`post`、`sendMessage`、`sendMessageDelay`方法，先挨个看看

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

我们看完了消息入队，接下来看消息出队的位置。自然在`MessageQueue`中查找，很容易就定位到`next`方法，调用它的的方法是`Loope#oopOnce`，再往上找就是`Looper#loop`(注意我是Android12)。这个方法很熟悉吧。

```java
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        ……
        // 【关键点10】死循环，只有loopOnce方法返回false时退出
        for (;;) {
            if (!loopOnce(me, ident, thresholdOverride)) {
                return;
            }
        }
    }

		
    private static boolean loopOnce(final Looper me,
            final long ident, final int thresholdOverride) {
        Message msg = me.mQueue.next(); // might block 【注意注释：可能阻塞】
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return false;
        }
       ……
        try {
          ……
            // 【关键点11】还记得这个target是什么吗？
            msg.target.dispatchMessage(msg);
           ……
        } catch (Exception exception) {
            ……
        } finally {
           ……
        }
				……
        return true;
    }
```

`loopOnce`方法中主要就是调用`me.mQueue.next()`获取一个消息msg，注意这里可能阻塞。【关键点11】这里调用了`msg.target.dispatchMessage(msg)`可以回头看看【关键点6】赋值的地方。所以这里就将消息分发给了对应的Handler去处理了。待会儿再看`Handler#dispatchMessage`，我们接着看next方法是怎么取消息的。

```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        ……
        int nextPollTimeoutMillis = 0;
      	// 【关键点12】继续死循环
        for (;;) { 
            ……
            nativePollOnce(ptr, nextPollTimeoutMillis);

          // 加锁
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis(); // 当前时间
                Message prevMsg = null;
                Message msg = mMessages; // msg 指向链表头结点
                if (msg != null && msg.target == null) { // 这个if可以先忽略
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                  // 【关键点13】如果当前时间小于头结点的when，更新nextPollTimeoutMillis，并在对应时间就绪后poll通知
                    if (now < msg.when) { 
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message. 开始获取消息
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next; // 更新头结点 
                        }
                        msg.next = null; // 断链处理，等待返回
                        ……
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null; 
                }

                ……
            }

            ……
            // 下面是IdleHandler的处理，还不是很了解
        }
    }
```

`next`方法中也是一个死循环，**不断的尝试获取当前消息队列已经到时间的消息，如果没有满足的消息，就会一直循环，这就是为什么会`next`会阻塞的原因**。

看完了`next`方法，获取到了msg，回到刚才的`msg.target.dispatchMessage(msg)`，接着看Handler是如何处理消息的。

```java
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
          // 【关键点14】如果消息有CallBack则直接，优先调用callback
            handleCallback(msg);
        } else {
          	// 【关键点15】如果Handler存在mCallback，优先处理Handler的Callback
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
          	// 【关键点16】此方法会被重写，用户用于处理具体的消息
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

`dispatchMessage`方法中优先处理`Message`的`callback`，再回头看看`Handler#post`方法应该知道`callback`是个啥了吧。

【关键点15】如果Handler设置了`mCallback`, 则优先判断`mCallback#handleMessage`的返回值，这个机制可以让我们做一些勾子，监听Handler上的一些消息。

【关键点16】应该不用多说了，这个方法一般在创建`Handler`时被重写，用于接收消息。

---

到这里，我们就简单的把Handler的消息机制看完了，我们结合开篇的几个问题，做一个总结吧。

## 总结

1. 一个线程只能有一个`Looper`，如何确保？
   1. ThreadLocal
2. `Handler#post(Runnable)`会将`Runnable`封装到一个`Message`中。
3. `MessageQueue`采用单链表的实现方式，并且在存取消息时都会进行加锁。
4. `Looper#loop`采用死循环的方式，会阻塞线程。那么为什么主线程不会被阻塞？
   1. 因为Android是事件驱动的，很多的系统事件（点击事件、屏幕刷新等）都是通过Handler处理的，因此主线程的消息队列，会一直有消息的。
5. Handler是如何实现线程切换的？
   1. `Looper`和`MessageQueue`和线程绑定的，**也就是说这个消息队列中的所有消息，最后分给对应的Handler都是在创建Looper的线程**。所以无论Handler在什么线程发送消息，最后都回到创建Looper的线程中执行。（有点饶，可以自己好好捋一下，可以看一下第一节的使用）
6. Thread和Looper、MessageQueue是一对一的关系，Looper、MessageQueue对于Handler和是一对多的关系。**这里要注意，一个具体的Handler实例，肯定只关联一个Looper和queue的哟**。

上面就是整个Handler的消息机制，还有一些扩展内容，一起看看。

## 扩展

### 消息屏障

Android 系统为了保证某些高优先级的 Message（异步消息） 能够被尽快执行，采用了一种消息屏障（Barrier）机制。其大致流程是：先发送一个屏障消息到 MessageQueue 中，当 MessageQueue 遍历到该屏障消息时，就会判断当前队列中是否存在**异步消息**，有的话则先跳过同步消息（开发者主动发送的都属于同步消息），优先执行异步消息。这种机制就会使得在异步消息被执行完之前，同步消息都不会得到处理。我们可以通过调用 Message 对象的 `setAsynchronous(boolean async)` 方法来主动发送异步消息。而如果想要主动发送屏障消息的话，就需要通过反射来调用 MessageQueue 的 `postSyncBarrier()` 方法了，该方法被系统隐藏起来了

屏障消息的作用就是可以确保高优先级的异步消息可以优先被处理，ViewRootImpl 就通过该机制来使得 View 的绘制流程可以优先被处理

——————消息屏障部分摘抄自https://juejin.cn/post/6901682664617705485

### IdleHandler

IdleHandler 是 MessageQueue 的一个内部接口，可以用于在 Loop 线程处于空闲状态的时候执行一些优先级不高的操作，通过 MessageQueue 的 `addIdleHandler` 方法来提交要执行的操作。例如，ActivityThread 就向主线程 MessageQueue 添加了一个 GcIdler，用于在主线程空闲时尝试去执行 GC 操作。

——————IdleHandler部分摘抄自https://juejin.cn/post/6901682664617705485

### Handler内存泄露

Handler作为内部类使用时，持有外部类的引用，比如Activity，所以当Activity销毁时，如果Handler中还有延迟任务，则泄露发生。因此要在合理的时机移除Handler中的消息。`Handler.removeCallbacksAndMessages(null)`

---

> 水平有限，请狠狠指导。













