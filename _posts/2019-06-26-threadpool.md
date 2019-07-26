---
title: "ThreadPool实现原理"
category: Java
tags: [concurrency, java]
---
本文主要分析java.util.concurrent.ThreadPoolExecutor的实现原理，首先看它的构造函数：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
- corePoolSize:线程池中稳定保存的线程数（一开始会小于这个数）
- maximumPoolSize：线程池中最大线程数
- keepAliveTime and unit：大于最小线程数的线程空闲后存活时间
- workQueue：用于存放任务的阻塞队列
- threadFactory：用于创建线程的工厂类
- handler：当任务队列满了且线程数达到了最大时的饱和策略
对于IO密集型任务，线程数一般设为CPU数*2，对于计算密集型任务，线程数一般设为CPU数。

当调用execute方法时：
```java
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
```
其流程如图：

![threadpool_workflow](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/threadpool_workflow.png)

创建线程是通过addWorker创建内部Worker类，其中调用getThreadFactory().newThread(this)来创建执行自己的线程，之后在addWorker中start该线程，执行Worker run方法中的runWorker会不断的从任务队列中获取任务或阻塞，并且每次执行任务前会执行beforeExecute，之后会afterExecute，可以通过重写beforeExecute方法来给执行线程重命名。

线程池状态变化如图：
- RUNNING:  Accept new tasks and process queued tasks
- SHUTDOWN: Don't accept new tasks, but process queued tasks
- STOP: Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks
- TIDYING:  All tasks have terminated, workerCount is zero, the thread transitioning to state TIDYING will run the terminated() hook method
- TERMINATED: terminated() has completed

![thread_lifecycle](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/thread_lifecycle.png)

shutdownNow终止线程的方法是通过调用Thread.interrupt()方法来实现的:
```
 * <p> If this thread is blocked in an invocation of the {@link
 * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
 * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
 * class, or of the {@link #join()}, {@link #join(long)}, {@link
 * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
 * methods of this class, then its interrupt status will be cleared and it
 * will receive an {@link InterruptedException}.
 *
 * <p> If this thread is blocked in an I/O operation upon an {@link
 * java.nio.channels.InterruptibleChannel InterruptibleChannel}
 * then the channel will be closed, the thread's interrupt
 * status will be set, and the thread will receive a {@link
 * java.nio.channels.ClosedByInterruptException}.
 *
 * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
 * then the thread's interrupt status will be set and it will return
 * immediately from the selection operation, possibly with a non-zero
 * value, just as if the selector's {@link
 * java.nio.channels.Selector#wakeup wakeup} method were invoked.
 *
 * <p> If none of the previous conditions hold then this thread's interrupt
 * status will be set. </p>
```
可以看到如果线程处于正常活动状态，那么会将该线程的中断标志设置为true，而无法中断当前的线程。所以，shutdownNow并不代表线程池就一定立即就能退出，它也可能必须要等待所有正在执行的任务都执行完成了才能退出。
