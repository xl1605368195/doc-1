# 线程变量

## ThreadLocal常用api以及使用

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




## 使用常见




## 常用陷阱



## 


https://xie.infoq.cn/article/baf3b63fab9e932422ec56edf