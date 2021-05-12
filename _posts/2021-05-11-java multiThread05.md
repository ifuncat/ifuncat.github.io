---
title: Java并发05-深入理解Synchronized关键字
author: ifuncat
date: 2021-05-11 22:22:22 +0800
categories: [Java核心]
tags: [Java并发]
---
<style>
img{
    padding-left: 3%;
}
</style>

### 一. 前言
- 造成线程安全问题主要有三个条件: 多线程环境, 存在共享资源, 多线程对共享资源进行非原子操作.
- 解决线程安全问题的通常做法是：当存在多个线程操作共享资源时，需要保证同一时刻只有一个线程在操作数据, 其他线程等待这个线程操作完数据，然后再去操作这个更新的数据．
- 上述解决方案也被称为`互斥锁`, 当一个共享数据被正在访问的线程加上互斥锁后, 此时其他线程要访问共享数据只能等待, 直到正在访问的线程处理完毕后释放锁, 其他线程竞争这个锁才能访问共享资源.
- Java的Synchronized关键字可以保证在同一时刻, 只有一个线程可以执行某个方法或代码块(在方法或代码块中操作共享数据). 另外synchronized还能保证可见性, 即共享数据被一个线程修改后能够立即刷新到主内存, 保证其他线程访问的是最新的共享数据(详见volatile).

### 二. synchronized的三种使用方式
#### 1. 修饰静态方法
- 当synchronized作用于静态方法时, 锁就是当前的类对象. 静态成员(静态属性,静态方法等)不专属于任何一个实例对象, 因此可以控制静态成员的并发操作.
- 演示: 分别开启两条线程测试普通静态方法addAge()和同步静态方法synAddAge(), 每条线程中循环执行10000次的自增方法. 从结果看出addAge()得到的值小于20000, 而synAddAge()得到的值为20000, 同步静态方法执行结果符合预期.

```java
public class UserServiceImpl {
    static int age = 0; //共享变量

    public static void addAge() { age++; } //对比测试,未加synchronized

    public synchronized static void synAddAge() { age++; } //同步方法

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                UserServiceImpl.addAge(); 
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                UserServiceImpl.addAge();
            }
        }, "t2").start();

        Thread.sleep(3000); //主线程休眠3s,确保t1/t2线程执行完毕
        System.out.println("age: " + age);

        UserServiceImpl.age = 0; //注意归零,继续测试同步静态方法

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                UserServiceImpl.synAddAge();
            }
        }, "t3").start();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                UserServiceImpl.synAddAge();
            }
        }, "t4").start();

        Thread.sleep(3000); //主线程休眠3s,确保t3/t4线程执行完毕
        System.out.println("synchronized age: " + age);
    }
}

//打印结果如下:
//age: 17558
//synchronized age: 20000
```

#### 2. 修饰实例方法/普通方法
- 当synchronized作用于实例方法时, 锁就是当前的实例对象. 实例成员(成员变量,普通方法等)专属于一个实例对象, 可以控制这一个实例对象的并发操作.

```java
public class AgentServiceImpl {
    int age = 0;

    public void addAge() { this.age++; }

    public synchronized void synAddAge() { this.age++; }

    public static void main(String[] args) throws InterruptedException {
        AgentServiceImpl agent = new AgentServiceImpl();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                agent.addAge();
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                agent.addAge();
            }
        }, "t2").start();

        Thread.sleep(3000); //主线程休眠3s,确保t1/t2线程执行完毕
        System.out.println("age: " + agent.age);

        agent.age = 0; //注意归零,继续测试同步方法

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                agent.synAddAge();
            }
        }, "t3").start();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                agent.synAddAge();
            }
        }, "t4").start();

        Thread.sleep(3000); //主线程休眠3s,确保t3/t4线程执行完毕
        System.out.println("synchronized age: " + agent.age);
    }
}
```

#### 3. 修饰代码块
- 通常需要使用同步的地方只是某几行操作共享变量的代码, 方法体的其他都无需使用同步.
- JVM执行代码时并不是一行行顺序执行下去, 而是会在编译成字节码时进行优化, 可能打乱执行顺序, 好处是执行效率更高, 但是保证执行结果一直. 如果对整个方法进行同步, 其中的一些不需要同步的代码不能被JVM优化, 相比于优化过的代码执行效率偏低, 因此synchronized修饰代码块效率可能更高.
```java
public class StaffServiceImpl {
    int age = 0;

    public void addAge() {
        //....其他操作
        synchronized (this) { //对象锁
            age++;  //只需synchronized修饰操作共享变量的代码块
        }
    }

    public void add2Age() {
        //....其他操作
        synchronized (StaffServiceImpl.class) { //类锁
            age += 2;
        }
    }
}
```

### 三. 使用Synchronized注意点
#### 1. 锁对象不一致
- 场景一. 如果一个静态变量分别被普通同步方法, 静态同步方法更新, 多线程分别执行这两个同步方法, 同样会产生线程安全问题, 得到的静态变量的值将不是预期值. (静态方法只能操作静态变量)
- 因为普通同步方法对应的锁对象是实例对象, 静态同步方法对应的锁是类对象, 线程持有的锁不是同一个锁, 不能起到互斥作用, 就会产生同步问题.
```java
public class EmpServiceImpl {
    static int age = 0; //全局变量

    public synchronized void addAge() { age++; } //普通同步方法操作全局变量

    public synchronized static void staticAddAge() { age++; } //静态同步方法操作全局变量

    public static void main(String[] args) throws InterruptedException {
        EmpServiceImpl emp = new EmpServiceImpl();

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                emp.addAge();
            }
        }, "t1").start(); //t1线程调用普通同步方法累加age

        new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                emp.staticAddAge();
            }
        }, "t2").start(); //t2线程调用静态同步方法累加age

        Thread.sleep(3000); //主线程休眠3s,确保t1/t2线程执行完毕
        System.out.println("age: " + emp.age);

    }
}
```
- 场景二: 如果一个静态/成员变量分别被两个普通方法更新, 一个方法中的同步代码块的锁对象为实例对象, 另一个方法中的同步代码块的锁对象为类对象. 多线程分别执行这两个普通方法, 同样会产生线程安全问题, 得到的变量的值不是预期值. (普通方法可以操作静态/成员变量)
- 因为线程拿到的锁对象不同, 不能起到互斥作用, 就会产生同步问题.

```java
public class EmpServiceImpl {
    int age = 0; //全局变量

    public void addAge() {
        synchronized (this) { //锁对象为实例对象
            age++;
        }
    }

    public void staticAddAge() {
        synchronized (EmpServiceImpl.class) { //锁对象为类对象
            age++;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        EmpServiceImpl emp = new EmpServiceImpl();

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                emp.addAge();
            }
        }, "t1").start(); //t1持有的锁为实例对象

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                emp.staticAddAge();
            }
        }, "t2").start(); //t2持有的锁为类对象

        Thread.sleep(3000); //主线程休眠3s,确保t1/t2线程执行完毕
        System.out.println("age: " + emp.age);
    }
}
```

### 四. synchronized底层语义
- JVM中的同步基于进入和退出管理对象(也叫监视器Monitor)实现, 无论是显式同步(即同步代码块, 有明确的`monitorenter, monitorexit`指令)还是隐式同步(如synchronized方法)都是如此.
- Java中synchronized方法并不是由`monitorenter, monitorexit`指令实现的, 而是由方法调用指令读取运行时常量池的`ACC_SYNCHRONIZED`标志来隐式实现的.

#### 1. 理解Java对象头与Monitor
JVM中, 一个实例对象在内存布局中分为三块: 对象头(Header), 实例数据(Instance)和对齐填充(Padding)
- 实例数据: 存放对象的属性数据, 包括父类的属性信息, 如果是数组, 还包括数组的长度等.
- 填充对齐: 由于虚拟机要求对象起始地址必须是8字节的整数倍, 填充数据不是必须存在, 仅仅为了字节对齐

##### 对象头
- 实现Synchronized锁对象的基础是Java头对象, 一般而言, synchronized使用的锁对象存于Java对象头中.
- JVM中采用2个字节来存储对象头, 如果对象是数组则会分配3个字节, 其中一个记录数组长度. 主要结构由`Mark Word`和`Class Metadata Address`组成.
- MarkWord(标记字):存储对象的hashCode, 锁信息, 分代年龄, GC标志等信息.
- Class Metadata Address: 类型指针指向对象的类的元数据, JVM通过这个指针确定对象是哪个类的实例

##### MarkWord(标记字)
- 对象头的信息与对象的自身定义的数据没有关系, 对象头信息作为额外的存储成本, JVM出于优化空间考虑, Markword被设计成非固定的数据结构, 以便存储更多的数据.
- 如32位JVM下除了上述列出的markword默认存储结构外, 还有如下可能变化的结构

<img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-thread-05-01.jpg" style="zoom:100%" />

- 其中轻量级锁和偏向锁是JDK1.6对synchronized优化时新增的.
- 重量级锁也就是synchronized锁, 其锁标志位为10, 其中指针指向的是monitor对象(也被称为管程/监视器锁),

##### Monitor对象
- 每个对象都存在着一个monitor与之关联, 对象与其monitor之间的关系有多种实现方式: 1. monitor与对象一起创建及销毁; 2. 当线程试图获得对象锁时, monitor对象自动生成, 当一个monitor被某个线程持有时, 它便处于锁定状态.
- JVM中, monitor对象由ObjectMonitor类实现的 ,位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的, 其主要数据结构如下
```c++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录线程个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
- ObjectMonitor中有两个队列, _WaitSet和_EntryList用来保存ObjectWaiter对象列表(每个等待锁的线程都会被封装成ObjectWaiter对象), _owner指向持有monitor的线程.
- 当多个线程同时访问同一同步代码时, 先会进入_EntryList, 当对象获取到对象的monitor后, 进入_owner区域并把monitor的owner变量设置为当前线程, 同时会把monitor中的计数器count+1.
- 若线程调用wait()方法, 将释放当前持有的monitor, owner变量变为null, count-1. 同时该线程进入_waitset中等待被唤醒.
- 若当前线程执行完毕将释放monitor, 并且复位owner变量的值, 以便其他线程获得monitor.

<img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-thread-05-02-2.jpg" style="zoom:100%" />


参考: [https://blog.csdn.net/javazejian/article/details/72828483](https://blog.csdn.net/javazejian/article/details/72828483)