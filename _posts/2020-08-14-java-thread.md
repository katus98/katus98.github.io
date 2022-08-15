---
title: Java开发技术之多线程
author: katus
date: 2020-08-14 19:53:17 +0800
categories: [Java]
tags: [Java]
---

> 暑假期间，个人对一些未来研究生阶段可能会常用的编程技术进行重新一轮的系统复习和学习，及希望能够查缺补漏，有所提升。本文也是作为复习和学习过程中的笔记，用于长久的记录。不排除其中可能含有部分疏漏和错误，如有发现，希望各位能够批评指正，谢谢。

## 一、多线程概述

### （一）概述

+ 每个进程有独立的方法区、堆空间、虚拟机栈和程序计数器。
+ 一个进程可能拥有多个线程。
+ 每个线程有独立的虚拟机栈和程序计数器，但是一个进程下的多线程共享方法区和堆空间。

### （二）线程的优先级

+ 线程存在优先级，优先级越高，CPU将资源分配到该线程的可能性越大，但是这并不意味着高优先级的线程一定先运行。
+ Java线程优先级分为十个级别，从1~10（整数），默认是5。
  + 最低优先级`Thread.MIN_PRIORITY`（1）
  + 默认优先级`Thread.NORM_PRIORITY`（5）
  + 最高优先级`Thread.MAX_PRIORITY`（10）

## 二、线程的生命周期

+ 线程的状态分为五种。
  + 新建：完成线程创建。
  + 就绪：线程可以随时开始运行，但是尚未处于运行状态。
  + 运行：线程正在运行。
  + 死亡：线程完成运行。
  + 阻塞：线程被限制无法运行，只能通过其他命令使其重新进入就绪/运行状态。

## 三、创建多线程的方法

### （一）创建Thread子类

+ 创建`Thread`子类，并重写run()方法。

```java
package com.katus.thread;

/**
 * 创建多线程的方式 1: 创建Thread子类
 * @author katus
 * @version 1.0, 2020-08-09
 */
class MyThread extends Thread {   // 因为要求继承Thread类 导致本类不能是其他类的子类
    public MyThread(String name) {
        super(name);
    }

    @Override
    public void run() {   // 此线程执行的操作
        try {
            sleep(1000);   // 本线程阻塞1000ms
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        for (int i = 0; i < 50; i++) {
            if (i % 2 == 0) {
                System.out.println(this.getName() + ":" + i);
            }
//            if (i % 20 == 0) {
//                Thread.yield();   // 释放当前线程CPU的执行权, 但是有可能又被CPU分配到了
//            }
        }
    }
}

public class ThreadTest {
    public static void main(String[] args) {
        Thread.currentThread().setName("Thread-main");
        Thread.currentThread().setPriority(Thread.MIN_PRIORITY);   // 设置最低优先级
        // 创建线程对象
        MyThread t1 = new MyThread("Thread-1st");
        t1.setPriority(Thread.MAX_PRIORITY);   // 设置最高优先级
        // 开始执行线程, 主线程继续
        // t1.setName("Thread-1st");
        t1.start();   // 启动当前线程并调用当前线程的run()
        // t1.run();   // 错误的启动线程的方法
        // t1.start();   // 不能让已经start的线程再次start
        for (int i = 0; i < 50; i++) {
            if (i % 2 == 0) {
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
            if (i == 30) {
                try {
                    t1.join();   // 调用后本线程阻塞, t1线程执行至结束, 本线程取消阻塞
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        System.out.println(t1.isAlive());   // 判断线程是否存活 如果执行完毕即不存活
    }
}
```

### （二）创建Runnable接口的实现类

- 创建一个类实现`Runnable`接口，重写其中的run()方法。
- 本方法的优势在于创建的类可以作为其他有意义自定义类的子类，（相对于上一种方法）不必受单继承的限制。

```java
package com.katus.thread;

/**
 * 创建多线程的方式 2: 实现Runnable接口
 * @author katus
 * @version 1.0, 2020-08-10
 */
class MThread implements Runnable {
    // 实现Runnable接口中的run方法
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            if (i % 2 == 0) {
                System.out.println(i);
            }
        }
    }
}

public class ThreadTest1 {
    public static void main(String[] args) {
        MThread mThread = new MThread();   // 创建实现类对象
        Thread t1 = new Thread(mThread);   // 构造线程
        t1.start();   // 启动线程

        // 再次启动一个线程
        Thread t2 = new Thread(mThread);
        t2.start();
    }
}
```

### （三）创建Callable接口的实现类

- 创建一个类，实现`Callable`接口，重写其中的call()方法。
- 该方法要求Java 5以上版本。
- 本方法的优势在于创建的线程允许抛出异常，允许有返回值，同时支持泛型。

```java
package com.katus.thread;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**
 * 创建多线程的方式 3: 实现Callable接口 (Java5 新特性)
 * @author katus
 * @version 1.0, 2020-08-10
 */
// 创建Callable接口的实现类
class NumThread implements Callable {
    // 重写call方法 内部为线程的操作
    @Override
    public Object call() throws Exception {   // 优势：可以抛出异常 可以有返回值 支持泛型
        int sum = 0;
        for (int i = 0; i <= 100; i++) {
            if (i % 2 == 0) {
                System.out.println(i);
                sum += i;
            }
        }
        return sum;
    }
}

public class ThreadTest2 {
    public static void main(String[] args) {
        // 创建Callable实现类的对象
        NumThread numThread = new NumThread();
        // 通过Callable实现类对象构建FutureTask类的对象
        FutureTask futureTask = new FutureTask(numThread);
        // 通过FutureTask类对象构建Thread类的对象 即构建线程
        new Thread(futureTask).start();
        try {
            // 下方法的返回值即为Callable实现类中call方法的返回值
            Object sum = futureTask.get();
            System.out.println("sum = " + sum);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

### （四）使用线程池

- 通过`Executors`类创建线程池，设定线程池参数，通过线程池提交/执行多线程。
- 该方法要求Java 5以上版本。
- 优势在于方便进行线程管理和调度。

```java
package com.katus.thread;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * 创建多线程的方式 4: 使用线程池 (Java5 新特性)
 * @author katus
 * @version 1.0, 2020-08-10
 */
class NumberThread implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i <= 100; i++) {
            if (i % 2 == 0) {
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        }
    }
}

class NumberThread1 implements Callable {
    @Override
    public Object call() throws Exception {
        for (int i = 0; i <= 100; i++) {
            if (i % 2 != 0) {
                System.out.println(Thread.currentThread().getName() + ":" + i);
            }
        }
        return null;
    }
}

public class ThreadTest3 {
    public static void main(String[] args) {
        // 提供指定线程数量的线程池
        ExecutorService service = Executors.newFixedThreadPool(10);   // ExecutorService 是返回对象实现的接口

        System.out.println(service.getClass());   // 验证实际返回的对象的类

        // 可选设置线程池的属性
        ThreadPoolExecutor executor = (ThreadPoolExecutor) service;   // ThreadPoolExecutor 才是实际返回的对象
        executor.setCorePoolSize(15);
        executor.setMaximumPoolSize(12);
        // executor.setKeepAliveTime(...);

        NumberThread numberThread = new NumberThread();
        service.execute(numberThread);   // 接受Runnable实现类

        NumberThread1 numberThread1 = new NumberThread1();
        service.submit(numberThread1);   // 接受Callable/Runnable实现类

        // 关闭线程池
        service.shutdown();
    }
}
```

## 四、线程安全

### （一）同步代码块

- 一个被`synchronized`关键字修饰的代码块，需要传入一个Object对象作为同步监视器，该Object对象作为是否需要限制的标志，即需要同一个同步监视器的代码块中的代码同时只能有一个线程在运行。
- 同步代码块执行完成之后，相应的同步监视器会被释放。

```java
package com.katus.thread;

/**
 * 线程安全之同步代码块
 * @author katus
 * @version 1.0, 2020-08-10
 */
class TicketSaleWindow implements Runnable {
    private int ticket;
    private final Object lock = new Object();

    public TicketSaleWindow(int number) {
        ticket = number;
    }

    @Override
    public void run() {
        while (true) {
            // 同步代码块 接受同步监视器（锁）任何类的对象 要求多线程的锁是同一个对象
            synchronized (lock) {   // synchronized (this) 最简单的写法   // synchronized (TicketSaleWindow.class)
                if (ticket > 0) {
                    System.out.println(Thread.currentThread().getName() + ":" + ticket--);
                } else {
                    break;
                }
            }
        }
    }
}

public class TicketSale {
    public static void main(String[] args) {
        TicketSaleWindow window = new TicketSaleWindow(250);
        Thread t1 = new Thread(window), t2 = new Thread(window), t3 = new Thread(window);
        t1.start();
        t2.start();
        t3.start();
    }
}
```

### （二）同步方法

+ 同步方法就是具有同步功能的方法，相当于整个方法体都是同步的，方法声明被`synchronized`关键字修饰。
+ 同步方法也是需要同步监视器的，但是是隐性的。
  + 静态同步方法，同步监视器是类名.class。
  + 非静态同步方法，同步监视器是this。
+ 同步方法体执行完成之后会释放同步监视器。

```java
package com.katus.thread;

/**
 * 线程安全之同步方法 本质上就是把同步代码块变成了整个方法
 * @author katus
 * @version 1.0, 2020-08-10
 */
class TicketSaleWindow2 implements Runnable {
    private int ticket;

    public TicketSaleWindow2(int number) {
        ticket = number;
    }

    @Override
    public void run() {
        do {
            show();
        } while (ticket > 0);
    }

    /*要注意同步方法在继承Thread类的方式下要改写成静态方法才能获得线程安全!!! 同步监视器为TicketSaleWindow2.class*/
    private synchronized void show() {   // 非静态同步方法的默认监视器是this
        if (ticket > 0) {
            System.out.println(Thread.currentThread().getName() + ":" + ticket--);
        }
    }
}

public class TicketSale2 {
    public static void main(String[] args) {
        TicketSaleWindow2 window = new TicketSaleWindow2(250);
        Thread t1 = new Thread(window), t2 = new Thread(window), t3 = new Thread(window);
        t1.start();
        t2.start();
        t3.start();
    }
}
```

### （三）AQS框架实现类（API锁）

- `ReentrantLock`类对象，顾名思义，其lock()方法和unlock()方法成对使用，lock()方法调用后，其他线程都无法执行，只有等unlock()方法调用之后才会解锁。
- 该方法要求Java 5以上版本。

```java
package com.katus.thread;

import java.util.concurrent.locks.ReentrantLock;

/**
 * 线程安全之Lock锁 (Java5新特性)
 * @author katus
 * @version 1.0, 2020-08-10
 */
class TicketSaleWindow3 implements Runnable {
    private int ticket;
    private final ReentrantLock lock = new ReentrantLock();

    public TicketSaleWindow3(int number) {
        ticket = number;
    }

    @Override
    public void run() {
        while (true) {
            try {
                lock.lock();
                if (ticket > 0) {
                    System.out.println(Thread.currentThread().getName() + ":" + ticket--);
                } else {
                    break;
                }
            } finally {
                lock.unlock();
            }
        }
    }
}

public class LockTest {
}
```

## 五、线程通信

- notify() notifyAll() wait() 只能用在同步方法和同步代码块中。
- 三个方法的调用者必须是当前同步代码块或者同步方法的同步监视器。
- 三个方法是定义在Object类中的所有的对象均有资格调用。

```java
package com.katus.thread;

/**
 * 线程通信
 * @author katus
 * @version 1.0, 2020-08-10
 */
class NumberPainter implements Runnable {
    private int number;

    public NumberPainter() {
        this(0);
    }

    public NumberPainter(int number) {
        this.number = number;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (this) {
                // 一旦执行此方法 会有一个wait中的阻塞线程被唤醒(选取其中优先级高的线程)
                notify();   // this.notify();
                // 一旦执行此方法 全部wait中的阻塞线程被唤醒
                // notifyAll();
                if (number <= 100) {
                    System.out.println(Thread.currentThread().getName() + ":" + number++);
                } else {
                    break;
                }
                try {
                    // 一旦执行此方法 本线程立刻阻塞 同时释放同步监视器
                    wait();   // this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

public class ThreadCommunicationTest {
    public static void main(String[] args) {
        NumberPainter numberPainter = new NumberPainter();
        Thread t1 = new Thread(numberPainter), t2 = new Thread(numberPainter);

        t1.start();
        t2.start();
    }
}
```
