---
title: Java序列化与反序列化
date: 2023-03-23 13:36:04
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304012041744.png
---

# Java序列化与反序列化

## 什么是序列化和反序列化

序列化: 把Java对象转换为字节序列的过程

反序列化：把字节序列恢复为Java对象的过程

## 序列化和反序列化的意义

序列化与反序列化的设计就是用来传输数据的。

当两个进程进行通信的时候，可以通过序列化反序列化来进行传输。

**序列化的好处:**

* 能够实现数据的持久化，通过序列化可以把数据永久的保存在硬盘上，也可以理解为通过序列化将数据保存在文件中。

* 利用序列化实现远程通信，在网络上传送对象的字节序列。

**序列化与反序列化应用的场景: **

*  想把内存中的对象保存到一个文件中或者是数据库当中。
*  用套接字在网络上传输对象。
*  通过 RMI 传输对象的时候。



## 常见的序列化和反序列化协议

* XML&SOAP

XML 是一种常用的序列化和反序列化协议，具有跨机器，跨语言等优点，SOAP（Simple Object Access protocol） 是一种被广泛应用的，基于 XML 为序列化和反序列化协议的结构化消息传递协议

* JSON
* Protobuf

## 实现序列化和反序列化的方法

- java.io.ObjectOutputStream
  序列化：首先给该类传入一个文件对象(用于写入序列化结果)，然后通过调用该类的 writeObject(目标对象) 方法将目标对象写入到文件
- java.io.ObjectInputStream
  反序列化：首先给该类传入一个文件对象(用于读取文件中的序列化结果)，然后通过调用该类的 readObject() 方法将其反序列化为目标对象

## 简单实现序列化

### Serializable

将要序列化的类实现 `Serializabel` 接口（Serializable 接口是一个标记接口，不用实现任何方法。一旦实现了此接口，则表明该类的对象就是可序列化的），而且所有属性必须是可序列化的，就是如果一个可序列化的类的成员不是基本类型，也不是 String 类型，比如自己自定义的类，那这个引用类型也必须是可序列化的，否则，会导致此类不能序列化 (用 `transient` 关键字修饰的属性除外，不参与序列化过程) 。



例如：

实现一个Person类，其中`money`是被`transient`修饰的

```java
import java.io.*;

public class Person implements Serializable {
    public String name;
    public int age;
    public transient int money;
    public Person(String name, int age,int money) {
        this.name = name;
        this.age = age;
        this.money = money;
    }

}
```



实现序列化：序列化成功后会生成person.ser文件，里面存储的是person对象的所有属性和状态的字节序列

```java
import java.io.*;
public class SerializationExample {
    public static void main(String[] args) throws Exception {
        // 创建一个Person对象
        Person person = new Person("T0dis", 18,100);

                // 创建ObjectOutputStream对象
                FileOutputStream fileOut = new FileOutputStream("person.ser");
                ObjectOutputStream out = new ObjectOutputStream(fileOut);

        // 序列化Java对象
        out.writeObject(person);

        // 关闭流
        out.close();
        fileOut.close();
        System.out.println("反序列化成功");
    }
}
```



实现反序列化： 

```java
import java.io.*;
public class DeserializationExample {
    public static void main(String[] args) throws Exception {
        // 创建ObjectInputStream对象
        FileInputStream fileIn = new FileInputStream("person.ser");
                ObjectInputStream in = new ObjectInputStream(fileIn);

        // 反序列化Java对象
        Person person = (Person) in.readObject();

        // 输出反序列化后的对象属性
        System.out.println(person.name);
        System.out.println(person.age);
        System.out.println(person.money);
        // 关闭流
        in.close();
        fileIn.close();//当io流不需要使用到时，一定要进行关闭流操作，否则很可能引起内存泄漏
    }
}
```

反序列化的结果：

![image-20230401150831703](https://raw.githubusercontent.com/todis21/image/main/202304011508055.png)

可以发现transient 修饰的 money没有被序列化，所以反序列化结果中没有值，如果是字符串，它的结果将是null

###  Externalizable

Externalizable进行序列化和反序列化会比较麻烦，因为需要重写序列化和反序列化的方法，序列化的细节需要手动完成。当读取对象时，会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中。因此，实现Externalizable接口的类必须要提供一个无参的构造器，且它的访问权限为public。



创建了一个 `Person` 对象

```java
import java.io.*;

class Person implements Externalizable {
    private String name;
    private int age;

    public Person() {} // 构造函数需要提供无参构造方法

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
	
    //下面是重写方法，在下面的代码中，transient关键字就不起作用了
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(name); // 写入 name 属性
        out.writeInt(age); // 写入 age 属性
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        name = (String) in.readObject(); // 读取 name 属性
        age = in.readInt(); // 读取 age 属性
    }
}
```



序列化和反序列化的过程

```java
public class Main {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Person person = new Person("Lucy", 18);
        System.out.println("Before serialization: " + person);

        // 序列化
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(person);

        // 反序列化
        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        Person newPerson = (Person) ois.readObject();
        System.out.println("After deserialization: " + newPerson);
    }
}
```



## 反序列化后命令执行

```java
import java.io.*;

class RUN_C implements java.io.Serializable
{
    public String name;
    public String motto;
    // 自定义 readObject 方法
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException{
        //执行默认的readObject()方法
        in.defaultReadObject();
        //执行命令
        Runtime.getRuntime().exec("calc.exe");
    }
}

public class SerializeTest {
    public static void main(String [] args) throws IOException, ClassNotFoundException{
        //实例化一个可序列化对象
        RUN_C testClass = new RUN_C();
        testClass.name = "Haker by xxxxxx";
        testClass.motto = "Haker!";

        //序列化
        //将序列化后的对象写入到文件
        FileOutputStream fos = new FileOutputStream("test.ser");
        ObjectOutputStream os = new ObjectOutputStream(fos);
        os.writeObject(testClass);
        os.close();
        fos.close();

        //反序列化
        RUN_C obj = null;
        //从文件读取序列化的结果后进行反序列化
        FileInputStream fis = new FileInputStream("test.ser");
        ObjectInputStream ois = new ObjectInputStream(fis);
        obj = (RUN_C)ois.readObject();
        ois.close();
        fis.close();


        System.out.println(obj.name);
        //System.out.println(obj.motto);
    }
}

```



![image-20230401202615699](https://raw.githubusercontent.com/todis21/image/main/202304012026631.png)

使用WinHex查看序列化生成的test.ser文件：

![image-20230401202927501](https://raw.githubusercontent.com/todis21/image/main/202304012029028.png)





- AC ED：Java序列化文件的魔数。
- 00 05：版本号，其中00 05表示主版本号为0，次版本号为5。
- 73 72：常量流（常量池）的标识符，代表下面的字节流是常量池信息。
- 00 05：常量池信息的长度，即5个字节。



