---
title : '☕ Java Concurrency Volatile'
date :  2023-11-29T12:11:34+08:00
draft : false
categories: [Java]
---

[原文地址](https://jenkov.com/tutorials/java-concurrency/volatile.html#:~:text=The%20Java%20volatile%20keyword%20is%20intended%20to%20address%20variable%20visibility,read%20directly%20from%20main%20memory.)

Java中的volatile关键字的作用是将一个变量标记为“存储在主存中”（being stored in main memory）。准确地说，每次都是从主存中读取volatile变量的值，而不是从CPU寄存器中，并且每次写都是将volatile变量的值写到主存中，而不仅仅是CPU寄存器中。

实际上，自从Java 5以来，volatile关键字不仅仅是保证读和写都是从内存中。我将会在随后介绍。

## 变量可见性问题

Java中volatile关键字保证了对于变量的改变对其他线程可见。这听起来有点抽象，且听我娓娓道来。

在一个多线程应用中，假设多个线程操作非volatile变量，每个线程在操作这个变量的时候可能会将变量的值从主存拷贝到CPU寄存器中，这样做主要是为了性能考虑。如果你的计算机是多核的，每个线程跑在一个CPU上。这意味着，每个线程可能将变量的值拷贝到不同CPU的寄存器中，如下图所示：

![Pasted image 20231129094642](http://beijing-my-blog-image-bucket.oss-cn-beijing.aliyuncs.com/uPic/Pasted%20image%2020231129094642.png)

对于非volatile类型的变量，JVM无法保证何时将变量的值从主存读到CPU寄存器中，或者何时将CPU寄存器中的值写回主存。这可能会导致一些问题，我将会在随后介绍。

假设多个线程同时操作一个counter变量：
```java
public class SharedObject {
	public int counter = 0;
}
```

假设只有Thread1增加counter变量的值，同时Thread1和Thread2不断地读取counter变量的值。

如果counter变量没有用volatile修饰，那么我们就无法保证counter的值何时会从寄存器写回主存。这意味着，CPU寄存器中的counter的值和主存中的值可能不同，如下图所示：

![Pasted image 20231129095127](http://beijing-my-blog-image-bucket.oss-cn-beijing.aliyuncs.com/uPic/Pasted%20image%2020231129095127.png)

上面这个问题，即由于变量还没写回主存导致其他线程无法看到变量的最新值的问题，又被称为“可见性”问题。

## Java volatile 可见性保证

Java中的volatile关键字可以解决变量可见性问题。将counter变量声明为volatile，即可确保所有对于counter变量的写，将会立刻写回到主存。同时，所有对于counter变量的读，也会直接从主存读。

下面是如何将counter声明为volatile：
```java
public class SharedObject {
	public volatile int counter = 0;
}
```

将一个变量声明为volatile，能够确保对于这个变量的写对其他线程可见。

在上面的那个场景中，线程T1更改counter的值，线程T2读取counter的值（但是不更改它），将counter变量声明为volatile就能够保证对于counter变量的写对线程T2都可见。

然而，假如T1和T2都增加counter的值，那么将counter声明为volatile可能并不能满足要求，稍后会解释。

### volatile可见性保证的全部内容

实际上，Java volatile的可见性保证超出了volatile变量自身。可见性保证如下：
- 如果线程A写了一个volatile变量，随后线程B读同一个volatile变量，那么在线程A写volatile变量之前对它可见的所有变量，在线程B读取volatile变量之后，同样对B可见。
- 如果线程A读取一个volatile变量，那么在线程A读取volatile变量之前对它可见的变量，在线程A读取volatile变量时，都将会从内存中重新读取。

下面是一段示例代码：
```java
public class MyClass {
	private int years;
	private int months;
	private volatile int days;

	public void update(int years, int months, int days) {
		this.years = years;
		this.months = months;
		this.days = days;
	}
}
```

update方法中更新了三个变量，其中只有days是volatile的。

volatile的可见性保证意味着，如果一个值写入了days，那么所有对于线程可见的变量同样被写入主存。这意味着，当一个值写入days的时候，years和months的值也被写入主存。

当读取years，months和days的时候，你可能会这样做：
```java
public class MyClass {
	private int years;
	private int months;
	private volatile int days;

	public int totalDays() {
		int total = this.days;
		total += months * 30;
		total += years * 365;
		return total;
	}

	public void update(int years, int months, int days) {
		this.years = years;
		this.months = months;
		this.days = days;
	}
}
```

注意在totalDays方法中，首先将days的值读到total中。当读取days的值的时候，months和days的值也会从内存中读取。因此，通过上面的读取顺序，你能够保证读到days, months和days的最新值。

## 指令重排带来的挑战

JVM和CPU允许通过重排指令来提升性能，只要指令的语义不变。比如，看下面这个例子：
```java
int a = 1;
int b = 2;

a++;
b++
```

这些指令可以被重排为如下的序列，这并没有改变程序的语义：
```java
int a = 1;
a++;

int b = 2;
b++;
```

然而，当有volatile变量时，指令重排可能会带来一些问题。让我们以上面的MyClass为例：
```java
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

在update方法中，当写days变量时，新写入years和months的值也会被写入内存中。然而，如果JVM对指令做了如下重排：
```java
public void update(int years, int months, int days){
    this.days   = days;
    this.months = months;
    this.years  = years;
}
```

重排之后，写days时months和years的值也会被写回到内存，但是这发生在months和days的值更新之前。因此，不能保证months和years的新值对其他线程可见。指令重排之后语义发生了改变。

对于这个问题，Java有一个解决方法，我们稍后会介绍。

## Java volatile对于happens-before的保证

为了解决指令重排所带来的问题，Java的volatile关键字在可见性保证之外， 又提供了一种“happens-before”保证。happens-before保证：
- 对于其他变量的读或者写，不能重排到对volatile变量的写之后，如果这些读/写最初位于对volatile变量的写之前的话。在volatile变量写之前的读/写，能够确保发生在（happens-before）对volatile变量的写之前。注意，对于其他变量的读/写，如果位于对volatile变量的写之后的话，仍然有可能会被重排到对volatile变量的写之前。
- 对于其他变量的读或者写，不能重排到对volatile变量的读之前，如果这些读/写最初位于对volatile变量的读之后的话。注意，对于其他变量的读/写，如果位于对volatile变量的读之前的话，仍然有可能会被重排到对volatile变量的读之后。

![java_volatile_happens_before.excalidraw](http://beijing-my-blog-image-bucket.oss-cn-beijing.aliyuncs.com/uPic/java_volatile_happens_before.excalidraw.png)

## volatile不是万金油

即使volatile关键字能够确保对volatile变量的所有读都是直接从主存读、写都是直接写到主存，仍然有一些情况，仅仅使用volatile是不够的。

在前面所讲的例子中，只有线程1会写共享变量counter，那么将counter声明为volatile能够确保线程2总能读到counter的最新值。

实际上，多个线程可以同时写一个volatile变量，并且将正确的值存储在内存中，如果写入的新值不依赖于之前的值的话。或者说，如果一个线程在写volatile变量之前不需要先读它的值，使用volatile还是能够保证正确性的。

只要一个线程需要首先读取volatile变量的值，然后依赖于读取到的值产生一个新值的话，使用volatile就无法再保证正确性了。在读取volatile变量的值和写入新值之间的这一段时间间隔，产生了竞争关系，其他线程可能会在这段时间内读取到相同的值，产生一个新值，然后将这个值写回主存，这样就把其他线程写的值给覆盖了。

上面所说的这种情况，即多个线程同时增加一个counter的值，就是volatile无法保证正确性的情况。下面对此做详细解释。

假设线程1将共享变量counter的值0读取到CPU寄存器中，对它的值加1，然后没有将新值写回内存。线程2从内存中读取到的counter的值还是0，然后将它的值加1，并且也没有将新值写回内存。如下图：

![Pasted image 20231129112449](http://beijing-my-blog-image-bucket.oss-cn-beijing.aliyuncs.com/uPic/Pasted%20image%2020231129112449.png)

线程1和线程2现在已经不同步了。counter的实际值应该是2，但是每个线程所看到的值，都在各自CPU寄存器里，都是1。同时，主存中的值是0。全乱套了！即使线程最终将counter的值写回主存，这个值也是个错的。

## 什么时候可以使用volatile？

正如我之前提到的那样，如果两个线程都对一个共享变量进行读写，那么使用volatile关键字是不够的。在这种情况下，你需要使用synchronized关键字来确保对变量的读写操作是原子的。读写一个volatile变量并不会阻塞其他线程来进行读写。为了处理这种情况，你需要使用synchronized将critical sections包裹起来。

除了使用synchronized之外，你还可以使用java.util.concurrent包中提供的原子数据结构。比如，AtomicLong或者AtomicReference或者其他类型。

如果只有一个线程读写volatile变量，其他线程都读这个变量，那么volatile可以保证读线程每次都读到最新的值。如果不使用volatile的话，就无法提供这种保证。

volatile关键字能够保证对32位或者64位的变量有效。

## volatile的性能考虑

读或者写volatile变量将会导致从主存中读或者写这个变量。从主存中进行读写比从CPU寄存器中进行读写的代价要高得多。访问volatile变量同样会导致禁止一些指令重排，指令重排是提升性能的一种常用技术。因此，只有当你真的需要强制保证变量的可见性的时候，再去使用volatile。

实际上，CPU寄存器的值通常只会写到CPU的L1 cache，这非常快。虽然比不上写CPU寄存器快，但是也是很快的了。将L1 cache的值写回L2,L3以及主存是通过一个独立的芯片来控制的，CPU并不负责干这个。

即使如此，还是要记住，只有真正需要的时候再用volatile。

