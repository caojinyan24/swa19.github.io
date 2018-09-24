---
layout: post
title: "JAVA之内存溢出的排查和解决"
date: 2017-09-26 09:07:03 +0800
categories: 基础
tags: java
---

* TOC
{:toc}


# 问题介绍
最近做项目，遇到了内存溢出的问题：`java.lang.OutOfMemoryError: PermGen space`，第一次遇到这种问题，总结下问题排查方法。  

## OutOfMemoryError问题排查  

内存泄露的产生是因为申请的内存空间一直得不到释放，最终导致无法分配更多的内存空间。jvm有自己的内存回收的机制，当一个对象不存在指向它的引用时，这个对象所在的内存空间会被标记，在某种条件下，对这个对象的内存进行回收。  
那么当一个对象已经不再使用，但它的引用依旧被其他对象持有的时候就容易发生内存泄露。  
jdk提供了一个很好的可视化工具，在jkd的bin目录下有一个可执行程序：jvisualvm（windows系统双击打开，ubuntu系统在控制台中执行:`./jvisualvm`）  
打开后左侧显示目前运行的java程序，双击打开一个java应用，在Monitor中显示了各个系统参数，观察对应的内存区域所占用的空间是否在不断上升。如果目前占用的内存空间和已经达到最大内存空间，首先尝试提高最大内存配置，根据开发环境的不同，配置方式会存在区别。
![](/_pic/201710/visualvm.png)
如果已经提高了最大内存配置，而应用所占用的空间依旧在上升，这时候要排查下是否存在内存泄露的情况。  
查找内存泄露有很多方法，目前也存在一些成熟的商业软件。我简单总结下使用jdk自带的几个工具排查。  

* 通过java命令行查看  

看永久代的内存使用情况  

`jstat -gcpermcapactity <pid> `

> 当使用jdk6启动项目时，报的异常信息为PermGen space不足。但是在jdk8中，原有的PermGen space已经被metaspace内存空间所取代，相应的jstat命令也不再存在`-gcpermcapactity`的选项,取而代之的是`-gcmetacapacity`,这个不再详细写了，具体参考：![永久代(PermGen)和元空间(Metaspace)](http://www.cnblogs.com/paddix/p/5309550.html)
> jstat是一个很强大的用来查看jvm参数的工具，它利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控。其他jdk自带的工具还有 jstack, jconsole, jinfo, jmap, jdb，后边可以再详细写一个总结。

查看内存空间的分配情况  

`jmap -histo <pid>`

* 通过jvisualvm可视化工具查看

在jvisualvm的Profile标签页中，展示了内存的动态分配情况，如果发生了内存泄露，那么泄露的对象所占用的内存空间会不断提高的，通过这个工具可以很方便地看到内存的分配情况。  

## PermGen Space 相关

首先明确下PermGen space用来存放什么信息，PermGen space其实指的就是方法区。不过方法区和“PermGen space”又有着本质的区别。前者是 JVM 的规范，而后者则是 JVM 规范的一种实现，并且只有 HotSpot 才有 “PermGen space”，而对于其他类型的虚拟机，如 JRockit（Oracle）、J9（IBM） 并没有“PermGen space”。由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久代的内存溢出用来存放类相关的信息。在编译时，类加载器从class文件中，提取出类的相关信息，并存储在方法区中。

在实例化一个类的对象时，jvm将会从方法区中获取这个类的相关信息，并完成初始化工作。方法区的内存也是可以进行回收的，当某个类不再被使用时，jvm将卸载这个类，进行垃圾回收  
和方法区相关的jvm参数有两个，-XX:PermSize和-XX:MaxPermSize
如果使用tomcat启动服务，可以修改Tomcat的配置文件catalina.sh文件，添加配置：`JAVA_OPTS="$JAVA_OPTS -Xms512m -Xmx1024m -XX:PermSize=256M -XX:MaxPermSize=512m"`

## 问题解决过程

出现“java.lang.OutOfMemoryError: PermGen space”的问题时，启动jvisualvm，查看PemGen的内存展示显示最大80M，已使用80M，于是首先尝试修改最大PermSpace的大小。我的开发环境是IDEA+Tomcat，首先按照网上的资料修改Tomcat安装目录下的bin/catalina.sh文件，添加配置：`JAVA_OPTS="$JAVA_OPTS -Xms512m -Xmx1024m -XX:PermSize=256M -XX:MaxPermSize=512m"`，但是配置并未生效，在visualvm中显示最大内存依旧是80M。  
其实可以理解，因为IDEA在运行Web应用时，每个应用会新建一个Tomcat实例（实例配置在IDEA的安装目录下），所以修改Tomcat的配置，对启动的Web服务是不会生效的。这点通过IDEA启动Web服务的日志中也可以看到，

![](/_pic/201710/log.PNG)

其中显示的CATALINA_BASE目录在IDEA的配置路径下，我在对应的目录下并没有找到catalina.sh文件，好在IDEA的应用配置中有一个“VM options”，配置后，visualvm中显示的最大内存得到了更新。重启应用后，内存不足的问题消失。

![](/_pic/201710/vm.PNG)

这个内存不足问题不是由于内存泄露引起的，没有机会能够做进一步的实际排查。
