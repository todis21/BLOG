---
title: Java反射
date: 2023-03-16 11:17:12
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202303162204108.png
---

# java反射





## 什么是java反射？

Java反射机制是在运行状态时，对于任意一个类，都能够获取到这个类的所有属性和方法，对于任意一个对象，都能够调用它的任意一个方法和属性(包括私有的方法和属性)，这种动态获取的信息以及动态调用对象的方法的功能就称为java语言的反射机制。



## Java反射的主要类

- 类：java.lang.Class;
- 构造器：java.lang.reflect.Constructor;
- 字段：java.lang.reflect.Field;
- 方法：java.lang.reflect.Method;
- 修饰符：java.lang.reflect.Modifier;



## Java如何获取一个类

JVM为每个加载的class创建了对应的`Class`实例，并在实例中保存了该`class`的所有信息；因此，如果获取了某个`Class`实例，我们就可以通过这个`Class`实例获取到该实例对应的`class`的所有信息(下面用`String`类举例)

* 直接通过一个`class`的静态变量`class`获取

```
Class cls = String.class;
```



* 通过该实例变量提供的`getClass()`方法获取

```
String s = "hello";
Class cls = s.getClass();
```



* 知道一个`class`的*完整类名*，可以通过静态方法`Class.forName()`获取

```
Class cls = Class.forName("java.lang.String");
```



上面3种方法获取的结果都是`class java.lang.String`



## 一个简单的反射示例

先简单写个类

```java
class MyClass {
    private int privateField;
    public String publicField;

    public MyClass() {}

    public void doSomething() {}
    
}
```



```java
MyClass obj = new MyClass(); //创建一个对象
Class c = obj.getClass(); //获取类名

Method[] methods = c.getDeclaredMethods(); //获取全部方法

Constructor[] constructors = c.getDeclaredConstructors(); //获取类中声明的全部构造函数

       
Field[] fields = c.getDeclaredFields(); // 获取类的字段

```



完整代码

```java
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args) {
        MyClass obj = new MyClass();
        Class c = obj.getClass();
        System.out.println("类名："+c);
        Method[] methods = c.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println("方法的名称：" + method.getName());
        }

        Constructor[] constructors = c.getDeclaredConstructors();
        System.out.println("构造函数的数量：" + constructors.length);
        for(Constructor con:constructors)
        {
            System.out.println("构造函数的名称：" + con.getName());
        }

        // 获取类的字段
        Field[] fields = c.getDeclaredFields();
        for (Field field : fields) {
            System.out.println("字段的名称：" + field.getName());
        }


    }
}


class MyClass {
    private int privateField;
    public String publicField;

    public MyClass() {}//构造函数

    public void doSomething() {}
}
```

运行结果：

```
类名：class MyClass
方法的名称：doSomething
构造函数的数量：1
构造函数名称：MyClass
字段的名称：privateField
字段的名称：publicField
```



## 获取字段

上面的反射示例中，使用了`getDeclaredFields()`方法获取字段，还有一个方法`getFields()`



两者的区别：

getDeclaredFields()：获得某个类的所有声明的字段，即包括public、private和proteced，但是不包括父类的申明字段。

getFields()：获得某个类的所有的公共（public）的字段，包括父类中的字段。

例如：

使用：getDeclaredFields()

```java
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args) {
       Class cls = String.class;
       System.out.println("类名："+cls);
       Field[] fields = cls.getDeclaredFields();
       for (Field field:fields){
           System.out.println("字段名: "+field);
       }
    }

}

```

输出：

```
类名：class java.lang.String
字段名: private final byte[] java.lang.String.value
字段名: private final byte java.lang.String.coder
字段名: private int java.lang.String.hash
字段名: private boolean java.lang.String.hashIsZero
字段名: private static final long java.lang.String.serialVersionUID
字段名: static final boolean java.lang.String.COMPACT_STRINGS
字段名: private static final java.io.ObjectStreamField[] java.lang.String.serialPersistentFields
字段名: private static final char java.lang.String.REPL
字段名: public static final java.util.Comparator java.lang.String.CASE_INSENSITIVE_ORDER
字段名: static final byte java.lang.String.LATIN1
字段名: static final byte java.lang.String.UTF16
```

使用：getFields()

```
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args) {
       Class cls = String.class;
       System.out.println("类名："+cls);
        Field[] fields = cls.getFields();
        for (Field field:fields){
            System.out.println("字段名: "+field);
        }

    }

}

```

输出：

```
类名：class java.lang.String
字段名: public static final java.util.Comparator java.lang.String.CASE_INSENSITIVE_ORDER
```



可以通过getDeclaredFields()或getFields()获取指定的字段并修改值

例如：

先创建一个类StuInfo并继承PersonInfo：

```java
class StuInfo extends PersonInfo{
    public int age;
    private int money;

    @Override
    public String toString() {
        return "StuInfo{" +
                "name=" + name +
                ", money=" + money +
                '}';
    }
}

class PersonInfo{
    public String name = "TOM";
}
```



反射部分代码：
```java

public class Reflectdemo {
    public static void main(String[] args) throws Exception {
        Class stiClass = StuInfo.class;
        // 获取public字段"age":
        System.out.println(stiClass.getField("age"));
        // 获取继承的public字段"name":
        System.out.println(stiClass.getField("name"));
        // 获取private字段"grade":
        System.out.println(stiClass.getDeclaredField("money"));
        // 获得值,name.get里面参数需要该类对象，而不是.class
        Field name = stiClass.getField("name");
        System.out.println(name.get(stiClass.newInstance()));  //获取name的值，这里是TOM
        // 设置值
        StuInfo stuInfo = new StuInfo();
        Field money = stiClass.getDeclaredField("money");
        money.setAccessible(true);
        money.set(stuInfo,1000);
        System.out.println(stuInfo);

    }


}
```

输出结果：
```
public int StuInfo.age
public java.lang.String PersonInfo.name
private int StuInfo.money
TOM
StuInfo{name=TOM, money=1000}
```



## 获取方法

有一下几种方式可以获取：

- `Method getMethod(name, Class...)`：获取某个`public`的`Method`（包括父类）

- `Method getDeclaredMethod(name, Class...)`：获取当前类的某个`Method`（不包括父类）

- `Method[] getMethods()`：获取所有`public`的`Method`（包括父类）

```java
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args)  {
        String  s = "T0dis";
        Class cls = s.getClass();
        System.out.println(cls);

        Method[] m2 = cls.getMethods();
		System.out.println(m2.length);//输出个数90
    }
}

```



- `Method[] getDeclaredMethods()`：获取当前类的所有`Method`（不包括父类）  

```java
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args)  {
        String  s = "T0dis";
        Class cls = s.getClass();
        System.out.println(cls);
        Method[] m1 = cls.getDeclaredMethods();
		System.out.println(m1.length);//输出个数141

    }
}

```



简单来通过反射来使用`substring`方法:

```java
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args) throws Exception {
        Class stringClass = String.class;
        Method substringMethod = stringClass.getMethod("substring", int.class, int.class);
        String str = "Hello, World!";
        Object result = substringMethod.invoke(str, 7, 12);
        String subStr = (String) result;
        System.out.println(subStr); // 输出 World
    }

}
```



这里的示例中获取的是有参数方法的Method对象，若是获取无参数方法的Method对象，例如：

```java
Class stringClass = String.class;
Method lengthMethod = stringClass.getMethod("length");
```

`getDeclaredMethod`也是类似的用法，这里就不介绍了



**注意**：如果调用的方法是静态方法。那么invoke`方法传入的第一个参数永远为`null

```
// 获取Integer.parseInt(String)方法，参数为String:
Method m = Integer.class.getMethod("parseInt", String.class);
// 调用该静态方法并获取结果:
Integer n = (Integer) m.invoke(null, "23333");
System.out.println(n);
```



## 获取构造方法

构造方法（Constructor）是一种特殊的方法，用于在对象创建时初始化对象的成员变量。它的名称必须与类名相同，没有返回类型（包括 void），可以有参数，也可以没有参数。当创建一个对象时，构造方法会自动调用，为对象的成员变量赋初值。如果没有定义构造方法，Java会默认提供一个无参构造方法。如果定义了构造方法，Java不会再提供默认的构造方法。



通过Class实例获取Constructor的方法有下面几种：

- `getConstructor(Class...)`：获取某个`public`的`Constructor`；
- `getDeclaredConstructor(Class...)`：获取某个`Constructor`；
- `getConstructors()`：获取所有`public`的`Constructor`；
- `getDeclaredConstructors()`：获取所有`Constructor`。

调用非`public`的`Constructor`时，必须首先通过`setAccessible(true)`设置允许访问。`setAccessible(true)`可能会失败。

示例：

简单的写个类：

```java
class Person {
    public String name;
    public int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    private Person(String name) {
        this.name = name;
    }

    public void sayHello() {
        System.out.println("Hello, my name is " + name + ", I'm " + age + " years old.");
    }
}
```





获取Person类的Class对象

```java
 Class<?> clazz = Person.class;//这里的<?>是Java中的泛型语法，表示"未知类型"。在这个例子中，它表示我们不知道要使用的类的具体类型，但我们知道它是一个类。这种语法通常用于在编译时检查类型安全性，并在运行时使用具体类型
```

通过下面代码可以看出这几个方法的区别

代码中`getModifiers()` 方法是 Java 反射 API 的一部分，它用于获取指定构造函数的修饰符。修饰符是一个整数值，用于描述类，字段，方法或构造函数的访问级别等信息。

```java
// 获取所有公有构造方法
Constructor<?>[] constructors = clazz.getConstructors();
System.out.println("getConstructors():");
Arrays.stream(constructors).forEach(constructor -> {
      System.out.println(constructor.getName() + "(" + Modifier.toString(constructor.getModifiers()) + ")");
   });

 // 获取所有构造方法（包括私有构造方法）
Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();
System.out.println("\ngetDeclaredConstructors():");
Arrays.stream(declaredConstructors).forEach(declaredConstructor -> {
      System.out.println(declaredConstructor.getName() + "(" + Modifier.toString(declaredConstructor.getModifiers()) + ")");
    });

// 获取指定公有构造方法
Constructor<?> constructor1 = clazz.getConstructor(String.class, int.class);
System.out.println("\ngetConstructor():");
System.out.println(constructor1.getName() + "(" + Modifier.toString(constructor1.getModifiers()) + ")");

// 获取指定构造方法（包括私有构造方法）
Constructor<?> constructor2 = clazz.getDeclaredConstructor(String.class);
System.out.println("\ngetDeclaredConstructor():");
System.out.println(constructor2.getName() + "(" + Modifier.toString(constructor2.getModifiers()) + ")");
```

输出结果：

```
getConstructors():
Person(public)

getDeclaredConstructors():
Person(public)
Person(private)

getConstructor():
Person(public)

getDeclaredConstructor():
Person(private)
```



## 如何通过反射执行命令



简单的弹个计算机

```
import java.lang.reflect.*;

public class Reflectdemo {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Class cls = Class.forName("java.lang.Runtime");
        Method EXEC = cls.getMethod("exec",String.class);
        String comm = "calc";
        Process p = (Process) EXEC.invoke(Runtime.getRuntime(), comm);
    }

}
```





## 结束

还有很多反射的知识没写，遇到再补充吧
