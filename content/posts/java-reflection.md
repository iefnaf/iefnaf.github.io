+++
title = '🧙Reflection: the black magic of Java'
date = 2023-10-26T16:13:13+08:00

+++

反射（Reflection）是Java语言的一个特性。它允许一个运行中的Java程序来自检，操作程序内部的属性。比如，Java类可以获得所有成员的名字，并且展示它们。

反射的一个实际应用是JavaBeans，使用它可以通过一个构建工具可视化地操作一个软件的组件。这个工具使用反射来在动态加载的过程中获得Java组件（类）的属性。

## 一个简单的例子

为了展示反射是如何工作的，考虑下面这个简单的例子：

```java
import java.lang.reflect.*;

public class DumpMethods {
  public static void main(String args[])
  {
     try {
        Class c = Class.forName(args[0]);
        Method m[] = c.getDeclaredMethods();
        for (int i = 0; i < m.length; i++)
        System.out.println(m[i].toString());
     }
     catch (Throwable e) {
        System.err.println(e);
     }
  }
}
```

运行这个程序：

```shell
$ javac DumpMethods.java
$ java DumpMethods
```

将会产生如下输出：

```
public boolean java.util.Stack.empty()
public synchronized java.lang.Object java.util.Stack.peek()
public synchronized int java.util.Stack.search(java.lang.Object)
public java.lang.Object java.util.Stack.push(java.lang.Object)
public synchronized java.lang.Object java.util.Stack.pop()
```

输出的是`java.util.Stack`的方法名，以及方法的参数类型和返回值。

这段程序使用`class.forName`加载了指定的类，然后调用`getDeclaredMethods`方法来获得这个类定义的方法的一个列表。`java.lang.reflect.Method`是一个类，它代表一个类方法。

## 开始使用反射

反射相关的类，比如`Method`，在`java.lang.reflect`中。为了使用这些类，总共需要三步。

**第一步**：获得你想要操纵的类的一个`java.lang.Class`对象。`java.lang.Class`用来代表一个运行中的Java程序的类或者接口。

一种获得Class对象的方式是使用`Class.forName`方法。

```java
Class c = Class.forName("java.lang.String");
```

另一种方式是使用`Class c = int.class;`或者`Class c = Integer.TYPE;`来获得基本类型的类信息。

**第二步：**调用`getDeclaredMethods`之类的方法，获得这个类的所有方法的一个列表。

第三步：使用反射API来操作第二步获得的信息。

## 模拟instanceOf

一旦我们拿到了类相关的信息，那么接下来就可以问Class对象一些基本的问题。比如，`Class.isInstance`方法就可以用来模拟`instanceof`运算符。

```java
public class ReflectTest {
    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName(args[0]);
            System.out.println(c.isInstance(0));
            System.out.println(c.isInstance(new ArrayList<>()));
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

编译运行：

```shell
$ javac ReflectTest.java
$ java ReflectTest java.util.Stack
false
false
$ java ReflectTest java.util.ArrayList
false
true
```

## 得到一个类的方法

反射的一个最基础，也是最重要的用法是用来得到一个类中定义的所有方法。下面是一个例子：

程序首先使用`Class.forName`获得了A类的一个Class对象，然后通过这个对象拿到了A类的方法（只有一个，也就是f1）。

如果你使用的是`getMethods`而不是`getDeclaredMethods`，那么你也可以得到继承自父类的方法。

一旦拿到一个`Method`对象的列表，展示每个方法的参数类型、异常类型、返回值类型就是小菜一碟了。所有的这些类型，无论是基础类型还是非基础类型，都是用一个Class来表示。

```java
public class ReflectTest {
    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest$A");

            Method[] methods = c.getDeclaredMethods();
            for (Method m : methods) {
                System.out.println("name = " + m.getName());
                System.out.println("declaring class  = " + m.getDeclaringClass());
                Class<?>[] parameters = m.getParameterTypes();
                for (int j = 0; j < parameters.length; j++) {
                    System.out.println("parameter " + j + " = " + parameters[j].getName());
                }
                Class<?>[] exceptions = m.getExceptionTypes();
                for (int j = 0; j < exceptions.length; j++) {
                    System.out.println("exception " + j + " = " + exceptions[j].getName());
                }
                Class<?> returnType = m.getReturnType();
                System.out.println("return type = " + returnType.getName());
            }
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }

    public static class A {
        private int f1(Object p, int x) throws NullPointerException {
            if (p == null) {
                throw new NullPointerException();
            }
            return x;
        }
    }
}
```

编译运行，得到以下输出：

```
name = f1
declaring class  = class ReflectTest$A
parameter 0 = java.lang.Object
parameter 1 = int
exception 0 = java.lang.NullPointerException
return type = int
```

## 获得构造函数相关信息

有了反射这个黑魔法，我们可以很简单地获得一个类的构造函数。下面是一个例子。

```java
public class ReflectTest {
    ReflectTest() {
    }

    ReflectTest(int i, double d) {
    }

    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest");

            Constructor<?>[] constructorList = c.getDeclaredConstructors();
            for (Constructor<?> ct : constructorList) {
                System.out.println("name = " + ct.getName());
                System.out.println("declaring class = " + ct.getDeclaringClass());
                Class<?>[] parameterTypes = ct.getParameterTypes();
                for (int i = 0; i < parameterTypes.length; i++) {
                    System.out.println("param #" + i + " " + parameterTypes[i]);
                }
                Class<?>[] exceptionTypes = ct.getExceptionTypes();
                for (int i = 0; i < exceptionTypes.length; i++) {
                    System.out.println("exception #" + i + " " + exceptionTypes[i]);
                }
                System.out.println("--------------------------");
            }
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

编译运行，得到以下输出：

```
name = ReflectTest
declaring class = class ReflectTest
--------------------------
name = ReflectTest
declaring class = class ReflectTest
param #0 int
param #1 double
--------------------------
```

## 获得类的成员

利用反射，我们还可以获得一个类的所有成员。

整体代码与前一个类似。这里用到的一个新特性是`Modifer`。这个反射类用来表示类成员的修饰语(`modifier`)，比如`private int`。modifier本身是用数字来表示的，`Modifier.toString`用来返回一个modifier的字符串表示。

```java
public class ReflectTest {
    private double d;
    public static final int i = 37;
    String s = "testing";

    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest");

            Field[] fields = c.getDeclaredFields();
            for (Field fld : fields) {
                System.out.println("name = " + fld.getName());
                System.out.println("type = " + fld.getType());
                int mod = fld.getModifiers();
                System.out.println("modifiers = " + Modifier.toString(mod));
                System.out.println("----");
            }
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

输出：

```
name = d
type = double
modifiers = private
----
name = i
type = int
modifiers = public static final
----
name = s
type = class java.lang.String
modifiers = 
----
```

## 通过名字调用方法

以上的例子展示了如何获得类的相关信息。除此之外，反射还有一些其他的骚操作。比如，使用名字来调用方法。

下面是一个具体的例子。假设一个程序想要调用`add`方法，但是知道运行时才知道这件事。也就是，这个方法的名字是在运行时确定的。`getMethod`用来寻找这个类中的一个方法，这个方法的名字是add，有两个int类型的参数。找到我们想调用的方法之后，我们拿到了一个`Method`类型的对象，为了调用这个方法，我们需要构造这个类的一个实例和这个方法的参数。

```java
public class ReflectTest {
    public int add(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest");
            Class<?>[] partTypes = new Class[2];
            // 注意，这里不要写成Integer.class了
            partTypes[0] = Integer.TYPE;
            partTypes[1] = Integer.TYPE;
            Method method = c.getMethod("add", partTypes);
            ReflectTest classObj = new ReflectTest();
            Object[] argList = new Object[2];
            argList[0] = 3;
            argList[1] = 4;
            Object retObj = method.invoke(classObj, argList);
            Integer retVal = (Integer) retObj;
            System.out.println(retVal);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 创建对象

下面是使用反射创建对象的一个例子。首先类似于通过名字和参数类型获得方法，这里我们通过构造函数的参数类型获得一个Constructor，然后创建构造函数的参数，通过对构造器调用`newInstance`来创建对象。

```java
public class ReflectTest {
    public ReflectTest() {
    }

    public ReflectTest(int a, int b) {
        System.out.println("a = " + a + ", b = " + b);
    }

    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest");
            Class<?>[] partTypes = new Class[2];
            // 注意，这里不要写成Integer.class了
            partTypes[0] = Integer.TYPE;
            partTypes[1] = Integer.TYPE;
            Constructor<?> ct = c.getConstructor(partTypes);
            Object[] argList = new Object[2];
            argList[0] = 3;
            argList[1] = 4;
            Object retObj = ct.newInstance(argList);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 改变成员变量的值

反射的另一个骚操作是用来改变对象中的成员变量的值。我们可以通过成员变量的名字，得到一个`Field`，然后通过Field的set方法来改变成员变量的值。(看到这我都震惊了...感觉反射像是从高维空间伸过来的一只手😂)

```java
public class ReflectTest {
    public double d;

    public static void main(String[] args) {
        try {
            Class<?> c = Class.forName("ReflectTest");
            Field fld = c.getField("d");
            ReflectTest obj = new ReflectTest();
            System.out.println("d = " + obj.d);
            fld.setDouble(obj, 3.14);
            System.out.println("d = " + obj.d);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 使用数组

最后一个骚操作：使用反射来创建并操作数组。Java语言中的数组是一种特殊的类，一个数组的引用可以赋值给一个Object引用。

下面是一个简答的例子。在这个例子中，我们创建了一个string的数组，数组长度是10，然后设置了数组中下标为5的元素。

```java
public class ReflectTest {
    public static void main(String[] args) {
        try {
            Class<?> clazz = Class.forName("java.lang.String");
            Object arr = Array.newInstance(clazz, 10);
            Array.set(arr, 5, "this is a test");
            String s = (String) Array.get(arr, 5);
            System.out.println(s);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 总结

Java反射很有用，因为它支持我们在运行时动态地根据名字来获得类的信息，并且支持在一个运行中的程序中操作它们。这个特性非常牛逼，在其他编程语言，比如C, C++, Fortan中都没有。

