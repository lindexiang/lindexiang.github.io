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







