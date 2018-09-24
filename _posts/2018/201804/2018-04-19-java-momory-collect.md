---
layout: post
title:  "Java实现系统内存监控及报警"
date:   2018-04-18 21:04:08 +0800
categories: 基础
tags: java
---


* TOC
{:toc}


自上次出现内存泄露的故障之后,review故障的时候说要加内存的监控,本来该运维做的时候交给了开发来做.    
百度之后找到了几个解决办法,其中一个是引用一个专门用来监控系统参数的第三方jar包:

~~~
<dependency>
    <groupId>org.hyperic</groupId>
    <artifactId>sigar</artifactId>
    <version>1.6.5.132-7</version>
</dependency>
~~~

但引入之后,发现这个jar包是jdk1.4,在jdk1.8的环境下已经不能支持.只能转而寻觅其他方案.

最后发现jdk自带的一个很方便的工具类`ManagementFactory`,下午尝试了下其中的各个接口,针对内存的监控的代码如下:


~~~
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.mail.javamail.JavaMailSenderImpl;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.mail.Authenticator;
import javax.mail.PasswordAuthentication;
import javax.mail.Session;
import javax.mail.internet.MimeMessage;
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;
import java.util.Properties;

/**
 * 每隔${INTERVAL}s收集应用占用jvm的堆内存和非堆内存,对一分钟的数据取平均值.
 * 若一分钟内的内存使用率/总内存>${ALERT_RATE},则发送邮件报警
 *
 * @author jinyan
 * @date 4/17/18 6:44 PM
 */
@Component
public class JavaMonitor {
    private static final Logger logger = LoggerFactory.getLogger(JavaMonitor.class);
    private JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
    private static final Integer INTERVAL = 10;
    private static final Integer MONITOR_INTERVAL = 60;
    private static final double ALERT_RATE = 0.75;

    @PostConstruct
    public void start() {
        mailSender.setHost("smtp.163.com");
        mailSender.setPort(25);
        mailSender.setUsername("**@163.com");
        mailSender.setPassword("**");
        mailSender.setDefaultEncoding("Utf-8");
        Properties p = new Properties();
        p.setProperty("mail.smtp.timeout", "25000");
        p.setProperty("mail.smtp.auth", "true");
        Authentication authentication = new Authentication("**@163.com", "**");
        Session session = Session.getDefaultInstance(p, authentication);

        mailSender.setJavaMailProperties(p);
        new Thread(new Runnable() {
            @Override
            public void run() {
                doMomitor();
            }
        }).start();
    }


    private void doMomitor() {
        while (true) {
            HeapBody heapBody = collect(MONITOR_INTERVAL / INTERVAL);
            if (heapBody.getHeapBytesRate() > ALERT_RATE) {
                sendMail("jinyan.cao@...com", "MemoryUsageAlert", "heapMomoryUsage over 75%,last minute value is " + heapBody.getHeapBytesRate());
            }
            if (heapBody.getNoneHeapBytesRate() > ALERT_RATE) {
                sendMail("jinyan.cao@...com", "MemoryUsageAlert", "noneHeapMomoryUsage over 75%,last minute value is " + heapBody.getNoneHeapBytesRate());
            }
        }
    }

    /**
     * 收集内存信息:10s内取一次统计
     */
    private HeapBody collect(Integer count) {
        HeapBody result = new HeapBody();
        Long heapBytes = 1L;
        Long noneHeapBytes = 1L;
        Long heapMaxBytes = 1L;
        Long noneHeapMaxBytes = 1L;
        try {
            for (int i = 0; i < count; i++) {
                MemoryMXBean memorymbean = ManagementFactory.getMemoryMXBean();
                MemoryUsage heapUsage = memorymbean.getHeapMemoryUsage();
                MemoryUsage nonHeapUsage = memorymbean.getNonHeapMemoryUsage();
                heapBytes += heapUsage.getUsed();
                heapMaxBytes += heapUsage.getMax();
                noneHeapBytes += nonHeapUsage.getUsed();
                noneHeapMaxBytes += nonHeapUsage.getMax();
                logger.info("{},{},{},{}", heapBytes, heapMaxBytes, noneHeapBytes, noneHeapMaxBytes);
                Thread.sleep(INTERVAL * 1000);
            }
            result.setHeapBytesRate(heapBytes.doubleValue() / heapMaxBytes.doubleValue());
            result.setNoneHeapBytesRate(noneHeapBytes.doubleValue() / noneHeapMaxBytes.doubleValue());
            logger.info("result:{}", result);
            return result;
        } catch (Exception e) {
            logger.error("collect error:", e);
            return result;
        }
    }

    private void sendMail(String to, String subject, String text) {
        try {

            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper messageHelper = new MimeMessageHelper(mimeMessage, true, "UTF-8");
            messageHelper.setFrom("caojinyanivy@163.com", "superviser");
            messageHelper.setTo(to);
            messageHelper.setSubject(subject);
            messageHelper.setText(text, true);
            mailSender.send(mimeMessage);
        } catch (Exception e) {
            logger.error("send error:", e);
        }
    }

    class Authentication extends Authenticator {
        String username = null;
        String password = null;

        public Authentication() {
        }

        public Authentication(String username, String password) {
            this.username = username;
            this.password = password;
        }

        @Override
        protected PasswordAuthentication getPasswordAuthentication() {
            PasswordAuthentication pa = new PasswordAuthentication(username, password);
            return pa;
        }
    }

    class HeapBody {
        double heapBytesRate;
        double noneHeapBytesRate;

        public double getHeapBytesRate() {
            return heapBytesRate;
        }

        public void setHeapBytesRate(double heapBytesRate) {
            this.heapBytesRate = heapBytesRate;
        }

        public double getNoneHeapBytesRate() {
            return noneHeapBytesRate;
        }

        public void setNoneHeapBytesRate(double noneHeapBytesRate) {
            this.noneHeapBytesRate = noneHeapBytesRate;
        }

        @Override
        public String toString() {
            final StringBuffer sb = new StringBuffer("HeapBody{");
            sb.append("heapBytesRate=").append(heapBytesRate);
            sb.append(", noneHeapBytesRate=").append(noneHeapBytesRate);
            sb.append('}');
            return sb.toString();
        }
    }
}

~~~

写了一个内存泄露的代码测试,运行日志为

~~~
2018-04-18 19:09:24 |-INFO  monitor.JavaMonitor - 41895017,63766529,60472657,1392508929
2018-04-18 19:09:34 |-INFO  monitor.JavaMonitor - 89117601,127533057,126750585,2785017857
2018-04-18 19:09:44 |-INFO  monitor.JavaMonitor - 145963825,191299585,193055585,4177526785
2018-04-18 19:09:54 |-INFO  monitor.JavaMonitor - 185836393,255066113,259156105,5570035713
2018-04-18 19:10:04 |-INFO  monitor.JavaMonitor - 237182081,318832641,325318865,6962544641
2018-04-18 19:10:14 |-INFO  monitor.JavaMonitor - 274110313,382599169,391491929,8355053569
2018-04-18 19:10:24 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.7164425205534097, noneHeapBytesRate=0.04685690232466779}
2018-04-18 19:10:24 |-INFO  monitor.JavaMonitor - 48212145,63766529,66200137,1392508929
2018-04-18 19:10:34 |-INFO  monitor.JavaMonitor - 107396129,127533057,132418257,2785017857
2018-04-18 19:10:44 |-INFO  monitor.JavaMonitor - 147937657,191299585,198655033,4177526785
2018-04-18 19:10:54 |-INFO  monitor.JavaMonitor - 199036393,255066113,264944577,5570035713
2018-04-18 19:11:04 |-INFO  monitor.JavaMonitor - 261287689,318832641,331291601,6962544641
2018-04-18 19:11:14 |-INFO  monitor.JavaMonitor - 309692433,382599169,397647417,8355053569
2018-04-18 19:11:24 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.80944355893256, noneHeapBytesRate=0.04759364062911611}
2018-04-18 19:11:25 |-INFO  monitor.JavaMonitor - 42179153,63766529,66942633,1392508929
2018-04-18 19:11:35 |-INFO  monitor.JavaMonitor - 99116017,127533057,133936945,2785017857
2018-04-18 19:11:45 |-INFO  monitor.JavaMonitor - 148418113,191299585,201010441,4177526785
2018-04-18 19:11:55 |-INFO  monitor.JavaMonitor - 190817097,255066113,268084897,5570035713
2018-04-18 19:12:05 |-INFO  monitor.JavaMonitor - 249932393,318832641,335177913,6962544641
2018-04-18 19:12:15 |-INFO  monitor.JavaMonitor - 304857849,382599169,402292881,8355053569
2018-04-18 19:12:25 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.7968073997567935, noneHeapBytesRate=0.04814964711807942}
2018-04-18 19:12:25 |-INFO  monitor.JavaMonitor - 53773257,63766529,67158121,1392508929
2018-04-18 19:12:35 |-INFO  monitor.JavaMonitor - 106946177,127533057,134344337,2785017857
2018-04-18 19:12:45 |-INFO  monitor.JavaMonitor - 162097257,191299585,201546425,4177526785
2018-04-18 19:12:55 |-INFO  monitor.JavaMonitor - 220515193,255066113,268752097,5570035713
2018-04-18 19:13:05 |-INFO  monitor.JavaMonitor - 283796233,318832641,335957769,6962544641
2018-04-18 19:13:15 |-INFO  monitor.JavaMonitor - 338285297,382599169,403206225,8355053569
2018-04-18 19:13:25 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.8841767688209485, noneHeapBytesRate=0.0482589634728409}
2018-04-18 19:13:26 |-INFO  monitor.JavaMonitor - 49881681,63766529,67225481,1392508929
2018-04-18 19:13:36 |-INFO  monitor.JavaMonitor - 110077289,127533057,134516657,2785017857
2018-04-18 19:13:46 |-INFO  monitor.JavaMonitor - 166334065,191299585,201858833,4177526785
2018-04-18 19:13:56 |-INFO  monitor.JavaMonitor - 222298665,255066113,269207809,5570035713
2018-04-18 19:14:06 |-INFO  monitor.JavaMonitor - 278796817,318832641,336561393,6962544641
2018-04-18 19:14:16 |-INFO  monitor.JavaMonitor - 337396065,382599169,403915105,8355053569
2018-04-18 19:14:26 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.8818525818596329, noneHeapBytesRate=0.048343807931843556}
2018-04-18 19:14:26 |-INFO  monitor.JavaMonitor - 57390489,63766529,67361585,1392508929
2018-04-18 19:14:36 |-INFO  monitor.JavaMonitor - 114686401,127533057,134725089,2785017857
2018-04-18 19:14:46 |-INFO  monitor.JavaMonitor - 176347073,191299585,202088593,4177526785
2018-04-18 19:14:56 |-INFO  monitor.JavaMonitor - 237309441,255066113,269452097,5570035713
2018-04-18 19:15:06 |-INFO  monitor.JavaMonitor - 300931529,318832641,336815601,6962544641
2018-04-18 19:15:16 |-INFO  monitor.JavaMonitor - 364255545,382599169,404191521,8355053569
2018-04-18 19:15:26 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.952055243486428, noneHeapBytesRate=0.04837689162157902}
2018-04-18 19:15:27 |-INFO  monitor.JavaMonitor - 63374305,63766529,67395505,1392508929
2018-04-18 19:15:37 |-INFO  monitor.JavaMonitor - 125096137,127533057,134812385,2785017857
2018-04-18 19:15:47 |-INFO  monitor.JavaMonitor - 186544569,191299585,202270737,4177526785
2018-04-18 19:15:57 |-INFO  monitor.JavaMonitor - 249520777,255066113,269794369,5570035713
2018-04-18 19:16:07 |-INFO  monitor.JavaMonitor - 312010649,318832641,337318001,6962544641
2018-04-18 19:16:17 |-INFO  monitor.JavaMonitor - 374398689,382599169,404843553,8355053569
2018-04-18 19:16:27 |-INFO  monitor.JavaMonitor - result:HeapBody{heapBytesRate=0.9785663935929771, noneHeapBytesRate=0.048454932054786924}
2018-04-18 19:16:27 |-INFO  monitor.JavaMonitor - 62916201,63766529,67529137,1392508929
2018-04-18 19:16:37 |-INFO  monitor.JavaMonitor - 126160449,127533057,135060065,2785017857
2018-04-18 19:16:47 |-INFO  monitor.JavaMonitor - 189704185,191299585,202590993,4177526785
2018-04-18 19:16:57 |-INFO  monitor.JavaMonitor - 253403241,255066113,270121921,5570035713
2018-04-18 19:17:07 |-INFO  monitor.JavaMonitor - 317148257,318832641,337657201,6962544641
2018-04-18 19:17:17 |-INFO  monitor.JavaMonitor - 380906217,382599169,405192481,8355053569
Exception in thread "Thread-24" java.lang.OutOfMemoryError: Java heap space
	at monitor.Test$1.run(Test.java:26)
	at java.lang.Thread.run(Thread.java:748)
Exception in thread "AsyncAppender-Worker-stdoutAppender" Exception in thread "AsyncAppender-Worker-infoAppender" java.lang.OutOfMemoryError: Java heap space
java.lang.OutOfMemoryError: Java heap space
~~~

这个类可以打包到公共jar包中,这样引用了这个jar包的系统(spring框架),都可以在运行的时候获得内存监控和报警的功能.


除了这个监控方案之外,还考虑了根据gc的次数来判断,但测试之后,发现系统初始10s内gc2次,最后达到OOM的时候,gc15次;而且这个gc次数根据jvm参数设置和系统运行情况的不同而不同,如果拿来做内存监控不太合理;    
另外由于存在频繁的gc,堆内存的统计会存在波动,所以上边的方案采用了取平均值的方式.根据测试情况来看,输出的堆内存占用比率在逐渐增加,直到最终发生OOM,所以这种方案还是比较有参考意义的.    

以上只是`ManagementFactory`的一个小的应用方案,在我考虑有个这个工具,还可以做到更多    

* 1. 通过接口`List<MemoryPoolMXBean> getMemoryPoolMXBeans()`可以获得各个内存区域的详细信息.

heap usage:
Par Eden Space,type:Heap memory
Par Survivor Space,type:Heap memory
CMS Old Gen,type:Heap memory
none heap:
name:Code Cache,type:Non-heap memory
Metaspace,type:Non-heap memory
Compressed Class Space,type:Non-heap memory

* 2. 线上服务器一般是无法通过jdk的可视化工具查看jvm的各项参数信息的,通过这个工具类可以获取到各个时间点的各项信息,然后通过打点展示在页面上,自己实现虚拟机的实时监控.

最后根据其中类的命名(MXBean结尾),我猜想jdk的可视化工具如jconsole、jvisuam是不是也是使用这个工具类打点的呢?
