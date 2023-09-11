# 线程变量

为变量在线程中都创建副本，线程可访问自己内部的副本变量。
该类提供了线程局部 (thread-local) 变量，
访问这个变量（通过其 get 或 set 方法）的每个线程都有自己的局部变量，
它独立于变量的初始化副本
实现的原理：每个线程都有一个ThreadLocalMap类型变量threadLocals。
ThreadLocal的set会在threadLocals中保存以ThreadLocal对象为key，
以保存的变量为value的值，get会获取该值。

## ThreadLocal常用api以及使用

https://zhuanlan.zhihu.com/p/576975260
```java
public class ThreadLocalDemo {

    // 定义一个String类型的线程变量
    static ThreadLocal<Integer> context = new ThreadLocal<>();

    public static void main(String[] args) {
        // 设置线程变量的值
        context.set(1000);

        // 从线程变量中取出值
        Integer id = context.get();
        System.out.println(id);

        // 删除线程变量中的值
        context.remove();
        // 输出null
        System.out.println(context.get());
    }
}
```
ThreadLocal的用法非常简单，创建ThreadLocal的时候指定泛型类型，然后就是赋值、取值、删除值的操作。
不同线程之间，ThreadLocal数据是线程隔离的，测试一下：
```java
public class ThreadLocalDemo {
    // 1. 创建ThreadLocal
    static ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {
        IntStream.range(0, 5).forEach(i -> {
          	// 创建5个线程，分别给threadLocal赋值、取值
            new Thread(() -> {
                // 2. 给ThreadLocal赋值
                threadLocal.set(i);
                // 3. 从ThreadLocal中取值
                System.out.println(Thread.currentThread().getName()
                        + "," + threadLocal.get());
            }).start();
        });
    }

}
```

可以看出不同线程之间的ThreadLocal数据相互隔离，互不影响，这样的实现效果有哪些应用场景呢？

ThreadLocal的应用场景主要分为两类：
1.避免对象在方法之间层层传递，打破层次间约束。
比如用户信息，在很多地方都需要用到，层层往下传递，比较麻烦。这时候就可以把用户信息放到ThreadLocal中，需要的地方可以直接使用。拷贝对象副本，减少初始化操作，并保证数据安全。
比如数据库连接、Spring事务管理、SimpleDataFormat格式化日期，都是使用的ThreadLocal，即避免每个线程都初始化一个对象，又保证了多线程下的数据安全。

2.使用ThreadLocal保证SimpleDataFormat格式化日期的线程安全，代码类似下面这样：

```java
/**
 * @author 一灯架构
 * @apiNote ThreadLocal示例
 **/
public class ThreadLocalDemo {
    // 1. 创建ThreadLocal
    static ThreadLocal<SimpleDateFormat> threadLocal =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));


    public static void main(String[] args) {
        IntStream.range(0, 5).forEach(i -> {
            // 创建5个线程，分别从threadLocal取出SimpleDateFormat，然后格式化日期
            new Thread(() -> {
                try {
                    System.out.println(threadLocal.get().parse("2022-11-11 00:00:00"));
                } catch (ParseException e) {
                    throw new RuntimeException(e);
                }
            }).start();
        });
    }

}
```


```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadId {
    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
        new ThreadLocal<Integer>() {
            @Override protected Integer initialValue() {
                return nextId.getAndIncrement();
        }
    };

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }
}
```
```java
public final class TestThreadId extends Thread {

    // number of times to create threads and gather their ids
    private static final int ITERATIONCOUNT = 50;

    // Threads constructed per iteration. ITERATIONCOUNT=50 and
    // THREADCOUNT=50 takes about one second on a sun Blade 1000 (2x750mhz)
    private static final int THREADCOUNT = 50;

    // The thread local storage object for holding per-thread ids
    private static ThreadId id = new ThreadId();

    // Holds the per-thread so main method thread can collect it. JMM
    // guarantees this is valid after this thread joins main method thread.
    private int value;

    private synchronized int getIdValue() {
        return value;
    }

    // Each child thread just publishes its id value for validation
    public void run() {
        value = id.get();
    }

    public static void main(String args[]) throws Throwable {

        // holds true corresponding to a used id value
        boolean check[] = new boolean[THREADCOUNT*ITERATIONCOUNT];

        // the test threads
        TestThreadId u[] = new TestThreadId[THREADCOUNT];

        for (int i = 0; i < ITERATIONCOUNT; i++) {
            // Create and start the threads
            for (int t=0;t<THREADCOUNT;t++) {
                u[t] = new TestThreadId();
                u[t].start();
            }
            // Join with each thread and get/check its id
            for (int t=0;t<THREADCOUNT;t++) {
                try {
                    u[t].join();
                } catch (InterruptedException e) {
                     throw new RuntimeException(
                        "TestThreadId: Failed with unexpected exception" + e);
                }
                try {
                    if (check[u[t].getIdValue()]) {
                        throw new RuntimeException(
                            "TestThreadId: Failed with duplicated id: " +
                                u[t].getIdValue());
                    } else {
                        check[u[t].getIdValue()] = true;
                    }
                } catch (Exception e) {
                    throw new RuntimeException(
                        "TestThreadId: Failed with unexpected id value" + e);
                }
            }
        }
    } // main
} // TestThreadId
```

## 源码解析

### 初始化

```text
    public ThreadLocal() {
    }
```

```text
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
}
```


```text
protected T initialValue() {
        return null;
}
```

获取线程变量的值

返回此线程局部变量的当前线程副本中的值。如果变量没有当前线程的值，则首先将其初始化为initialValue方法调用返回的值。
```text
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

```text
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

```text
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

```text
    public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }

    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```


线程中与 threadlocal 有关系的

```text

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
初始化的逻辑在ThreadLocal中
```text
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    /**
     * Factory method to create map of inherited thread locals.
     * Designed to be called only from Thread constructor.
     *
     * @param  parentMap the map associated with parent thread
     * @return a map containing the parent's inheritable bindings
     */
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
```

返回此线程本地变量的当前线程的`初始值`。该方法将在线程第一次使用get方法访问变量时执行，
除非该线程以前调用过set方法，在这种情况下，不会为该线程调用initialValue方法。
通常，每个线程最多调用一次此方法，但在随后调用remove和get的情况下，可能会再次调用此方法。
此实现只返回null；如果程序员希望线程局部变量的初始值不是null，
那么ThreadLocal必须被子类化，并且这个方法被重写。通常，将使用匿名内部类。


ThreadLocal是线程本地变量，就是线程的私有变量，不同线程之间相互隔离，无法共享，相当于每个线程拷贝了一份变量的副本。

目的就是在多线程环境中，无需加锁，也能保证数据的安全性。

```text
static class ThreadLocalMap {
    // Entry对象，WeakReference是弱引用，当没有引用指向时，会被GC回收
    static class Entry extends WeakReference<ThreadLocal<?>> {
        // ThreadLocal泛型对象值
        Object value;
        // 构造方法，传参是key-value
        // key是ThreadLocal对象实例，value是ThreadLocal泛型对象值
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
  
    // Entry数组，用来存储ThreadLocal数据
    private Entry[] table;
    // 数组的默认容量大小
    private static final int INITIAL_CAPACITY = 16;
    // 扩容的阈值，默认是数组大小的三分之二
    private int threshold;

    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
}
```


## 使用常见




## 常用陷阱



## 


https://xie.infoq.cn/article/baf3b63fab9e932422ec56edf