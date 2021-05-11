---
title: Java并发04-线程安全性问题说明
author: ifuncat
date: 2021-05-10 22:22:22 +0800
categories: [Java核心]
tags: [Java并发]
---
<style>
img{
    padding-left: 3%;
}
</style>

## 一. 线程的安全性概述
### 1. 什么是线程的安全性
- 线程的安全性主要针对线程操作的对象的状态(如对象的实例变量或静态变量)而言的
- 如果在多线程环境下, 线程操作的对象状态不一致, 那么就是线程不安全的, 例如多线程对变量a进行累加操作, 每个线程读取到的a都是其他线程操作完的最新的值, 执行完后a的值符合预期.

### 2. 产生线程安全性问题的三个条件
- 多线程环境下<br/>
  不言而喻, 单线程执行不会产生线程安全性问题
- 存在共享资源 <br/>
  根据定义, 线程安全性问题是针对被操作的对象而言的, 只有多个线程操作共享变量时才可能引起这个变量的状态不一致, 从而产生线程不安全问题.
- 对共享变量进行非原子性操作<br/>
  线程操作共享变量导致其状态发生改变(如变量a进行累加操作)才可能引起变量的状态不一致, 这些操作一般为非原子操作. <br/>
  而如果线程的操作只是读取变量值而不修改值, 则不会产生线程安全性问题. 线程读取变量值的操作是一个原子操作.

## 二. 如何产生线程安全性问题
### 1. 线程安全类: 无状态类
```java
public class UserServiceImpl {
    public static Double getAvgAge(List<User> list) {
        int size = list.size();
        return Optional.ofNullable(list).map(items -> items.stream().mapToDouble(User::getAge).average().orElse(0.00)).orElse(0.00);
    }
}
```
- 无状态类: 简单来说就是没有属性的类, 如接口实现类, 不能保存数据, 是不变类, 多线程操作下不会改变这个类的实例的状态, 因为无状态, 所以线程安全. 参考: [https://www.cnblogs.com/east7/p/11665450.html](https://www.cnblogs.com/east7/p/11665450.html)
- UserServiceImpl的实例中只有局部变量如list, size, 都是存于栈中, 由线程独有, 不与其他线程共享. 堆中的变量(如类中的实例变量)能被多个线程访问, 因此如果某个类存在实例变量/全局变量, 且被多个线程访问, 就有可能产生线程安全性问题.

### 2. 类中有一个属性
```java
public class UnSafeDemo {
    int a = 0; //共享变量

    public static void main(String[] args) throws InterruptedException {
        UnSafeDemo demo = new UnSafeDemo();
        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                demo.a++; //多线程操作实例变量a, 使其状态发生改变, 而不只是读取a的值
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                demo.a++;
            }
        }, "t2").start();

        Thread.sleep(2000); //主线程休眠便于查看结果
        System.out.println("a: " + demo.a++); //不是每次都是20000
    }
}
```
#### 2.1. 结果分析
- 开启两个线程对实例变量a进行操作, 两线程均累加100000, 这样预期值为200000, 但是实际执行发现大多数情况下`a`的值小于200000.
- 对`a`进行累加操作并不是原子操作, 而是由三步构成: 线程从主存中获得a的值读取到自己的工作内存中, 在工作内存中进行+1, 然后将+1后的a的值写到主存中. 参考累加1操作的字节码执行过程.
- 单线程下无问题, 多线程情况下, 如果t1线程先读取到`a`并且在工作内存中+1, 但是此时还未刷新到主内存中, 然后t2获得CPU资源(t1等待)从主内存中读取`a`值, 此时读取到的`a`值是旧值, 导致最终累加的结果不一致. 详见: Java内存模型.
- 以上这种在并发编程中, 由于不恰当的执行时序而出现错误结果的情况称为"竞态条件"
- 由以上可知变量`a`不安全, 因此这个类也是不安全的

##### 2.2. 解决上述安全性问题
- 同步: 对操作共享变量的方法引入同步机制, 如synchronized, 显式锁, volatitle, 原子变量等.
- 不可变变量: 使用final修饰`a`, 修饰后`a`即为不可变, 也就变的安全了. 注意被修饰的变量的类型仅限基本数据类型和String, 其他引用类型不适用, 参考: [final关键字](https://www.cnblogs.com/xiaoxi/p/6392154.html)
- 不操作共享变量: 多线程中不操作共享变量, 自然变得安全.

##### 2.3. 使用原子变量等解决安全性问题
- AtomicInteger.incrementAndGet()是个原子操作, 也是同步操作
```java
public class UnSafeDemo {
    static AtomicInteger a = new AtomicInteger(); //共享变量

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                a.incrementAndGet();  //incrementAndGet() 原子操作
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                a.incrementAndGet();
            }
        }, "t2").start();

        Thread.sleep(2000); //主线程休眠便于查看结果
        System.out.println("a: " + a); //不是每次都是20000
    }
}
```

### 3. 类中有多个属性
- 如果类中只有一个实例变量, 那么可以用原子变量来保证线程安全. 那么类中存在多个实例变量/全局变量呢? 即便是对每个变量都用上对应的原子变量, 也是无法实现线程安全.
- 一个原子变量的操作是原子操作, 多个原子操作组合在一起其他就不是原子操作了.

```java
public class UnsafeAtomicDemo {
    static AtomicInteger a = new AtomicInteger(0);
    static AtomicInteger b = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                a.incrementAndGet();
                b.incrementAndGet();
                if (a.get() != b.get()) { //a,b的起始值相同, 每次循环都执行相同的+1操作, 正常情况下 a,b的值相同
                    System.out.println(1);
                }
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                a.incrementAndGet();
                b.incrementAndGet();
                if (a.get() != b.get()) {
                    System.out.println(2);
                }
            }
        }, "t2").start();

        Thread.sleep(2000); //主线程休眠便于查看结果
        System.out.println("a: " + a.get() + ", b: " + b.get()); //不是每次都是20000
    }
}
```
#### 3.1. 结果分析
- a, b的起始值相同, 每次循环都执行相同的+1操作, 正常情况下(单线程执行) a, b的值相同, 但是却出现a, b的值不相同的情况.
- 多个原子操作组合在一起就不是原子操作了. 因此 `a.incrementAndGet(); b.incrementAndGet();` 这两操作合在一起就是非原子操作, 可以看作是普通的int类型的值累加, 同样存在着线程安全问题.

#### 3.2 解决线程安全问题: 上锁
- 可以将`a.incrementAndGet(); b.incrementAndGet();`这两方法抽取为一个私有方法, 然后对这个方法使用synchronized修饰.
- 或者将`a.incrementAndGet(); b.incrementAndGet();`放到synchronized代码块中.
- 可以把synchronized方法或代码块看作是一个原子操作.
- 注意: synchronized必须使用同一个锁对象才有用
```java
public class UnsafeAtomicDemo {
    static AtomicInteger a = new AtomicInteger(0);
    static AtomicInteger b = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        UnsafeAtomicDemo lock = new UnsafeAtomicDemo();
        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                synchronized (lock) { //共用一个锁对象, 否则无法生效
                    a.incrementAndGet();
                    b.incrementAndGet();
                    if (a.get() != b.get()) { //a,b的起始值相同, 每次循环都执行相同的+1操作, 正常情况下 a,b的值相同
                        System.out.println(1);
                    }
                }
            }
        }, "t1").start();

        new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                synchronized (lock) {
                    a.incrementAndGet();
                    b.incrementAndGet();
                    if (a.get() != b.get()) {
                        System.out.println(2);
                    }
                }
            }
        }, "t2").start();

        Thread.sleep(2000); //主线程休眠便于查看结果
        System.out.println("a: " + a.get() + ", b: " + b.get()); //不是每次都是20000
    }
}
```

## 三. 总结
- 产生线程安全问题的三大条件: 多线程环境, 存在共享变量, 多线程对共享变量进行非原子的操作.
- 什么样的类是线程安全的: 线程安全类(无属性)
- 产生线程安全问题解决思路: 优化共享变量(使用原子操作类, volatile修饰等), 对操作共享变量的方法进行加锁(synchronized, ReentrantLock等)