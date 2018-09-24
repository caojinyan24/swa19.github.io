---
layout: post
title:  "Java源码学习"
date:   2018-04-23 21:04:08 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

# CopyOnWriteArrayList

jdk中对Collection操作有一个共识：就是在循环中是不能对列表做修改的。前几天写个小工具，要求对一个List做遍历，并在遍历过程中向List添加元素。翻了Collection的子类之后，发现`CopyOnWriteArrayList`可以解决这个问题。

看下其中的add方法：

~~~
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}
~~~

首先，`CopyOnWriteArrayList`对内部数组的操作通过`getArray`和`setArray`进行。当需要向数组中添加元素时，通过`  Object[] newElements = Arrays.copyOf(elements, len + 1);`创建一个原数组的拷贝，在向新数组中添加元素后，将内部数组指向新数组的引用。

同时对`CopyOnWriteArrayList`对象的任何修改都需要首先获取到对象锁：`final transient ReentrantLock lock = new ReentrantLock();`从而保证了线程的安全性。

# ConcurrentHashMap

1. 根据key做hash后，进行分段，不同段使用不同的锁控制，提高并发访问效率；另外当需要rehash的时候，只对某个分段做rehash
2. 所有需要共享的变量使用volitale关键字，乐观锁提高了并发量

# hashTable

各个方法使用synchronized修饰来保证多线程并发访问的安全，效率较低

# TreeMap，TreeSet

一个基于红黑树实现的有序的map和set，线程不安全

# ReentrantLock

ReentrantLock的锁的控制是通过AbstractQueuedSynchronizer类的子类（`abstract static class Sync extends AbstractQueuedSynchronizer`）实现的。

Sync类提供的方法有：

1. 锁定：`abstract void lock()`
2. 获取锁：`final boolean nonfairTryAcquire(int acquires) `
3. 释放锁：`protected final boolean tryRelease(int releases) `
4. 判断当前线程是否获得当前锁：`protected final boolean isHeldExclusively()`，如果当前锁由当前线程获得，则返回true
5. 新建condition：`final ConditionObject newCondition()`
6. 查询获得当前锁的线程：`final Thread getOwner() `。其中的getExclusiveOwnerThread()方法来自`AbstractOwnableSynchronizer`，在这个类中有一个属性`private transient Thread exclusiveOwnerThread;`用来表示当前获取排他锁的线程。
7. 查询锁定的次数：`final int getHoldCount()`，0代表未被当前线程获得锁
8. 查询是否是锁定状态：`final boolean isLocked() `，getState为0代表未锁定
9. 反序列化：`private void readObject(java.io.ObjectInputStream s)`

通过Sync的一个实例来实现线程控制，根据Sync的实现的不同，分为公平锁和不公平锁两种，这两种锁的实现如下：

~~~
   static final class NonfairSync extends Sync {
       private static final long serialVersionUID = 7316153563782823691L;

       /**
        * Performs lock.  Try immediate barge, backing up to normal
        * acquire on failure.
        */
       final void lock() {
           if (compareAndSetState(0, 1))
               setExclusiveOwnerThread(Thread.currentThread());
           else
               acquire(1);
       }

       protected final boolean tryAcquire(int acquires) {
           return nonfairTryAcquire(acquires);
       }
   }
~~~

~~~
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */    
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
~~~

~~~
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
~~~


公平锁和sychronized类似，根据获取锁的先后顺序执行

对不公平锁，在执行lock()函数时，首先利用cas进行锁定，如果锁定成功，直接插入，否则，正常等待锁释放。

在创建ReentrantLock实例时，默认是不公平锁

相比sychronized的特性：

1. 在获取锁的时候，如果获取不到可直接返回；避免阻塞线程
2. 可设置优先级，中断正在执行的线程
3. 将锁的相关信息以对象的形式做封装，可以获取到当前锁的信息，如等待的线程队列，当前获取锁的线程等

# ThreadPoolExecutor

`private final BlockingQueue<Runnable> workQueue;`：存放任务

execute

~~~
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}

~~~

1. 获取当前线程数，如果小于核心线程数，则执行addWorker操作
2. 如果当前标示线程池未销毁，并且当前任务队列存在空余位置
2.1. 再次check，如果当前线程池未运行，并且当前任务可成功移除，拒绝当前任务
2.2. 如果再次check，核心线程已满，重新起线程做addWork操作
3. 如果条件2不成立
3.1. 新建线程执行addWork操作
3.2. 如果3.1失败，拒绝当前任务

addWorker首先再次核对当前的线程池状态，跟任务列表，如果可以执行，则添加任务到任务列表中，并且启动线程

在向任务队列中添加任务时，使用ReentraintLock做并发控制（因为这里的workers的类型是HashSet，不能保证并发的安全问题，根据HashSet文档，通过`Set s = Collections.synchronizedSet(new HashSet(...));`可以保证并发安全）

~~~
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
~~~

每个Worker实例是一个Runnable子类

~~~
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
~~~

`  while (task != null || (task = getTask()) != null)`在while循环中，从当前的`BlockingQueue<Runnable> workQueue`和
`private final HashSet<Worker> workers`取出task，这两个的区别是，workers是线程池中加入的所有的Worker，而workQueue则是所有即将运行的任务队列，当添加任务时，workers添加数据，添加失败或者任务退出（执行完毕）的时候移走，而workQueue则是在执行addWorker前添加，表示要执行的线程队列。

# LinkedBlockedQueue

通过ReentrantLock的Condition实现添加和删除的等待。
