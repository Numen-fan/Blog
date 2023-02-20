# 源码阅读—ThreadLocal

## 前言

都知道`ThreadLocal`能够保存变量的副本，使得各个线程的访问不受影响，今天看看是如何做到这一点的。

## 使用

先来看一个`ThreadLocal`的简单用法

```java
// 定义一个ThreadLocal变量，存储类型是Integer
private static final ThreadLocal<Integer> mThreadLocal = new ThreadLocal<>();

public static void main(String[] args) {
  			// 主线程set
        mThreadLocal.set(1);
        System.out.println(Thread.currentThread().getName() + "——>" + mThreadLocal.get());

        new Thread("Thread#1") {
            @Override
            public void run() {
                super.run();
              // 先读取
               System.out.println(this.getName() + "——>" + mThreadLocal.get());
              // 子线程set
                mThreadLocal.set(2);
                System.out.println(this.getName() + "——>" + mThreadLocal.get());
            }
        }.start();

        new Thread("Thread#2") {
            @Override
            public void run() {
                super.run();
              // 子线程get
                System.out.println(this.getName() + "——>" + mThreadLocal.get());
            }
        }.start();
    }

```

上面的例子很简单，声明了`ThreadLocal`变量，存储类型是Integer, 在main函数中，先set(1)，并读取打印。随后启动两个子线程，分别操作这个`ThreadLocal`变量。我们来看一下这个程序的运行结果。

```java
main——>1
  
Thread#1——>null
Thread#1——>2
  
Thread#2——>null
```

从运行结果来看，各个线程对`ThreadLcoal`变量的操作，只在其线程内部生效，并不会影响其它线程，接下来我们看看这点是如何做到的。

## 源码分析

先看看类的定义

```java

/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
*/
public class ThreadLocal<T> {}
```

从注释就可以看出，关键的方法是`set`和`get`，拿我们就直奔主题。

### set方法

```java
public void set(T value) {
    Thread t = Thread.currentThread(); // 首先拿到当前的线程对象
    ThreadLocalMap map = getMap(t); // 拿到当前线程对象中的一个ThreadLocalMap实例
    if (map != null) { // 操作map对象
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

这个方法最终是将value值存储到一个`ThreadLocalMap`对象中，key是当前当前`ThreadLocal`对象，value为实际值。

接下来看看`ThreadLocalMap map = getMap(t);`这句话

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

返回的是`Thread`对象中的变量

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这里注意`ThreadLocalMap`是定义在`ThreadLocal`中的一个静态类，他并不是我们广义理解的Map，看看他的定义。

```java
static class ThreadLocalMap {
		// 内部继续定一个Entry, 封装key value
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    private static final int INITIAL_CAPACITY = 16;
    private Entry[] table; // 一个Entry数组
    private int size = 0;
    ……
 }
```

`ThreadLocalMap`内部又定一个Entry类，并声明一个Entry数组；**因此我们应该清楚了此map非彼map**。

那么继续看看这个所谓的map是如何做set处理的。

```
 private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 拿到这个ThreadLocal的一个hash值
    
    // 处理hash冲突，这里使用简单的线性hash解决hash冲突
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        
        // 说明是重置，直接修改当前entry的value即可
        if (k == key) {
            e.value = value;
            return;
        }
        
        // 说明已经被回收了，需要启动清理，key是弱引用
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
		
		// 在当前位置插入元素
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) // 扩容处理
        rehash();
}
```



> 到这里我们应该清楚了set方法干得事，也应该明白了，为什么`ThreadLocal`变量做到了线程数据备份，并且线程之间不会相互影响。
>
> - 因为每次调用set，都是操作的当前线程中的`ThreadLocalMap`对象。

---

### get方法

其实了解了set的实现机制，get方法就不难了。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

get方法同样是拿到当前线程的ThreadLocalMap对象，在该对象上去查找当前`ThreadLocal`变量的存储value。

这里我们重点分析一下`ThreadLocalMap.Entry e = map.getEntry(this);`

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1); // 计算下标
    Entry e = table[i]; // 从table数组中获取Entry
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

---



## 结论

1. `ThreadLocal`变量在赋值（set）和读取（get）的时候都先拿到了当前线程中的`ThreadLocalMap`对象，该对象是当前的线程的成员变量；这回答了`ThreadLocal`能够实现线程数据备份的原因。
2. `ThreadLocalMap`并不是平时所说的Map结构，他是`ThreadLocal`的静态内部类；并且在其内部还定义了Entry数据结构和一个Entry数组。set和get都作用在这个数组之上。
3. 今天简单的看了set和get方法，以及其中牵扯到的一些东西。其中还有许多其它值得探究的问题，比如内存泄漏，map中的value是强引用。还有hash计算等。





