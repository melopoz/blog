---
title: java.util.concurrent
tags:
  - JUC
  - 并发
  - 锁
  - 阻塞队列
categories: Java
---



这是JUC包的总体结构：

<img src="https://i.loli.net/2021/04/02/1sVvczI6hmJY3oN.png" style="zoom:80%;" />

- 阻塞队列（线程池、锁要用阻塞队列来实现）
- [线程池 ThreadPoolExecutor](#线程池 ThreadPoolExecutor)
- 锁
- 并发集合









# 线程池 ThreadPoolExecutor

用一个原子数字  **ctl** 表示 线程池的状态 **state** 和 工作线程数`workerCount` **wc**。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; } // 高3位 表示 state
private static int workerCountOf(int c)  { return c & CAPACITY; }// 低29位 表示 wc
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



## 生命周期

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;// 运行：接受新任务并处理排队的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;// 关机：不接受新任务，但处理排队的任务
private static final int STOP       =  1 << COUNT_BITS;// 停止：不接受新任务，不处理排队的任务，并中断进行中的任务
private static final int TIDYING    =  2 << COUNT_BITS;// 整理：所有任务已终止，workerCount为零，转换到状态TIDYING的线程将运行terminated（）钩子方法
private static final int TERMINATED =  3 << COUNT_BITS;// terminated（）已完成
```



## 关键参数

常见的创建方式：

- 构造函数

  ```java
  public ThreadPoolExecutor(
      int corePoolSize,// 核心线程数
      int maximumPoolSize,// 最大线程数
      long keepAliveTime,// 多余的空闲线程(若allowCoreThreadTimeOut=true则含coreWorker)将在终止之前等待新任务的最长时间
      TimeUnit unit,// 时间单位
      BlockingQueue<Runnable> workQueue,// 使用的阻塞队列
      // 下边两个参数任选1 或 都传
      ThreadFactory threadFactory,// 可以指定线程名，增加日志可读性 默认Executors.DefaultThreadFactory 命名规则为pool-i-thread-
      RejectedExecutionHandler handler)// 拒绝策略
  {... ...}
  // 这是默认的拒绝策略：抛异常
  private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
  ```

- **Executors** 的静态方法

  - **newFixedThreadPool**(int **nThreads**, ThreadFactory threadFactory^可选^)：**固定线程数**、**无界队列**、**keepAliveTime=0**

    ```java
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
    ```

  - **newWorkStealingPool**(int parallelism^可选^)：**多个队列并行**（也就不能保证任务按照提交顺序被执行）、可以从其他队列窃取任务

    ```java
    // 该线程池维护足以支持给定并行度级别的线程，并且可以使用多个队列来减少争用。
    // 并行度级别对应于活跃参与或可用于参与任务处理的最大线程数。实际的线程数可能会动态增长和收缩。
    // 工作窃取池不能保证提交任务的执行顺序
    public static ExecutorService newWorkStealingPool(int parallelism/*无参则使用 ncpu */) {
        return new ForkJoinPool(parallelism, ForkJoinPool.defaultForkJoinWorkerThreadFactory, null, true);
    }
    ```

  - **newSingleThreadExecutor**(ThreadFactory threadFactory^可选^)：**corePoolSize、maxPoolSize 均为 1**、无界队列、**keepAliveTime=0**

  - **newCacheThreadPool**(ThreadFactory threadFactory^可选^)：来一个任务就创建线程执行

    ```java
    //core:0，maxPS无上限、使用`SynchronousQueue`（所以`submit(task)`就会新建一个线程）
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      // capacity为0。take()之后才能再put()
                                      threadFactory);
    }
    ```

  - **newScheduledThreadPool**(int **corePoolSize**, ThreadFactory threadFactory^可选^)：

  - **newSingleThreadScheduledExecutor**(ThreadFactory threadFactory^可选^)：

    [ScheduledThreadPoolExecutor](#ScheduledThreadPoolExecutor)

> todo 上边两个 和下边详解！！！



### 拒绝策略

- **AbortPolicy**：抛异常（默认）

  ```java
  public static class AbortPolicy implements RejectedExecutionHandler {
      public AbortPolicy() { }
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          throw new RejectedExecutionException(
              "Task " + r.toString() + " rejected from " + e.toString());
      }
  }
  ```

- **CallerRunsPolicy**：在当前线程(调用execut()的线程)执行

  ```java
  public static class CallerRunsPolicy implements RejectedExecutionHandler {
      public CallerRunsPolicy() { }
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          if (!e.isShutdown()) { r.run();}
      }
  }
  ```

- **DiscardPolicy**：直接丢弃

  ```java
  public static class DiscardPolicy implements RejectedExecutionHandler {
      public DiscardPolicy() { }
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) { }
  }
  ```

- **DiscardOldestPolicy**：丢弃最早进入queue的任务（即将执行的任务）

  ```java
  public static class DiscardOldestPolicy implements RejectedExecutionHandler {
      public DiscardOldestPolicy() { }
      public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          if (!e.isShutdown()) {
              e.getQueue().poll();
              e.execute(r);
          }
      }
  }
  ```

  



## 运行原理

> todo 流程图



### 常见问题：

worker线程什么时候回收？

> 也就是什么时候调用`workers.remove(w);`。可以搜到只有两处调用了。
>
> **addWorkerFailed()** 和 **processWorkerExit()**。
>
> 而 `processWorkerExit()` 只在 **runWorker(Worker w)** 方法中调用，runWorker 方法又是 worker 的run方法的主体循环。
>
> 可以看下边[Worker线程](#Worker线程)第三步。
>
> 所以只有 `getTask() == null` 时才会 `跳出while` 并 `回收空闲且多余的` worker线程。
>
> 就看看[getTask()方法](#getTask()方法)什么时候 `return null;`。

怎么区分核心线程和非核心线程？

> 不区分，就计数就行了呗。区分出来又有什么用？挺离谱的问题。如果头脑不清醒的话被问到 可能还真不太敢回答。。



### Worker线程

非核心线程是Pool在timeout时间之内没有收到任务，才会销毁。

`Worker#run()` 调用 `j.u.c.ThreadPoolExecutor#runWorker(this)`：（`Workder extends AQS implements Runnable`！）

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    public void run() {
        runWorker(this);
    }
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask; // 1.创建worker的时候有绑定的任务就直接执行
    w.firstTask = null;
    w.unlock(); // allow interrupts  2.锁定this 允许中断
    boolean completedAbruptly = true;
    try {
        // 重点！ 如果task是null，就会跳出while。所以去看看getTask()
        while (task != null || (task = getTask()) != null) { // 3.while循环 从队列取出任务并执行
            w.lock();
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted()
               ) {
                wt.interrupt(); // 4. 检查中断和清除中断等等（线程池shutdown会中断所有空闲worker：interruptIdleWorkers()）
                // interruptIdleWorkers方法：能拿到worker.tryLock能成功，就是空闲的worker
            }
            try {
                beforeExecute(wt, task); // 空方法。可以自定义子类 完成一些前置工作
                Throwable thrown = null;
                try { task.run(); } catch {...} // 5.执行任务
                } finally { afterExecute(task, thrown);}// 空方法。可以自定义子类 实现一些收尾工作
            } finally { w.unlock(); }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

### getTask()方法

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {// 1. pool 已关闭
            decrementWorkerCount();
            return null;// 
        }
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;// 2. 允许回收核心线程 或 wc超过coreSize
        if ((wc > maximumPoolSize || (timed && timedOut)) && // 超过max(不太可能) 或 设置timeout且已超时
            (wc > 1 || workQueue.isEmpty())// 无论如何必须 wc>1 且 队列为空
           ) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```





## 配置

> todo 
>
> 动态配置
>
> 线程池属性的set方法 原理



## ScheduledThreadPoolExecutor

先看类结构图。

> todo





## Spring线程池

> todo ThreadPoolTaskExecutor



## Tomcat线程池

与Java的普通线程池不太一样，Tomcat需要优先响应链接，所以 请求进来时，如果没有空闲的核心线程，就创建线程来执行任务，当线程数达到`maxPoolSize`的时候，再把任务放到`BlockingQueue`。

> todo 源码证明