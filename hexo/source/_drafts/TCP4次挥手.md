---
title: TCP4次挥手
tags: Network
categories: Network
comments: true
copyright: true
abbrlink: tcp_sequence_number
date: 2018-12-15 01:47:33
updated: 2018-12-15 01:47:33
---

### 客户端发起关闭


### 服务器发起关闭

客户端代码：[https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_cli.cc](https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_cli.cc)

服务器代码：[https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_srv.cc](https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_srv.cc)

客户端对服务器发起连接，连接后，服务器将当前时间戳发给客户端，然后关闭连接。通过`Wireshark`抓包如下：
![tcp_4_times_close_1]()

序号5-8属于TCP三次握手的内容，这里不做介绍。具体看序号9-14的报文内容。

序号9的报文将当前时间字符串发送给客户端，其中的数据长度为`4097`，序号为11的报文则是客户端告诉服务器这部分的字节数据全部收到。

**产生四次挥手操作的报文即10、12、13、14。**


1. 序号10报文。服务端发送FIN给客户端，告知客户端服务器已经没有数据发送。Seq_Num=X
2. 序号12报文。客户端收到FIN，返回ACK=1，Ack_Num=X+1，并且关闭客户端的读通道.
3. 服务器收到客户端返回的ACK报文，关闭服务器的写通道。（此时服务器仍能通过读通道读取客户端发过来的数据）
4. 序号13报文。客户端发送FIN，告知服务器客户端也没有数据发送。
5. 序号14报文。服务器收到FIN，返回ACK=1。关闭服务器读通道。
6. 客户端收到ACK=1的报文，关闭客户端写通道。

### 四次挥手中的各种状态