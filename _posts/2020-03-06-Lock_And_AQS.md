---
title: Java中的锁以及队列同步器(AQS)
categories: [concurrent]
comments: true
---

一、Lock接口：

    在Lock接口出现之前，Java靠synchronized关键字实现锁的功能。但是synchronized关键字将锁的获取与释放固化了，显得并没有这么灵活。在jdk1.5以后，新增加了Lock接口。lock主要具备synchronized关键字锁不具备的以下同步特性：

        （1）.尝试非阻塞地获取锁：tryLock()；

        （2）.能被中断地获取锁：lockInterruptibly();

        （3）.超时获取锁：tryLock(Long time,TimeUnit)

    下面是Lock接口的API:

         1）void lock():

         获取锁，电泳该方法当前线程会获取锁，获得锁后，从该方法返回；

         2）void lockInterruptibly():

         可中断地获取锁，和lock()方法的不同之处在于该方法会响应中断，即在锁的获取中可以中断当前线程；

         3）boolean tryLock():

         尝试非阻塞的获取锁，调用该方法后立刻返回，如果能够获取则返回true，否则返回false；

         4）boolean tryLock(long time,TimeUnit unit) throws InterruptedException:

         超时获取锁，当前线程在以下三种情况下会返回：

            ①：当前线程在超时时间内获得了锁

            ②：当前线程在超时时间内被中断

            ③：超时时间结束，返回false

         5）void unlock(): 释放锁；

         6）Condition newCondition():

         获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件的wait()方法，而调用后，当前线程将释 放锁

二、队列同步器(AbstactQueuedSynchronizer)：——下面简称同步器。

    1.定义：是用来构建锁或者其他同步组件的基础框架。使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。

    2.同步器与锁的关系：同步器是实现锁的关键。锁是面向使用者的，它定义了使用者与锁交互的接口，隐藏了实现的细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程排队、等待与唤醒等底层操作。

    3.同步器可重写的方法：

        protected boolean tryAcquire(int arg)——独占式获取同步状态；

        protected boolean tryRelease(int arg)——独占式释放同步状态；

        protected boolean tryAcquireShared(int arg)——共享式获取同步状态；

        protected boolean tryReleaseShared(int arg)——共享式释放同步状态；

        protected boolean isHeldExclusively()——表示同步器是否在独占模式下被线程占用。
    
三、自定义一个独占式锁

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
 
public class Mutex implements Lock {
    //静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer{
        //是否处于占用状态
        protected boolean isHeldExclusively(){
            return getState() == 1;
        }
        //当状态为0时获取锁
        protected boolean tryAcquire(int arg){
            if(compareAndSetState(0,1)){
                setExclusiveOwnerThread(Thread.currentThread());    //设置成功，则代表获取了同步状态。
                return true;
            }
            return false;
        }
 
        //释放锁，将状态设置为0
        protected boolean tryRelease(int releases){
            if(getState()==0)
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
 
        //返回一个Condition,每个condition都包含了一个condition队列
        Condition newCondition(){
            return new ConditionObject();
        }
    }
    //仅仅需要将操作代理到Sync上即可
    private final Sync sync = new Sync();
 
    public boolean isLocked(){
        return sync.isHeldExclusively();    //是否处于同步状态
    }
    @Override
    public void lock() {
        sync.acquire(1);    //获取锁，将状态设置为1
    }
 
    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
 
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);//尝试获取锁
    }
 
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1,unit.toNanos(1000));
    }
 
    @Override
    public void unlock() {
        sync.release(1);    //释放锁
    }
 
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```
上述示例中，独占锁Mutex是一个自定义同步组件，它在同一时刻值允许一个线程占有锁。Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。