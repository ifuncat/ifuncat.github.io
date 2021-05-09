---
title: Java多线程03-Thread.join/sleep/yield方法详解
author: ifuncat
date: 2021-05-09 10:22:22 +0800
categories: [Java核心]
tags: [Java多线程]
---
<style>
img{
    padding-left: 3%;
}
</style>
## 一. Thread.join(): 线程插队

### 1. join()的理解
- 源码中的注释: waits for this thread to die, 等待这个线程结束
- join()方法更形象的说法: 线程插队, 等插队的线程执行完毕, 被插队的才继续执行
- 举例说明: 程序中有两个线程t1, t2, t1的run()方法中调用了t2.join(), 则t1线程会在此段代码`t2.join()`执行后退出, 即暂停t1的run()方法中`t2.join()`这段代码的下面的代码, 直到t2线程执行完毕后, t1线程才会执行后续的未完代码.


### 2.join()的使用
#### 示例场景
- 程序中有两线程t1, t2, t1独立运行耗时100ms, t2独立运行耗时200ms. t2对共享变量config进行赋值, t1读取config的值.
- 期待结果: t1读取到的config的值是t2赋值后的新值.
- 问题: 不做处理情况下,t1耗时短读取到的config必然是旧值. 即便两个线程耗时相同, t1, t2操作共享变量会产生同步问题, t1可能先于t2执行, 读取到的仍然是config的旧值.
- 解决: t2在t1线程中插队, 因此需要等待t2执行完毕,t1继续执行,此时共享变量是t2修改后的新值

#### 代码演示
```java
public class JoinDemo {
    private static Integer config = 0; //共享变量

    @Test //一般情况下,t1由于耗时短,不等t2赋值完毕, t1已经执行完毕,读取config为旧值
    public void testNoJoin() throws InterruptedException {
        Thread t2 = new Thread(() -> {
            try {
                Thread.sleep(200); //模拟t2耗时200ms
                config = 1; //t2线程赋值config
                System.out.println("t2 set config: " + config);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t2");

        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(100); //模拟t1耗时100ms
                System.out.println("t1 get config: " + config); //读取共享变量的值
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t1");

        t1.start();
        t2.start();

        Thread.sleep(1000); //模拟主线程暂停,便于查看执行效果
    }
    //打印结果: t1 get config: 0   t2 set config: 1

    @Test//t1线程中,t2线程插队,因此t1线程执行到 t2.join(); 代码时停下等待, 直到t2执行完毕,此时config已经时t2修改后的新值
    public void testInJoin() throws InterruptedException {
        Thread t2 = new Thread(() -> {
            try {
                Thread.sleep(200); //模拟t2耗时200ms
                config = 1; //t2线程赋值config
                System.out.println("t2 set config: " + config);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t2");

        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(100); //模拟t1耗时100ms
                t2.join(); //t1线程中,t2线程插队
                System.out.println("t1 get config: " + config);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "t1");

        t1.start();
        t2.start();

        Thread.sleep(1000); //模拟主线程暂停,便于查看执行效果
    }

    //打印结果: t1 get config: 1   t2 set config: 1
}
```
### 3.join()的用途
确保能够完整的获得插队线程的处理结果<br/>
以上示例中可以看出, 如果t1需要读取共享变量经过t2更新后的新值, 需要先让t2完成操作, 故需让t2插队, t1才能读到新值.

## 二. Thread.yield(): 线程退让
### 1. yield()的理解
- 当前线程提示资源调度器, 该线程愿意放弃其当前正在使用的CPU资源, 即停止执行.
- 资源调度器可以选择忽视这个请求提示, 则请求的线程将继续执行.
- yield()方法执行成功, 则这个线程的状态由runnable(运行) -> running(就绪), 否则仍为runnable.
- 举例说明: 有两个线程t1,t2, 两者优先级一致都为默认的5, 在t1的run()方法中调用了Thread.yield(), 请求让出CPU资源, 让其他同等级的线程如t2先执行, 如果请求被忽略, t1继续执行, 如果请求允许, 则t2得到CPU时间片开始执行

### 2.yield()的使用
#### 示例场景
- 驾校里有四名学员分别为t1, t2, t3, t4, 其中t1的优先练车的级别最高为1, t4优先级最低为10, t2/t3的优先级相等均为5.
- t1 优先级最高,最先练车, 且不能将练车机会让给其他低级别的学员. t4优先级最低, 只能等其他人练完后才能开始练车.
- t2/t3优先级相同可以礼让练车机会给对方.

#### 代码演示
```java
@Test
public void testYield() {
    //定义练车行为
    Runnable drive = () -> {
        for (int i = 0; i < 5; i++) {
            if (i == 3) { //当这个学员练习3次开车之后，尝试把机会让给同级别的学员
                System.out.println(Thread.currentThread().getName() + " try to yield..");
                Thread.yield(); //当前线程请求退让CPU资源, 是否允许由CPU决定
            }
            //这个学员正在练习开车
            System.out.println(Thread.currentThread().getName() + " is practicing driving....");
        }
    };
    //定义4个学员,并设置优先级
    Thread t1 = new Thread(drive, "t1");
    Thread t2 = new Thread(drive, "t2"); //t2,t3默认优先级5
    Thread t3 = new Thread(drive, "t3");
    Thread t4 = new Thread(drive, "t4");
    t1.setPriority(Thread.MAX_PRIORITY); //最高优先级10
    t4.setPriority(Thread.MIN_PRIORITY); //最低优先级1

    //开始练车
    t1.start();
    t2.start();
    t3.start();
    t4.start();
}
```
#### 结果分析
可能实际情况并不是预想的: t1先执行完毕, t2, t3交叉执行, t4最后执行, 而是杂乱无章的. 原因如下: 
- 线程请求退让CPU资源, 并不一定会被允许, 可能被否而继续执行. 因此t2线程请求退让时被否而继续执行.
- 线程的优先级高低只是表明抢占CPU资源的概率的大小, 并不表明优先级高的线程一定会先执行, 因此t4可能最先执行

### 3. 注意点
- 线程执行yield(), 不一定能成功退让CPU资源, 退让成功概率由CPU决定.
- 关于线程优先级, 只表示线程竞争CPU时间片的概率, 不保证优先级高的一定先执行.

## 三. Thread.sleep(): 线程休眠
### 1. sleep()的理解
让当前正在运行的线程休眠指定时间
  
### 2. sleep()与yield()的异同
#### 相同点
- 都会暂停执行当前线程, 如果yield()方法的退让CPU资源请求被允许的话.
- 如果当前线程持有锁, 执行sleep()/yield()后, 在等待竞争CPU资源过程中都不会释放锁.
#### 不同点
- sleep()可以精确指定线程休眠的时间且线程一定会暂停执行, 而yield()则不一定
- sleep的线程在休眠过程中可能抛出异常, 能被打断, yield()不能被打断.
  