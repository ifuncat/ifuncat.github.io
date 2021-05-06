---
title: Java多线程系列01
author: ifuncat
date: 2021-05-06 20:22:22 +0800
categories: [Java核心]
tags: [Java多线程]
---
<style>
img{
    padding-left: 6%;
}
</style>

## 一. 进程与线程
1. ### 进程(Process)的基本概念
  - 指一个内存中运行的应用程序, 每个进程都有一个独立的内存空间, 程序进入内存后就称为进程.
  - 一个应用程序可以同时运行多个进程, 进程也是程序的一次执行过程, 是系统运行程序的基本单位.
  - 系统运行一个程序即为从一个进程创建, 运行到死亡的过程.
2. ### 线程的基本概念(Thread)
  - 线程是进程中的一个执行单元, 负责当前进程中程序的执行.
  - 一个程序至少有一个进程, 一个进程至少由一个线程.

  <img src="https://cdn.jsdelivr.net/gh/ifuncat/blog-images/post/javacore/multi-trhead-01-01.jpg" style="zoom:50%" />