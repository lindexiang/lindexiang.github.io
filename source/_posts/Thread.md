---
title: Thread知识点
date: 2018-10-24 10:43:20
tags: 
    - java并发
    - 线程中断
categories: java同步包
image: http://pbhb4py13.bkt.clouddn.com/jess-watters-559478-unsplash.jpg
top: 100

---

# java线程介绍
## 线程的状态

1. New   
创建后未启动的线程
2. Runnable   
包括Running 和 Ready  运行和就绪
3. Waiting  
不会被分配CPU时间，等待被其他线程显式唤醒 以下会被无限期等待
Object.wait() 没有设置Timeout
没有设置Timeout的thread.join()
LockSupport.park()方法   和这个lock.await()方法  比较类似 
<!-- more -->
4. Timed Waiting 
不会被分配cpu时间，被其他线程显示唤醒或者时间到了自动唤醒 
Thread.sleeping()
Object.wait() 设置了时间
Thread.join() 设置了时间
LockSupport.parkNanos() 设置了时间 lock.await(Time)
LockSupport.parkutil()  就是trylock  
5. blocked  
blocked和waiting的区别是blocked是等待获取一个排它锁，这个事件在另一个线程放弃这个锁的时候发生。Waiting是在等待唤醒动作的发生
6. Terminated  
终止线程或者结束线程
     ![97C18F3C-1273-486B-841E-2E65C028A282](http://pbhb4py13.bkt.clouddn.com/97C18F3C-1273-486B-841E-2E65C028A282.jpg)

### wait和block的区别
Wait状态是线程主动放弃了锁，进入等待状态，需要其他线程去主动notify和notifyall唤醒
blocked是线程获取锁失败而进入阻塞状态，无需唤醒，当锁空闲时，由JVM从blocked队列唤醒这个线程

```java
synchonorized(obj) {
    Obj.wait();
}
```

T1和T2两个线程分别进入这段代码，T1获取了obj的锁，然后wait放弃了锁并进入等待状态
接着T2想获取obj的锁，失败进入blocked阻塞状态，当T1线程wait后得到锁被JVM唤醒
## 线程中断
每个线程都有一个boolean类型的变量来标志该线程的中断状态，Thread类中包含三个与中断状态相关的方法：

```java
Public void Thread{
     // 中断一个线程，将其中断标识位置为true
     public void interrupt() {} 
     // 返回目标线程的当前中断标识 不清除标志位 
     Public void isInterrupted(){}
     //清除线程的中断状态，返回之前的值
     Public static boolean interrupted(){}  
     //私有方法 返回中断表示，并且设置是否清除
     private native boolean isInterrupted(boolean ClearInterrupted)  
}
```
## 如何处理线程中断
### 线程不会被挂起，使用volatile来代替
在任务内没有阻塞的情况下可以使用，但是当任务存在wait()方法时，它不能响应，会无限阻塞下去
```java
public class MyTask implements Runnable
{
   private volatile  running = true;

   public void run() {
        while(running){
            //...操作
        }
   }
    public  void stop() {
        running = false;
    }
}
```

### 线程挂起与线程中断
阻塞方法会使线程进入阻塞的状态，例如：等待获得一个锁、Thread.sleep方法、BlockingQueue的put、take方法等。
大部分的阻塞方法都是响应中断的，即这些方法在线程中执行时如果发现线程被中断，会清除线程的中断状态，并抛出InterruptedException表示该方法的执行过程被从外部中断。响应中断的阻塞方法通常会在入口处先检查线程的中断状态，线程不是中断状态时，才会继续执行
在下面这个例子中，包含了阻塞方法sleep，在线程外部通过interrupt方法请求结束线程，sleep会清除中断标识位并抛出InterruptedException 
阻塞方法抛出的异常有两种解决的办法

* 第一种是重新抛出InterruptedException，将该异常的处理权交给方法调用者，这样该方法也成为了阻塞方法（调用了阻塞方法并且抛出InterruptedException）
* 第二种是通过interrupt方法恢复线程的中断状态，这样可以使得处理该线程的其他代码能够检测到线程的中断状态；使用该方法后，处理该线程的其他方法可以检测到线程中断 。比如有两个任务被阻塞了

```java
public MyTask implements Runnable{
   public void run() {
       try {
          while(!Thread.currentThread().isInterrupted()) {
               Thread.sleep(3000);  //阻塞方法 会抛出中断异常
               //...其他操作
               return;
          }
        }
       catch(InterruptedException ex) {
           Thread.currentThread().interrupt();  //恢复中断状态
       }
   }
}
public class  Test{
    public void method(){
         Thread thread = new Thread(new MyTask()).start();
          //....
          thread.interrupt();      //通过中断机制请求结束线程的执行
    }
}
```

### 不支持中断并且相应中断的任务
对于不支持任务取消操作但是仍然响应中断的阻塞方法应该在本地先保存中断标识位，等任务结束了恢复中断，而不是在捕获InterrptedException的时候中断

```java
public  class  MyTask implements Runnable {
     boolean interrupted = false;
     public void run(){
        try{
           //不支持取消操作
           while(true) { 
              try{
                 Thread.sleep(3000);
                 //...其他操作
                 return;
              }
              catch(InterruptedException ex) {
                 //在本地保存中断状态
                 interrupted = true;  
                 //Thread.currentThread().interrupt();     
                  //不要在这儿立即恢复中断
                 //在这个地方中断，任务将无法完成，无限循环
              }
           }
         }
         finally {
              if(interrupted)
                 Thread.currentThread().interrupt();  //恢复中断
         }
     }
}
```

### 不会抛出中断异常的方法

* 有些方法阻塞方法不响应中断的，即在收到中断请求时不会抛出InterruptedException，如：java.io包中的Socket I/O方法、java.nio.channels包中的InterruptibleChannel类的相关阻塞方法、java.nio.channels包中的Selector类的select方法等。 可以使用其他方法来中断
* 对于java.io包的Socket I/O方法，可以通过关闭套接字，从而使得read或者write方法抛出SocketException而跳出阻塞
* java.nio.channels包中的InterruptibleChannel类的方法其实是响应线程的interrupt方法的，只是抛出的不是InterruptedException，而是ClosedByInterruptedException，除此之外，也可以通过调用InterruptibleChannel的close方法来使线程跳出阻塞方法，并抛出AsynchronousClosedException
* 对于java.nio.channels包的Selector类的select方法，可以通过调用Selector类的close方法或者wakeup方法从而抛出ClosedSelectorExeception

```java
public class ReadThread extends Thread
{
     private final  Socket client;
     private final  InputStream in;

     public ReadThread(Socket client) throws IOException 
     {
         this.client = client;
         in = client.getInputStream();
     }

     public void interrupt()
    {
        try
        {
             socket.close();
        }
        catch(IOException ignore){}
        finally
        {
            super.interrupt();
        }
    }

    public void run()
    {
        //调用in.read方法
    }

}
```


