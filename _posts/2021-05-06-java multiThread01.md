---
title: Java多线程01-进程与线程/线程生命周期/创建线程的几种方式
author: ifuncat
date: 2021-05-06 20:22:22 +0800
categories: [Java核心]
tags: [Java多线程]
---
<style>
img{
    padding-left: 3%;
}
</style>

## 一. 进程与线程

### 1. 进程(Process)的基本概念
- 指一个内存中运行的应用程序, 每个进程都有一个独立的内存空间, 程序进入内存后就称为进程.
- 一个应用程序可以同时运行多个进程, 进程也是程序的一次执行过程, 是系统运行程序的基本单位.
- 系统运行一个程序即为从一个进程创建, 运行到死亡的过程.
  
### 2. 线程的基本概念(Thread)
- 线程是进程中的一个执行单元, 负责当前进程中程序的执行.
- 一个程序至少有一个进程, 一个进程至少由一个线程.

  <img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-thread-01-01.jpg" style="zoom:50%" />

### 3. 线程工作内存与进程内存(主内存)
- Java内存模型规定, 所有变量都存储在主内存中, 每个线程中有自己的工作内存 , 线程的工作内存保存了该线程所使用到的变量, 这些变量都是从主内存中拷贝而来.
- 线程对变量的所有操作, 如读取, 赋值等都必须在工作内存中进行, 不同线程间无法直接访问. 对于线程工作内存中的变量, 多个线程间的变量值的传递需要通过主内存来完成
- 基于上述内存模型, 便产生了多线程编程中的数据"脏读"问题, 如A线程对变量a赋值, 需要先从主内存中拷贝a到自己的工作内存中进行赋值操作, 此时更新后的a还未返回到主内存是, 线程B就从主内存中读取a, 此时B线程读取到的a的值不是最新的, 因此线程不安全.
- 详见: [https://blog.csdn.net/qq877728715/article/details/101547608](https://blog.csdn.net/qq877728715/article/details/101547608)

<img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-thread-01-02.png" style="zoom:100%" />

### 4. 多进程与多线程的区别
- 本质区别在于每个进程都有自己的一套变量, 而同一进程下的多个线程会共享这个进程的变量
- 共享变量的使用让线程间的通信比进程间的通信更为有效, 容易
- 在某些操作系统中, 相较于进程, 线程更为轻量级, 创建/销毁一个线程比启动新进程的开销小得多

### 5. 线程调度
- 大部分操作系统都支持多线程并发运行, 现在的操作系统几乎都支持同时运行多个程序, 此时这些程序是在同时运行着, 感觉这些程序是在同一时刻运行, 实际上CPU使用抢占式调度模式在多个线程间进行着高速的切换
- 对于CPU的一个核而言, 在某个时刻只能运行一个线程, 而CPU在多个线程之间切换速度非常快, 让人感觉是在同一时刻运行, 其实多线程程序并不能提高程序的运行速度, 但能提高程序运行效率, 让CPU的使用效率更高
- 分时调度: 所有线程轮流使用CPU的使用权, 平均分配给每个线程占用CPU的时间
- 抢占式调度: 优先让优先级高的线程使用CPU, 如果线程的优先级相同则会随机选择一个线程, java使用此类调度方式

## 二. 线程的生命周期

### 1. 线程生命周期简介
当线程被创建并启动以后, 它既不是一启动就进入了执行状态, 也不是一直处于执行状态, 在线程的生命周期中, 线程要经过新建(new), 就绪(runnable), 运行(running), 阻塞(blocked)和死亡(dead)五种状态. 尤其是当线程启动后, 它不能一直占着CPU独自运行, CPU在多条线程间进行高速的切换, 于是线程状态也会在运行, 阻塞之间切换

<img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-thread-01-03.png" style="zoom:100%" />

### 2. 新建和就绪状态
- 当程序使用new关键字创建了一个线程后, 该线程即处于新建状态, 此时这个线程对象和其他java对象一样, 仅仅由JVM为其分配了内存并初始化了成员变量值, 但未表现出任何线程的动态特征, 即不会执行任何动作
- 当线程对象调用了start()后, 线程处于就绪状态, JVM会为其创建方法调用栈和程序计数器, 处于此状态的线程并未开始运行, 只是表示这个线程可以运行了, 至于何时开始运行, 取决于JVM中线程调度器的调度, 线程获得CPU的时间片后才开始执行

### 3. 运行状态
- 如果处于就绪状态的线程经JVM调度后拿到了CPU的时间片, 则开始执行run方法的线程执行体, 此时线程处于运行状态
- 当发生如下的某一状态, 线程将会进入阻塞状态:
  - 线程调用sleep方法, 主动放弃所占用的CPU资源
  - 线程调用了一个阻塞式IO方法, 如读取文件IO流等，在该方法返回前，线程将会被阻塞
  - 线程试图获得一个同步监视器，但该同步监视器被其他线程锁持有
  - 线程在等待某个通知
  - 程序调用了线程的suspend方法将该线程挂起, 该方法容易导致死锁, 不建议使用该方法

### 4. 阻塞状态
- 当前正在执行的线程被阻塞后, 其他线程就可以获得执行的机会了. 被阻塞的线程会在合适的时候重写进入就绪状态, 注意是就绪状态而非运行状态, 也就是说被阻塞的线程的阻塞解除后, 必须重新等待线程调度器再次调度它
- 针对以上运行状态进入阻塞状态的几种情况, 当发生如下情况线程则会从阻塞状态重新进入就绪状态
  - 调用sleep方法的线程经过了指定的sleep时间
  - 线程调用的阻塞式IO方法已经返回, 如IO读取文件完毕获得了字节流等
  - 线程成功获得了同步监视器
  - 线程正在等待某个通知时, 其他线程发出了一个通知
  - 处于挂起状态的线程被调用了resume恢复方法

### 5. 死亡状态
  线程会以以下三种方式之一结束, 从而处于死亡状态
- run() 执行完毕, 线程正常结束
- 线程抛出一个未捕获的exception或error
- 直接调用该线程的stop()来结束该线程, 此方法容易导致死锁,不推荐使用

## 三. 线程创建的几种方式
### 1. 继承Thread类
```java
public class ReadBookThread extends Thread{
    @Override
    public void run() { //重点在于重写run方法
        System.out.println(this.getName()+" readBook");
    }
}

@Test
public void testThreadSubClass(){
    ReadBookThread t1 = new ReadBookThread();
    t1.setName("t1");
    t1.start();
}
```
### 2. 实现Runnable接口
```java
public class ReadBookRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " readBook....");
    }
}

@Test
public void testRunnableImpl(){
    new Thread(new ReadBookRunnable(),"t1").start();
}
```
### 3. 使用匿名内部类/lambda表达式
```java
@Test
public void testLambda() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " readBook....");
        }
    }, "t1").start();

    new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " readBook....");
    }, "t2").start();
}
```
### 4. Callable + Future接口
会返回线程的执行结果, 如果有的话会抛出异常
```java
public class ReadBookCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName()+" readBook...");
        return Integer.MAX_VALUE;
    }
}

@Test
public void testCallableFutureTask() throws Exception {
    ReadBookCallable callable = new ReadBookCallable();
    FutureTask<Integer> futureTask1 = new FutureTask<>(callable);

    //匿名内部类或lambda表达式实现Callable接口
    FutureTask<Integer> futureTask2 = new FutureTask<>(() -> {
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName() + " readBook...");
        return Integer.MAX_VALUE - 1;
    });

    new Thread(futureTask1, "t1").start();
    new Thread(futureTask2, "t2").start();

    Thread.sleep(1000); //main线程先干点别的
    Integer result1 = futureTask1.get(); //接收t1线程执行结果,并且抛出异常,如果有的话
    Integer result2 = futureTask2.get();
    System.out.println("t1 result: " + result1);
    System.out.println("t2 result: " + result2);
}
```
### 5. 定时器创建线程
```java
@Test
public void testTimer() {
    Timer timer = new Timer();
    timer.schedule(new TimerTask() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " timerTask....");
        }
    }, 0, 2000); //延迟0ms开始执行一次, 每隔2000ms执行一次
}
```
### 6. 线程池
推荐使用线程池, 便于线程管理且节约线程资源
```java
@Test
public void testThreadPool() throws Exception {
    //executor作为线程池,可以使用spring提供的ThreadPoolTaskExecutor,此处在before方法中初始化了一个ThreadPoolTaskExecutor对象
    executor.execute(() -> { //传入一个runnable对象,无返回值,不抛出异常
        System.out.println(Thread.currentThread().getName() + " runnable...");
    });

    Future<Integer> future = executor.submit(() -> { //传入一个callable对象,有返回值,抛出异常如果有的话
        System.out.println(Thread.currentThread().getName() + " callable...");
        return Integer.MAX_VALUE;
    });

    Thread.sleep(1000); //main线程休息1s

    Integer result = future.get();
    System.out.println("future result: " + result);
}
```
## 四. 注意点
### 1. run()与start()
  创建并运行一个线程所犯的常见错误是调用线程的run()方法而非start()方法，如下所示, run方法确实被调用了, 但是调用的线程不是new出来的t1线程, 而是被当前线程即main线程所执行. 想让t1线程执行, 必须调用start()
```java
public static void main(String[] args) {
    new Thread(()-> System.out.println("do something"),"t1").run();
}
```
### 2. 线程的执行顺序问题
  如下所示依次初始化并启动多个线程, 但是各个线程的实际执行顺序却是随机的, 这是由于操作系统和JVM共同决定的, 默认情况下JVM的线程调度机制为随机, 每个线程获得CPU时间片的概率随机, 从而执行顺序随机. 但可以设置成按照优先级执行, 优先级高的线程获得CPU时间片的概率大, 但是不代表优先级高的线程一定先于优先级低的线程先执行
```java
@Test //顺序随机,无法复现多执行几次
public void testExecuteSequence() {
    for (int i = 0; i < 10; i++) {
        new Thread(() -> System.out.println(Thread.currentThread().getName() + " do something"), i + "").start();
    }
}
```