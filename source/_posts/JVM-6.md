---
title: 深入理解JVM虚拟机 第6章 类文件结构
date: 2018-07-27 09:32:51
tags: 虚拟机
---

# 深入理解Java虚拟机 第六章 类文件结构
## class类文件的结构
![-c](http://pbhb4py13.bkt.clouddn.com/15319300709181.jpg)


```java
ClassFile {
              u4             magic;
              u2             minor_version;
              u2             major_version;
              u2             constant_pool_count;
              cp_info        constant_pool[constant_pool_count-1]; //常量池，字面量和符号引用
              u2             access_flags; //访问标志
              u2             this_class; //全限定名
              u2             super_class; //父类全限定名
              u2             interfaces_count; //接口数量
              u2             interfaces[interfaces_count]; //接口的全限定名
              u2             fields_count;
              field_info     fields[fields_count]; //类或接口的字段
              u2             methods_count;
              method_info    methods[methods_count]; //方法表
              u2             attributes_count;
              attribute_info attributes[attributes_count]; //属性表，code，exception等
}
```

1. class文件是以8字节为基础的二进制流，各个数据都是按一定的顺序排列，如果需要占用8字节以上的空间数据，按照高位在前分割存储
2. class文件的存储结构只有2种，无符号数和表


* 无符号数 基本的数据类型，u1,u2,u4,u8类表示1，2，4，8个字节的无符号数，可以描述数字，索引引用，数字量，按UTF-8编码的字符串。
* 表 是由多个无符号数组成的复合数据类型，所有表以_info结尾，整个class就是一张表

## 6.2 魔数
Class文件的头4个字节，作用：确定这个文件是否为一个能被虚拟机接受的Class文件。值为：0xCAFEBABE(咖啡宝宝)
## 6.3 版本号
紧接着魔数的4个字节。第5，6字节是次版本号，第7，8字节是主版本号
## 6.4 常量池
常量池可以理解为class文件的资源仓库。主要存放了两大类常量：字面量和符号引用。
> 字面量 string类型的字面量和final类型常量
> 符号引用 包括3中类常量  1.类和接口的全限定名  2.字段的名称和描述符  3.方法的名称和描述符

java在虚拟机加载class文件时才会动态链接，class文件中不会保存各个方法，字段的最终布局，这些字段，方法的符号引用需要在运行时的转换才能得到真正的内存入口。在虚拟机运行时会从常量池中得到对应的符号引用，在类创建时解析。
常量池的14中常量结构 比如CONSTANT_CLASS_info,代表类或接口的符号引用，表中的name_index指向CONSTANT_UTF8_info类型的数据，这个存放着我们类的全限定名。
![](http://pbhb4py13.bkt.clouddn.com/15325344651412.jpg)

![](http://pbhb4py13.bkt.clouddn.com/15325344711713.jpg)

## 6.5访问标志
常量池结束后的两个字节是表示访问标志，用于识别类或者接口的访问信息，比如是类还是接口，访问类型，是否final等等。

##  6.6类索引，父类索引，接口索引集合
class文件由这三个数据来确定类的继承关系。类索引(this_class)和父类索引(super_class)是u2类型的数据，接口索引集合是u2类型的数据集合。
> 类索引可以确定类的全限定名，父类索引可以确定类的父类全限定名，除了Object类，其他类都有1个父类。接口索引集合按照implements顺序从左到右排列在集合索引中

## 6.7字段表集合
字段表描述类或者接口中申明的变量。字段包括类变量和实例变量，不包括方法内部的局部变量。字段表的格式如下所示，access_flags是字段的访问标志，public，可变性final，并发性violatile等等。其中name_index和descriptor_index是对常量池的引用，代表字段的简单名称和方法的描述符。
![](http://pbhb4py13.bkt.clouddn.com/15325352191240.jpg)

### 全限定名，简单名称，描述符的区别
1. 全限定名是`org/fenixsoft/clazz/TestClass`是类的全限定名
2. 简单名称 指没有类型和参数修饰符的方法或者字段名称 inc()方法和m字段的简单名称为inc 和m
3. 字段和方法的描述符的解释如下

字段的描述符如下所示
![](http://pbhb4py13.bkt.clouddn.com/15325355600922.jpg)
对于数组类型，每一维度用[表示，比如java.lang.string[][]的描述符为[[Ljava/lang/String, int[]的描述符为[I
描述符描述方法时按照先参数列表再返回值，比如void inc() 表示为()V 方法`java.lang.String toString()` 表示为 （）Ljava/lang/String ， 方法int indexOf(char[]source, int sourceOffset)可以表示为([CI)I。

## 6.8方法表集合
方法表和字段表类似，结构也是访问标志，名称索引，描述符索引，属性表集合等。
![](http://pbhb4py13.bkt.clouddn.com/15326196731134.jpg)

方法的定义在可以通过方法的访问标志，名称索引，描述符索引表达清楚，**方法内部的代码是经过java的编译器编译成字节码后存放在方法属性集合中的名为“code”的属性中**

在java语言中，重载(override)一个方法，除了和原方法的简单名称一样，还需要和原方法有一个不同的**特征签名**,特征签名是一个方法中不同参数在常量池中的字段符号引用的合集，所以返回值不会包含在特征签名中，所以**无法通过返回值来重载方法**

## 6.9 属性表
![](http://pbhb4py13.bkt.clouddn.com/15326201581064.jpg)

### code属性
**java代码经过javac编译后会变成字节码指令存储在code属性中**。code属性存放在方法表的属性中，但是不是所有的方法表中都存在code属性，比如接口和抽象类的方法就不存在code属性
![](http://pbhb4py13.bkt.clouddn.com/15326207069614.jpg)

max_stack是操作数栈深度的最大值
max_locals是局部变量所需的空间 
> max_locals的单位值slot，slot是虚拟机为局部变量分配内存的最小单位，除了double和long两者是需要2个slot存放，其他的都是1个slot来存放
> 方法参数包括this，异常处理的参数，即try catch中定义的异常，方法体中的局部变量都是用slot来存放。slot可以被复用，只要保证正确性。

code_length和code是存储的字节码
exception是方法中可能抛出的受查异常，也就是throws的异常
ConstantValue属性是为静态变量赋值

### 异常表
编译器是采用异常表而不是简单的跳转命令来实现java的异常和finally处理机制
![](http://pbhb4py13.bkt.clouddn.com/15326221650816.jpg)
看出0-9行是正常返回，10-20是exception型的异常返回，21-25是非exception得异常返回。3中路径都有finally中的代码，finally的代码是会嵌套在3种路径的代码之后在return之前。
结果是没有异常，返回1，出现exception异常，返回是2，出现exception以外的异常，方法没有返回值。

### 字节码指令简介
* 加载和存储指令
加载和存储指令将数据在栈帧的局部变量表和操作数栈之间传输
* 运算指令
将连个操作数栈的值进行运算，再将结果存回操作数栈顶
* 类型转换
转换类型，虚拟机直接支持从小范围类型向大范围类型的安全转换，大数到小数就要使用转换指令
* 对象创建和访问指令
new，newarray，访问类字段 getstatic 访问非类字段 getfield
* 方法调用指令
> invokevirtual 调用方法的虚方法，根据方法的实际类型进行分派
> invokeinterface 调用接口的方法，搜索实现接口方法的实例对象并找出最合适的方法调用
> invokespecial 调用特殊的方法，比如实力初始化方法和私有方法和父类方法
> invokespecial 类方法static
> invokedynamic 运行时动态解析出引用的方法。

### 同步指令
java虚拟机支持方法级的同步和方法内部指令的同步，两种的同步结构都是使用**管程(Monitor)**来支持。比如`synchronize`的语句实现是monitorenter和monitorexit来实现，执行的线程必须先成功持有管程，然后才能执行方法，方法最后成功或者非正常完成都会释放管程。**同步方法在执行时跑出异常，在内部无法处理异常，同步方法持有的管程在异常被抛到同步方法之外时也被自动释放**。





