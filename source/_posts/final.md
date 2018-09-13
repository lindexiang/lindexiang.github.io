---
title: 深入理解final关键字的作用
date: 2018-08-30 22:55:46
tags: 
   - 内存模型
   - final关键字
   - 线程安全
   - 内存可见性
categories: java并发编程
image: http://pbhb4py13.bkt.clouddn.com/2018-08-30-david-beatz-798135-unsplash.jpg
top : 99

---

# final关键字特性
final关键字在java中使用非常广泛，可以申明成员变量、方法、类、本地变量。一旦将引用声明为final，将无法再改变这个引用。final关键字还能保证内存同步，本博客将会从final关键字的特性到从java内存层面保证同步讲解。这个内容在面试中也有可能会出现。

## final使用
### final变量
final变量有成员变量或者是本地变量(方法内的局部变量)，在类成员中final经常和static一起使用，作为类常量使用。**其中类常量必须在声明时初始化，final成员常量可以在构造函数初始化。**

```java
public class Main {
    public static final int i; //报错，必须初始化 因为常量在常量池中就存在了，调用时不需要类的初始化，所以必须在声明时初始化
    public static final int j;
    Main() {
        i = 2;
        j = 3;
    }
}
```

就如上所说的，对于类常量，JVM会缓存在常量池中，在读取该变量时不会加载这个类。

```java

public class Main {
    public static final int i = 2;
    Main() {
        System.out.println("调用构造函数"); // 该方法不会调用
    }
    public static void main(String[] args) {
        System.out.println(Main.i);
    }
}
```
### final方法
final方法表示该方法不能被子类的方法重写，将方法声明为final，在编译的时候就已经静态绑定了，不需要在运行时动态绑定。final方法调用时使用的是invokespecial指令。 

```java
class PersonalLoan{
    public final String getName(){
        return"personal loan”;
    }
}
 
class CheapPersonalLoan extends PersonalLoan{
    @Override
    public final String getName(){
        return"cheap personal loan";//编译错误，无法被重载
    }
    
    public String test() {
        return getName(); //可以调用，因为是public方法
    }
}
```
### final类
final类不能被继承，final类中的方法默认也会是final类型的，java中的String类和Integer类都是final类型的。

```java
final class PersonalLoan{}
 
class CheapPersonalLoan extends PersonalLoan {  //编译错误，无法被继承 
}
```

### final关键字的知识点
1. final成员变量必须在声明的时候初始化或者在构造器中初始化，否则就会报编译错误。final变量一旦被初始化后不能再次赋值。
2. 本地变量必须在声明时赋值。 因为没有初始化的过程
3. 在匿名类中所有变量都必须是final变量。
4. final方法不能被重写, final类不能被继承
5. 接口中声明的所有变量本身是final的。类似于匿名类
6. final和abstract这两个关键字是反相关的，final类就不可能是abstract的。
7. final方法在编译阶段绑定，称为静态绑定(static binding)。
8. 将类、方法、变量声明为final能够提高性能，这样JVM就有机会进行估计，然后优化。

final方法的好处:

1. 提高了性能，JVM在常量池中会缓存final变量
2. final变量在多线程中并发安全，无需额外的同步开销
3. final方法是静态编译的，提高了调用速度
4. **final类创建的对象是只可读的，在多线程可以安全共享**

## 从java内存模型中理解final关键字

java内存模型对final域遵守如下两个重拍序规则 

1. 初次读一个包含final域的对象的引用和随后初次写这个final域，不能重拍序。
2. 在构造函数内对final域写入，随后将构造函数的引用赋值给一个引用变量，操作不能重排序。

**以上两个规则就限制了final域的初始化必须在构造函数内，不能重拍序到构造函数之外，普通变量可以。**

具体的操作是

1. java内存模型在final域写入和构造函数返回之前，插入一个StoreStore内存屏障，静止处理器将final域重拍序到构造函数之外。
2. java内存模型在初次读final域的对象和读对象内final域之间插入一个LoadLoad内存屏障。

new一个对象至少有以下3个步骤

1. 在堆中申请一块内存空间
2. 对象进行初始化
3. 将内存空间的引用赋值给一个引用变量，可以理解为调用invokespecial指令

**普通成员变量在初始化时可以重排序为1-3-2，即被重拍序到构造函数之外去了。 final变量在初始化必须为1-2-3。**

### 读写final域重拍序规则

```java
public class FinalExample {
    int i;               
    final int j;
    static FinalExample obj;

    public void FinalExample () {
        i = 1;                   // 1
        j = 2;                   // 2
    }

    public static void writer () {  //写线程A  
        obj = new FinalExample ();  // 3
    }

    public static void reader () {       //读线程B执行
        if(obj != null) {               //4
            int a = object.i;           //5
            int b = object.j;           //6
        }
    }
}
```

我们可以用happens-before来分析可见性。结果是保证a读取到的值可能为0，或者1 而b读取的值一定为2。
首先，**由final的重拍序规则决定3HB2**，但是3和1不存在HB关系，原因在上面说过了。 因为线程B在线程A之后执行，所以3HB4。
那么2和4的HB关系怎么确定?? **final的重拍序规则规定final的赋值必须在构造函数的return之前**。所以2HB4。因为在一个线程内4HB6.所以可以得出结论2HB5。则b一定能得到j的最新值。而a就不一定了，因为没有HB关系，可以读到任意值。

HB判断可见性关系真是太方便了。可以参考我的另外一个博客http://medesqure.top/2018/08/25/happen-before/

可能发生的执行时序如下所示。
![](http://pbhb4py13.bkt.clouddn.com/2018-08-31-F06EA061-418C-4952-959A-D3A8CAE347C6.jpg)

### final对象是引用类型
如果final域是一个引用类型，比如引用的是一个int类型的数组。对于引用类型，写final域的重拍序规则增加了如下的约束

1. 在构造函数内对一个final引用的对象的成员域的写入和随后在构造函数外将被构造对象的引用赋值给引用变量之间不能重拍序。 即先写int[]数组的内容，再将引用抛出去。

```java
public class FinalReferenceExample {
    final int[] intArray;                     //final是引用类型
    static FinalReferenceExample obj;
    
    public FinalReferenceExample () {        //构造函数  在构造函数中不能被重排序 final类型在声明或者在构造函数中要赋值。
        intArray = new int[1];              //1
        intArray[0] = 1;                   //2
    }
    
    public static void writerOne () {          //写线程A执行
        obj = new FinalReferenceExample ();  //3
    }
    
    public static void writerTwo () {          //写线程B执行
        obj.intArray[0] = 2;                 //4
    }
    
    public static void reader () {              //读线程C执行
        if (obj != null) {                    //5
            int temp1 = obj.intArray[0];       //6
        }
    }
}
```

JMM保证了3和2之间的有序性。同样可以使用HB原则去分析，这里就不分析了。执行顺序如下所示。
![6DBA7734-EFF8-4AC2-8E3B-E1645889A109](http://pbhb4py13.bkt.clouddn.com/2018-08-31-6DBA7734-EFF8-4AC2-8E3B-E1645889A109.png)

## final引用不能从构造函数“逸出”

**JMM对final域的重拍序规则保证了能安全读取final域时已经在构造函数中被正确的初始化了。**
但是如果在构造函数内将被构造函数的引用为其他线程可见，那么久存在对象引用在构造函数中逸出，final的可见性就不能保证。 其实理解起来很简单，**就是在其他线程的角度去观察另一个线程的指令其实是重拍序的。** 

```java
public class FinalReferenceEscapeExample {
    final int i;
    static FinalReferenceEscapeExample obj;
    
    public FinalReferenceEscapeExample () {
        i = 1;       //1写final域
        obj = this;  //2 this引用在此“逸出”  因为obj不是final类型的，所以不用遵守可见性  }
    
    public static void writer() {
        new FinalReferenceEscapeExample ();
    }

    public static void reader {
        if (obj != null) {                     //3
            int temp = obj.i;                 //4
        }
    }
}
```
操作1的和操作2可能被重拍序。在其他线程观察时就会访问到未被初始化的变量i，可能的执行顺序如图所示。
![AAF34760-7112-463C-852F-25CB775AFD62](http://pbhb4py13.bkt.clouddn.com/2018-08-31-AAF34760-7112-463C-852F-25CB775AFD62.png)

本文结束，欢迎阅读。
本人博客 http://medesqure.top/ 欢迎观看















