---
layout: post
title:  "线程同步"
date:   2017-12-20 16:31:45 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

线程同步的几种方式

# volatile

当一个线程修改了变量的值，会立刻把这个变量同步到主内存中；而其他线程在读取这个变量的值时，会从主内存中获取变量

![](/_pic/201712/640.png)

保证变量在线程之间的可见性和禁止指令重排，但是需要注意的是，volatile对于非原子性的操作，并不能保证线程安全。volatile不能保证线程安全，所以，当对该变量的读写不依赖当前值的时候，才可以考虑使用volatile关键字来做线程同步。

volatile变量禁止了指令重排是怎么实现的？jvm为volatile变量做了内存屏障

1. 在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障
2. 在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障

内存屏障也称为内存栅栏或栅栏指令，是一种屏障指令，它使CPU或编译器对屏障指令之前和之后发出的内存操作执行一个排序约束。 这通常意味着在屏障之前发布的操作被保证在屏障之后发布的操作之前执行。

内存屏障共分为四种类型：

* LoadLoad屏障：

抽象场景：Load1; LoadLoad; Load2

Load1 和 Load2 代表两条读取指令。在Load2要读取的数据被访问前，保证Load1要读取的数据被读取完毕。

* StoreStore屏障：

抽象场景：Store1; StoreStore; Store2

Store1 和 Store2代表两条写入指令。在Store2写入执行前，保证Store1的写入操作对其它处理器可见

* LoadStore屏障：

抽象场景：Load1; LoadStore; Store2

在Store2被写入前，保证Load1要读取的数据被读取完毕。

* StoreLoad屏障：

抽象场景：Store1; StoreLoad; Load2

在Load2读取操作执行前，保证Store1的写入对所有处理器可见。StoreLoad屏障的开销是四种屏障中最大的。

对于volatile修饰的变量，更新时使用jdk的native方法，compareAndSet，保证了更新操作的原子性。

# ReentrantLock

把锁的实现作为一个java类，而不是语言特性来实现,通过这种方式可以提供更多的锁的特性。

非阻塞，当当前线程处于空闲状态，其他线程可重新获得锁。空闲状态的线程在重新获得锁之前会一直保持休眠状态。
对于已经获得锁的线程，可以多次获得锁

使用示例：

# synchronized

synchronized可以用来修饰代码块，普通方法，静态方法。其中代码块和普同方法添加对象锁，而静态方法添加类锁。

首先，java对象头中中保存了可用于同步的锁，如下是对象头的存储结构

![](/_pic/201712/sync.jpg)

随着程序的运行，对象头的状态会发生变化：

![](/_pic/201712/syc.jpg)

每一个线程会维护一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个Monitor关联，在对象头的LockWord中会保存指向Monitor的起始地址，同时Monitor中保存拥有这个锁的线程标识。

# 类锁和对象锁

根据synchronize修饰对象的不同，加的锁的类型也有所不同，如果加载static域或类上的话，是类锁，其他的是对象锁。

对于对象锁，同时只能有一个线程可以获取到这个对象的对象锁。

~~~
public class TestSync2 implements Runnable {
    int b = 100;
    synchronized void m1() throws InterruptedException {
        b = 1000;
        Thread.sleep(500); //6
        System.out.println("b=" + b);
    }
    synchronized void m2() throws InterruptedException {
        Thread.sleep(250); //5
        b = 2000;
    }
    public static void main(String[] args) throws InterruptedException {
        TestSync2 tt = new TestSync2();
        Thread t = new Thread(tt);  //1
        t.start(); //2
        tt.m2(); //3
        System.out.println("main thread b=" + tt.b); //4
    }
    @Override
    public void run() {
        try {
            m1();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
~~~

以上代码运行结果为：

~~~
main thread b=1000
b=1000
如果m1方法调度的线程启动较晚，也可能会出现结果：
main thread b=2000
b=1000
~~~

# 参考：
[volatile对指令重排的影响](https://mp.weixin.qq.com/s/g9J39yvNdPI2a5aXhMrv_Q)

[死磕 Java 并发：深入分析 synchronized 的实现原理 ](https://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651478216&idx=1&sn=0a78b71d5b80277f33d3ecfddd657e54&chksm=bd2534b78a52bda1df9f204f633a2c49069efe09dc3d1783888099d935d54e5617b38d9c6ebe&scene=21##)
