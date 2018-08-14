---
title: JVM-13
images: /images/摘要配图/
date: 2018-08-14 23:43:27
tags:
    - JVM原理
    - 线程安全与锁优化
categories: JVM虚拟机原理
image: 
---

# 线程安全与锁优化
==线程安全的定义:==    当多个线程访问一个对象时，如果不用考虑这些线程在运行时的调度和交替运行，也不需要执行额外的同步，或者在调用方法时进行其他的协调操作，调用这个对象的行为都可以获得正确的结果，那么这个对象就是线程安全的。

## 线程安全的5种数据
1. 不可变</br>
&nbsp;&nbsp;&nbsp;&nbsp;一个final的对象被正确的构造出来后(在构造过程中没有发生this指针溢出)，那么这个对象永远不会处于多个线程中不一致的情况，即是线程安全的。</br>
&nbsp;&nbsp;&nbsp;&nbsp;如果共享数据是基本数据类型，只要使用final关键字修饰就可以了。如果是对象，则保证对象的行为对状态不会有影响。比如java.lang.String类。调用他的substring()等方法都是返回一个新构造的对象，线程安全的。</br>
&nbsp;&nbsp;&nbsp;&nbsp;对象行为不影响自己状态可以把对象中带有状态的变量都申明为final，这样在构造函数结束后就是不可变的。比如java.lang.Integer的构造函数。

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

2.绝对线程安全</br>
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
3. 相对线程安全</br>
大部分的容器都是相对线程安全的， Vector、HashTable、Collections 的 synchronizedCollection() 方法包装的集合等。

## 线程安全的实现方法
### 互斥同步
&nbsp;&nbsp;&nbsp;&nbsp;多个线程并发访问时保证共享书只会被一个线程使用，实现互斥同步的手段有==临界区、互斥量、信号量==。
&nbsp;&nbsp;&nbsp;&nbsp;在java中使用synchronized关键字，被编译后会在同步块的前后分别加入monitorenter和monitorexit两个字节码指令，这两个字节码都需要一个reference类型的参数来指明锁定的对象。
&nbsp;&nbsp;&nbsp;&nbsp;**synchronized明确指明了锁定的对象，那么就是这个对象，如果没有指明，那么就要看syn修饰的是实例方法还是类方法，分别取对应的对象实例和class对象作为锁对象。**
&nbsp;&nbsp;&nbsp;&nbsp;==根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加 1，相应的，在执行 monitorexit 指令时将锁计数器减 1，当计数器为 0 时，锁就被释放。如果获取对象锁失败，那当前线程就要阻塞等待，知道对象锁被另外一个线程释放为止。==
&nbsp;&nbsp;&nbsp;&nbsp;使用synchronized同步代码块有2点需要注意：

* synchronized 同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题。采用计数器的方式+1即可。
* **java线程是通过操作系统的内核线程或者称为轻量级进程来实现阻塞和唤醒。**则每次阻塞线程，需要操作系统先将用户态切换到内核态，状态转换需要耗费很多的时间，可能比代码执行的时间还长。因此synchronized操作是重量级锁，**虚拟机本身对synchronized进行了优化，比如操作系统在阻塞线程前进行一段时间的自旋等待过程，避免频繁切换状态**

除了synchonorized，还可以使用java.util.concurrent包的ReentrantLock来实现同步。用法上也可以实现重入锁，ReentrantLock加入了许多高级功能，==等待可中断、实现公平锁、锁绑定多个条件==

* 等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。
* 公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。
* 锁绑定多个条件是指一个 ReentrantLock 对象可以同时绑定多个 Condition 对象，而在 synchronized 中，锁对象的 wait() 和 notify() 或 notifyAll() 方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外添加一个锁，而 ReentrantLock 则无须这样做，只需要多次调用 newCondition() 方法即可。

### 非阻塞同步




