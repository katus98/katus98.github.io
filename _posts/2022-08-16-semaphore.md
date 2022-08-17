---
title: 信号量机制
author: katus
date: 2022-08-16 00:54:08 +0800
categories: [Operating System]
tags: [Operating System, Java, Multiprocessing, Multithreading]
typora-root-url: ..
---

## 一、信号量概述

### 1.概念

用户进程可以通过使用操作系统提供的一对原语来对信号量进行操作，从而很方便的实现了进程互斥、进程同步。

信号量其实就是一个变量（可以是一个整数，也可以是更复杂的记录型变量），可以用一个信号量来表示系统中某种资源的数量，比如：系统中只有一台打印机，就可以设置一个初值为1的信号量。

原语是一种特殊的程序段，其执行只能一气呵成，不可被中断。原语是由关中断和开中断指令实现的。

一对原语：`wait(S)`原语和`signal(S)`原语，可以把原语理解为我们自己写的函数，函数名分别为`wait`和`signal`，括号里的信号量`S`其实就是函数调用时传入的一个参数。`wait`、`signal`原语常简称为`P`、`V`操作（来自荷兰语`proberen`和`verhogen`）。因此，常把`wait(S)`、`signal(S)`两个操作分别写为`P(S)`、`V(S)`。

### 2.类别

+ 整型信号量

  + 用一个整型变量作为信号量，表示系统中某种资源的数量。

    ```c
    int S = 1;   // 初始化信号量, 表示某种资源的剩余数
    // 注意wait与signal都是通过原语实现的, 是原子性的, 这里仅仅是通过C语言模拟
    void wait(int S) {
        while (S <= 0);   // 如果不满足需求, 则忙等待
        S = S - 1;
    }
    // 注意wait与signal都是通过原语实现的, 是原子性的, 这里仅仅是通过C语言模拟
    void signal(int S) {
        S = S + 1;
    }
    ```

  + 进程调用逻辑。

    ```c
    wait(S);   // P操作
    useResource();   // 使用临界资源
    signal(S);   // V操作
    ```

  + 由于只有单一的整型信号量，只能通过忙等待阻塞进程，不满足“让权等待”的原则。

+ 记录型信号量

  + 整型信号量存在“忙等”问题，因此人们又提出了“记录型信号量”，即用记录型数据结构表示的信号量。

    ```c
    typedef struct {
        int value;
        Struct process *L;
    } Semaphore;
    // 注意wait与signal都是通过原语实现的, 是原子性的, 这里仅仅是通过C语言模拟
    void wait(Semaphore S) {
        S.value--;
        if (S.value < 0) {   // 无法获取到资源
            block(S.L);   // 进程调用block原语进行自我阻塞
        }
    }
    // 注意wait与signal都是通过原语实现的, 是原子性的, 这里仅仅是通过C语言模拟
    void signal(Semaphore S) {
        S.value++;
        if (S.value <= 0) {   // 如果当前进程释放资源之后value仍不大于0, 说明有其他进程在排队
            wakeup(S.L);   // 调用wakeup原语唤醒等待队列中的第一个进程
        }
    }
    ```

  + 可以解决让权等待的问题；可以实现进程互斥和进程同步。

## 二、信号量机制实现进程互斥、同步与前驱

### 1.进程互斥

进程互斥可以通过`mutex`信号量的`P`、`V`操作将临界区包裹来实现。

```c
Semaphore mutex = new Semaphore(1);
void process {
    wait(mutex);
    criticalArea();
    signal(mutex);
}
```

### 2.进程同步

信号量初始化为0，需要先执行的进程代码执行完之后执行信号量`V`操作，后执行进程代码执行之前调用信号量`P`操作。

```c
Semaphore s = new Semaphore(0);
void processA {
    // 进程A临界区先执行
    criticalAreaA();
    signal(s);
}
void processB {
    wait(s);
    // 进程B临界区后执行
    criticalAreaB();
}
```

### 3.进程前驱

实际上是多个进程形成有向无环图，产生一个具有先后关系多进程同步情况。可以为每一对前驱关系都按照进程同步的方式设计一个信号量。

## 三、信号量机制解决经典进程问题

> 以下例子均通过Java多线程的方式模拟多进程

### 1.单生产者-消费者问题

有一类生产者生产一种产品，一类消费者消费同一种产品。生产者将生产的产品放入一个有限缓冲区，消费者从这个缓冲区内取走产品消费。现在需要保证各方对缓冲区的操作是互斥的，同时缓冲区满了则暂停生产，缓冲区空了则暂停消费。

解法：增删缓冲区的操作作为临界区互斥，同时将缓冲区空位的个数和产品的个数都作为资源信号量引导进程同步。

```java
package com.katus;

import lombok.extern.slf4j.Slf4j;

import java.util.Random;
import java.util.concurrent.Semaphore;

/**
 * 通过信号量机制实现的单生产者-消费者模型
 *
 * @author SUN Katus
 * @version 1.0, 2022-08-12
 */
@Slf4j
public class SingleProducerConsumer {
    private static final int CAPACITY = 5, BOUND = 20;
    private final Random random;
    private final Buffer buffer;
    private final Semaphore mutex, full, empty;

    public SingleProducerConsumer() {
        this.random = new Random();
        this.buffer = new Buffer();
        this.mutex = new Semaphore(1);
        this.full = new Semaphore(0);
        this.empty = new Semaphore(CAPACITY);
    }

    public class Producer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    int product = random.nextInt(BOUND);
                    Thread.sleep(100);
                    // 互斥操作必须在同步操作之后, 否则会死锁 (保证只有满足了同步关系才允许访问临界资源)
                    empty.acquire();
                    mutex.acquire();
                    buffer.write(product);
                    log.info("Produced a product: {}", product);
                    mutex.release();
                    full.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class Consumer implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    full.acquire();
                    mutex.acquire();
                    int product = buffer.read();
                    log.info("Consumed a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public static class Buffer {
        private final int[] buffer;
        private int index;

        public Buffer() {
            this.buffer = new int[CAPACITY];
            this.index = 0;
        }

        public boolean isFull() {
            return index == CAPACITY;
        }

        public boolean isEmpty() {
            return index == 0;
        }

        public void write(int product) {
            buffer[index++] = product;
        }

        public int read() {
            return buffer[--index];
        }
    }

    public static void main(String[] args) {
        SingleProducerConsumer singleProducerConsumer = new SingleProducerConsumer();
        new Thread(singleProducerConsumer.new Consumer(), "Consumer").start();
        new Thread(singleProducerConsumer.new Producer(), "Producer").start();
    }
}
```

### 2.多生产者-消费者问题

多类生产者消费者生产、消费不同的产品，但是共享同一个有限缓冲区。现在需要保证各方对缓冲区的操作是互斥的，同时能够满足各类生产者消费者正常生产消费。

解法：增删缓冲区的操作作为临界区互斥，同时将缓冲区空位的个数和各类产品的个数都作为资源信号量引导进程同步。

```java
package com.katus;

import lombok.ToString;
import lombok.extern.slf4j.Slf4j;

import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.Semaphore;

/**
 * 通过信号量机制实现的多类别生产者-消费者模型
 *
 * @author SUN Katus
 * @version 1.0, 2022-08-12
 */
@Slf4j
public class MultiProducerConsumer {
    private static final int CAPACITY = 5, BOUND = 20;
    private final Buffer buffer;
    private final Semaphore mutex, a, b, empty;

    public MultiProducerConsumer() {
        this.buffer = new Buffer();
        this.mutex = new Semaphore(1);
        this.a = new Semaphore(0);
        this.b = new Semaphore(0);
        this.empty = new Semaphore(CAPACITY);
    }

    public class ProducerA implements Runnable {
        @Override
        public void run() {
            int i = 0;
            while (true) {
                try {
                    Thread.sleep(100);
                    Product product = new ProductA(i);
                    i = (i + 1) % BOUND;
                    empty.acquire();
                    mutex.acquire();
                    buffer.write(product);
                    log.info("Produced a product: {}", product);
                    mutex.release();
                    a.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class ProducerB implements Runnable {
        @Override
        public void run() {
            int i = 0;
            while (true) {
                try {
                    Thread.sleep(100);
                    Product product = new ProductB(i);
                    i = (i + 1) % BOUND;
                    empty.acquire();
                    mutex.acquire();
                    buffer.write(product);
                    log.info("Produced a product: {}", product);
                    mutex.release();
                    b.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class ConsumerA implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    a.acquire();
                    mutex.acquire();
                    Product product = buffer.readByType("A");
                    log.info("Consumed a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class ConsumerB implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    b.acquire();
                    mutex.acquire();
                    Product product = buffer.readByType("B");
                    log.info("Consumed a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public static class Buffer {
        private final Set<Product> set;

        public Buffer() {
            this.set = new HashSet<>();
        }

        public void write(Product product) {
            set.add(product);
        }

        public Product readByType(String productName) {
            for (Product product : set) {
                if (productName.equals(product.getName())) {
                    set.remove(product);
                    return product;
                }
            }
            return null;
        }
    }

    public static abstract class Product {
        public abstract String getName();
    }

    @ToString
    public static class ProductA extends Product {
        private final int id;

        public ProductA(int id) {
            this.id = id;
        }

        @Override
        public String getName() {
            return "A";
        }
    }

    @ToString
    public static class ProductB extends Product {
        private final int id;

        public ProductB(int id) {
            this.id = id;
        }

        @Override
        public String getName() {
            return "B";
        }
    }

    public static void main(String[] args) {
        MultiProducerConsumer multiProducerConsumer = new MultiProducerConsumer();
        new Thread(multiProducerConsumer.new ProducerA(), "ProducerA").start();
        new Thread(multiProducerConsumer.new ProducerB(), "ProducerB").start();
        new Thread(multiProducerConsumer.new ConsumerA(), "ConsumerA").start();
        new Thread(multiProducerConsumer.new ConsumerB(), "ConsumerB").start();
    }
}
```

### 3.吸烟者问题

一类生产者（提供者）生产多种产品，多类消费者（吸烟者）消费各自的产品类型，使用同一个共享缓冲区。需要保证各方对缓冲区的操作是互斥的，同时能够满足生产者和各类消费者正常生产消费。

解法：增删缓冲区的操作作为临界区互斥，同时将缓冲区空位的个数和各类产品的个数都作为资源信号量引导进程同步。

```java
package com.katus;

import lombok.ToString;
import lombok.extern.slf4j.Slf4j;

import java.util.HashSet;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.Semaphore;

/**
 * 通过信号量解决吸烟者问题(单生产者生产多种产品, 多种消费者消费不同的产品)
 *
 * @author SUN Katus
 * @version 1.0, 2022-08-12
 */
@Slf4j
public class ProviderSmoker {
    private static final int CAPACITY = 5, BOUND = 20;
    private final Buffer buffer;
    private final Semaphore mutex, a, b, c, empty;

    public ProviderSmoker() {
        this.buffer = new Buffer();
        this.mutex = new Semaphore(1);
        this.a = new Semaphore(0);
        this.b = new Semaphore(0);
        this.c = new Semaphore(0);
        this.empty = new Semaphore(CAPACITY);
    }

    public class Provider implements Runnable {
        private final Random random;

        public Provider() {
            this.random = new Random();
        }

        @Override
        public void run() {
            int i = 0;
            while (true) {
                try {
                    Thread.sleep(50);
                    int r = random.nextInt(3);
                    Product product;
                    switch (r) {
                        case 0:
                            product = new ProductA(i);
                            break;
                        case 1:
                            product = new ProductB(i);
                            break;
                        default:
                            product = new ProductC(i);
                    }
                    i = (i + 1) % BOUND;
                    empty.acquire();
                    mutex.acquire();
                    buffer.write(product);
                    log.info("Provide a product: {}", product);
                    mutex.release();
                    switch (r) {
                        case 0:
                            a.release();
                            break;
                        case 1:
                            b.release();
                            break;
                        default:
                            c.release();
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class SmokerA implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    a.acquire();
                    mutex.acquire();
                    Product product = buffer.readByType("A");
                    log.info("Smoke a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class SmokerB implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    b.acquire();
                    mutex.acquire();
                    Product product = buffer.readByType("B");
                    log.info("Smoke a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class SmokerC implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    c.acquire();
                    mutex.acquire();
                    Product product = buffer.readByType("C");
                    log.info("Smoke a product: {}", product);
                    mutex.release();
                    empty.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public static class Buffer {
        private final Set<Product> set;

        public Buffer() {
            this.set = new HashSet<>();
        }

        public void write(Product product) {
            set.add(product);
        }

        public Product readByType(String productName) {
            for (Product product : set) {
                if (productName.equals(product.getName())) {
                    set.remove(product);
                    return product;
                }
            }
            return null;
        }
    }

    public static abstract class Product {
        public abstract String getName();
    }

    @ToString
    public static class ProductA extends Product {
        private final int id;

        public ProductA(int id) {
            this.id = id;
        }

        @Override
        public String getName() {
            return "A";
        }
    }

    @ToString
    public static class ProductB extends Product {
        private final int id;

        public ProductB(int id) {
            this.id = id;
        }

        @Override
        public String getName() {
            return "B";
        }
    }

    @ToString
    public static class ProductC extends Product {
        private final int id;

        public ProductC(int id) {
            this.id = id;
        }

        @Override
        public String getName() {
            return "C";
        }
    }

    public static void main(String[] args) {
        ProviderSmoker providerSmoker = new ProviderSmoker();
        new Thread(providerSmoker.new Provider(), "Provider").start();
        new Thread(providerSmoker.new SmokerA(), "SmokerA").start();
        new Thread(providerSmoker.new SmokerB(), "SmokerB").start();
        new Thread(providerSmoker.new SmokerC(), "SmokerC").start();
    }
}
```

### 4.读者-写者问题

多个读写进程对同一文件资源进行读写，需要保证读进程之间可以并发执行，而写进程只能单独执行。

解法：通过一个整型变量记录当前有多少个读进程正在读，一个互斥信号量保证对前述整型变量的操作是原子性的；一个读写信号量表示读写资源只能被各种读进程或者一个写进程占用，一个写信号量保证写进程不会饿死。

```java
package com.katus;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.Semaphore;

/**
 * 用信号量解决读者-写者问题
 *
 * @author SUN Katus
 * @version 1.0, 2022-08-12
 */
@Slf4j
public class ReaderWriter {
    private static final int BOUND = 20;
    /**
     * 保证记录读进程数量的操作互斥(保证对readCount的操作是原子化的, 当然也可以使用原子类)
     */
    private final Semaphore mutex;
    /**
     * 保证写进程和其他进程之间的互斥关系
     */
    private final Semaphore rw;
    /**
     * 保证写进程不会饿死
     */
    private final Semaphore w;
    /**
     * 读进程正在读的数量
     */
    private int readCount;

    public ReaderWriter() {
        this.mutex = new Semaphore(1);
        this.rw = new Semaphore(1);
        this.w = new Semaphore(1);
        this.readCount = 0;
    }

    public class Reader implements Runnable {
        @Override
        public void run() {
            int i = 0;
            while (true) {
                try {
                    Thread.sleep(50);
                    w.acquire();
                    mutex.acquire();
                    if (readCount == 0) {
                        rw.acquire();
                    }
                    readCount++;
                    mutex.release();
                    w.release();
                    log.info("Reading {} ...", i);
                    i = (i + 1) % BOUND;
                    mutex.acquire();
                    if (readCount == 1) {
                        rw.release();
                    }
                    readCount--;
                    mutex.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public class Writer implements Runnable {
        @Override
        public void run() {
            int i = 0;
            while (true) {
                try {
                    Thread.sleep(100);
                    w.acquire();
                    rw.acquire();
                    log.info("Writing {} ...", i);
                    i = (i + 1) % BOUND;
                    rw.release();
                    w.release();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public static void main(String[] args) {
        ReaderWriter readerWriter = new ReaderWriter();
        new Thread(readerWriter.new Reader(), "Reader").start();
        new Thread(readerWriter.new Writer(), "Writer").start();
    }
}
```

### 5.哲学家进餐问题

五个哲学家围着一个圆桌吃饭，总计五支筷子，分别放在每个哲学家之间。每个哲学家只能思考或者吃饭，吃饭需要同时拿取左右两边的筷子。

解法：规定哲学家获取筷子的顺序，比如要求偶数号哲学家先拿左筷子，而奇数号哲学家先拿右筷子。

```java
package com.katus;

import lombok.ToString;
import lombok.extern.slf4j.Slf4j;

import java.util.Random;
import java.util.concurrent.Semaphore;

/**
 * 用信号量解决哲学家进餐问题 (预防死锁)
 * * 策略1 总计N个哲学家, 最多只允许N-1个哲学家同时吃饭
 * * 策略2 要求偶数编号哲学家和奇数编号哲学家拿筷子的顺序相反 (本实现)
 * * 策略3 将哲学家拿筷子的过程互斥
 *
 * @author SUN Katus
 * @version 1.0, 2022-08-12
 */
@Slf4j
public class PhilosopherMeal {
    private static final int CAPACITY = 5;
    private final Semaphore[] chopsticks;
    private final Random random;

    public PhilosopherMeal() {
        this.chopsticks = new Semaphore[CAPACITY];
        for (int i = 0; i < CAPACITY; i++) {
            chopsticks[i] = new Semaphore(1);
        }
        this.random = new Random();
    }

    @ToString
    public class Philosopher implements Runnable {
        private final int id;

        public Philosopher(int id) {
            this.id = id;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(100);
                    boolean meal = random.nextBoolean();
                    if (meal) {
                        int rightId = (id + 1) % CAPACITY;
                        if (id % 2 == 0) {
                            chopsticks[id].acquire();
                            chopsticks[rightId].acquire();
                        } else {
                            chopsticks[rightId].acquire();
                            chopsticks[id].acquire();
                        }
                        log.info("{} is eating...", this);
                        chopsticks[id].release();
                        chopsticks[rightId].release();
                    } else {
                        log.info("{} is thinking...", this);
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public static void main(String[] args) {
        PhilosopherMeal philosopherMeal = new PhilosopherMeal();
        for (int i = 0; i < CAPACITY; i++) {
            new Thread(philosopherMeal.new Philosopher(i), "Philosopher-" + i).start();
        }
    }
}
```

