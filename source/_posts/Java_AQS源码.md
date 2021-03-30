---
title: AQS-JUC中锁的抽象模板
tags: 
  - JUC
  - 并发
  - 同步
categories:
  - [Java,JUC]
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---





todo 

https://www.cnblogs.com/waterystone/p/4920797.html





# AQS源码分析

维护一个 `volatile int` 变量 `state` 代表加锁状态

维护一个 队列代表请求锁资源的线程，head持有锁，后边的节点（线程）等待锁

提供一套模板，实现类必须实现以下方法来实现自己加锁解锁的逻辑：

- boolean tryAcquire(int)：独占式 尝试获取资源，成功则返回true，失败则返回false。
- boolean tryRelease(int)：独占式 尝试释放资源，返回值同上。
- int tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- int tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。
- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。





java同步机制的底层支持(1.6及以上)：LockSupport

# LockSupport

帮助AQS挂起/恢复当前线程

> https://www.cnblogs.com/yonghengzh/p/14280670.html

这个接口声明的方法很像Object的 wait() 和 notify()

void await()  

void awaitUniterruptibl()

boolean await(long, TimeUnit)

long awaitNanos(long)

boolean awaitUntil(Date)

void signal()

void signalAll()

在java层面只是对`Unsafe#park()`、`Unsafe#unpark()`的简单封装，在JVM的C语言实现中，每个线程持有一个Parker对象，该Parker对象有三个变量： `_counter`、``_cond`、`_mutex`，`_counter`初始值为`0`



## park()

HotSpot源码（部分）：

```c
void Parker::park(bool isAbsolute, jlong time) {
	if (Atomic::xchg(0, &_counter) > 0) return;// 获取_counter的值并将其置为0，如果原值为1，则什么也不做
    Thread* thread = Thread::current();
    assert(thread->is Java_thread(), "Must be JavaThread");
    JavaThread *jt = (JavaThread *)thread;
    
    assert(_cur_index == -1, "invariant");
    if (time == 0) {
        _cur_index = REL_INDEX;
        
        // 使当前线程加入操作系统的条件等待队列，同时释放mutex锁，并挂起当前线程（也就是 阻塞在这里！！！！！）
        statue = pthread_cond_wait (&_cond[_cur_index], _mutex);// pthread_cond_wait 是Linux标准线程库的一个系统调用
        // Java中的wait()、await()如果是在Linux中调用，也是通过native调用的这个函数

    } else {
        _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
        status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime);//??????
        if (status != 0 && WorkAroundNPTLTimedWaitHang) {
            pthread_cond_destroy (&_cond[_cur_index]);
            pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
        }
    }
    _cur_index = -1;
    assert_status(status == 0 || status == EINTR || ...)
        ...

    _counter = 0; // 计数器再次被置为0
    status = pthread_mutex_unlock(_mutex);// 线程释放锁   结束一个park()操作
    
    assert_status(status == 0, status, "invariant");
    OrderAccess::fence();
    if (jt -> handle_special_suspend_equivalent_condition()) {
        jt->java_suspend_self();
    }
}
```

1. 获取当前线程关联的 Parker 对象。
2. 将计数器置为 0，同时检查计数器的原值是否为 1，如果是则放弃后续操作。
3. 在互斥量上加锁。
4. **在条件变量上阻塞**，同时**释放锁**并等待被其他线程唤醒，当被唤醒后，将重新获取锁。
5. 当线程恢复至运行状态后，将计数器的值再次置为 0。
6. 释放锁。`最后都要释放锁`

简单说：

> - 如果`_counter==0`，则线程t暂停（wait）,直到被唤醒（ unpark(t) ）；
> - 如果`_counter==1`，则将`_counter`置为`0`，线程继续运行；

## unpark(Thread)

HotSpot源码：

```c
void Parker::unpark() {
    int s, status ;
    status = pthread_mutex_lock(_mutex);// 给当前线程加锁           这里加锁了
    assert (status == 0, "invariant");
    s = _counter;
    _counter = 1;// 然后将_counter置为1
    if (s < 1) {// 然后判断Parker对象关联的线程是否被park(),
        // thread might be parked 线程可能已经停止了
        if (_cur_index != 1) {
            if (WorkAroundNPTLTimedWaitHang) {
                status = pthread_cond_signal (&_cond[_cur_index]);// 如果被park():通过 pthread_mutex_signal 函数唤醒该线程
                assert (status == 0, "invariant");
                status = pthread_mutex_unlock(_mutex);// 最后释放锁
                assert (status == 0, "invariant");
            } else {
                status = pthread_mutex_unlock(_mutext);
                assert (status == 0, "invariant");
                status = pthread_cond_signal (&_cond[_cur_index]);
                assert (status == 0, "invariant");
            }
        } else {
            pthread_mutex_unlock(_mutex);
            assert (status == 0, "invariant");
        }
    } else {
        pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
    }    
}
//   --该线程恢复至运行状态(先拿到mutex锁)然后从pthread_cond_wait方法返回--------关联park()源码的pthread_cond_wait函数调用！！！！！
```

1. 获取目标线程关联的 Parker 对象（注意目标线程不是当前线程，是Java中unpark(jt)的参数对应的线程）。------jt：JavaThread
2. 在互斥量上加锁。------jt在park()函数中阻塞的时候是释放了锁的
3. 将计数器置为 1。------jt在park()函数开始时将`_counter`置为了 0，这里置为 1，jt被唤醒之后还会把`_counter`置为 0
4. 唤醒在条件变量上等待着的线程。------jt调用park()函数并阻塞在了系统函数`pthread_cond_wait`调用的地方
5. 释放锁。\------jt继续运行需要拿到对象的mutex锁

简单说：

> - 如果线程已暂停，则唤醒它
> - 如果线程正在运行，`_counter==0`，将`_counter`置为`1`
> - 如果线程正在运行，`_counter==1`，do nothing
>
> > 可以在改线程调用park()之前调用unpark()，效果就是 该线程调用park()的时候不会暂停
>
> > 相比Object的`wait()-notify()/notifyAll()`来说更准确，可以控制到指定的线程（只要持有该线程Thread的引用）



# Exclusive独占模式

## 加锁

### acquire(int arg)

独占式获取锁

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && 				// 如果尝试拿锁成功 直接return  没拿到就入队然后阻塞直到获取到锁
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))// 如果尝试拿锁失败，就以独占模式入队
        selfInterrupt();// currentThread.interrupt(); 
    // 调用interrupt()是因为acquireQueued(...)方法可能判断过线程是否被中断，用的是isInterrupted()方法，会清除中断标识，acquireQueued(...)的返回值就代表了线程是否被中断过，所以这里根据这个返回值决定是否需要补充currentThread的中断标识
}
```

#### acquireQueued(...) 独占模式入队并阻塞获取锁

```java
final boolean acquireQueued(final Node node, int arg) {
    //标记是否成功拿到资源，默认false
    boolean failed = true;
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过
        for (;;) {
            final Node p = node.predecessor();// 相当于 p = node.prev  判断了prev是不是null
            // 如果当前节点是头节点 且 尝试获取锁成功 就return
            if (p == head && tryAcquire(arg)) {// node.prev == head 并且 try=true
                setHead(node);// 清理引用： head = node; node.thread = null; node.prev = null;
                p.next = null; // help GC
                failed = false;
                return interrupted;// return false 表示没有中断, acquire()会直接return
            }
            // 如果上边try失败了就检查当前线程是否应该park(),如果需要 就检查当前线程是否被中断了（会清空中断标识，然后继续自旋，所以说是不可被中断的，说的就是acquireQueued(...)这个方法！ 只能是最后将原本的中断标识返回出去，由acquire()方法在设置上当前线程的中断标识）
            if (shouldParkAfterFailedAcquire(p, node) && // 检查当前线程是否需要park()
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

值得一提的是：`Condition#await()`的调用链中的一环——`doAcquireSharedInterruptibly()`的大致逻辑也是这样！

##### addWaiter(node)将当前线程包装成node入队  将curNode通过自旋置为tail

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 如果pred==null说明队列需要初始化
    enq(node);// 在这里自旋，CAS设置tail为自己
    return node;
}

// enq(node) 将node置为tail
private Node enq(final Node node) {
    for (;;) {			// 自旋                1
        Node t = tail;
        if (t == null) { // Must initialize  必须初始化，因为发现队列没有初始化才调用的enq()，这里再验证一下
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;                       // 2
            if (compareAndSetTail(t, node)) {    // 3  其实就是自选执行这三步 所以就算多个线程在同时跑这三行代码 总会排着队入队的
                t.next = node;// 必须把自己设置为tail才行
                return t; // 唯一出口
            }
        }
    }
}
```

##### shouldParkAfterFailedAcquire(prev, curNode) 检查当前线程是否应该park()

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)//Signal： -1，指示后续线程需要unpark()
        //prev的状态是signal，要求释放以发出信号，所以可以安全地park()
        return true;
    if (ws > 0) {
        // prev对应的线程被cancelled(取消)
        do {
            // 跳过prev并重试 直到找到waitStatus<=0的node
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        // 让(最靠后的、waitStatus<=0的)node作为当前节点的prev
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

##### parkAndCheckInterrupt() park当前线程并中断

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);// 挂起当前线程
    return Thread.interrupted();//最后在查询线程是否被中断，并返回该中断标识（清除掉了，不处理，return出去交给调用方处理）
}
```



## 解锁





# Shared共享模式

## 加锁

## 解锁