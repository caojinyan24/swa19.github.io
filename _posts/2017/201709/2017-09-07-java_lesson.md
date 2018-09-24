---
layout: post
title: "JAVA开发踩到过的坑"
date: 2017-09-07 16:49:48 +0800
categories: 基础
tags: java
---

* TOC
{:toc}


# String转Json

今天发现一个差点被带到线上的bug,为了解析json串,我新建了一个类,将Json串转成这个类的java对象.测试进行很顺利,但后边diff代码的时候,发现这个类的一堆gettersetter方法未被调用,在idea的提示下,我把这些方法删掉了,果然代码好看了很多.
但就在刚刚,在测试已经测试完成之后,我自己又跑了一下测试数据,并且出狱莫名的无聊,我加了行日志,打印了这个java对象,发现所有的属性都是null?!而这个问题发生的源头就在我出于洁癖删除的gettersetter方法.
同样删除gettersetter方法可能会导致的问题就是Mybatis查询,因为Mybatis同样是通过无参的构造函数和gettersetter方法来给Entity实体赋值的.
所以,在完成全面测试之后,不要轻易改动东西;如果真要改,再次做全面测试,bug往往生于一个不经意.
另一个就是代码洁癖也是要稍稍控制下的,编译器的提示仅仅是提示.

# 通过offset,limit做查询和更新(2018-03-20)

看下代码:

~~~
Long offset = 0L;
        Long pageSize = 1000L;
        while (true) {
            List<Long> ids = ylKxCustContactMapper.queryEmptyTel(offset, pageSize);
            if (CollectionUtils.isEmpty(ids)) {
                break;
            }
            ylKxCustContactMapper.deleteEmptyTel(ids);
            offset += pageSize;
        }
~~~

代码执行后,检查数据发现空的电话号码并未被全部删除,哪里出了问题呢?检查代码后恍然大悟:这个代码片段,首先按照offset查找空的电话号码,然后对查出的数据做删除;接着offset增加,再次查询时,空的电话号码总数将变少,当增加offset后,将会有部分未处理的数据被跳过.
所以这就是根据某个状态做分页查询,再根据查询结果对这个状态做变更后,再次查询下一页数据做同样的操作时,将会导致部分数据被遗漏.


# 全局变量初始化

上段代码：
~~~
@Service
public class DemoImpl implements Demo{
    private static CountDownLatch count = new CountDownLatch(1);
    private ExecutorService executorService = Executors.newFixedThreadPool(5);
    public Boolean doSth() {        
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                doThings();
            }
        });      
        try {
            count.await();
        } catch (InterruptedException e) {
            throw new RuntimeException("run deduct data sync interrupted");
        }
        return true;
    }
}
~~~

这个方法会没有通过定时任务执行，每天执行一次。有个很奇怪的情况，正常`doThings()`这个方法需要执行20分钟左右，但这个定时任务执行时间一般只有十几ms，某些时候又会变成正常的20分钟左右。一开始以为同步锁没锁上，是我在使用的时候哪里有问题，但是仔细理了下代码之后发现了问题。
由于该服务部署在tomcat服务器，使用spring框架。在做类的初始化时默认都是单例，所以这个count只会在重新部署启动时初始化为1，在一次执行完成后，count值变为0；第二次再次执行时，count依旧是0.所以就出现了以上的问题。
怎么解决这个问题呢?

1. 首先需要保证每次在执行`doSth`方法时对`count`变量重新做初化
2. 其次保证同一时刻只有一个线程执行`doSth`方法，也就是做线程同步

修改后的代码为：

~~~
@Service
public class DemoImpl implements Demo{
    private static CountDownLatch count;
    private ExecutorService executorService = Executors.newFixedThreadPool(5);
    public sychronized Boolean doSth() {        
        count = new CountDownLatch(1);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                doThings();
            }
        });      
        try {
            count.await();
        } catch (InterruptedException e) {
            throw new RuntimeException("run deduct data sync interrupted");
        }
        return true;
    }
}
~~~
修改后定时任务的时间记录正常


# 循环中的continue和break

前几天出的一个线上问题,符合条件的数据有3000条,但实际插入到数据库的只有两条.最后查看代码发现这批数据通过for循环插入,而在这个循环中,一个判断条件中增加了`break`导致循环中断.
所以对于循环中出现break和continue的,一定要重点检查下是否符合业务逻辑.


# RocketMqsubject和tag配置问题导致消息重复发送

在A系统中会发送给B系统一个RocketMq消息，B系统在接收到这个消息之后，发送消息给C系统  
根据日志，ProducerA发送了一条消息，但是到ConsumerB的之后，首先接收到和Producer发送消息一样的msgId的消息，然后不断刷日志，显示ConsumerB持续不断地接收到同样消息内容，但是msgId不同的消息。  
开始因为初次配置使用RocketMq，怀疑RocketMq的处理机制问题，但其他系统应用不存在这个问题；后来在梳理业务逻辑的时候，发现C系统接收的是来自A系统的消息，而这个是不合理的，需要对C系统和B系统接收消息的tag做一个区分，由此才意识到，原来B系统一直在接收消息，然后发送消息，发送的消息由于subject和tag配置的问题，B系统会接收到自己发送的消息并处理，由此，一个死循环导致了消息被不断地重复发送和接收和处理  
遇到问题而不得其解的时候，不要怀疑人生。有句话说：事出反常必有妖

# java引用

对变量做临时赋值后，后续在用的时候没有恢复成原有的值   
由于java中的变量传递的是变量的引用，所以在方法内对一个变量做的改变在这个方法调用结束之后会继续保留，这个错误很弱智，但也很容易被忽略

# static变量

对于一个类中定义的static变量，这个变量在多个类实例中共用。看一个类的定义:

~~~
public class ScheduleJobList {
    private static final Logger logger = LoggerFactory.getLogger(ScheduleJobList.class);
    private  static PriorityBlockingQueue<JobInfoWrapper> list;
}
~~~

使用时：

~~~
    public static void addRegisteredJob(String jobInfoStr) {
        Map<JobInfo, ScheduleJobList> jobMap = new ConcurrentHashMap<JobInfo, ScheduleJobList>();
        JobScheduleParser.ScheduleMsgBody msg = JSON.parseObject(jobInfoStr, JobScheduleParser.ScheduleMsgBody.class);
        ScheduleJobList scheduleJobList = new ScheduleJobList();
        scheduleJobList.addJob(JobInfoWrapper.newWrapperJob(msg.getNewJob()));//将当前需要执行的任务添加到任务列表中
        jobMap.remove(msg.getOldJob());
        jobMap.put(msg.getNewJob(), scheduleJobList);
    }
~~~

问题是，当我对一个JobInfo实例的ScheduleJobList做了dequeue操作，其他JobInfo实例的ScheduleJobList中的数据也会减少，一开始百思不得其解，直到看ScheduleJobList类的时候注意到static关键词，才恍然大悟。
static关键词修饰的静态变量只会保存在一个内存空间，当需要它在各个类中不共享的时候，坑就出现了。
