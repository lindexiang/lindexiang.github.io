---
title: 深入理解JVM虚拟机 第7章 虚拟机的类加载机制
date: 2018-07-12 01:03:01
tags: 
  - JVM原理 
  - 虚拟机类加载机制
categories: JVM虚拟机原理
image: http://pbhb4py13.bkt.clouddn.com/damon-lam-774304-unsplash.jpg

---

# 虚拟机类加载机制

>虚拟机类加载机制虚拟机把描述类的数据从class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型。Java语言里，类型的加载和连接过程是在程序运行期间完成的。

## 类加载的时机
![C03B21D5-7E81-419C-9C63-76CCE16DE2A8](http://pbhb4py13.bkt.clouddn.com/C03B21D5-7E81-419C-9C63-76CCE16DE2A8.png)

类加载的生命周期：
1. 加载
2. 验证
3. 准备
4. 解析
5. 初始化
6. 使用
7. 卸载
**加载，验证，准备，初始化，卸载的顺序是确定的，为了支持java的动态绑定，解析过程可以在初始化之后调用，即动态绑定或者称为晚期绑定**
### 4种必须对类进行“初始化”的情况
- new，getstatic，putstatic，invokestatic这4个字节码，对类没有初始化必须先对类进行初始化，即使用new生成对象，读取或者设置类的static变量，final修饰的static变量在编译器把结果放在常量池了除外，调用类的static方法
- 使用java.lang.reflect包的方法对类进行反射调用
- 初始化一个类时，其父类未初始化，先触发父类的初始化过程
- main方法包含的类要先初始化

### 被动引用:
* 通过子类调用父类的静态字段不会导致子类的初始化(对于静态字段，只有直接定义这个字段的类才会被初始化)
* 通过数组定义应用类 classA[] array = new classA[10]；不会初始化classA，只有该数组去访问classA对象的成员时才会加载类
* 常量会在编译期间存入调用类的常量池 final字段


```java
//对于static字段，只有直接定义这个字段的类会被加载
class A ｛
     public static int i = 2;
｝
class B extends A{
     public static void main(String[] args) {
        System.out.println(B.i);
     }   
}
//数组不会触发类的加载,只有访问i才会加载
class A ｛
     public static int i = 2;
｝
class B {
    public static void main(String[] args) {
        A[] a = new A[10];
    }
}
//final字段是不会触发A类的加载，会触发B的加载，因为i会被转化成B对自身常量池的引用
class A ｛
     public static final int i = 2;
｝
class B {
     public static void main(String[] args) {
        System.out.println(A.i);
     }   
}
```

### java方法的绑定
java方法的调用需要先将方法和调用方法的类绑定起来，绑定分为静态绑定和动态绑定：

1. 静态绑定：即前期绑定。在程序执行前方法已经被绑定，此时由编译器或其它连接程序实现。针对java，简单的可以理解为程序编译期的绑定。java当中的方法只有final，static，private和构造方法是前期绑定的。
2. 动态绑定：即晚期绑定，也叫运行时绑定。在运行时根据具体对象的类型进行绑定。在java中，几乎所有的方法都是后期绑定的。

## 加载
加载是“类加载”的一个过程。加载过程主要完成3件事情：
1. 通过类的全限定名来获取定义类的二进制字节流
2. 将字节流表示的静态存储结果转化成方法区中的运行时数据结构
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据结构的访问入口

### 类加载器
对任何一个类，需要由它的类加载器和本身的class文件来确定在虚拟机中的唯一性。即一个class文件用两个类加载器加载出来的类是不想等的。equals()，isInstance 等方法的返回结果不同,使用instanceof的返回结果也不同。

```java
public class Main {

    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {

        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1)+ ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null)
                        return super.loadClass(name);
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }

            }
        };
        Object obj = myLoader.loadClass("Main").newInstance();
        System.out.println(obj.getClass()); //class Main
        System.out.println(obj instanceof Main); //false
        Main main = Main.class.newInstance();
        System.out.println(main.getClass().isInstance(obj)); //false
    }
}
```
上述的两个对象一个是使用应用程序类加载器加载的，一个是自定义的类加载器加载，虽然来自同一个clas文件，但是属于两个不同的类。

#### 双亲委派模型
 ![-c](http://pbhb4py13.bkt.clouddn.com/15310663051213.jpg)
 类加载器的层次图如图所示 ,称为**类加载器的双亲委派模型**, 双亲委派模型要求顶层的启动类加载器外，其余的类均要有自己的父类加载器，而类加载器不是以继承关系实现，而是使用组合的方式来加载父加载器。
 
 ```java
    @CallerSensitive
    public ClassLoader getClassLoader() {
        ClassLoader cl = getClassLoader0(); //获取自身classloader
        if (cl == null)
            return null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // caller can be null if the VM is requesting it
            ClassLoader ccl = getClassLoader(caller);
            //isAncestor方法将类加载器派给了引导类加载器
            if(cc1 != null && cc1 != c1 && !c1.iaAncestor(cc1)) {
                sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);
            }
        }
        return cl;
    }
    //判断c1是不是this的父类加载器 同时c1的引用也会改变，变成c1的parent
    boolean isAncestor(ClassLoader cl) {
        ClassLoader acl = this;
        do {
            acl = acl.parent;
            if (cl == acl) {
                return true;
            }
        } while (acl != null);
        return false;
    }

 ```
1.  启动类加载器 BootStrap ClassLoader
    用c++编写，该加载器加载java_home/lib的库文件，将库类加载器到虚拟机内存中。启动类加载器无法被java程序使用
2. 扩展类加载器 extension classLoader
    该加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载JDK\jre\lib\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。
3. 应用程序类加载器：Application ClassLoader，
    该类加载器由sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

应用程序由这三种类加载器配合加载，我们也可以自定义类加载器。因为JVM自带的类加载器只是从本地的文件系统中加载标准的java class文件，使用自己编写的classLoader，可以
1. 执行非致信代码前，自动检验数字签名
2. 动态创建自定义的类
3. 从特定的场所取得java class，比如数据库和网络IO中

#### 双亲委派模型的工作流程
一个类加载器收到类加载的请求，不会自己先尝试加载这个类，而是将请求委托类父加载器区完成，所有的类加载器都会传递到顶层的启动类加载器区加载，当父加载器无法找到需要的类时，自加载器才会尝试自己去加载这个类。
使用该模型去组织类加载器的好处是java类带有层次性，比如类java.lang.Object存放在JDK\jre\lib下的rt.jar，即(java_home\lib)路径下，最终都会使用启动类加载器区加载，保证了Object类在各个类及载器中都是同一个类。
  

## 类加载的验证阶段
验证：确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。在idea中的报错的东西都是在这个阶段检查的
虚拟机规范：如果验证到输入的字节流不符合Class文件的存储格式，就抛出一个java.lang.VerifyError异常或其子类异常

## 类加载的准备阶段
 准备阶段是正式为类变量分配内存并设置类变量初始值的阶段**类变量是static变量，不包括实例变量**，这些变量所使用的内存都在方法区中分配。**初始值一般指零值**
 `public static int value = 123` 
 在准备阶段后value的值为0，而不是123，在类的初始化阶段才会赋值为123。
 `public static final int value = 123`
 final类型的变量在准备阶段就会初始化成指定的值，存在运行时常量池中。 ![-c](http://pbhb4py13.bkt.clouddn.com/F8F341CD-91EF-4A78-A9BD-F71B50F78C0C.png)
 注意点：
1.  对于基本数据类型，static变量和成员变量没有显示赋值会使用默认值，但是局部变量在使用前必须显示赋值，否则编译不通过
2. static final 类型的变量在申明时必须显示赋值，否则编译不通过。final类型的变量在申明时赋值，或者类初始化后赋值，总之final类型的变量必须显示赋值。static类型的变量在准备阶段会赋零值
3. 对于reference，数组引用，对象引用，没有显示赋值而直接使用，系统会赋null 数组中的元素没有赋值，会使用零值。

```java
public class main1 {
    public static int i;
    public  final int ii; //final可以在构造函数内初始化
    public main1() {
        ii = 1;
    }
    public static final int iii; //必须显示初始化，因为static会在准备阶段赋零值
    static {
        iii = 1; //这里赋值也可以，因为都是在准备阶段执行
    }
}
```
## 类加载的解析阶段
解析阶段可能在初始化之后，阶段不固定。
解析阶段就是将加载的类中常量池里的符号引用转化成直接引用的过程。
* 符号引用
符号引用是用符号来表示引用的对象，只要符号可以无歧义的定位到目标对象即可。目标对象可以在内存中还不存在。
* 直接引用
直接引用时可以直接指向目标中的指针，相对偏移量或者能间接定位到目标的句柄。直接引用的目标在内存中必须存在。
**A.f1(),符号引用指的是方法区中的偏移量，直接引用指的是直接指向类的方法的入口地址，f1()具体的方法地址**

* 类的解析
当前的类为D，将一个符号引用N解析为类或接口C的直接引用，
1 C N = new D() 如果C不是数组类型，那么会将N的权限定名给D，用D的类加载器去加载。
2 C[] N = new D[100] C时数组类型，不会去启动D的类加载器去加载，但是虚拟机会生成一个表示这个数组的对象。
* 字段解析
对字段表中的class_index的索引中的CONSTANT_Class_info符号引用进行解析，也就是字段所属的类或者接口的符号引用。比如在方法里调用了C.a 将字段所属的类定义为C
如果C本身存在字段a，直接返回a的直接引用。
如果C实现了借口，按照继承关系从下往上递归搜索各个接口和父接口，找到字段a的直接引用。
如果C不是Object，则从下往上递归搜索父类中的字段，直到找到a
都没有找到，抛出异常
* 类方法解析 C.a()解析
先在类方法表中的索引的方法所属的类或者接口的符号引用。在C中找到a的直接引用。否则在C的父类中找到和这个方法的直接引用，查找结束。否则在C的接口中找到这个方法的引用，如果存在，说明C是抽象类，则查找结束，抛出异常。
* 接口方法解析 c.a() c是一个接口 
则先查c自身的接口，没有就查父接口 ，直到查到a的直接引用。
## 类加载的初始化阶段
类初始化阶段是类加载过程的最后一步，在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源。 
在初始化阶段实际是执行类构造器的方法`<clinit>()`方法，
1. `<clinit>()`会由编译器自动收集类中的类变量的赋值动作和静态语句块(static{}块)语句合并，并且顺序由原文件的顺序决定。**静态语句块中只能给访问到静态语句块之前的变量，在它之后的变量可以访问，但是不能赋值**

```java
public class Test {
    static {
        i = 0;                       //给变量赋值可以正常编译通过
        System.out.print(i);         //编译器会提示“非法向前引用”
        }
    static int i = 1;
}
```

2. `<clinit>()`和类的实例构造器不同`<init>()`，它不需要显示调用父类的构造器，虚拟机会保证在子类初始化之前父类已经初始化完毕。因此虚拟机中第一个被执行`<init>()`一定是java.lang.Object类。
3. 父类的static代码块一定会优先于子类的static代码块先执行。
4.  `<clinit>()`方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。
5.  接口中不能使用静态语句块，但仍然有变量初始化的操作，因此接口与类一样都会生成`<clinit>()`方法，但与类不同的是，执行接口的初始化方法之前，不需要先执行父接口的初始化方法。只有当父接口中定义的变量使用时，才会执行父接口的初始化方法。另外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。
6. 虚拟机会保证一个类的`<clinit>()`方法在多线程环境中被正确的加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的clinit()方法，其他线程都需要阻塞等待，直到活动线程执行类初始化方法完毕。

很简单，下面代码执行的结果为2，而不是1

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
        }
    }
static class Sub extends Parent {
    public static int B = A;
    }

public static void main(String[] args) {
    System.out.println(Sub.B);
    }
```



 





