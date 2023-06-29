---
title: s q l注入小结
date: 2021-05-26 11:54:09
tags: web
cover: https://browser9.qhimg.com/bdr/__85/t01c3dca97966573c74.jpg
---

# s q l 注入的小总结

## 原理：

​	就是通过把SQL命令插入到Web表单递交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。具体来说，它是利用现有应用程序，将（恶意）的SQL命令注入到后台数据库引擎执行的能力，它可以通过在Web表单中输入（恶意）SQL语句得到一个存在安全漏洞的网站上的数据库，而不是按照设计者意图去执行SQL语句。

## s q l注入攻击一般流程：

## 1.判断注入点：

​	判断一个链接是否存在注入漏洞，可以通过对其传入的参数（但不仅仅只限于参数，还有cookie注	入，HTTP头注入等） 进行构造，然后对服务器返回的内容进行判断来查看是否存在注入点。

## 2.判断注入点类型

* 按照参数类型分类：

  ​    (1)数字型注入： 如id=1 ，传入的参数是数字，注入时该参数不需要用单或双引号构造闭合。      				  

  ​	(2)字符型注入：如username=admin  ,传入的参数是字符或字符串，注入时要注意去构造闭合。

* 按照数据请求方式分类:

  ​	(1)GET注入

  ​	(2)POST注入

  ​	(3)HTTP请求头注入

* 按照语句执行效果分类：

  ​	(1)union联合查询注入

  ​	(2)报错注入

  ​	(3)堆叠注入

  ​	(4)宽字节注入

  ​	(5)基于布尔的盲注

  ​	(6)基于时间的盲注

  ​	(7)二次注入

  ​	(8)cookie注入 - http请求头参数注入

  ​	(9)base64注入攻击

### 3.判断数据库类型

* 常见网页类型对应的数据库：

| 网页类型 | 数据库            |
| -------- | ----------------- |
| PHP      | MySQL             |
| asp      | Access/SQL server |
| .net     | SQL server        |
| java     | oracle/MySQL      |
| ...      | ...               |

* 可以通过数据库报错信息来判断数据库类型，例如输入单双引号；
* 可用特殊字符或注释判断：
  1,“null”和“%00”是Access支持的注释。
  2，“#”是MySQL中的注释符，返回错误说明该注入点可能不是MySQL，另外也支持’-- ',和/* */注释（注意mysql使用-- 时需要后面添加空格）
  3，“–”和/* */是Oracle，SQL server和MSSQL支持的注释符，如果正常，说明可能就是这仨了。
  4，“;”是子句查询标识符，在Oracle中不支持多行查询，返回错误，很可能是Oracle数据库。
  这样一串下来，基本就知道了数据库类型了

### 4.获取数据提权

这方面鄙人不熟，各位大佬轻喷....

### 5.过滤绕过：

在实战或ctf中，网站前后端会将用户输入的字符进行过滤，这时就要通过特殊手段绕过过滤。以下是网上文章整合内容

#### (1)进行fuzz模糊测试：

输入一些特殊字符或关键字进行测试，这个主要是为了知道数据 库对那些字符或关键字进行了过滤 ，然后对症下药。可以用burpsuite 结合字典来测试。字典部分截图如下，



​	 ![截图 2020-05-28 172126](/img/sql注入字典.png)

#### (2)空格被过滤：

> 1.使用注释符/ * * /，内联注释：/ * !不带小数点的版本号+关键字*/
> （实战和CTF简单题有些用）
> 2.利用apache的CVE（并且是mysql 5.0版本以上）：
> %09 TAB键（水平）、%0a 新建一行、%0c 新的一页、%0d return功能、
> %0b TAB键（垂直）、%a0 ，%20空格(这两个有点拉胯)
> 上述的几个字符对付不严格的正则匹配比较有用
> 举例：id=-1 union %0A select 1,2,3 from xxx
>
> 用联合查询时（union）用“（）”来代替空格
>
> 如 select(username)from(users)where(id=1)

#### (3)引号被过滤：

> 1.用16进制：0x加上16进制转换后的字符串
>
> 2.未使用addslashes,且注释和反斜杠没有被过滤的情况下：使用单引号转义，构造playload使其拼接成如下语句：
>
> ```sql
> select * from user where username='admin\' and passwd='123456';#后面剩下的语句;
> ```

> 3. 在GBK编码条件下使用宽字节注入。

#### (4)等价替换：



> and=&&=%26%26
> or=||=%7c%7c
> xor=^
>
> select在堆叠开启的时候可以用handler代替
>
> 等价函数：
> ascii()=ord()，也可以把这个换成hex,bin,to_base64等等
> char()=chr()
> mid(xxx,a,b)=substring(xxx,a,b)=substr(xxx,a,b)
> 还有一个来自我的突发奇想，但是有个缺点就是读到最后会重复返回最后一个字符
> mid(xxx,a,b)=right(left(xxx,a+b),b)
> strcmp(left(xxx,a),b)，功能类似于like
>
> where被过滤：
> from table where id=1 等价于
> from table  a join table b on a(或者b).id=1 
>
> 时间盲注中相关函数被过滤可以使用笛卡尔积
> information_schema.collations,information_schema.collations,information_schema.collations
> 三个collations表做笛卡尔积时间大约为6秒，这个表在一般情况下都是固定的数量，所以对于不同地方的数据库，时间大致都相同。
> 但是注意要在mysql 5以上才可以（只有mysql5以上才有元数据的这个表）
> 这个方法可以直接拿来替换sqlmap掉payloadxml中的sleep(5)，因为sqlmap对time-sec参数的优化是用一种数学方法，所以sleep(5)
> 只是个形式。

#### (5)逗号被过滤：

> 对于列名
>
> ```
> select 1,2 from xxx = select * from (select 1)a join (select 2)b
> ```

> 对于mid(),substr(),substring()这样的三参数字符串截取函数可以使用
>
> ```
> mid(database() from 2 for 1) = mid(database(),2,1)
> ```

> 还有一种很常用的使用like或者regexp
>
> like 'a%'
> regexp '^a'
>
> 然后逐次增加后面的字母

> limit的第二个参数用offset绕过
>
> ```
> limit 0,1 = llimit 0 offset 1
> ```

#### (6)特殊操作：

对付正则用双写绕过，如selselectect.

大小写绕过

