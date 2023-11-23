---
title: '☕ Java Concurrency Thread'
date: 2023-11-23T16:30:33+08:00
draft:  false
categories: [Java]
---

原文地址：https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html

计算机的使用者理所当然地认为他们的系统可以同时处理多个事情。他们假设他们可以在其他应用下载文件的时候使用word，控制打印队列以及视频。即使是一个非常简单的应用，也被认为同时可以处理多个事情。比如：视频处理软件必须同时从网络上下载视频数据，解压，播放。即使是word也要同时处理键盘和鼠标的动作，无论它当前是正在处理文本格式化还是更新显示内容。这样的软件被称为并发（concurrent）软件。
Java语言生来就支持并发编程，在语言层面提供了基本的并发支持，以及并发库。自从5.0版本以来，Java引入了更高层次的并发API。本教程介绍Java提供的基础并发支持，并对java.util.concurrent包所提供的一些API进行介绍。

## 进程和线程
在并发编程中，与执行相关的有两个基本的单元：进程（process）和线程（thread）。
一个计算机系统通常有很多活跃的进程和线程，即使计算机只有一个核（core）也是如此。单核系统同时只有一个线程在执行，多个线程之间使用分时（time slicing）的方式来共享这个处理器核心。

目前，计算机系统多核架构已经成为主流，这极大地增强了一个系统的并发处理能力，但是并不是说单核系统不能搞并发，即使一个核也可以并发！

### 进程
一个进程是一个自包含的执行环境（self-contained environment）。一个进程通常有一个完成的，私有的运行时资源。特别地，每个进程都有它自己的内存空间。
进程通常被视为是和程序或者应用相同的概念。然而，用户眼中的一个应用可能实际上是由多个进程组成。为了处理进程间的通信，绝大多数操作系统支持进程内通信资源（Inner Process Communication，IPC），比如管道（pipe）和套接字（socket）。IPC不仅可以用于同一个系统内线程间的通信，也可以用于不同系统之间的通信。
绝大多数JVM都是作为一个进程来跑的。一个Java应用可以使用ProcessBuilder来创建进程。多进程应用超出了本教程的讲述范围。
### 线程
线程有时又被称为轻量级进程。进程和线程都提供了一种执行环境，但是创建线程比创建进程需要的资源更少。
线程在进程之内——每个进程最少有一个线程，线程之间共享进程的资源，包括内存和打开文件。这样使线程之间的通信变得更加简单，但是有时会引入一些问题。
多线程是Java程序的一个核心特性。每个应用至少有一个线程——如果你把系统线程，比如内存管理线程，信号处理线程也算上，那就有多个线程。但是从应用程序员的视角来看，开始的时候只有一个线程，被称作主线程（main thread）。这个线程可以创建其他的线程，稍后我们会介绍这一点。

## Thread对象
每个线程都与Thread类的一个实例所对应。有两种使用Thread创建并发应用的基本方法：
- 直接使用Thread类来进行线程的创建和管理，每当应用需要初始化一个异步任务时，实例化一个Thread。
- 将线程的管理从应用中抽象出来，将应用的任务提交给执行器（executor）
本节介绍Thread类的使用方法，Executors在并发进阶中介绍。

### 线程的创建和启动
为了创建一个线程，你需要提供这个线程中运行的代码。可以有两种方式来做这件事：
- 提供一个Runnable对象。Runnable接口定义了一个方法：run，它是这个线程中运行的代码，然后把Runnable结构体传递给Thread的构造函数，下面是一个例子：
```java
public class HelloRunnable implements Runnable {  
  public static void main(String[] args) {  
    new Thread(new HelloRunnable()).start();  
  }  
  
  @Override  
  public void run() {  
    System.out.println("hello from a thread");  
  }  
}
```

- 创建Thread的子类。Thread类本身实现了Runnable接口，尽管它的run方法什么也不干。所以我们可以继承Thread类，覆盖它的run方法。下面是一个例子：
```java
public class HelloThread extends Thread {  
  @Override  public void run() {  
    System.out.println("hello from a thread");  
  }  
  
  public static void main(String[] args) {  
    new HelloThread().start();  
  }  
}
```

多一句，Thread类中run方法：
```java
public  
class Thread implements Runnable {

	/* What will be run. */  
	private Runnable target;

	@Override  
	public void run() {  
	    if (target != null) {  
	        target.run();  
	    }  
	}
}
```

无论使用上述两种方法的哪一种，都是使用Thread.start来启动新的线程。
那么应该用哪一种方法呢？第一种方法，也就是实现Runnable接口，更加通用。因为还可以继承其他的类。第二种方法实现起来更加简单，但是因为已经继承自Thread类，所以不能继承自其他的类了。本课程更加推荐第一种做法，这样做可以把Runnable任务和Thread对象解耦，这样做更加灵活，并且对后面要介绍的并发高阶API也同样适用。
Thread类定义了线程管理相关的一些方法，这些方法包含静态方法，提供关于线程的信息、状态等。
### 线程的暂停
Thread.sleep方法会将当前正在执行的线程挂起一段时间。这样做可以将处理器时间让给应用内的其他线程，或者同一个系统内的其他应用。sleep方法也可以用于步调（pacing），就像下面这个例子展示的一样，也可以用来等待另外一个线程，SimpleThreads例子会展示这一点。
sleep方法有两个覆盖的版本：一个指定了休眠的时间，单位是毫秒。另一个单位是纳秒。然而，不能保证准确休眠指定的时间，因为这还取决于底层操作系统。同样，休眠周期可以被中断所终结，我们会在稍后介绍中断。不论在哪种情况下，你都不能假定调用sleep就会将当前的线程准确休眠指定的时间。
SleepMessages使用sleep来每隔4秒打印一个消息：
```java
public class SleepMessages {  
  public static void main(String[] args) throws InterruptedException {  
    String[] importantInfo = {  
            "Mares eat oats",  
            "Does eat oats",  
            "Little lambs eat ivy",  
            "A kid will eat ivy too"  
    };  
    for (String s : importantInfo) {  
      //Pause for 4 seconds  
      Thread.sleep(4000);  
      //Print a message  
      System.out.println(s);  
    }
  }
}
```
注意，main方法声明了抛出`InterruptedException`异常。前面我们提到另外一个线程可能会中断正在休眠的线程，但是这是一个例外。因为这个应用只有一个线程，所以它可以好好睡一觉，不用担心被其他人打扰，也就不需要catch这个异常。
### 线程的中断
中断是对线程的一个提示，它告诉这个线程应该停下手头正在做的事情，然后去做一些别的。线程如何响应中断由程序员决定，但是常用的一种处理方法是终结这个线程的执行。
一个线程通过在一个Thread对象上调用`interrupt`方法来给另一个线程发送一个中断信号。为了确保中断机制工作正常，被中断的线程需要支持被中断。
一个线程如何支持被中断呢？这取决于这个线程正在做什么。如果一个线程经常调用抛出`InterruptedException`异常的方法，在catch到中断异常之后，它简单地从run方法返回。比如：假设SleepMessages的for循环在一个run方法中，那么应该像下面这样修改，以支持中断。
```java
for (String s : importantInfo) {  
  //Pause for 4 seconds
  try {
	  Thread.sleep(4000);
  } catch (InterruptedException e) {
	  // We've been interrupted: no more messages.
	  return;
  }
    
  //Print a message  
  System.out.println(s);  
}
```
很多抛出`InterruptedException`异常的方法，比如sleep，都是这样设计的：如果抛出了中断异常，那么就应该直接返回。

那么如果一个线程很长时间都没有调用一个抛出中断异常的方法呢？如果是这样的话，它应该调用Thread.interrupted，这个方法会返回true，如果接收到一个中断信号的话。下面是一个例子：
```java
for (int i = 0; i < inputs.length; i++) {
    heavyCrunch(inputs[i]);
    if (Thread.interrupted()) {
        // We've been interrupted: no more crunching.
        return;
    }
}
```
在上面这个简单的例子中，仅仅是测试一下中断状态，如果收到了中断信号，就直接退出。在更加复杂的应用中，可能抛出一个`InterruptedException`会更有意义。
```java
if (Thread.interrupted()) {
    throw new InterruptedException();
}
```
这样做可以让中断处理相关的代码都放在catch中。

中断机制是通过使用内部的一个flag——中断状态来实现的。调用Thread.interrupt会设置这个flag。当一个线程通过调用静态方法Thread.interrupted之后，这个flag会被清空。非静态方法isInterrupted不会清空中断状态flag。

## 一个简单的例子
下面这个简单的例子综合了本节所讲的知识。SimpleThreads包含两个线程。第一个是每个Java应用都会有的主线程。主线程通过Runnable对象创建了一个新线程MessageLoop，然后等待它执行结束。如果MessageLoop太墨迹，执行了太久，主线程会中断它。
MessageLoop线程打印一系列的消息。如果在打印所有消息之前被中断了，MessageLoop线程会打印一个消息然后退出。
```java
public class SimpleThreads {  
  static void threadMessage(String message) {  
    String threadName = Thread.currentThread().getName();  
    System.out.format("%s: %s%n", threadName, message);  
  }  
  
  private static class MessageLoop implements Runnable {  
  
    @Override  
    public void run() {  
      String[] importantInfo = {  
              "Mares eat oats",  
              "Does eat oats",  
              "Little lambs eat ivy",  
              "A kid will eat ivy too"  
      };  
  
      try {  
        for (String s : importantInfo) {  
          // Pause for 4 seconds  
          Thread.sleep(4000);  
          // Print a message  
          threadMessage(s);  
        }  
      } catch (InterruptedException e) {  
        threadMessage("I wasn't done!");  
      }  
    }  
  }  
  
  public static void main(String[] args) throws InterruptedException {  
    // Delay, in milliseconds before  
    // we interrupt MessageLoop    // thread (default one hour).    long patience = 1000 * 60 * 60;  
  
    // If command line argument  
    // present, gives patience    // in seconds.    if (args.length > 0) {  
      try {  
        patience = Long.parseLong(args[0]) * 1000;  
      } catch (NumberFormatException e) {  
        System.err.println("Argument must be an integer.");  
        System.exit(1);  
      }  
    }  
  
    threadMessage("Starting MessageLoop thread");  
    long startTime = System.currentTimeMillis();  
    Thread t = new Thread(new MessageLoop());  
    t.start();  
  
    threadMessage("Waiting for MessageLoop thread to finish");  
    while (t.isAlive()) {  
      threadMessage("Still waiting...");  
      // Wait maximum of 1 second for MessageLoop thread to finish.  
      t.join(1000);  
      if (((System.currentTimeMillis() - startTime) > patience) && t.isAlive()) {  
        threadMessage("Tired of waiting!");  
        t.interrupt();  
        // Shouldn't be long now  
        // -- wait indefinitely        t.join();  
      }  
    }  
    threadMessage("Finally!");  
  }  
}
```
