---
layout: post
title:  "jvm内存相关及问题排查"
date:   2018-04-12 22:49:48 +0800
categories: 基础
tags: java
---


* TOC
{:toc}

# 问题排查

前几天一个项目代码报Heap OutOfMemory错误，排查了下代码没有发现问题，没办法只能采用最笨的删代码方式排查，经过艰难的排查，最后定位到了出问题的地方是一个一个第三方提供的工具类，用来做加解密。   

~~~
private static byte[] encrypt(RSAPublicKey publicKey, byte[] plainTextData) throws Exception {
    if (publicKey == null) {
        throw new Exception("加密公钥为空, 请设置");
    } else {
        try {
            Cipher cipher = Cipher.getInstance("RSA", new BouncyCastlededeProvider());
            cipher.init(1, publicKey);
            byte[] output = cipher.doFinal(plainTextData);
            return output;
        } catch (NoSuchAlgorithmException var4) {
            throw new Exception("无此加密算法");
        } catch (NoSuchPaddingException var5) {
            return null;
        } catch (InvalidKeyException var6) {
            throw new Exception("加密公钥非法,请检查");
        } catch (IllegalBlockSizeException var7) {
            throw new Exception("明文长度非法");
        } catch (BadPaddingException var8) {
            throw new Exception("明文数据已损坏");
        } catch (Exception var9) {
            var9.printStackTrace();
            return null;
        }
    }
}
~~~

其中的`BouncyCastlededeProvider`来自   

~~~
<dependency>
<group>org.bouncycastle</group>
<artifact>bcprov-jdk15on</artifact>
<version>1.56<version>
~~~

~~~
public BouncyCastleProvider() {
        super("BC", 1.56D, info);
        AccessController.doPrivileged(new PrivilegedAction() {
            public Object run() {
                BouncyCastleProvider.this.setup();
                return null;
            }
        });
    }  
~~~

乍一看代码似乎没问题，毕竟一般印象中，局部的临时变量会自动被回收掉的。此时发现网络上也存在过相同加解密的内存泄露问题，但其中提出的解释细究起来的话并不经得起推敲。    
于是再次排查代码，知道是哪个对象没有被回收之后，排查起来就有目的得多了。    

~~~
//javax.crypto.JceSecurity类
private static final Map<Provider, Object> verificationResults = new IdentityHashMap();

static synchronized Exception getVerificationResult(Provider var0) {
        Object var1 = verificationResults.get(var0);
        if (var1 == PROVIDER_VERIFIED) {
            return null;
        } else if (var1 != null) {
            return (Exception)var1;
        } else if (verifyingProviders.get(var0) != null) {
            return new NoSuchProviderException("Recursion during verification");
        } else {
            Exception var3;
            try {
                verifyingProviders.put(var0, Boolean.FALSE);
                URL var2 = getCodeBase(var0.getClass());
                verifyProviderJar(var2);
                verificationResults.put(var0, PROVIDER_VERIFIED);
                var3 = null;
                return var3;
            } catch (Exception var7) {
                verificationResults.put(var0, var7);
                var3 = var7;
            } finally {
                verifyingProviders.remove(var0);
            }
            return var3;
        }
    }
~~~

出现内存泄露的原因在于Cipher类的273行getInstance方法中`Exception var9 = JceSecurity.getVerificationResult(var1);`会在`verificationResults`中添加一个Provider实例的引用。那么由于在方法调用时，每次都是新建一个`BouncyCastleProvider`对象，而这个对象由于被一个静态的Map引用，这个Map不会被回收，相应的它持有的对象也无法被内存回收，也由此导致内存占用不断增加，最终导致OOM。

最后总结下这次排查过程中涉及到的小知识点。

***

# jvm内存配置

`-Xmx`：JAVA HEAP的最大值、默认为物理内存的1/4    
`-Xms`：JAVA HEAP的初始值，server端最好Xms与Xmx一样    
`-Xmn`：JAVA HEAP young区的大小    
`-XX:MetaspaceSize`：元空间的初始值    
`-XX:MaxMetaspaceSize`：元空间的最大值  

设置jvm参数    

~~~
-Xms64m -Xmx64m -XX:NewRatio=1 -XX:MaxMetaspaceSize=128m -XX:MetaspaceSize=128m -Xverify:none  -XX:MaxTenuringThreshold=7 -XX:GCTimeRatio=19 -XX:+DisableExplicitGC -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=70 -Ddubbo.shutdown.hook=true   -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -XX:+PrintGCApplicationStoppedTime
~~~


# jvm内存溢出与内存泄露

内存泄露：使用资源后未及时释放，导致可用的内存空间越来越少，最终会导致OutOfMemory异常
内存溢出：需要申请的资源超出了内存的大小；如果排查不存在内存泄露的情况的话，增加jvm的内存大小就可以解决这个问题。


# jvm内存溢出排查    

## top：查看当前进程的占用情况

通过Top命令查看到当前占用内存最大的进程ID

~~~
top - 19:02:54 up  8:39,  1 user,  load average: 1.07, 1.40, 1.52
Tasks: 265 total,   1 running, 263 sleeping,   0 stopped,   1 zombie
%Cpu(s): 17.6 us, 11.8 sy,  0.0 ni, 69.5 id,  1.0 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem :  8055528 total,   200152 free,  5935260 used,  1920116 buff/cache
KiB Swap:  8268796 total,  6317248 free,  1951548 used.  1044616 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                             
 2540 jinyan    20   0 3451680 636596 596264 S  33.1  7.9  97:19.71 VirtualBox                                                                                                          
 3375 jinyan    20   0 2114904 363400 172636 S  30.8  4.5   4:22.18 Web Content                                                                                                         
 1275 root      20   0  812292 176828 132744 S  11.9  2.2  48:02.33 Xorg                                                                                                                
 2168 jinyan    20   0 1577936  69508  31128 S  10.9  0.9  40:07.43 compiz                                                                                                              
29065 jinyan    20   0 1380620 181444  38224 S   8.9  2.3  61:25.38 chrome                                                                                                              
 8164 jinyan    20   0 5177944 1.864g  17408 S   7.9 24.3 144:10.11 java                                                                                                                
15841 jinyan    20   0  604880  19592  11132 S   5.0  0.2   0:19.73 gnome-terminal-                                                                                                     
 3291 jinyan    20   0 2465916 365664 196344 S   4.0  4.5   2:29.34 firefox                                                                                                             
 1227 root      20   0  259140   2092   1912 S   1.7  0.0   7:55.14 ECAgent                                                                                                             
 8628 jinyan    20   0 5680828 418008   6016 S   1.3  5.2   2:30.30 java                                                                                                                
 2496 jinyan    20   0  830468   9772   7188 S   1.0  0.1   4:40.63 VBoxSVC                                                                                                             
 2491 jinyan    20   0  168348   2252   2016 S   0.7  0.0   2:13.66 VBoxXPCOMIPCD                                                                                                       
 3332 jinyan    20   0 1833268  96644  32300 S   0.7  1.2   5:43.61 atom                                                                                                                
 3465 jinyan    20   0 1887036 188088 118676 S   0.7  2.3   1:17.25 Web Content                                                                                                         
 4461 jinyan    20   0 1418752 152312  42148 S   0.7  1.9   2:24.72 atom                                                                                                                
 9135 jinyan    20   0   43676   3828   3076 R   0.7  0.0   0:00.11 top   
~~~

分析：    
上边几行比较容易理解，看下下边的几个变量
PR：优先级    
VIRT：进程使用的虚拟内存总量    
RES：进程使用的、未被换出的物理内存大小    
SHR：共享内存大小    
S：进程状态（D不可中断的睡眠状态；R运行；S睡眠；T跟踪/停止；Z：僵尸进程）    
%CPU：上次更新到现在的CPU时间占用百分比    
%MEM：进程使用的物理内存百分比    
TIME+：进程使用的CPU时间总计    
COMMAND：命令名/命令行    

[TOP命令参考地址](http://www.jb51.net/LINUXjishu/34604.html)


## jmap命令   

查看指定进程的jvm物理内存的占用情况     
`jmap -histo:live ${pid} `：  查看当前内存中的类实例及个数    
`jps`：查看本地的java进程

## jstat

`jstat -gcutil pid 1000` 每1000ms打印一次各个内存分区的占用情况

[jstat详细使用](http://www.51testing.com/html/92/77492-203728.html)

## jdk提供的可视化的两个工具    

`jconsole`，`jvisualvm`：这两个工具就我的使用来看，功能差不多，我习惯用jvisualvm，可以查看各项jvm参数，在排查内存泄露问题的时候，可以动态查看各种类的实例个数和内存占用情况。

# jvm内存结构

![](/_pic/201804/jvm-memory.png)






## 延伸学习

[面向GC的Java编程](http://ifeve.com/gc-oriented-java-programming/)    
[浅析Java虚拟机结构与机制](http://blog.hesey.net/2011/04/introduction-to-java-virtual-machine.html)    
[JVM初探 -JVM内存模型](https://blog.csdn.net/zjf280441589/article/details/53437703)
