---
title: 为什么Netty的FastThreadLocal这么快【翻译】
link:
date: 2020-12-27 17:19:08
tags:
---

## 性能测试

ThreadLocal一般在多线程环境用来保存当前线程的数据。用户可以很方便地使用，并且不关心、不感知多线程的问题。下面我会用两个场景来展示多线程的问题：

- 多个线程同时操作一个ThreadLocal
- 一个线程操作多个ThreadLocal

### 1. 多个线程同时操作一个ThreadLocal

测试代码分别用于ThreadLocal和FastThreadLocal。 代码如下:

> ```java
> package com.github.hezhangjian.demo.netty;
> 
> import lombok.extern.slf4j.Slf4j;
> import org.junit.Test;
> 
> import java.util.concurrent.CountDownLatch;
> 
> /**
>  * @author hezhangjian
>  */
> @Slf4j
> public class ThreadLocalTest {
> 
>     @Test
>     public void testThreadLocal() throws Exception {
>         CountDownLatch cdl = new CountDownLatch(10000);
>         ThreadLocal<String> threadLocal = new ThreadLocal<String>();
>         long starTime = System.currentTimeMillis();
>         for (int i = 0; i < 10000; i++) {
>             new Thread(new Runnable() {
> 
>                 @Override
>                 public void run() {
>                     threadLocal.set(Thread.currentThread().getName());
>                     for (int k = 0; k < 100000; k++) {
>                         threadLocal.get();
>                     }
>                     cdl.countDown();
>                 }
>             }, "Thread" + (i + 1)).start();
>         }
>         cdl.await();
>         System.out.println(System.currentTimeMillis() - starTime + "ms");
>     }
> 
> }
> 
> ```

上述的代码创建了一万个线程，并将线程名设置在ThreadLocal中，随后获取这个值十万次，然后通过CountDownLoatch计算总耗时。运行这个程序大概耗时1000ms。

接下来，测试FastThreadLocal，代码基本上相似:

> ```java
> package com.github.hezhangjian.demo.netty;
> 
> import io.netty.util.concurrent.FastThreadLocal;
> import io.netty.util.concurrent.FastThreadLocalThread;
> import lombok.extern.slf4j.Slf4j;
> import org.junit.Test;
> 
> import java.util.concurrent.CountDownLatch;
> 
> /**
>  * @author hezhangjian
>  */
> @Slf4j
> public class FastThreadLocalTest {
> 
>     @Test
>     public void testFastThreadLocal() throws Exception {
>         CountDownLatch cdl = new CountDownLatch(10000);
>         FastThreadLocal<String> threadLocal = new FastThreadLocal<String>();
>         long starTime = System.currentTimeMillis();
>         for (int i = 0; i < 10000; i++) {
>             new FastThreadLocalThread(new Runnable() {
> 
>                 @Override
>                 public void run() {
>                     threadLocal.set(Thread.currentThread().getName());
>                     for (int k = 0; k < 100000; k++) {
>                         threadLocal.get();
>                     }
>                     cdl.countDown();
>                 }
>             }, "Thread" + (i + 1)).start();
>         }
> 
>         cdl.await();
>         System.out.println(System.currentTimeMillis() - starTime);
>     }
> }
> ```

跑完之后，用时还是差不多1000ms。这证明了两者在这个场景下没有什么差别

### 2. 单个线程操作多个ThreadLocal

先看ThreadLocal的:

> ```java
> package com.github.hezhangjian.demo.netty;
> 
> import lombok.extern.slf4j.Slf4j;
> import org.junit.Test;
> 
> import java.util.concurrent.CountDownLatch;
> 
> /**
>  * @author hezhangjian
>  */
> @Slf4j
> public class ThreadLocalSingleThreadTest {
> 
>     @Test
>     public void testThreadLocal() throws Exception {
>         CountDownLatch cdl = new CountDownLatch(1);
>         int size = 10000;
>         ThreadLocal<String> tls[] = new ThreadLocal[size];
>         for (int i = 0; i < size; i++) {
>             tls[i] = new ThreadLocal<String>();
>         }
> 
>         new Thread(new Runnable() {
>             @Override
>             public void run() {
>                 long starTime = System.currentTimeMillis();
>                 for (int i = 0; i < size; i++) {
>                     tls[i].set("value" + i);
>                 }
>                 for (int i = 0; i < size; i++) {
>                     for (int k = 0; k < 100000; k++) {
>                         tls[i].get();
>                     }
>                 }
>                 System.out.println(System.currentTimeMillis() - starTime + "ms");
>                 cdl.countDown();
>             }
>         }).start();
>         cdl.await();
>     }
> 
> }
> 
> ```

上述的代码创建了一万个ThreadLocal，然后设置一个值，随后获取十万次数值，大概耗时2000ms

接下来我们测试FastThreadLocal

> ```
>     public static void test1() {
>         int size = 10000;
>         FastThreadLocal<String> tls[] = new FastThreadLocal[size];
>         for (int i = 0; i < size; i++) {
>             tls[i] = new FastThreadLocal<String>();
>         }
>         
>         new FastThreadLocalThread(new Runnable() {
> 
>             @Override
>             public void run() {
>                 long starTime = System.currentTimeMillis();
>                 for (int i = 0; i < size; i++) {
>                     tls[i].set("value" + i);
>                 }
>                 for (int i = 0; i < size; i++) {
>                     for (int k = 0; k < 100000; k++) {
>                         tls[i].get();
>                     }
>                 }
>                 System.out.println(System.currentTimeMillis() - starTime + "ms");
>             }
>         }).start();
>     }
> ```

运行结果大概只有30ms; 可以发现存在了数量级的差距。接下来重点分析ThreadLocal的机制和FastThreadLocal为什么比ThreadLocal快

### ThreadLocal机制

我们经常会使用到set和get方法，我们分别查看一下源代码:

> ```java
>     public void set(T value) {
>         Thread t = Thread.currentThread();
>         ThreadLocalMap map = getMap(t);
>         if (map != null)
>             map.set(this, value);
>         else
>             createMap(t, value);
>     }
>     
>     ThreadLocalMap getMap(Thread t) {
>         return t.threadLocals;
>     }
> ```

首先，获取当前的线程，然后获取存储在当前线程中的ThreadLocal变量。变量其实是一个ThreadLocalMap。最后，查看ThreadLocalMap是否为空，如果为空，则创建一个新的空Map，如果key不为空，则以ThreadLocal为key，存储这个数据

> ```java
> private void set(ThreadLocal<?> key, Object value) {
> 
>             // We don't use a fast path as with get() because it is at
>             // least as common to use set() to create new entries as
>             // it is to replace existing ones, in which case, a fast
>             // path would fail more often than not.
> 
>             Entry[] tab = table;
>             int len = tab.length;
>             int i = key.threadLocalHashCode & (len-1);
> 
>             for (Entry e = tab[i];
>                  e != null;
>                  e = tab[i = nextIndex(i, len)]) {
>                 ThreadLocal<?> k = e.get();
> 
>                 if (k == key) {
>                     e.value = value;
>                     return;
>                 }
> 
>                 if (k == null) {
>                     replaceStaleEntry(key, value, i);
>                     return;
>                 }
>             }
> 
>             tab[i] = new Entry(key, value);
>             int sz = ++size;
>             if (!cleanSomeSlots(i, sz) && sz >= threshold)
>                 rehash();
>         }
> ```

一般来说，ThreadLocal Map使用数组来存储数据，类似于HashMap。 每个ThreadLocal在初始化时都会分配一个threadLocal HashCode，然后按照数组的长度执行模块化操作，因此会发生哈希冲突。 在HashMap中，使用数组+链表来处理冲突，而在ThreadLocal Map中，也是一样的。 Next索引用于执行遍历操作，这显然具有较差的性能。 让我们再次看一下get方法。

> ```java
>     public T get() {
>         Thread t = Thread.currentThread();
>         ThreadLocalMap map = getMap(t);
>         if (map != null) {
>             ThreadLocalMap.Entry e = map.getEntry(this);
>             if (e != null) {
>                 @SuppressWarnings("unchecked")
>                 T result = (T)e.value;
>                 return result;
>             }
>         }
>         return setInitialValue();
>     }
> ```

同样，首先获取当前线程，然后获取当前线程中的ThreadLocal映射，然后以当前ThreadLocal作为键来获取ThreadLocal映射中的值：

> ```java
>         private Entry getEntry(ThreadLocal<?> key) {
>             int i = key.threadLocalHashCode & (table.length - 1);
>             Entry e = table[i];
>             if (e != null && e.get() == key)
>                 return e;
>             else
>                 return getEntryAfterMiss(key, i, e);
>         }
>         
>          private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
>             Entry[] tab = table;
>             int len = tab.length;
> 
>             while (e != null) {
>                 ThreadLocal<?> k = e.get();
>                 if (k == key)
>                     return e;
>                 if (k == null)
>                     expungeStaleEntry(i);
>                 else
>                     i = nextIndex(i, len);
>                 e = tab[i];
>             }
>             return null;
>         }
> ```

在相同的设置模式下，数组下标通过模块化获取来获取，否则，如果没有冲突，将遍历数据，因此可以通过分析大致了解以下问题：

- ThreadLocal Map是存储在Thread下的，ThreadLocal是键，因此多个线程在同一个ThreadLocal上进行操作实际上是在每个ThreadLocal Map线程中插入的一条记录，没有冲突问题；
- ThreadLocalMap在解决冲突时会通过遍历极大地影响性能。
- FastThreadLocal通过其他方式解决冲突以优化性能
让我们继续看看FastThreadLocal如何实现性能优化

译者说：为什么set的时候不适用fastPath()，因为往往大家使用完ThreadLocal都会remove，这个时候，经常是createEntry，而非updateEntry

## 为什么Netty的FastThreadLocal这么快

Netty分别提供了两类FastThreadLocal和FastThreadLocalThread。 FastThreadLocalThread继承自Thread。 以下也是常用的set和get方法的源代码分析：

> ```java
>    public final void set(V value) {
>         if (value != InternalThreadLocalMap.UNSET) {
>             set(InternalThreadLocalMap.get(), value);
>         } else {
>             remove();
>         }
>     }
> 
>     public final void set(InternalThreadLocalMap threadLocalMap, V value) {
>         if (value != InternalThreadLocalMap.UNSET) {
>             if (threadLocalMap.setIndexedVariable(index, value)) {
>                 addToVariablesToRemove(threadLocalMap, this);
>             }
>         } else {
>             remove(threadLocalMap);
>         }
>     }
> ```

首先，将值确定为Internal ThreadLocalMap。 UNSET，然后内部ThreadLocalMap也用于存储数据:

> ```java
>     public static InternalThreadLocalMap get() {
>         Thread thread = Thread.currentThread();
>         if (thread instanceof FastThreadLocalThread) {
>             return fastGet((FastThreadLocalThread) thread);
>         } else {
>             return slowGet();
>         }
>     }
> 
>     private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
>         InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
>         if (threadLocalMap == null) {
>             thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
>         }
>         return threadLocalMap;
>     }
> ```

可以发现内部ThreadLocal映射也存储在FastThreadLocalThread中。 不同之处在于，它直接使用FastThreadLocal的index属性，而不是使用ThreadLocal的相应哈希值对位置进行建模。 实例化时初始化索引：

> ```java
>     private final int index;
> 
>     public FastThreadLocal() {
>         index = InternalThreadLocalMap.nextVariableIndex();
>     }
> ```

Then enter the nextVariableIndex method:

> ```java
>     static final AtomicInteger nextIndex = new AtomicInteger();
>      
>     public static int nextVariableIndex() {
>         int index = nextIndex.getAndIncrement();
>         if (index < 0) {
>             nextIndex.decrementAndGet();
>             throw new IllegalStateException("too many thread-local indexed variables");
>         }
>         return index;
>     }
> ```

内部ThreadLocal映射中有一个静态nextIndex对象，用于生成数组下标，因为它是静态的，所以每个FastThreadLocal生成的索引都是连续的。 让我们看看如何在内部ThreadLocal映射中设置索引变量：

> ```java
>     public boolean setIndexedVariable(int index, Object value) {
>         Object[] lookup = indexedVariables;
>         if (index < lookup.length) {
>             Object oldValue = lookup[index];
>             lookup[index] = value;
>             return oldValue == UNSET;
>         } else {
>             expandIndexedVariableTableAndSet(index, value);
>             return true;
>         }
>     }
> ```

索引变量是存储值s的对象数组； 直接使用index作为数组下标进行存储； 如果index大于数组的长度，则将其展开； get方法通过FastThreadLocal中的索引快速读取：

> ```java
>    public final V get(InternalThreadLocalMap threadLocalMap) {
>         Object v = threadLocalMap.indexedVariable(index);
>         if (v != InternalThreadLocalMap.UNSET) {
>             return (V) v;
>         }
> 
>         return initialize(threadLocalMap);
>     }
>     
>     public Object indexedVariable(int index) {
>         Object[] lookup = indexedVariables;
>         return index < lookup.length? lookup[index] : UNSET;
>     }
> ```

通过下标直接阅读非常快，这是牺牲空间换来的速度

## 总结

通过以上分析，我们可以知道，当有很多ThreadLocal读写操作时，我们可能会遇到性能问题； 另外，FastThreadLocal实现了O（1）通过空间读取数据的时间； 还有一个问题，为什么不直接使用HashMap（数组+黑红树林）代替ThreadLocalMap。
