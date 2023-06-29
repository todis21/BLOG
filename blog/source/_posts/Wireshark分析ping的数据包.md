---
title: Wireshark分析ping的数据包
date: 2021-10-11 18:14:07
tags:
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-ymrvxx_1920x1080.png
---

#  Wireshark分析ping的数据包

## 1.抓取ping数据包

打开Wireshark  开启抓包，打开cmd输入ping github.com 回车

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20133742.png)

在Wireshark的中条件过滤栏输入“icmp"

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20133907.png)

可以看到8个报文，其中有4个请求报文和4个应答报文

## 2.分析其中一个报文（request）

### 物理层

可以看到ping request数据链路层报文一共有74个字节，帧序号为258，使用的协议为icmp协议

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20134012.png)



### 数据链路层

这里有发送数据帧的源节点MAC地址和接收数据帧的目标节点MAC地址

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20134135.png)



### 网络层

这里包含着发送数据帧的源节点IP地址和接收数据帧的目标节点IP地址

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20134222.png)

### 传输层

![](https://raw.githubusercontent.com/todis21/image/main/img/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202021-10-18%20134331.png)



