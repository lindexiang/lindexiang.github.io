---
title: 深入理解JVM虚拟机 第13章 线程安全和锁优化
date: 2018-08-14 23:43:27
tags:
    - JVM原理
    - 线程安全与锁优化
categories: JVM虚拟机原理
image: https://medesqure.oss-cn-hangzhou.aliyuncs.com/minimalistic-landscape-mountains-forest-bird-sky-artwork-others-13757.jpg

---


# 线程安全与锁优化

**线程安全的定义**: 当多个线程访问一个对象时，如果不用考虑这些线程在运行时的调度和交替运行，也不需要执行额外的同步，或者在调用方法时进行其他的协调操作，调用这个对象的行为都可以获得正确的结果，那么这个对象就是线程安全的。
<!-- more -->
## 线程安全的5种数据
### 不可变
一个final的对象被正确的构造出来后(在构造过程中没有发生this指针溢出)，那么这个对象永远不会处于多个线程中不一致的情况，即是线程安全的。
如果共享数据是基本数据类型，只要使用final关键字修饰就可以了。如果是对象，则保证对象的行为对状态不会有影响。比如java.lang.String类。调用他的substring()等方法都是返回一个新构造的对象，线程安全的。
对象行为不影响自己状态可以把对象中带有状态的变量都申明为final，这样在构造函数结束后就是不可变的。比如java.lang.Integer的构造函数。**final类型的变量由JVM保证读取变量的值必须在变量赋值之后**。

```java
private final int value;
 
    /**
     * Constructs a newly allocated {@code Integer} object that
     * represents the specified {@code int} value.
     *
     * @param   value   the value to be represented by the
     *                  {@code Integer} object.
     */
    public Integer(int value) {
        this.value = value;
```

### 绝对线程安全
java.util.Vector是一个线程安全的容器，因为它的方法都加上了synchronized关键字，虽然效率比较低，但是是线程安全的。

```java
private static Vector<Integer> vector = new Vector<Integer>();
 
public static void main(String[] args) {
	while (true) {
		for (int i = 0; i < 10; i++) {
			vector.add(i);
		}
		
		Thread removeThread = new Thread(new Runnable() {
			@Override
			public void run() {
				for (int i = 0; i < vector.size(); i++) {
					vector.remove(i);
				}
			}
		});
		
		Thread printThread = new Thread(new Runnable() {
			@Override
			public void run() {
				for (int i = 0; i < vector.size(); i++) {
					System.out.println(vector.get(i));
				}
			}
		});
		
		removeThread.start();
		printThread.start();
		
		// 不要同时产生过多的线程，否则会导致操作系统假死
		while (Thread.activeCount() > 20);
	}
```
上述的操作会导致数据越界，因为组合操作不是线程安全的，需要对Vector对象进行加锁操作。

```java
Thread removeThread = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized(vector) {
				for (int i = 0; i < vector.size(); i++) {
					vector.remove(i);
				}
			}
		}
	});
	
	Thread printThread = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized(vector) {
				for (int i = 0; i < vector.size(); i++) {
					System.out.println(vector.get(i));
				}
			}
		}

```
### 相对线程安全
大部分的容器都是相对线程安全的， Vector、HashTable、Collections 的 synchronizedCollection() 方法包装的集合等。

## 线程安全的实现方法
### 互斥同步
多个线程并发访问时保证共享书只会被一个线程使用，**实现互斥同步的手段有临界区、互斥量、信号量**。
在java中使用synchronized关键字，被编译后会在同步块的前后分别加入monitorenter和monitorexit两个字节码指令，这两个字节码都需要一个reference类型的参数来指明锁定的对象。
**synchronized明确指明了锁定的对象，那么就是这个对象，如果没有指明，那么就要看synchronized修饰的是实例方法还是类方法，分别取对应的对象实例和class对象作为锁对象。**
**根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加 1，相应的，在执行 monitorexit 指令时将锁计数器减 1，当计数器为 0 时，锁就被释放。如果获取对象锁失败，那当前线程就要阻塞等待，知道对象锁被另外一个线程释放为止。**
使用synchronized同步代码块有2点需要注意：

* synchronized 同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题。采用计数器的方式+1即可。
* **java线程是通过操作系统的内核线程或者称为轻量级线程来实现阻塞和唤醒。**则每次阻塞线程，需要操作系统先将用户态切换到内核态，状态转换需要耗费很多的时间，可能比代码执行的时间还长。因此synchronized操作是重量级锁，**虚拟机本身对synchronized进行了优化，比如操作系统在阻塞线程前进行一段时间的自旋等待过程，避免频繁切换状态**

除了synchonorized，还可以使用java.util.concurrent包的ReentrantLock来实现同步。用法上也可以实现重入锁，ReentrantLock加入了许多高级功能，**等待可中断、实现公平锁、锁绑定多个条件**

* 等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。
* 公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。
* 锁绑定多个条件是指一个 ReentrantLock 对象可以同时绑定多个 Condition 对象，而在 synchronized 中，锁对象的 wait() 和 notify() 或 notifyAll() 方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外添加一个锁，而 ReentrantLock 则无须这样做，只需要多次调用 newCondition() 方法即可。

### 非阻塞同步
互斥同步最主要的问题是进行线程切换和唤醒所带来的性能问题，因为要从用户态切换到核心态，使用轻量级线程来调度用户线程的。并且互斥同步属于悲观锁，无论数据是否存在竞争，都会进行加锁的操作。采用乐观的并发策略的实现可以不用将线程挂起，这种同步操作称为非阻塞同步。
java中一般使用CAS指令完成乐观锁的操作，该指令有3个操作元素，分别是内存位置V、旧的预期值A和新值B。CAS 指令执行时，当且仅当V符合旧值预期值A时，处理器用新值B更新V的值，否则它就不执行更新，但是无论是否更新了V的值，都会返回V的旧值，上述的处理过程是一个原子操作。

```java
public final int incrementAndGet() {
        for (;;) {
            int current = get(); //使用volatile保证可见性
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }
```
incrementAndGet()方法在一个无限循环中，不断尝试将一个比当前值大 1 的新值赋给自己(内部维护了volatile的值)。如果失败了，那说明在执行 “获取-设置” 操作的时候值已经有了修改，于是再次循环进行下一次操作，直到设置成功为止。

但是CAS存在漏洞，即ABA问题，初始值为A，在准备赋值时检查仍然为A，但是其中可能先变成B。即ABA问题。

### ThreadLocal无同步方案
如果一个变量希望只在一个线程中使用，那么可以把这个变量的可见性范围限制在一个线程之中。比如web服务端中，一个请求对应一个服务器线程的处理方式可以使用线程本地储存完成。
如果一个变量会被多个线程访问，可以使用volatile变量，该变量的原理前面已经讲过了。如果一个变量需要被一个线程独享，那么可以使用ThreadLocal类来实现线程本地存储。这个类的原理另外开一篇博客说明。

## 锁优化
### 代码层
减少锁的持有时间(锁粗化)，减少锁的粒度(concurrentHashMap,分段锁机制),锁分离(读写锁)
### JVM层面

1. 锁消除 如果运行时不会出现竞争，直接将锁消除。</br> 
2. 偏向锁 为了避免一个线程对访问的对象重复加锁和解锁，浪费资源。JVM将对象设置为可偏向的，这个是在对象的对象头的Mark Word里面设置的。默认将Mark Word的ThreadId设置为0，当第一个线程访问这个对象时，通过CAS操作将ThreadID设置为线程的ID，线程运行同步代码块。执行完不释放偏向锁，线程下次访问的时候不需要加锁和解锁。如果另外的线程竞争这个锁时，当前线程的锁升级为轻量级锁，另外的线程进行自选等待，如果自旋结束前，上一个线程释放了轻量级锁，该线程获得轻量级锁，如果自旋结束后锁没有被释放，那么轻量级锁会升级为重量级锁，自旋结束的线程进入阻塞状态。
3. 自旋锁 当一个物理机器存在一个以上的处理器能让两个线程同时进行，那么可以让后面请求锁的线程先忙循环(自旋)，但是不放弃处理器的执行时间，称为自旋锁。 **自旋锁虽然避免了线程切换的开销，但是也是会占用处理器的执行时间，如果锁占用的时间很短，自旋等待的效果会很好。否则会性能浪费。**引入了自适应的自旋锁，默认是10次的自旋时间，如果锁在同一个对象，并且上一次自旋成功了，那么这次的自旋时间会加长，否则会缩短。
3. 锁消除

对于如下的代码，因为String是final类型的，所以一定是线程安全的。

```java
public static String concatString(String s1, String s2, String s3) {
	return s1 + s2 + s3;
}
```
但是在代码编译后会变成如下的样子

```java

public static String concatString(String s1, String s2, String s3) {
	StringBuffer sb = new StringBuffer();
	sb.append(s1);
	sb.append(s2);
	sb.append(s3);
	return sb.toString();
}
```
StringBuffer的append()方式使用了synchonorized同步代码块。虚拟机观察了变量sb后，发现它无法逃逸到concatString()方法之外，其他线程无法访问到，则这里虽然有锁，但是会被消除，代码编译后不会添加monitorenter和moniterexit。

### 锁的状态和转化
对象头
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474827770462.jpg)

锁的状态主要有 **无锁状态、偏向锁状态、轻量级锁状态，重量级锁状态**
状态转化

1. 当JVM设置偏向状态时，则默认对象处于偏向锁状态 (CAS操作)
2. 当两个线程竞争锁时，获得偏向锁的线程升级为轻量级锁，如果不再活动，转化成无锁状态，如果再活动，升级为重量级锁，然后解锁，进入无锁状态。
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474827095905.jpg)
轻量级锁CAS操作之前堆栈与对象的状态
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474828219845.jpg)
轻量级锁CAS操作之后堆栈与对象的状态

轻量级锁能提升程序的同步性能的依据是绝大多数的锁，在整个同步周期内是不存在竞争的。因为没有竞争，轻量级锁使用CAS操作避免了互斥量的开销，如果存在锁竞争，除了互斥量的开销，还要发生CAS操作，因此在有竞争的情况下，使用轻量级锁比重量级锁更慢。
#### 偏向锁的执行过程
 代码进入同步快，如果没有被锁定(标志位为"01")，那么在当前线程的栈帧中建立一个锁记录(Lock Record)的空间，存储当前对象的Mark Word(**因为对象头需要记录锁相关的内容**)
![](
https://medesqure.oss-cn-hangzhou.aliyuncs.com/15474827335163.jpg)

 然后虚拟机使用CAS操作将对象的Mark Word更新为 Lock Record的指针，如果成功了，那么线程有了对象的锁，将Mark Word的锁标志变成“00”
如果跟新失败了，那么检查对象的Mark Word是否指向当前线程的栈帧，如果是则已经获得锁直接进入同步快，否则锁被其他线程抢占了。如果多个线程竞争同一个锁，那么轻量级锁没用了，升级为重量级锁。

