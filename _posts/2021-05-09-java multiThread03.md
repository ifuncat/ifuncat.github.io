---
title: Java并发03-守护线程/线程的中断
author: ifuncat
date: 2021-05-09 10:22:22 +0800
categories: [Java核心]
tags: [Java并发]
---
<style>
img{
    padding-left: 3%;
}
</style>
## 一. 守护线程

### 1. 什么是守护线程
- 简单来说就是为其他线程服务的线程.
- 也可以称之为后台线程, 非用户线程, 即随系统结束而结束的线程.
- JVM中, 所有非守护线程都执行完毕后, 无论是否存在守护线程, JVM都会自动退出
- 编写代码时注意: 守护线程不能持有任何需要关闭的资源, 如IO流等, 因为JVM退出时, 守护线程没有机会来释放资源, 会导致数据丢失.

### 2. 代码演示
```java
@Test
public void testDaemon(){
    Thread t1 = new Thread(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("t1 running...");
    });
    t1.setDaemon(true); //设置为守护线程, 默认为非守护线程
    t1.start();
}
```

## 二. 线程的中断

### 1. 线程中断说明
- 如果需要线程来执行一个长时间的任务, 可能需要能够在执行过程中中断这个线程的机制. 例如下载一个下载速度很慢的文件, 想要中断这次下载, 换个资源再下载, 这时程序就要中断下载线程的执行
- interrupt()方法并不是中断线程, 只是将线程的中断标志位设置为true.
- 对于非阻塞的线程, 执行interrupt()方法只是将线程的中断标志位设置为true, 但是该线程的状态不会改变, 如果原来线程状态为运行中, 那么这个线程还会执行下去, 详见非阻塞线程的interrupt操作.
- 对于阻塞的线程, 如执行了sleep() / join() / wait()方法的线程, 对该线程执行interrupt()会产生一个InterruptedException, 而且会清除掉线程中断标志位, 即此时中断标志位的值仍为默认的旧值false(表示未中断), 也不会结束该线程而是会一直执行下去, 也就是处于阻塞状态的线程被调用了interrupted()方法非但没有中断该线程, 还会产生一个InterruptedException.

### 2. 非阻塞线程的interrupt操作
```java
@Test
public void testInterrupt() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        boolean interruptFlag = Thread.currentThread().isInterrupted(); //线程是否中断的标志位
        for (int i = 0; i < 1000; i++) { //循环打印1000次
            System.out.println("t1 add to " + i + " interrupted: " + interruptFlag);
        }
    }, "t1");

    Thread t2 = new Thread(() -> {
        try {
            Thread.sleep(2);  //t2线程执行1ms后开始执行中断t1操作
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("t2 start interrupt t1...");
        t1.interrupt();
    }, "t2");

    t1.start();
    t2.start();

    Thread.sleep(5000); //暂停主线程,便于观察结果
}

//执行结果如下
//t1 add to 348 interrupted: false
//t1 add to 349 interrupted: false
//t1 add to 350 interrupted: false
//t2 start interrupt t1...
//t1 add to 351 interrupted: false
//t1 add to 352 interrupted: false
//t1 add to 353 interrupted: false
//t1 add to 354 interrupted: false
//...... //直到add to 999, t1线程执行完全,不会中断
```
执行结果分析:
- t1线程循环1000次并打印index, t2线程则在其开始执行的第2ms后触发中断t1操作, 此时t1线程并未立即执行停止, 而是一直执行完全, 打印了1000次
- 解释: t1.interrupt()方法只是让t1线程的中断标志位变为了true, 并未中断t1线程, 所以t1线程还是保持原来的运行中状态, 就会一直执行下去

### 3. 非阻塞线程的interrupt()+isInterrupted()控制while循环
```java
@Test
public void testWhileInterrupt() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) { //当线程中断标志位不为true时,循环执行打印
            System.out.println("t1 is running....");
        }
        System.out.println("t1 end...");
    }, "t1");

    Thread t2 = new Thread(() -> {
        try {
            Thread.sleep(1);  //t2线程执行1ms后开始执行中断t1操作
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("t2 start interrupt t1...");
        t1.interrupt();
    }, "t2");

    t1.start();
    t2.start();

    Thread.sleep(1000); //暂停主线程,便于观察结果
}
```
执行结果分析
- 当线程中断标志位不为true时, t1线程会while循环打印. 
- t2线程在执行到2ms时触发执行t1.interrupt(), 此时t1中断标志位变为true, 从而跳出while循环, t1执行结束.

### 4. 阻塞线程的interrupt()+isInterrupted()控制while循环
```java
@Test
public void testBlockWhileInterrupt() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) { //如果当前线程没被中断，则一直进行
            try {
                Thread.sleep(100); //每次循环时休眠100ms, 转为阻塞状态
            } catch (InterruptedException e) { //虽然抛出异常但是在循环内部, 因此不会跳出循环或者说结束循环
                e.printStackTrace();
            }
            System.out.println("t1 is running....");
        }
        System.out.println("t1 end...");
    }, "t1");

    Thread t2 = new Thread(() -> {
        try {
            Thread.sleep(800);  //t2线程执行1ms后开始执行中断t1操作
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("t2 start interrupt t1...");
        t1.interrupt();
    }, "t2");

    t1.start();
    t2.start();

    Thread.sleep(2000); //暂停主线程,便于观察结果
}

//执行结果如下
//t1 is running....
//t1 is running....
//t1 is running....
//t2 start interrupt t1...
//java.lang.InterruptedException: sleep interrupted
//	at java.lang.Thread.sleep(Native Method)
//	at com.ifuncat.demo.test.normal.javacore.multithread.b_basefunction.JoinDemo.//lambda$testBlockWhileInterrupt$11(JoinDemo.java:182)
//	at java.lang.Thread.run(Thread.java:748)
//t1 is running....
//t1 is running....
//t1 is running....
//....//无限循环下去
```
执行结果分析:
- 线程中断标志位是默认的false时, t1线程会执行无需循环操作, 每次循环都会休眠100ms, 此时线程状态由running -> block.
- t2线程在执行800ms后执行中断t1线程的操作, 由于t1此时处于阻塞状态, 会抛出一个InterruptedException, 虽然抛出异常但是在循环内部, 因此不会跳出循环或结束循环, 另外t1的中断标志位仍为默认的false, 故t1会一直处于while循环下. 也就无法达到控制while循环的目的.

### 5. 阻塞线程的interrupt()+isInterrupted()控制while循环的正确做法
```java
@Test
public void testCorrectBlockWhileInterrupt() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        try { //在while循环外进行try/catch, 当发生异常时,会跳出while循环
            while (!Thread.currentThread().isInterrupted()) {
                Thread.sleep(100);
                System.out.println("t1 is running....");
            }
        } catch (InterruptedException e) {
            e.printStackTrace(); //可选择注释掉
            //处理发生
            System.out.println("t1 end...");
        }
    }, "t1");

    Thread t2 = new Thread(() -> {
        try {
            Thread.sleep(800);  //t2线程执行1ms后开始执行中断t1操作
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("t2 start interrupt t1...");
        t1.interrupt();
    }, "t2");

    t1.start();
    t2.start();

    Thread.sleep(2000); //暂停主线程,便于观察结果
}

//执行结果如下:
//t1 is running....
//t1 is running....
//t1 is running....
//t2 start interrupt t1...
//java.lang.InterruptedException: sleep interrupted
//	at java.lang.Thread.sleep(Native Method)
//	at com.ifuncat.demo.test.normal.javacore.multithread.b_basefunction.JoinDemo.lambda$testCorrectBlockWhileInterrupt$13(JoinDemo.java:212)
//	at java.lang.Thread.run(Thread.java:748)
//t1 end...
```
执行结果分析:
- 执行逻辑与阻塞线程的interrupt()+isInterrupted()控制while循环中几乎一致, 不同之处在于正确的做法是在while循环外进行try / catch.
- 这样做的好处是当发生了InterruptedException, 会跳出这个while循环, 线程执行结束. 而如果在while循环内部进行try / catch则不会跳出循环, 线程就会一直执行下去.
- 因此正确做法是对整个可能产生InterruptedException的代码块进行try / catch.