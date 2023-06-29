---
title: Java代理与动态代理
date: 2023-04-02 21:48:49
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304031628382.png
---

# Java代理与动态代理

## 代理

代理是一种设计模式，提供了对目标对象额外的访问方式，即通过代理对象访问目标对象，这样可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。

简言之，代理模式就是设置一个中间代理来控制访问原目标对象，以达到增强原对象的功能和简化访问方式。

![代理模式UML类图](https://raw.githubusercontent.com/todis21/image/main/202304031625114.png)

## 静态代理

静态代理是一种代理模式，它在程序运行之前就已经存在代理类的字节码文件，所以又称为编译时代理或者普通代理。在使用静态代理模式时，需要手动编写代理类，并在其中实现目标对象的方法调用。

下面举个例子： 黄*杰学长给学弟Tree买饭，买饭回来的路上还去买了奶茶

getrice.java 定义一个接口“买饭”

```java
//抽象对象：买饭
public interface getrice {
    public void Getrice();
}

```

Person.java 真实角色, 实现getrice接口

```java
public  class Person implements getrice {
    public void Getrice() {
        System.out.println("买饭");
    }
}
```

StaticProxy.java 静态代理, 

```java
public class StaticProxy implements getrice{
    private Person person;
    public StaticProxy(Person person)
    {
        this.person = person;
    }

    public void Getrice()
    {
        person.Getrice();
        System.out.println("路上买杯奶茶");
        System.out.println("把饭拿给学弟");
    }

}
```

Client.java

```java
public class Client {
    public static void main(String[] args) {
        //Tree要吃饭
        Person Tree = new Person();
        //找到在食堂吃饭的黄*杰学长帮忙带个饭
        StaticProxy jie = new StaticProxy(Tree);
        //学长买饭
        jie.Getrice();
    }
}
```

结果：

![image-20230403134900494](https://raw.githubusercontent.com/todis21/image/main/202304031625967.png)

在这个买饭的过程中，客户端Tree接触的是黄*杰（代理），看不到“饭”，当任然能买到饭。



## 动态代理

相对于静态代理而言，动态代理是一种更为灵活的代理模式。在使用动态代理时，代理类并不是在程序运行之前就已经存在，而是在运行时通过反射等机制动态生成。Java中实现动态代理需要借助java.lang.reflect包中的Proxy类和InvocationHandler接口。



动态代理的出现就是为了解决传统静态代理模式的中的缺点。

具备代理模式的优点的同时，巧妙的解决了静态代理代码冗余，难以维护的缺点。

在Java中常用的有如下几种方式：

### JDK 原生动态代理

getrice.java和Person.java代码不变

1. 首先实现一个InvocationHandler，方法调用会被转发到该类的invoke()方法。
2. 然后在需要使用getrice的时候，通过JDK动态代理获取getrice的代理对象。

DynamicProxy.java:

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class DynamicProxy implements InvocationHandler {
    private Object target;
    public  DynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = method.invoke(target, args);
        System.out.println("路上买杯奶茶");
        System.out.println("把饭拿给学弟");
        return result;
    }
}
```



客户使用动态代理调用

Client.java

```java
import java.lang.reflect.Proxy;
public class Client {
    public static void main(String[] args) {
        getrice Tree = new Person();

        DynamicProxy jie = new DynamicProxy(Tree);

        getrice proxy = (getrice) Proxy.newProxyInstance(
                Tree.getClass().getClassLoader(),
                Tree.getClass().getInterfaces(),
                jie);

        proxy.Getrice();
    }
}
```

运行结果是和上面的一样的





上述代码的核心关键是`Proxy.newProxyInstance`方法，该方法会根据指定的参数动态创建代理对象。

它三个参数的意义如下：

1. `loader`，指定代理对象的类加载器
2. `interfaces`，代理对象需要实现的接口，可以同时指定多个接口
3. `handler`，方法调用的实际处理者，代理对象的方法调用都会转发到这里

`Proxy.newProxyInstance`会返回一个实现了指定接口的代理对象，对该对象的所有方法调用都会转发给`InvocationHandler.invoke()`方法。

因此，在`invoke()`方法里我们可以加入任何逻辑，比如修改方法参数，加入日志功能、安全检查功能等等……



getClass().getClassLoader()：

getClass() 方法是 Java Object 类中的一个方法，它的作用是返回当前对象所属类的 Class 对象。而 getClassLoader() 则是 Class 类中的一个方法，它可以用来获取该类的类加载器。在这段代码中，getClass().getClassLoader() 表示获取当前对象所属类的类加载器。具体来说，getClass() 方法会返回 DynamicProxy 类的 Class 对象，而 getClassLoader() 方法则会获取该类的类加载器。



### cglib 动态代理

JDK动态代理是基于接口的,如果没有接口，就用cglib 动态代理

[CGLIB](https://github.com/cglib/cglib)(*Code Generation Library*)是一个基于[ASM](http://www.baeldung.com/java-asm)的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成。CGLIB通过继承方式实现代理。

首先，我们需要引入CGLIB库，例如Maven项目中的依赖：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.3.0</version>
</dependency>
```

然后，我们定义一个类`HelloService`，不需要实现任何接口：

```java
public class HelloService {
    public void sayHello() {
        System.out.println("Hello World!");
    }
}
```

接着，定义一个实现`MethodInterceptor`接口的代理处理器`HelloServiceProxy`：

```java
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class HelloServiceProxy implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before sayHello");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("After sayHello");
        return result;
    }
}
```

最后，在`main`方法中使用代理对象调用原始对象方法：

```java
import net.sf.cglib.proxy.Enhancer;

public class Main {
    public static void main(String[] args) {
        HelloService helloService = new HelloService();
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(HelloService.class);
        enhancer.setCallback(new HelloServiceProxy());

        HelloService proxy = (HelloService) enhancer.create();

        proxy.sayHello();
    }
}
```



输出结果：

```html
Before sayHello
Hello World!
After sayHello
```

其实CGLIB和JDK代理的思路大致相同

上述代码中，通过CGLIB的`Enhancer`来指定要代理的目标对象、实际处理代理逻辑的对象。

最终通过调用`create()`方法得到代理对象，对这个对象所有非final方法的调用都会转发给`MethodInterceptor.intercept()`方法。

在`intercept()`方法里我们可以加入任何逻辑，同JDK代理中的`invoke()`方法

通过调用`MethodProxy.invokeSuper()`方法，我们将调用转发给原始对象，具体到本例，就是`Landlord`的具体方法。CGLIG中`MethodInterceptor`的作用跟JDK代理中的`InvocationHandler`很类似，都是方法调用的中转站。





### javasist 动态代理

偷懒

pass



## 静态代理和动态代理的区别

静态代理：由程序员创建或者是由特定工具创建，在代码编译时就确定了被代理的类是一个静态代理。静态代理通常只代理一个类；
动态代理：在代码运行期间，运用反射机制动态创建生成。动态代理代理的是一个接口下的多个实现类。

