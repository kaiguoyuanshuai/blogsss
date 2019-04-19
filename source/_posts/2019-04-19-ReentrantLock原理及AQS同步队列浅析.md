---
title: ReentrantLock原理及AQS同步队列浅析
tags:
  - java并发
categories:
  - java并发专题
top: false
date: 2019-04-19 09:05:46
---
# ReentrantLock原理及AQS同步队列浅析

## 可重入锁的概念 
> 当前持有该锁的线程能够再次获取到该锁，不需要进行等待 

## 公平与非公平获取锁的区别
> 公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。 

## 使用方法  

```java
public class ReentrantLockDemo extends Thread {

    private Lock lock;

    public ReentrantLockDemo(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        try {
            lock.lock();
            for (int i = 0; i < 2; i++) {
                System.out.println("ThreadName = " + Thread.currentThread().getName() +
                        ", i  = " + i);
            }
        } catch (Exception e) {
        } finally {
            lock.unlock();
        }
    }
    
    @Test
    public void testReentrantLock() throws InterruptedException {
    
        Lock lock = new ReentrantLock();
    
        Thread t = new ReentrantLockDemo(lock,"t1");
        t.start();
        Thread t2 = new ReentrantLockDemo(lock,"t2");
        t2.start();
        t.join();
        t2.join();
    }
}
```

## 类结构图

{% asset_img 类图.png 人民日报点名 %}

## 源码 

### 默认构造函数
```java
/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync();
}
```
默认构造一个 NonfairSync 的非公平锁  

### locks.ReentrantLock.NonfairSync#lock 锁
```java
final void lock() {
    //设置 state 
    if (compareAndSetState(0, 1)) //先判断 state的值是不是0
    //如果成功  设置线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //获取锁
        acquire(1);
}
```

#### locks.AbstractQueuedSynchronizer#acquire 
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && //1 
        acquireQueued(addWaiter(Node.EXCLUSIVE) //2, arg)) //3
        selfInterrupt();
}
```

##### locks.ReentrantLock.NonfairSync#tryAcquire 获取锁 
```java
final boolean nonfairTryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取信号
    int c = getState();
    
    //如果信号了是0  则表示当前没有线程获取锁
    if (c == 0) {
        //CAS 设置 state 值
        if (compareAndSetState(0, acquires)) {
            //设置当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果是当前线程
    else if (current == getExclusiveOwnerThread()) {
        //新增 state的值  
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
            //设置 state
        setState(nextc);
        
        return true;
    }
    return false;
}
```
当这个方法返回false的时候(锁已经被线程占用且不是当前线程占用)才会执行 入同步队列的功能 

##### locks.AbstractQueuedSynchronizer#addWaiter 添加到队列中
```java
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //设置到头节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //入队列
    enq(node);
    return node;
}
```
/添加了一个独占锁 Node.EXCLUSIVE, 并且加入到队列中 ,具体相关入队和数据结构操作 看后续内容 

##### locks.AbstractQueuedSynchronizer#acquireQueued 等待锁释放
```java

/*
 * Various flavors of acquire, varying in exclusive/shared and
 * control modes.  Each is mostly the same, but annoyingly
 * different.  Only a little bit of factoring is possible due to
 * interactions of exception mechanics (including ensuring that we
 * cancel if tryAcquire throws exception) and other control, at
 * least not without hurting performance too much.
 */

/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //线程先走这边
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

** parkAndCheckInterrupt **

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```



### locks.ReentrantLock#unlock 释放锁 
```java
public void unlock() {
    sync.release(1);
}

```

#### locks.AbstractQueuedSynchronizer#release
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { //重点1 
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);   //重点2 
        return true;
    }
    return false;
}

```
##### locks.ReentrantLock.Sync#tryRelease 释放节点 
```java
protected final boolean tryRelease(int releases) {
    //递减 state
    int c = getState() - releases;
    //如果不是对应的线程解锁，则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    //是当前线程
    boolean free = false; 
    
    //如果是最后一次释放 
    if (c == 0) {
        //设置绑定的线程为null
        free = true;
        setExclusiveOwnerThread(null);
    }
    //设置 state 
    setState(c);
    
    //返回结果如果是  true 表示 全部释放了  为false 表示当前线程还需要继续解锁
    return free;
}

```
##### locks.AbstractQueuedSynchronizer#unparkSuccessor 唤醒线程
```java

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0) //设置等待的信号
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒线程
        LockSupport.unpark(s.thread);
}
```



//TODO    
 队列原理(组建队列，入队，出队)
