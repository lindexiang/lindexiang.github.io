---
title: AQS
images: /images/摘要配图/
date: 2018-10-07 22:40:04
tags:
---

# AQS的原理解析
## 简介
java中实现线程同步有synchronized和ReentrantLock方法，当然还有其他的方法。synchronized是使用`monitorenter`和`monitorexit`指令来同步代码块。而ReentrantLock是底层依赖AQS(AbstractQueuedSynchronizer)组件来实现的，即非阻塞同步队列。AQS相比于monitor可以实现多种同步方式，比如独占锁，共享锁，条件队列，公平锁等模式。在并发效率上，synchronized有自旋锁，偏向锁，轻量级锁，重量级锁的优化后，效率和ReentrantLock是差不多的。但是在同步的模式上只能是独占锁。
<!-- more -->
AQS使用CAS机制和用volatile的state值来记录获取锁，竞争锁，释放锁的操作。**AQS不关心如何挂起线程，AQS是判断资源是否能被访问，当线程不能被访问时对线程加入队列，挂起和唤醒等操作。** 

可以思考如下的问题

1. 线程如何访问AQS维护的资源
2. 当资源不可访问时，当前的线程如何挂起 
3. 当线程提前被中断或者其他原因退出访问资源，如何从AQS队列中退出

我们在使用时，**AQS主要的功能有独占锁和共享锁**。在实现了AQS的子类中，一般只会使用其中的一种。比如ReentrantLock实现了独占锁，CountDownLatch和Semphere实现了共享锁。

## AQS的数据结构
AQS维护了一个volatile int state(共享资源)和一个FIFO非阻塞等待队列来实现的。
### node节点
node节点的代码如下所示，

```java
static final class Node {
        // 共享锁模式
        static final Node SHARED = new Node();
        // 独占锁模式
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        /**     
        * CANCELLED，值为1，表示当前的线程被取消     
        * SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；     
        * CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中；           
        * PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行；     
        * 值为0，表示当前节点在sync队列中，等待着获取锁。     */
        // 线程的等待状态
        volatile int waitStatus;
        //前驱节点
        volatile Node prev;
        //后继节点
        volatile Node next;
        //该节点的线程
        volatile Thread thread;
        // 存储condition队列中的后继节点
        Node nextWaiter;
        //是否是共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        // 获取前驱节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    
        }
        // Used by addWaiter 
        Node(Thread thread, Node mode) {     
            this.nextWaiter = mode;
            this.thread = thread;
        }
        // Used by Condition
        Node(Thread thread, int waitStatus) { 
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```
![](http://pbhb4py13.bkt.clouddn.com/2018-10-07-15389236369509.jpg)
在node类中有prev和next，看出AQS的同步队列是双向队列。有thread来指向当前线程，nextWaiter 如果当前的节点时共享模式，值指向一个SHARE节点。当前节点是条件队列中，值会指向下一个等待条件的节点。waitstatus表示当前节点的状态，值如下所示。

1. -1 SIGNAL 当前节点的后继节点被阻塞，当前节点被释放后要唤醒后继节点
2. 1 CANCELLED 当前节点超时或者中断被取消，唯一大于0的值
3. -2 CONDITION 当前节点处于条件队列中，条件没达成不能获取锁
4. -3 PROPAGATE 当前节点处于传播模式，共享锁模式使用该值
5. 0 无 节点初始状态，head初始化条件队列时使用

在独占锁模式下的同步队列结构如下：
![](http://pbhb4py13.bkt.clouddn.com/2018-10-07-15389269065863.jpg)
head节点存储的是new出来的节点，它的waitStatus的值为0，tail指向队列的最后一个节点。

共享锁的同步队列如下：
![](http://pbhb4py13.bkt.clouddn.com/2018-10-07-15389269792122.jpg)
共享锁和独占锁时使用同一个同步队列，队列中的节点可以是共享类型也可以是独占类型。
除了以上的同步队列，还有一个条件队列入地下所示。
![](http://pbhb4py13.bkt.clouddn.com/2018-10-07-15389270569481.jpg)
使用的也是Node节点，是一个单向队列，用nextWaiter来指向下一个节点。

### CAS操作
AQS中有3个重要的变量,head、tail、state都是volatile类型的，保证了可见性。

```java
// 队头结点    
private transient volatile Node head;     
// 队尾结点    
private transient volatile Node tail;     
// 代表共享资源    
private volatile int state;     
protected final int getState() {        
    return state;    
}     
protected final void setState(int newState) {        
    state = newState;    
}     
//CAS操作，当stateoffset的内存的值state和expect相等时将内存的值设为update
protected final boolean compareAndSetState(int expect, int update) {        
    return unsafe.compareAndSwapInt(this,stateOffset, expect, update);    
}
```
## 源码解读
AQS定义了两种的资源共享方式

1. Exclusive 独占锁模式，只有一个线程能执行，ReentrantLock
2. Share 共享锁模式 多个线程可以同时执行，CountDownLatch/Semaphore

### AQS独占锁模式
一般情况下，ReentrantLock的释放方式为

```java
reentrantLock.lock();
//do something
reentrantlock.unlock();
```
ReentrantLock保证了在同一时刻只有一个线程能获取到锁，其余的线程都要挂起等待，**直到拥有锁的线程释放了锁，被挂起的线程被唤醒重新竞争锁。**
ReentrantLock的加锁都是由AQS完成的，它只是初始化了AQS的state资源的数量和获取资源。ReentrantLock分为公平锁和非公平锁。

获取独占锁的流程如下所示
![1D80EF6A-6E04-4586-9A0D-4DFE3A5C8216](http://pbhb4py13.bkt.clouddn.com/2018-10-10-1D80EF6A-6E04-4586-9A0D-4DFE3A5C8216.png)
结合ReentrantLock的源码分析
ReentrantLock 的构造函数中是初始化sync = new NonfairSync()，其中NonfairSync继承Sync，Sync继承了AQS

```java
public ReentrantLock() {
    sync = new NonfairSync(); //构造一个syn
}
```

ReentrantLock.lock得到获取锁的入口函数，调用sync.lock()

```java
public void lock() {
    sync.lock();
}
```

公平锁和非公平锁将lock方法重写了，根据不同的sync调用不同的lock

```java
//非公平锁
final void lock() {
    //用CAS修改state，如果state为0 设置为1 表示当前线程获取锁
    if (compareAndSetState(0, 1))               
        setExclusiveOwnerThread(Thread.currentThread()); //当前线程设置为独占锁
    else
        acquire(1); //尝试获取锁
}t

//公平锁
final void lock() {
    acquire(1);
}
```
**如果state为0，说明没有线程获取锁，可以设置当前线程获取独占锁，当前线程不加入队列，如果state为1，表示有线程占用资源，需要调用acquire()去获取锁**

```java
abstract static class Sync extends AbstractQueuedSynchronizer {}
static final class NonfairSync extends Sync {}
static final class FairSync extends Sync {}
```
* 公平锁 每个线程强占锁的顺序是先后调用lock方法的顺序，并依次获取锁。
* 非公平锁 每个线程强占锁的顺序不变，和调用lock方法的先后顺序无关。
**公平还是非公平是在获取锁的时候是直接获取锁还是先去队列中排队。**

#### acquire()方法

```java
public final void acquire(int arg) {
    //tryAcquire子类重写，不同的逻辑
    //addWaiter是将节点加入到队列的tail     
    //acquireQueued是不断循环，中断线程 
    if (!tryAcquire(arg) &&    
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
            selfInterrupt();
}
```
#### tryAcquire()方法






参考文献
https://blog.csdn.net/lingfenglangshao/article/details/78233414






