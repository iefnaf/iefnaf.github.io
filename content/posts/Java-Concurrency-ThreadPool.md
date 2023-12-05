---

title: '☕  Java Concurrency ThreadPool'
date:  2023-12-05T17:13:21+08:00
draft: false
categories: [Java]

---



原文地址：https://jenkov.com/tutorials/java-concurrency/thread-pools.html

线程池是一组可重复使用的线程，每个线程都可能执行不止一个任务。使用线程池是每次执行任务时都新创建一个线程的替代方案。

与重复使用一个已经创建好的线程相比，创建一个新的线程对性能造成的影响更大。这就是相比为每一个任务新创建一个线程，使用线程池的吞吐更高的原因。

除此之外，使用线程池可以更加方便地控制当前系统中活跃线程的个数。每个线程都会消耗系统中的一部分资源，比如内存，所以如果系统中活跃线程的个数太多的话，这些线程会消耗大量的资源，从而导致计算机变得很慢。比如，当内存消耗的太多时，操作系统会开始将RAM swap到磁盘。

在这个关于线程池的教程中，我将会向你解释线程池是如何工作的，如何使用线程池，以及如何实现一个Java线程池。记住，Java已经有了一个内置的线程池——Java ExecutorService，所以你不必自己实现它，就可以使用线程池。然而，有时你也可能需要实现自己的线程池，比如当你需要添加一些ExecutorService没有提供的功能时。或者，你只想造一个轮子玩一玩。

## 线程池是如何工作的

与为每个任务创建一个线程来并发跑不同，任务可以直接提交到线程池中。只要当前线程池中有idle的线程，这个任务就会被分配给它们执行。在内部，这些任务被插入到了一个BlockingQueue，线程池中的线程从这个队列中取任务。线程池中的剩余的idle的线程会被阻塞，等待从队列中取任务。

![Thread Pool Illustration](https://jenkov.com/images/java-concurrency/thread-pools-1.png)

## 线程池的使用场景

线程池经常在多线程服务器中使用。每个到达服务器的连接都被包装成一个任务，然后提交给线程池。线程池中的线程会在这些连接上并发地处理请求。下面我们会详细看一下如何在Java中实现一个多线程服务器。

## 内置的Java线程池

在java.util.concurrent包中有一个内置的线程池，所以你不必实现自己的线程池。你可以看看java.util.concurrent.ExecutorService这个类。

## Java线程池的实现

下面是一个简单版本的Java线程池实现方法。这个实现中使用了自Java 5以来支持的Java BlockingQueue。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ThreadPool {

    private BlockingQueue taskQueue = null;
    private List<PoolThreadRunnable> runnables = new ArrayList<>();
    private boolean isStopped = false;

    public ThreadPool(int noOfThreads, int maxNoOfTasks){
        taskQueue = new ArrayBlockingQueue(maxNoOfTasks);

        for(int i=0; i<noOfThreads; i++){
            PoolThreadRunnable poolThreadRunnable =
                    new PoolThreadRunnable(taskQueue);

            runnables.add(poolThreadRunnable);
        }
        for(PoolThreadRunnable runnable : runnables){
            new Thread(runnable).start();
        }
    }

    public synchronized void  execute(Runnable task) throws Exception{
        if(this.isStopped) throw
                new IllegalStateException("ThreadPool is stopped");

        this.taskQueue.offer(task);
    }

    public synchronized void stop(){
        this.isStopped = true;
        for(PoolThreadRunnable runnable : runnables){
            runnable.doStop();
        }
    }

    public synchronized void waitUntilAllTasksFinished() {
        while(this.taskQueue.size() > 0) {
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}
```

下面是PoolThreadRunnable类，该类实现了Runnable接口，所以它可以被Java的Thread来执行。

```java
import java.util.concurrent.BlockingQueue;

public class PoolThreadRunnable implements Runnable {

    private Thread        thread    = null;
    private BlockingQueue taskQueue = null;
    private boolean       isStopped = false;

    public PoolThreadRunnable(BlockingQueue queue){
        taskQueue = queue;
    }

    public void run(){
        this.thread = Thread.currentThread();
        while(!isStopped()){
            try{
                Runnable runnable = (Runnable) taskQueue.take();
                runnable.run();
            } catch(Exception e){
                //log or otherwise report exception,
                //but keep pool thread alive.
            }
        }
    }

    public synchronized void doStop(){
        isStopped = true;
        //break pool thread out of dequeue() call.
        this.thread.interrupt();
    }

    public synchronized boolean isStopped(){
        return isStopped;
    }
}
```

下面是如何使用ThreadPool的一个示例：

```java
public class ThreadPoolMain {

    public static void main(String[] args) throws Exception {

        ThreadPool threadPool = new ThreadPool(3, 10);

        for(int i=0; i<10; i++) {

            int taskNo = i;
            threadPool.execute( () -> {
                String message =
                        Thread.currentThread().getName()
                                + ": Task " + taskNo ;
                System.out.println(message);
            });
        }

        threadPool.waitUntilAllTasksFinished();
        threadPool.stop();

    }
}
```

线程池的实现包括两部分：ThreadPool类是线程池的公共接口，PoolThread类是实现执行任务的线程。

为了执行一个任务，使用一个Runnable的实现作为参数，调用ThreadPool.execute方法。这个Runnable会被插入到blocking queue中，等待从队列中被取出。

这个Runnable会被一个idle的线程从队列中取出并执行。你可以在PoolThread.run()中看到这些。当执行完毕之后，线程会尝试再次从队列中取出一个Runnable，直到停止为止。

为了停止一个ThreadPool，需要调用ThreadPool.stop方法。线程池中的每个线程都会被调用doStop方法。

注意PoolThread.doStop方法中的this.interrupt语句。这会保证卡在taskQueue.dequeue的wait方法上的线程退出，然后抛出`InterruptedException`异常并退出dequeue方法。这个异常会在PoolThread.run方法中捕获。然后会再次检查isStopped，并成功退出run方法，然后线程就gg了。

