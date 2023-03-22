---
title: Java Thread类使用方法和注意事项
date: 2023-02-23 00:08:30
tags:
    - Java
    - 多线程
top_img: https://www.apple.com.cn/newsroom/images/live-action/wwdc-2021/Apple_Developer-Tools-hero-animation-clip_060721.jpg.landing-regular_2x.jpg
cover: https://www.apple.com.cn/newsroom/images/live-action/wwdc-2021/Apple_Developer-Tools-hero-animation-clip_060721.jpg.landing-regular_2x.jpg
---
# Java Thread类使用方法和注意事项

Java的Thread类是Java多线程编程中非常重要的一个类，它提供了创建和操作线程的方法。本文将介绍Thread类的使用方法和一些注意事项，以及演示一些相关的代码示例。

# Thread类使用方法

## 创建线程
创建线程有两种方法：

1. 继承Thread类，重写run()方法。
``` java
class MyThread extends Thread {
    public void run() {
        System.out.println("MyThread running");
    }
}
```
通过创建一个类，继承Thread类，并重写run()方法来创建线程。在重写的run()方法中，定义线程的执行逻辑。创建该类的实例对象，并调用start()方法启动线程。
``` java
MyThread myThread = new MyThread();
myThread.start();
```
2. 实现Runnable接口，并重写run()方法。
``` java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("MyRunnable running");
    }
}
```
通过创建一个类，实现Runnable接口，并重写run()方法来创建线程。创建该类的实例对象，将其作为Thread的构造函数参数，并调用start()方法启动线程。
``` java
MyRunnable myRunnable = new MyRunnable();
Thread thread = new Thread(myRunnable);
thread.start();
```
## 启动线程
启动线程需要调用Thread类的start()方法，该方法将调用run()方法。

``` java
Thread thread = new Thread();
thread.start();
```
## 线程睡眠
可以通过Thread类的sleep()方法使线程暂停执行一段时间，单位为毫秒。例如，让线程睡眠1000毫秒（即1秒钟）。

``` java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
```
## 线程等待
线程等待需要使用Object类的wait()方法和notify()方法。调用wait()方法将使线程进入等待状态，直到另一个线程调用notify()方法才会继续执行。

``` java
synchronized (lock) {
    while (!flag) {
        try {
            lock.wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
## 线程加入
线程加入需要使用Thread类的join()方法。调用join()方法将使当前线程等待另一个线程执行完毕后再继续执行。

``` java
Thread thread = new Thread();
thread.start();
try {
    thread.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```
## 线程中断
线程中断需要使用Thread类的interrupt()方法。调用该方法将设置线程的中断标志位，如果线程处于阻塞状态，则会抛出InterruptedException异常。

``` java
Thread thread = new Thread();
thread.start();
thread.interrupt();
```
##注意事项

1. 线程的start()方法只能被调用一次，否则将抛出IllegalThreadStateException