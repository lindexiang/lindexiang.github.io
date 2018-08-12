---
title: 深入理解JVM原理 第12章 java内存模型和线程
date: 2018-08-07 01:09:53
tags: 
  - JVM原理 
  - java内存模型 
  - 线程
categories: JVM虚拟机原理
image: http://pbhb4py13.bkt.clouddn.com/senor-sosa-30861-unsplash.jpg
---






# java内存模型和线程
> JVM的内存模型是为了解决虚拟机实现多线程，但是多线程之间共享和竞争数据导致的问题。

## 硬件的效率与一致性
计算机的存储设备和cpu的运算能力之间存在数量级之间的差距，所以现代计算机系统会在内存和cpu之间再插入告诉缓存cache，cache的速度和cpu的速度一致。这样的好处是每次运算都会先将数据复制到缓存中，在进行cpu计算，计算结束后再讲结果从缓存同步到内存中，这个cpu无需每次都等待缓慢的内存读写了。
<!-- more -->
**存在问题：缓存一致性** 每个cpu都有自己的高速缓存，同时共享同一个主内存。当多个处理器处理涉及同一个内存，需要有**一致性协议**来保证数据一致性。
**同时为了使处理器内部的运算单元能被充分利用，处理器对输入的代码会进行乱序处理优化，处理器会保证结果的正确性**

![93F3194E-7D2F-454F-852D-B00B4A4D3E6F](http://pbhb4py13.bkt.clouddn.com/93F3194E-7D2F-454F-852D-B00B4A4D3E6F.png)

## java内存模型
1. java的内存模型定义了虚拟机将变量存储到内存和从内存中取出变量这样的细节，包括实例字段，静态字段和构成数组对象的元素，但是不包括局部变量和方法参数，因为这些是私有的，不会被共享，不存在竞争问题。
2. **Java内存模型中规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存（可以与前面将的处理器的高速缓存类比）**，线程的工作内存中保存了该线程使用到的变量到主内存副本拷贝，线程对变量的所有操作（读取、赋值）都必须在工作内存中进行，而不能直接读写主内存中的变量。不同线程之间无法直接访问对方工作内存中的变量，线程间变量值的传递均需要在主内存来完成，线程、主内存和工作内存的交互关系如下图所示

![D8A0CA7A-D144-4795-B7DF-7113C29AA36F](http://pbhb4py13.bkt.clouddn.com/D8A0CA7A-D144-4795-B7DF-7113C29AA36F.png)

注意：
1. reference变量是局部变量中，是线程私有的，但是它的实例对象是共享的。
2. 拷贝副本不会一次性拷贝10m的变量。volatile变量还是有工作内存的拷贝，但是它的操作顺序由特殊的规定，使它的操作就像在主内存中访问。
3. 这里的主内存、工作内存与Java内存区域的Java堆、栈、方法区不是同一层次内存划分。

## 内存间交互操作
java内存模型规定了如下8种操作来完成主内存和工作内存之间的数据交互，虚拟机保证如下8种操作是原子性的。
1. lock 作用主内存的变量 把一个变量标识为线程独占的状态
2. unlock 作用主内存的变量 把一个变量从锁定状态释放出来，该变量可以被其他线程锁定
3. read 作用主内存的变量 把一个变量从主内存传输到工作内存，使可以被loan动作使用
4. loan 作用于工作内存 把从主内存read的变量载入工作工作内存的变量副本中
5. use 作用于工作内存 把工作内存的变量传递给cpu，当虚拟机遇到需要读取变量的字节码会执行该指令
6. assign 作用于工作内存 把cpu的执行结果赋值给工作内存副本
7. store 作用于工作内存 把工作内存的变量传输到主内存
8. write 把工作内存传输的变量存到主内存的变量中。

**java内存模型规定了在执行上述的8中基本操作还要满足如下规则**
1. read 和loan 必须同时出现，即工作内存从主内存读取了对象后必须接收
2. 不能丢弃assign操作，即工作内存的变量变化了必须同步到主内存中
3. 没有发生assign，不能将工作内存的变量同步到主内存
4. lock时会清空工作内存的值，cpu需要使用这个值时必须重新loan 在unlock之前，会先将变量同步到主内存。

### volatile变量的特殊规定
**在使用了volatile变量后，当一个线程修改了这个值，另一个线程都会立刻得知。** 如果是普通的变量，只有A线程修改了值，写回主线程，B线程再读取内存的值才可以。
==volatile的操作规定，v和w都是volatile变量：
1. 线程T每次use变量v之前都要执行loan操作，loan操作和read必须同时出现。即（在工作内存中，每次使用v都会从主内存刷新得到最新的值，来保证能看见其他线程对变量v所做的改变。）
2. 线程assign后必须执行store，store和write会同步出现，即(在工作内存中，每次修改v后必须同步回主内存中，用来保证其他线程能立即看见自己对变量v的修改)
3. 保证volatile修饰的变量不会指令重排序优化，保证代码的执行顺序和程序的顺序相同。==

volatile保证了指令不会被乱序，如果initialized没有使用volatile，那么可能由于指令重排序导致线程A的最后的initialized = true被提前执行，则线程B使用配置文件会出错。
![ED9DBF9E-4748-413C-904A-C66E8A5797A1](http://pbhb4py13.bkt.clouddn.com/ED9DBF9E-4748-413C-904A-C66E8A5797A1.png)

### volatile解决DCL(双重检查)问题 单例

使用DCL写的懒汉式单例模式如下所示

```java
public class Singleton {
	
	private static Singleton instance = null;

	private int age;

	public static Singleton getInstance() {
		if(instance == null) { //1
			synchonorized(Singleton.class) { //2
				if(instance == null) { //3
					instance = new Singleton(); //4
				}
			}
		}
		return instance;  //5
	}

	public Singleton() {
		this.age = 18;
	}

	public int getAge() { //6
		return age;
	}
}
```
在一般情况下该单例模式可以正常工作，但是在多线程调用该单例还是会出现并发的问题。**因为可能线程会得到一个并未完全构造完成的对象。**比如当A线程访问getinstance()方法，在//1出instance == null 返回true，获得锁进入同步代码块，此时线程B也访问getInstance()方法，线程B在//1处instance==null可能会返回false，**但是此时instance并未完全初始化完成，线程B得到一个完全初始化的instance，**线程B在调用//6时可能不能拿到age=18的结果，此时DCL的问题就出来了。**问题就出在指令重排序的问题。**
问题出现的原因
==instance = new Singleton()这一句话不是原子操作，它的操作可以分为如下三个部分：
1. 分配内存空间
2. 实例化对象instance
3. 把instance引用指向的已分配的内存空间，此时instance有了内存地址，不再是null了==

_java允许对指令进行重排序，那么以上3步的执行顺序就可能是1-3-2，在这种情况下如果线程A执行完1-3后被阻塞了，此时线程B进来获得了instance的引用，因此此时instance不为空，直接到//1就返回了获得了没有实例完全的对象。_

使用volatile可以避免这个问题，因为volatile的对象保证不会被指令重拍序，在操作volatile对象之前的代码一定是执行完毕并且可见，在变量操作之后的代码一定是还没有被执行的。
所以当instance被定义成volatile时，保证创建的顺序一定是1-2-3，instance一定是null或者完全初始化完成的对象。
其实可以创建成一个static类并获取对象，因为虚拟机会自动保证静态变量的并发。

```java
public class Singleton {
	
	private volatile static Singleton instance = null;

	private int age;

	public static Singleton getInstance() {
		if(instance == null) { //1
			synchonorized(Singleton.class) { //2
				if(instance == null) { //3
					instance = new Singleton(); //4
				}
			}
		}
		return instance;  //5
	}

	public Singleton() {
		this.age = 18;
	}

	public int getAge() { //6
		return age;
	}
}
```



### volatile变量在并发下不安全
volatile变量规定对所有线程都是立即可见的，对volatile的所有写操作都是可以立刻反应到其他的线程中。虽然volatile变量不存在一致性问题，但是java运算操作不是原子性的，所以volatile变量的运算不是安全的。还volatile变量禁止指令重排序。比如自加操作。race++这个过程需要有多个步骤，将race的值取到栈顶，这个过程是正确的，但是接下来的自加操作中如果有其他的线程往主内存写数据就会使数据写回出错。写回去的值可能会比理论值小。
**什么情况下可以使用volatile变量**
1. 运算结果不依赖当前值，或者确保只能一个线程可以修改变量的值
2。 变量不需要其他的状态变量共同参与不变约束。

##原子性，可见性，有序性
java的内存模型都是围绕着在并发过程中如何处理原子性，可见性，有序性三个特征建立的
1. ==原子性== &nbsp;&nbsp;&nbsp;&nbsp;java内存模型直接保证8个基本操作是原子性的，如果要在大范围内保证原子性，必须使用monitorenter和monitorexit来隐式使用，这个反映到java代码就是同步代码块synchonorized关键字，即synchonorized的代码是原子性的。
2. ==可见性== &nbsp;&nbsp;&nbsp;&nbsp;一个线程修改了变量的值，其他的线程能立刻得到这个新的值。如volatile变量，除了这个变量，synchronized和final也可以完成可见性。final字段初始化后就能立刻被其他线程访问。
3. ==有序性== &nbsp;&nbsp;&nbsp;&nbsp;线程内表现为串行的操作，在其他的线程看来是无序的。包括指令重排序和主内存同步延迟现象。

> 对于可见性，violatile保证可见性，synchonorized和final也能保证可见性。synchonorized同步快的可见性是“对一个对象执行unlock前必须将变量同步到主内存(执行store，write)中”保证，**我理解同步块是保证了锁定的对象的可见性**。final关键字的可见性是被final修饰的字段在构造器中一旦被初始化完成，并且构造器没有把“this”引用传递出去，那么其他线程就能得到正确的值。==this指针逃逸是一件危险的事情，其他线程可能访问到初始化了一半的对象。==
对于有序性，violatile和synchonorized都能保证有序性，==violatile是防止指令重排序，synchonorized是由一个变量在同一时刻只能一个线程对其lock操作的这个规则获得的==。则决定了同一个锁的两个同步快只能串行的进入。

###先行发生原则
虚拟机可以对不满足先行原则的指令进行任意顺序的重排序。满足先行发生原则的规则如下：
1. 程序次序原则&nbsp;&nbsp;&nbsp;&nbsp;在一个线程内，代码按照顺序执行
2. 管程锁定规则&nbsp;&nbsp;&nbsp;&nbsp;对同一个锁，unlock操作时间上先行发生于后面的lock操作
3. volatile变量规则&nbsp;&nbsp;&nbsp;&nbsp;对一个volatile变量的写操作先于度操作
4. 线程启动原则&nbsp;&nbsp;&nbsp;&nbsp; Thread的start()先于该线程的任何操作
5. 线程终止原则&nbsp;&nbsp;&nbsp;&nbsp;Thread的所有操作都先于线程的终止检测。可以通过Thread.join()和Thread.isAlive()的返回值检测线程是否已终止
6. 线程终端规则 线程的interrupt()方法先于中断线程检测到中断事件的发生，即可以使用interrupted()方法检测到线程是否被中断了。
7. 对象终结原则 对象构造函数执行完毕先于finilized()方法
8. 传递性 A先于B，B先于C。保证A先于C

## java和线程
线程是比进程轻量级的调度单位。各个线程可以共享进程的资源(内存地址，文件I/O)等，又可以独立调度。==线程是CPU调度的基本单位==。
实现线程一般有3种实现方式，**内核线程实现，用户线程实现，用户线程加轻量级进程混合实现**。
1. 内核线程&nbsp;&nbsp;&nbsp;&nbsp;内核线程(Kernal-Level Thread KLT) 由操作系统内核支持的线程，这种线程由内核来完成线程切换，内核通过**调度器**对线程进行调度，支持多线程的内核叫做多线程内核。**程序一般会使用内核线程的一种高级接口--轻量级进程(Light Weight Process LWP)轻量级进程就是通常意义的线程，每个轻量级进程都是由一个内核线程支持，因此先有内核线程，才能有轻量级进程。**之间的数量关系为1：1。==内核线程耗时好资源，无法创建大规模的内核线程，并发数量低。==
2. 用户线程&nbsp;&nbsp;&nbsp;&nbsp;系统内核感知不到用户线程的存在。线程的创建，同步，销毁，调度都由用户态完成。优点：快速低耗，可以支持更大规模的线程数量。缺点：很难实现线程的调度。==因为CPU只会调度内核线程，用户线程没有内核线程的支持无法处理切换和调度。这个部分都需要用户自己去实现，比如当一个线程死循环不放弃CPU资源，其他线程将用户无法得到执行。==
3. 用户线程加上轻量级进程混合实现&nbsp;&nbsp;&nbsp;&nbsp;该方法使用户线程的创建，切换，调度方便，支持大规模的用户线程并发，并且操作系统提供了轻量级进程作为用户线程和内核线程的桥梁，使用内核的调度功能，即用轻量级进程来调度用户线程，用户线程和轻量级进程的数量比例为N:M。   

![WechatIMG92](http://pbhb4py13.bkt.clouddn.com/WechatIMG92.jpeg)
![WechatIMG93](http://pbhb4py13.bkt.clouddn.com/WechatIMG93.jpeg)

### java线程调度
1. 协同式线程调度 &nbsp;&nbsp;&nbsp;&nbsp;线程的执行时间由线程控制，线程完成工作后通知系统切换到另一个线程。优点:实现简单，不存在线程同步的问题。缺点：一个进程不退出cpu时间就会爆炸
2. 抢占式线程调度&nbsp;&nbsp;&nbsp;&nbsp;java使用该方法来调度。线程的执行时间由系统来决定。

##java线程的6种状态
1. 新建（New）:创建后尚未启动
2. 运行（Runable）：包括running和ready两个状态 状态可能是正在运行或者等待时间片
3. 无限期等待（Waiting）：线程无法获得CPU时间片，必须等待其他线程显示唤醒。 没有设置时间的Object.wait()，Thread.join()方法,LockSupport.park()
4. 限期等待(Timed Waiting) 线程不会被分配CPU时间片也无需被其他线程显示唤醒，到时间自动唤醒。Thread.sleep()方法，设置等待时间的Object.wait()，Thread.join()方法，LockSupport.parkNanos()方法，LockSupport.parkUntil()方法
5. 阻塞(Blocked) 阻塞状态和等待状态的区别是阻塞是等待获取一个排他锁，等待状态是等待唤醒动作的发生，这个过程在线程等待进入同步区域的时候，线程会进入这个状态 
6. 结束(Terminated)：线程被终止
![WechatIMG94](http://pbhb4py13.bkt.clouddn.com/WechatIMG94.jpeg)


