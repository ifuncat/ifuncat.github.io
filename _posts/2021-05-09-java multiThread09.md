---
title: Java并发09-Object.wait/notify/notifyAll方法详解
author: ifuncat
date: 2021-05-09 22:22:22 +0800
categories: [Java核心]
tags: [Java并发]
---
<style>
img{
    padding-left: 3%;
}
</style>
## 一. 概述
### Java
## 一. Object.wait(): 线程等待
### 1. wait()方法说明
- 线程在调用wait()时, 会释放当前持有的锁(), 然后让出CPU资源, 线程状态由running->wating.
- 只有当别的线程执行notify() / notifyAll(), 才会唤醒一个或多个处于waiting, 然后继续往下执行, 知道执行完synchronized代码块中的代码.
- 可以通过wait(long), 等待的线程一定时间后自动唤醒.
- 执行wait()方法时需要被try / catch, 以便发生异常也能使处于等待状态的线程被唤醒

### 2. wait()与sleep()的异同
- 二者都是让线程暂停执行
- wait()是Object类的方法, sleep()是Thread类的方法.
- sleep()不会释放任何锁, wait()则相反.
- sleep的线程正常恢复执行的方式只有等待休眠时间耗尽, wait的线程除了等待时间耗尽还能通过别的线程调用notify()/notifyAll()唤醒.

## 二. Object.notify()/notifyAll(): 线程唤醒
### 1. notify()方法说明
- 唤醒等待获得锁的单个线程
- 如果等待获得锁的线程有多个, 则唤醒其中一个进程. 选择唤醒哪个线程是随机的, 由CPU决定
### 2. notifyAll()方法说明
- 唤醒等待获得锁的所有线程
- notify()/notifyAll()方法的执行只是唤醒处于等待状态的线程, 而不会立即释放锁, 锁的释放需要看具体的同步代码块的执行情况
  
## 三. wait多线程中测试某个条件的变化
notify()/notifyAll()方法执行后唤醒的等待状态的线程会接着上次的执行情况继续执行下去, 因此在条件判断的
