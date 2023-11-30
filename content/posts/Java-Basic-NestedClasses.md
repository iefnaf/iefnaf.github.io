---

title: '☕ Java Basic NestedClasses'

date: 2023-11-30T09:39:30+08:00

draft: false

categories: [Java]

---

在Java中，嵌套类（nested class）就是定义在其他类内部的类。

嵌套类的作用就是清晰地将嵌套类和它的外部类组合在一起，表明这两个类要一起使用。或者，嵌套类只能通过它的外部类来使用。

Java开发者经常把嵌套类称为内部类（inner class），但是内部类（非静态嵌套类）仅是Java中嵌套类中的一种。

在Java中，嵌套类被认为是它们外部类的成员。因此，一个嵌套类可以被声明为public，package（没有存取修饰符），protected以及private（详细可见Java Basic AccessModifiers）。因此Java中的嵌套类也可以被继承。

你可以创建一种不同的嵌套类，它们的类型包括：

- 静态嵌套类（Static nested class）
- 非静态嵌套类（Non-static nested class）
- 本地类（Local class）
- 匿名类（Anonymous class）

## 静态嵌套类

静态嵌套类的声明方法如下：

```java
public class Outer {
  public static class Nested {
  }
}
```

为了创建一个Nested类的实例，你必须使用Outer类来作为前缀指代它，就像这样：

```java
Outer.Nested instance = new Outer.Nested();
```

在Java中，一个静态嵌套类实际上就是一个普通的类，只不过它被嵌套在其他的类中。作为静态类，嵌套静态类只能通过一个外部类实例的引用来访问外部类实例的实例变量。

## 非静态嵌套类（内部类）

Java中的非静态嵌套类也被称为内部类。内部类与外部类的某一个实例相关联。因此，你必须首先创建一个外部类的实例，然后才能创建一个内部类的实例。下面是一个定义内部类的例子：

```java
public class Outer {
  public class Inner {
  }
}
```

下面是你如何创建一个Inner类的实例：

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

注意，为了创建一个内部类的实例，你需要将new放在某个外部类实例的后面。

非静态嵌套类（内部类）可以访问外部类的fields，即使它们被声明为private的。下面是一个例子：

```java
public class Outer {
  private String text = "I am private";
  
  public class Inner {
    public void printText() {
      System.out.println(text);
    }
  }
}
```

注意内部类中的printText()方法是如何访问外部类中的私有text变量的。下面是如何调用printText方法：

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
inner.printText();
```

### 内部类shadowing

如果内部类声明了与外部类同名的变量或者方法，我们就称内部类的变量或者方法遮蔽了（shadow over）外部类的变量或者方法。下面是一个例子：

```java
public class Outer {
  private String text = "I am Outer private!";
  public class Inner {
    private String text = "I am Inner private";
    public void printText() {
      System.out.println(text);
    }
  }
}
```

在上面的例子中，Outer类和Inner类都有一个名为text的field。当Inner类访问text时，它访问的是它自己的text。当外部类访问text时，它访问的也是自己的text。

如果内部类想访问外部类的同名field，也是可以的。需要使用Outer.this.来指代。下面是一个例子：

```java
public class Outer {
  private String text = "I am outer private!";
  public class Inner {
    private String text = "I am inner private";
    public void printText() {
      System.out.println(text);
      System.out.println(Outer.this.text);
    }
  }
}
```

现在Inner.printText方法就可以输出Inner.text和Outer.text了。

## 本地类

Java中的本地类（Local class）和内部类相似，它定义在一个方法或者一个方法的块区域（{ ... }）内部。下面是一个例子：

```java
class Outer {
  public void printText() {
    class Local {
      
    }
    Local local = new Local();
  }
}
```

本地类只能在定义它们的方法，或者方法的块区域内部来访问。

本地类可以像内部类那样访问外部类的成员（变量或者方法）。

本地类也可以访问定义在同一个方法内或者同一个scope内的其他final类型的局部变量。

从Java8开始，本地类也可以访问所处方法的局部变量或者参数，参数需要是final或者等价final的。等价final意味着变量初始化之后不会再进行改变。方法的参数基本上都是等价final的。

内部类的shadow规则对于本地类同样适用。

## 匿名类

Java中的匿名类是没有名字的嵌套类。它们通常被声明为某个类的子类，或者某个接口的实现。

匿名类在实例化的同时进行定义。下面是一个例子：

```java
public class SuperClass {
  public void doIt() {
    System.out.println("SuperClass doIt()");
  }
}
```

```java
SuperClass instance = new SuperClass() {
  public void doIt() {
    System.out.println("Anonymous class doIt()");
  }
}

instance.doIt();
```

运行上面这段代码，匿名类的doIt方法将会被调用。匿名类继承了SuperClass，并且override了doIt方法。

匿名类也可以通过实现一个接口来创建，下面是一个例子：

```java
public interface MyInterface {
  public void doIt();
}
```

```java
MyInterface instance = new MyInterface() {
  public void doIt() {
    System.out.println("Anonymous class doIt()");
  }
}

instance.doIt();
```

如你所见，通过实现一个接口的方式来创建一个匿名类，与通过继承某个类的方式非常相似。

匿名类可以访问外部类的成员。在Java8之后，匿名类也可以访问final或者等效final的局部变量。

你可以在匿名类中定义成员或者函数，但是不能声明一个构造器。你可以声明一个静态初始化器（static initializer）。下面是一个例子：

```java
final String textToPrint = "Text...";
MyInterface instance = new MyInterface() {
  private String text;
  // static initializer
  { this.text = textToPrint; }
  
  public void doIt() {
    System.out.println(this.text);
  }
}
```

内部类的shadow规则对于匿名类同样适用。

## 嵌套类的好处

嵌套类的好处就是你可以把拥有所属关系的类组合在一起。你已经可以把他们放在同一个包中了，但是把一个类放在另一个类内是一种更强的组合。

一个嵌套类通常只被它的外部类使用，或者与外部类一同使用。某些场景中，嵌套类只对外部类可见。在另一些场景中，嵌套类可以对外部类可见，但是需要与外部类一同使用。

下面是一个具体的例子。在Cache类中，你可能会定义一个CacheEntry类，用来包含某个cache entry的信息。Cache的使用者可能永远也无法感知到CacheEntry的存在，如果他们不需要自己获取CacheEntry的信息的话。然而，Cache也可以让CacheEntry对外部可见。下面是两种实现的代码：

```java
public class Cache {
  private Map<String, CacheEntry> cacheMap = new HashMap<String, CacheEntry>();
  
  private class CacheEntry {
    public long timeInserted = 0;
    public object value = null;
  }
  
  public void store(String key, Object value) {
    CacheEntry entry = new CacheEntry();
    entry.value = value;
    entry.timeInserted = System.currentTimeMillis();
    this.cacheMap.put(key, entry);
  }
  
  public Object get(String key) {
    CacheEntry entry = this.cacheMap.get(key);
    if(entry == null) return null;
    return entry.value;
  }
}
```

```java
public class Cache {

    private Map<String, CacheEntry> cacheMap = new HashMap<String, CacheEntry>();

    public class CacheEntry {
        public long   timeInserted = 0;
        public object value        = null;
    }

    public void store(String key, Object value){
        CacheEntry entry = new CacheEntry();
        entry.value = value;
        entry.timeInserted = System.currentTimeMillis();
        this.cacheMap.put(key, entry);
    }

    public Object get(String key) {
        CacheEntry entry = this.cacheMap.get(key);
        if(entry == null) return null;
        return entry.value;
    }

    public CacheEntry getCacheEntry(String key) {
        return this.cacheMap.get(key);
        }

}
```

第一种实现对外隐藏了CacheEntry，第二种实现对外暴露了CacheEntry。

