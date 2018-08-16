---
title: threadLocal
images: /images/摘要配图/
date: 2018-08-17 00:06:03
tags:
---

# ThreadLocal 变量源码解析
> ThreadLocal是线程的局部变量，是每一个线程所独有的，其他线程无法对其访问，通常是类的private static变量



```java
public class ThreadlocalTest {

    public static void main(String[] args) {
        ThreadlocalTest threadlocalTest = new ThreadlocalTest();
        threadlocalTest.setLong(1l);
        threadlocalTest.setString("1-str");
        System.out.println("==main1==" + threadlocalTest.getLong() + " " + threadlocalTest.getString());

        new Thread(new Runnable() {
            @Override
            public void run() {
                threadlocalTest.setLong(2l);
                threadlocalTest.setString("2-str");
                System.out.println("==thread1==" + threadlocalTest.getLong() + " " + threadlocalTest.getString());
            }
        }).start();

        threadlocalTest.setLong(3l);
        threadlocalTest.setString("3-str");
        System.out.println("==main1==" + threadlocalTest.getLong() + " " + threadlocalTest.getString());
    }

    ThreadLocal<Long> longLocal = new ThreadLocal<>();
    ThreadLocal<String> stringLocal = new ThreadLocal<>();
    public void setLong(Long i) {
        longLocal.set(i);
    }

    public Long getLong() {
        return longLocal.get();
    }

    public void setString(String i) {
        stringLocal.set(i);
    }

    public String getString() {
        return stringLocal.get();
    }
}

//运行结果
==main1==1 1-str
==thread1==2 2-str
==main1==3 3-str
```




