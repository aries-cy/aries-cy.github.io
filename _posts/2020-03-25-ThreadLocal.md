---
title: ThreadLocal原理及其应用
categories: [concurrent]
comments: true
---

一、ThreadLocal定义

ThreadLocal是Java中提供的线程本地存储机制，可以利用该机制将数据缓存在某个线程内部，该线程可以在任意时刻获取缓存中的数据。

ThreadLocal底层是通过ThreadLocalMap实现的，每个Thread对象中都存在一个ThreadLocalMap，Map的Key为ThreadLocal对象，Map的Value为存储的对象。

二、存在的问题

如果在线程池中使用ThreadLocal会造成内存泄漏，因为当ThreadLocal对象使用完成后，应该把entry对象进行回收，但是线程池中的线程不会被回收，而线程对象是通过强引用指向ThreadLocalMap，ThreadLocalMap也是通过强引用指向entry对象，线程不被回收，entry对象也不会被回收，从而出现内存泄漏。

解决办法：在使用了ThreadLocal对象后，手动调用ThreadLocal的remove()方法。

三、使用ThreadLocal

假设有一个简单的需求，需要打印一个线程执行任务的时间。

首先定义一个Context，在Context中定义一个ThreadLocal，并提供set、get、remove方法：

```java
static class Context{
    static ThreadLocal<Long> threadLocal = new ThreadLocal<>();

     public static void set(Long time){
        threadLocal.set(time);
    }

    public static Long get(){
        return threadLocal.get();
    }

    public static void remove(){
        threadLocal.remove();
    }

}
```

然后定义一个Runable，在run()方法中，开始的时候在Threadlocal中设置线程开始的时间，然后在结束的时候，输出线程执行任务的时间：

```java
static class TimeThread implements Runnable{
    @Override
    public void run() {
        Context.set(System.currentTimeMillis());
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(System.currentTimeMillis() - Context.get());
        Context.remove();
        //执行remove方法后，输出null
        System.out.println(Context.get());
    }
}
```

在main()方法中测试一下

```java
public static void main(String[] args) {
    Thread thread = new Thread(new TimeThread());
    thread.start();
}
```

四、父线程传递本地变量到子线程

在实际的工作中，我们可能会遇到子线程中会用到父线程中的本地变量，那么这种情况应该怎么处理呢？

在Java中，提供了 `InheritableThreadLocal` 这样一个类，它继承了ThreadLocal，可以让子线程获取父线程中的本地变量，下面写代码测试一下。

```java
package com.cy.threadlocal;

public class ThreadLocalTest {


    public static void main(String[] args) {
        //测试子线程中能否获取父线程中设置的变量
        new Thread(new TimeThread()).start();
        //新启一个线程，看另外的线程能否获得TimeThread中设置的变量
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("其他线程获取变量 -> "+Context.get());
            }
        }).start();
    }

    static class TimeThread implements Runnable{
        @Override
        public void run() {
            //设置当前线程的参数（父线程）
            Context.set("父线程变量");
            //启动一个子线程
            new Thread(new Runnable() {
                @Override
                public void run() {
                    //在子线程中获取父线程设置的变量
                    System.out.println("子线程获取父线程设置的变量 -> "+Context.get());
                }
            }).start();
        }
    }

    static class Context{
        public static InheritableThreadLocal<String> threadLocal = new InheritableThreadLocal<>();

        public static void set(String param){
            threadLocal.set(param);
        }

        public static String get(){
            return threadLocal.get();
        }

        public static void remove(){
            threadLocal.remove();
        }
    }
}
```

执行上面的main()方法，控制台输出如下：

![](https://aries-cy.github.io/assets/note_img/threadlocal_ans.png)

可以看到子线程中确实获取到了父线程中的本地变量，而其他无关的线程并没有获取到该线程的本地变量。