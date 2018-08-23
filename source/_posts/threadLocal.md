---
title: threadLocal原理解析
date: 2018-08-17 00:06:03
tags: 
  - ThreadLocal 
  - 并发编程
categories: 并发编程
image: http://pbhb4py13.bkt.clouddn.com/5b7566b3d7d7d.jpg

---

# ThreadLocal 
## ThreadLocal的用法
> 在工作中使用到了ThreadLocal变量，但是对其原理不是非常的清楚，只是知道可以保存一个共享变量到本地线程的副本，线程之间不会竞争访问该变量。具体到在原理层面上如何去实现，还有ThreadLocal引发的内存泄漏问题都不是非常清楚。这篇博客将会讲讲我对源码的了解。
<!--more-->

ThreadLocalLocal的用法如下所示：

```java
public class ThreadlocalTest {

   //创建一个ThreadLocal对象，设置初始值为3
   private ThreadLocal<Integer> tlA = new ThreadLocal<Integer>() {
       @Override
       protected Integer initialValue() {
           return 3;
       }
   };

    private ThreadLocal<Integer> tlB = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 3;
        }
    };
    //信号量 每次允许一个线程进入
    Semaphore semaphore = new Semaphore(1);

    public class Worker implements Runnable {

        @Override
        public void run() {
            try {
                Thread.sleep(1000);
                semaphore.acquire();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            int valA = tlA.get();
            System.out.println(Thread.currentThread().getName() + "tlA 的初始值 = " + valA);
            valA = new Random().nextInt();
            tlA.set(valA);
            System.out.println(Thread.currentThread().getName() + "tlA 的新值 = " + valA);

            int valB = tlB.get();
            System.out.println(Thread.currentThread().getName() +"tlB 的初始值 = "+ valB);
            valB = new Random().nextInt();
            tlA.set(valB);
            System.out.println(Thread.currentThread().getName() +"tlB 的新值 =  "+ valB);
            semaphore.release();
        }
    }

    /*创建三个线程，每个线程都会对ThreadLocal对象tlA进行操作*/
    public static void main(String[] args){
        ExecutorService es = Executors.newFixedThreadPool(3);
        ThreadlocalTest tld = new ThreadlocalTest();
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.shutdown();
    }
}
```
运行结果如下所示：

```java
pool-1-thread-1tlA 的初始值 = 3
pool-1-thread-1tlA 的新值 = -1506777037
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  906618508
pool-1-thread-3tlA 的初始值 = 3
pool-1-thread-3tlA 的新值 = 1707618403
pool-1-thread-3tlB 的初始值 = 3
pool-1-thread-3tlB 的新值 =  -1088499016
pool-1-thread-2tlA 的初始值 = 3
pool-1-thread-2tlA 的新值 = -601273490
pool-1-thread-2tlB 的初始值 = 3
pool-1-thread-2tlB 的新值 =  1428640209
```
从运行结果来看，每次调用`ThreadLocal`对象的`get`方法都得到了初始值3，让3个线程按照顺序执行，从结果看`pool-1-thread-1`线程结束后设置的`tlA`的新值对`pool-1-thread-3`没有影响，线程3还是得到的是`ThreadLocal`对象的初始值3。相当于把该`ThreadLocal`对象当成是本地变量一样，但是该变量其实是一个共享全局变量。

骚一点，接着对上述的代码做一些简单的改变。
将`main`函数改变线程池的容量大小为1

```java
/*创建三个线程，每个线程都会对ThreadLocal对象tlA进行操作*/
    public static void main(String[] args){
        ExecutorService es = Executors.newFixedThreadPool(1);
        ThreadlocalTest tld = new ThreadlocalTest();
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.execute(tld.new Worker());
        es.shutdown();
    }
```

运行结果如下

```java
pool-1-thread-1tlA 的初始值 = 3
pool-1-thread-1tlA 的新值 = -1998579477
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  1571049844
pool-1-thread-1tlA 的初始值 = 1571049844
pool-1-thread-1tlA 的新值 = -1394637541
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  618157570
pool-1-thread-1tlA 的初始值 = 618157570
pool-1-thread-1tlA 的新值 = -732125710
pool-1-thread-1tlB 的初始值 = 3
pool-1-thread-1tlB 的新值 =  2035779705
```
从运行结果中看出`tlA`的值被多个线程共享了，其实是因为线程池用的都是同一个线程，所以访问的是共享的变量。 接着我们看其实现原理
## ThreadLocal的源码解析
在`Thread`类中定义了一个`threadLocals`,默认是null。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
在第一次调用`ThreadLocal`的get方法时，会为Thread线程创建一个`ThreadLocalMap`对象，这个是一个散列表，key是ThreadLocal对象，set方法中的值作为value，第一次调用get时，以`initValue()`方法返回的结果作为值。

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
    //设置初始值
private T setInitialValue() {
        T value = initialValue(); //这个方法可以被重写，设置自己的初始值
        Thread t = Thread.currentThread();
        ThreadLocalMap map = t.threadLocals;
        if (map != null)
            map.set(this, value);
        else
            t.threadLocals = new ThreadLocalMap(this, firstValue);
        return value;
    }
```

![](http://pbhb4py13.bkt.clouddn.com/15349553966324.png) 
图片出处 https://www.cnblogs.com/nullzx/p/7553538.html


## ThreadLocalMap对象
ThreadLocalMap是ThreadLocal对象内部的一个静态类，内部是维护了一个Entry的散列表，代码如下

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);//调用weakReference的构造函数 
            value = v;
        }
    }
    
    private Entry[] table;
    
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
         table = new Entry[INITIAL_CAPACITY];
         int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
         table[i] = new Entry(firstKey, firstValue);
         size = 1;
         setThreshold(INITIAL_CAPACITY);
    }
    
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }
    
    
    
}
```
每个线程都有一个ThreadLocalMap对象，key为ThreadLocal<?>对象，value为Object。可以在方法中定义多个threadLocal对象，但是一般都是讲ThreadLocal对象定义为static类型或者外部类中。相同的key在不同的Thread中的值是不同的。每个线程都操作各自的ThreadLocalmap对象。

> Entry继承了WeakReference，且设置key为弱引用，WeakReference是在gc时一定会被回收的对象，softReference是在gc后内存不足才会再gc一遍回收软引用指向的对象。
![未命名文件 -1-](http://pbhb4py13.bkt.clouddn.com/未命名文件 (1).jpg)

##ThreadLocal造成的内存泄露
>首先，我们要明确Entry是一个强引用，Entry的key即threadLocal是一个弱引用，当这个对象只有没弱引用持有时，一定是被gc掉的。

考虑一种情况，在一个方法内申明一个ThreadLocal对象，并设置了value值。当方法运行结束时，该threadLocal对象将没有被栈的变量指向，只有Entry的一个弱引用。**那么在gc时，会出现上图中所示的key为null，value存在的情况，而且该value将无法被访问。**则就出现了内存泄漏。

在get方法中的getEntry方法中存在如下的一段代码。当key为null时，会调用expungeStaleEntry()方法去遍历删除所有的key为null的Entry方法。**在调用get方法，set方法，remove方法，在key为null时会删除所有的为null的key**

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
 
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); //其实就是i++
        e = tab[i];
    }
    return null;
}

//遍历删除所有key为null的Entry
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            //重新hash
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

上述的方法并不能保证解决内存泄漏的问题，因为在调用get方法是不一定能获得key为null的对象。当线程结束时，ThreadLocalMap对象会被回收，那么完美。但是使用线程池时，如果ThreadLocal对象被回收，而线程是回收待使用，则value会一直存在堆中无法被访问。内存就会被一直泄漏。
使用线程池时使用不当还会发生bug。**当定义一个static的ThreadLocal对象，使用线程池，在线程中set了一个ThreadLocal对象。那么下一个线程会得到上一个线程的value，造成bug。就像最开始的代码中的运行结果。所以在线程结束时，手动remove掉该ThreadLocal。**


