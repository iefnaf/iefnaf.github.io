---

title : '☕ Java Concurrency Synchronization'
date : 2023-11-27T13:42:42+08:00
draft : false
categories: [Java]
---



线程之间的通信主要通过共享访问某个成员变量（field），以及这个成员变量所指向的对象。这种通信的方式非常有效，但是同样也带来了两个问题：

- 线程干扰（Thread interference）
- 内存一致性错误（Memory consistency error）

用来预防上述错误的方法叫做同步（synchronization）。
然而，同步会引入线程竞争（Thread Contention），这将会导致两个或者更多的线程同时去尝试访问相同的资源，导致JVM运行一个或者更多线程变得更慢，甚至会挂起线程，暂停它们的执行。饥饿（Starvation）和活锁（livelock）是线程竞争的某些表现形式。后面的章节会介绍Liveness。

本节主要包含以下主题：

- 线程干扰：描述了当多个线程尝试去访问共享数据时会发生什么问题
- 内存一致性错误：描述了由于共享内存不一致的视图（inconsistent views of memory）导致的问题
- 同步方法（synchronized method）：介绍了一个可以用来防止线程干扰以及内存一致性错误的简单套路
- 隐式锁（Implicit lock）以及同步：描述了一个更加通用的同步套路，以及如何基于隐式锁来做同步
- 原子访问（Atomic Access）：描述了一种更加通用的，不会被其他线程干扰的操作。

## 线程干扰
考虑一个简单的类，Counter：
```java
public class Counter {  
  private int c = 0;  
  
  private void increment() {  
    c++;  
  }  
  
  private void decrement() {  
    c--;  
  }  
  
  public int value() {  
    return c;  
  }  
}
```

计数器Counter是这样设计的：每次调用increment将会给c的值加一，每次调用decrement将会给c的值减一。然而，如果一个计数器Counter对象被多个线程引用，多个线程之间的相互干扰何能会让事情变得和我们的预期不太一样。
干扰发生在两个操作，运行在不同的线程中，对于同一个数据进行操作，并且操作是相互穿插着来的。这意味着两个操作包含了多个步骤，这些步骤之间有重叠。

看起来在Counter实例上的操作不会发生交叉，因为在c上的操作是如此的简单。然而，即使是这么简单的语句也会被虚拟机翻译成多个步骤。我们不会检查虚拟机具体执行了哪些步骤，只需要知道，一个如c++这样简单的表达式也会被拆分成如下三个步骤：
- 读取c当前的值
- 将c的值+1
- 将增加后的值存储到c中

c--也可以用同样的方法来拆分。
假设线程A调用了increment，与此同时线程B调用了decrement。如果c的初始值是0，它们的动作可能会以以下的方式进行交叉：

1. Thread A：Retrieve c
2. Thread B：Retrieve c
3. Thread A：Increment retrieved value; result is 1.
4. Thread B：Decrement retrieved value; result is -1.
5. Thread A：Store result in c; c is now 1.
6. Thread B：Store result in c; c is now -1.

线程A的结果丢失了，被线程B覆盖了。上面展示的这种交叉只是一种可能的情况。也有可能线程B的结果丢失，或者没有任何错误产生。因为执行的结果不确定，所以线程干扰导致的bug可能会很难发现以及修复。

## 内存一致性错误
当不同的线程看到某个数据的值不同，而这些值本应该相同时，就发生了内存一致性错误。触发内存一致性错误的原因很复杂，超过了本文的范畴。幸运的是，程序员并不需要搞清楚这些原因。我们只需要知道如何避免它们就行了。
防止内存一致性错误的关键在于理解happens-before关系。这个关系保证了一个语句的写对于另一个语句可见。考虑下面这个例子，假设定义一个int类型的变量，并对它进行初始化：
```java
int counter = 0;
```
counter被两个线程A和B所共享。假设A线程首先增加了counter的值。
```java
counter++;
```
然后，线程B打印了counter的值。
```java
System.out.println(counter);
```

如果上面这两个语句是在同一线程中执行的，那么我们可以预期输出的结果就是1。但是这两个语句是在两个不同的线程中执行的，输出的结果可能是0，因为无法保证线程A对于counter的改变对于线程B可见——除非程序员在这两个语句之间建立了happens-before关系。

建立happens-before关系的方法有几种。其中一种是通过同步，我们稍后会介绍。
我们已经看到了一些建立happens-before关系的两个动作：

- 当一个语句调用Thread.start，所有与这个语句有happens-before关系的语句，与这个线程要执行的所有语句都有happens-before关系。这段代码的效果就是新线程知道它是一个新创建的线程。
- 当一个线程结束，导致另一个线程中Thread.join返回时，结束线程中的所有语句都和join语句只有的语句有happens-before关系。结束线程的影响对于执行join线程可见。

哪些语句可以建立happens-before关系？详见java.util.concurrent包的summary部分。

## 同步方法
Java语言提供了两个同步的基本方法：同步方法（Synchronized methods）和同步语句（Synchronized statements）。更加复杂的同步语句将会在下一节介绍。本节先介绍一下同步方法。
为了让一个方法同步，仅仅在这个方法的声明中增加synchronized关键字就行了。
```java
public class SynchronizedCounter {  
  private int c = 0;  
  
  public synchronized void increment() {  
    c++;  
  }  
  
  public synchronized void decrement() {  
    c--;  
  }  
  
  public synchronized int value() {  
    return c;  
  }  
}
```

如果一个counter是SynchronizedCounter的实例，那么调用这些同步方法有以下两个效果：
- 首先，在同一个对象上的两个同步方法的调用之间不会产生重叠。当一个线程正在执行一个对象的同步方法时，所有其他调用这个对象的同步方法的线程将会被挂起，直到第一个线程结束。
- 其次，当一个同步方法退出时，会自动和相同对象的之后同步方法的调用之间建立起happens-before关系。这意味着对于对象的更改对于所有的thread都可见。

注意，构造器不能是同步的，在构造器上使用synchronized关键字是一个语法错误。同步构造器没有任何意义，因为当一个对象创建时，只有创建对象的这个线程才能访问对象。

⚠️警告：当创建一个需要在多个线程之间共享的对象时，要特别注意不要让这个对象的引用过早的泄漏。例如，假如你想要维护一个称为instance的List，保存这个类的所有实例。你可能会尝试在构造器中增加下面这一行：
```java
instances.add(this);
```
但是其他线程可以使用instance来在对象构造结束之前访问这个对象。

同步方法是一种防止线程竞争和内存不一致的简单有效的方法：如果一个对象对于多个线程可见，对这个对象的所有读写操作都是通过同步方法（一种重要的例外情况：final变量，在对象构造结束之后不能修改它的值，可以安全的通过非同步方法来读取它的值），这个策略很有效，但是可能会产生活性问题(liveness)，我们将会在之后介绍。

## 内部锁和同步
synchronized是基于一个被称为内部锁（intrinsic lock）或者管理锁（monitor lock）的内部实体来实现的。（在API中经常称这个实体为monitor）。
内部锁在同步中有两个作用：
- 强制对于对象状态的修改彼此隔离（enforcing exclusive access to an object's state）
- 建立happens-before关系，让修改可见

每个对象都有一个属于它的内部锁。通常来说，如果一个线程需要互斥、一致地访问一个对象的某个变量（field），那么它需要在访问变量之前获得内部锁，然后在操作结束之后释放内部锁。
在获得锁之后，释放锁之前，线程持有这个内部锁。只要线程持有内部锁，其他的线程就无法获得这个锁。如果其他线程尝试获得这个锁，它们将会被block。
当一个线程释放一个内部锁时，释放锁这个动作和这个锁之后的获取动作之间就建立起了happens-before关系。

### 同步方法中的锁
当一个线程调用同步方法时，它会自动地获取对象的内部锁，并在方法退出时自动地释放锁。即使方法结束是由于获异常导致的，锁的释放也会执行。
你可能会想，当一个静态同步方法调用的时候发生了什么的？因为一个静态方法是跟一个类绑定的，而不是一个对象。在这种情况下，尝试获得内部锁的线程会尝试寻找跟这个类相关的Class对象。因此，控制静态对象的锁和控制类的实例的锁是不同的。

### 同步语句
另一种创建同步代码的方法是使用同步语句。与同步方法不同，同步语句必须显式指定提供内部锁的对象：
```java
public void addName(String name) {
	synchronized(this) {
		lastName = name;
		nameCount++;
	}
	nameList.add(name);
}
```
在这个例子中，addName方法需要同步对于lastName和nameCount的修改，但是需要避免同步其他对象方法的调用（在同步语句中调用其他对象的方法可能会导致liveness问题）。如果没有同步语句，我们就必须创建一个单独的，非同步的，只调用nameLis.add的函数。
同步语句对于提升并发也很有用。假设MsLunch类有两个变量，c1和c2，它们永远也不会同时使用。所有对这两个变量的更改都必须进行同步，但是不需要保证c1和c2的同步是原子的，并且这样做会导致并发度降低。不使用同步方法和同步语句，我们创建了两个对象，用来提供锁：
```java
public class MsLunch {  
  private long c1 = 0;  
  private long c2 = 0;  
  private Object lock1 = new Object();  
  private Object lock2 = new Object();  
  
  public void inc1() {  
    synchronized (lock1) {  
      c1++;  
    }  
  }  
  
  public void inc2() {  
    synchronized (lock2) {  
      c2++;  
    }  
  }  
}
```

用这种方法的时候需要格外小心，你需要搞清楚是否需要保证c1和c2的更新是否需要保证原子。

### 可重入同步
一个线程可以获得由其他线程持有的锁，但是一个线程也可以获得它已经持有的锁。允许一个线程超过一次获得一个相同的锁，称为可重入同步。这描述了一个场景，同步代码，直接或者间接地调用了一个同样包含同步代码的方法，并且这两段代码使用相同的锁。如果没有可重入同步，同步代码可能需要特别小心，防止由于重入导致的block问题。

## 原子访问
在编程中，一个原子动作是指一下子发生的东西。一个原子操作无法在中途停止：它或者完全结束，或者一点都不开始。在原子操作结束之前，它所产生的影响对外不可见。
我们已经看到了一个递增表达式，比如c++，不是一个原子操作。有一些操作你可以定义成原子的：
- 对于引用变量以及大多数基础类型变量的读写操作（除了long和double之外）
- 对于所有使用volatile声明的变量的读写操作（包括long和double）

原子操作不会相互重叠，所以使用它们的时候不必担心线程干扰问题。然而，这并不能满足同步原子操作的所有需求，因为内存一致性错误依然存在。

使用volatile修饰的变量能够避免内存一致性错误，因为所有对于volatile变量的读和之后的写之间建立起了happens-before关系。这意味着所有对于volatile变量的修改都对其他线程可见。更重要的是，这意味着如果一个线程读volatile变量，它不仅仅能看到这个变量的最新修改，同样也可以看到导致这个变量改变的代码产生的其他影响。
使用简单的原子访问比通过同步代码访问更加有效，但是程序员需要格外小心内存一致性错误问题。
java.util.concurrent包提供了不依赖于同步的一些原子方法，我们将会在之后介绍。
