---
title: Java并发06-深入理解ThreadLocal
author: ifuncat
date: 2021-05-13 22:22:22 +0800
categories: [Java核心]
tags: [Java并发]
---
<style>
img{
    /* padding-left: 3%;
    margin: 0 auto; */
    align: center;
}
</style>

## 一. ThreadLocal 概述
### 1. 前言
- 多线程访问共享变量时容易发生同步问题, 特别是多个线程对共享数据进行写操作的时候.
- 为了保证线程安全, 在访问共享变量时需要进行额外的同步措施, 如加synchronized锁, volatile修饰共享变量等.
- ThreadLocal是一种规避多线程同步安全问题的新方法.

### 2. ThreadLocal是什么
- ThreadLocal译为线程本地变量(局部变量), 即这个变量是在存在于自身的工作内存中(非共享变量), 不会被其他线程访问到.
-  每个线程访问ThreadLocal对象时, 访问的都是线程自己的变量, 也就不会产生同步问题.

## 二. ThreadLocal 简单使用
- 在类中定义一个静态变量, 如果这个变量类型为常见类型如基本类型或String等, 多线程操作这个变量会产生线程安全问题.
- 定义线程本地变量, 每个线程访问的都是线程自己的局部变量, 不会产生线程安全问题.

```java
public class ThreadLocalTestDemo {
    private static final ThreadLocal<String> localVal = new ThreadLocal<>();

    static void printAndRemove() {
        System.out.println(Thread.currentThread().getName() + ": " + localVal.get());
        localVal.remove(); //清空线程本地变量
    }

    public static void main(String[] args) {
        new Thread(() -> {
            localVal.set("hello t1"); //设置线程本地变量的值
            printAndRemove(); //调用打印,清空方法
            System.out.println(Thread.currentThread().getName() + " after remove: " + localVal.get());
        }, "t1").start();

        new Thread(() -> {
            localVal.set("hello t2");
            printAndRemove();
            System.out.println(Thread.currentThread().getName() + " after remove: " + localVal.get());
        }, "t2").start();
    }
}
```

## 三. ThreadLocal的应用场景
- 在进行对象跨层传递的时候, 使用ThreadLocal可以避免多次传递, 打破层次间的约束.
- 线程间的数据隔离.
- 进行事务操作, 用于存储线程事务信息.
- 数据库连接Session会话管理.
- sl4j的MDC机制

```java
public class ConnectionManager {
  private static Connection connection = null;
  public static Connection openConnection(){
    if(null == connection){ connection = DriverManager.getConnection(); }
    return connection;
  }
  public static void closeConnection(){
    if(null != connection){ connection.close(); }
  }
}
```
以上为数据库连接的管理类, 使用数据库是都是先建立数据库连接, 使用完后关闭连接就好了. 如果很多用户多次频繁使用数据库, 就需要多次建立和关闭连接. 导致服务器压力很大.<br/>
如果使用ThreadLocal, 在每个线程中创建一个数据库连接实例, 在这个线程执行周期内任何地方都可以使用, 且线程之间互不影响. 这样即解决了线程同步问题, 也避免了频繁创建/关闭数据库连接, 提高了数据库的性能.


## 四. ThreadLocal源码解析

### 1. ThreadLocalMap
- `Thread`类中有一个属性`threadlocals`，类型为`ThreadLocal.ThreadLocalMap`, 详见Thread.java#182，也就是说每个线程实例对象都有自己的ThreadLocalMap.
- ThreadLocalMap作为ThreadLocal的静态内部类. 从源码看到ThreadLocalMap作为一个Map，其中的entry的结构为key：ThreadLocal实例对象，value：为代码中需要放入的值, 即需要线程各自持有的变量值, 如上述例子中的数据库连接对象, 注意:实际上key并非ThreadLocal对象本身，而是它的一个弱引用.

```java
static class ThreadLocalMap {
  static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;

      Entry(ThreadLocal<?> k, Object v) { //key：ThreadLocal实例对象，value：为代码中需要放入的值
          super(k);
          value = v;
      }
  }

  //省略
```

<div align=center>
    <img width=50% src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/2021-threadlocal-01.png">
</div>

### 2. ThreadLocal.set(T value)

```java
public void set(T value) {
    Thread t = Thread.currentThread(); //获得当前线程对象
    ThreadLocalMap map = getMap(t); //获得当前线程对象的ThreadLocalMap属性
    if (map != null) 
        map.set(this, value); //存在map.赋值, 注意this为用户定义的ThreadLocal对象
    else
        createMap(t, value); //不存在初始化ThreadLocalMap属性, 并赋值
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals; //获得传入线程对象的threadLocals属性
}
```

- 往线程本地变量存值时调用`ThreadLocal.set(T value)`, 其中先获得执行这段代码的线程对象的threadLocals属性对象, 其本质是个map.
- 如果没有threadLocals, 则初始化这个属性, 并向这个map中赋值, key为`this`, 即用户定义的ThreadLocal对象, value为需要代码中需要放入的值, 如上述例中的`localVal.set("hello t1");`, map中的key为类型为ThreadLocal的`localVal`对象, value为`"hello t1"` 字符串.

<div align=center>
    <img width=80% src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/threadlocal-00.png">
</div>

### 3. ThreadLocal.get()

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); //获得当前线程的threadLocals属性
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); //当前的ThreadLocal对象作为key, 从map中获得之前传入的值
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

- 获得线程本地变量中存入的值调用`ThreadLocal.get()`, 先获得当前线程的threadLocals属性, 其作为一个map, 当前ThreadLocal对象作为key, 从map中可以获得之前存入的值. 如上述例中的`localVal.get()`, map为执行这段代码的线程对象的threadLocals属性对象, key为`localVal`.

### 4. 堆/栈/线程/ThreadLocal在内存中的分布
- 每条线程都有自己的栈, 栈中的变量引用线程私有. 栈中的变量引用如threadLocalRef指向堆中的ThreadLocal对象, currentThreadRef指向堆中的当前线程的线程对象.
- 线程对象中持有一个属性, 类型为ThreadLocalMap的对象, 本质上是一个map.
- threadLocalMap中的key为threadLocal对象, value为存入的目标对象的引用, 指向堆中的目标对象.
 
<div align=center>
    <img width=80% src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/threadlocal-01.png">
</div>

## 四. ThreadLocal实战
[利用slfj.mdc实现请求链路追踪](https://snailclimb.gitee.io/javaguide/#/docs/java/multi-thread/%E4%B8%87%E5%AD%97%E8%AF%A6%E8%A7%A3ThreadLocal%E5%85%B3%E9%94%AE%E5%AD%97?id=threadlocal%e9%a1%b9%e7%9b%ae%e4%b8%ad%e4%bd%bf%e7%94%a8%e5%ae%9e%e6%88%98)
