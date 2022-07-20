---
title: TCP三次握手
<!-- description: 这是一个副标题 -->
date: 2022-07-20
slug: tcp_3_way_handshake
categories:
    - c

tags:
    - c
draft: true
---



### 概述

TCP是主机对主机层的传输控制协议，提供可靠的连接服务，采用三次握手确认建立一个连接。
这里通过`Wireshark`对简单的时间字符串TCP传输程序进行抓包，来分析三次握手的内容，理解TCP建立可靠连接的流程。

<!--more-->

### TCP报文一些内容介绍 
- `SYN`：建立连接，synchronous
- `ACK`：确认，acknowledgement
- `PSH`：传送，push
- `FIN`：结束，finish
- `RST`：重置，reset
- `URG`：紧急，urgent
- `Sequence number`：顺序号码，用来跟踪该端发送的数据量
- `Acknowledge number`：确认号码，用来通知发送端接收数据成功

### 三次握手流程抓包分析

编写一个示例代码，让客户端从服务器获取当前的时间。

服务器代码 `day_time_tcp_serv.cc`：
```c 
int main(int argc, char *argv[]) {
    int listen_fd = 0;
    int conn_fd;

    struct sockaddr_in serv_addr;
    char send_line[MAXLINE + 1];

    time_t ticks;

    listen_fd = Socket(AF_INET, SOCK_STREAM, 0);

    // 将对应字节全部置0
    bzero(&serv_addr, sizeof(serv_addr));

    // 协议族
    serv_addr.sin_family = AF_INET;

    // INADDR_ANY 指定0.0.0.0的地址，表示本机所有IP
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    // 端口指定
    serv_addr.sin_port = htons(9876);

    // 对套接字进行地址和端口绑定
    Bind(listen_fd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));

    Listen(listen_fd, LISTENQ);

    cout << "Server is running..." << endl;
    for (;;) {
        conn_fd = Accept(listen_fd, NULL, NULL);
        ticks = time(NULL);
        snprintf(send_line, sizeof(send_line), "%.24s\r\n", ctime(&ticks));
        Write(conn_fd, send_line, sizeof(send_line));
        Close(conn_fd);
    }

    return 0;
}
```

客户端代码 `day_time_tcp_cli.cc`：
```c 
int main(int argc, char *argv[]) {
    int n;
    int sock_fd;
    char receive_line[MAXLINE + 1];

    struct sockaddr_in sock_addr;

    if (argc != 2) {
        LogErrQuit("Param error");
    }

    sock_fd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&sock_addr, sizeof(sock_addr));
    sock_addr.sin_family = AF_INET;
    sock_addr.sin_port = htons(9876);

    // 将要连接的服务器IP地址进行转换，并赋值
    InetPton(AF_INET, argv[1], &sock_addr.sin_addr);

    Connect(sock_fd, (struct sockaddr *)&sock_addr, sizeof(sock_addr));

    while ((n = read(sock_fd, receive_line, MAXLINE)) > 0) {
        receive_line[n] = 0;
        if (fputs(receive_line, stdout) == EOF) {
            LogErr("Fputs error");
        }
    }
    if (n < 0) {
        LogErrQuit("Read error");
    }

    return 0;
}
```

服务器启动：
```shell 
$ ./tcp_serv_1
Server is running...
```

客户端启动：
```shell 
$ ./tcp_cli_1 127.0.0.1
Thu Dec 13 18:14:44 2018
$ 
```

代码实现的逻辑是启动服务器后，客户端连接服务器打印从服务器收到的时间字符串，然后客户端断掉连接。
接着，我们通过`Wireshark`来分析这整个流程。

抓包整个流程如下：
![tcp_3_way_handshake_1](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/tcp_3_way_handshake_1.png)

**`Wireshark`内容见附录**

#### 第一次握手 

如上图所示，选择**序号为1**的包，因为我们是要分析TCP报文，右键`Transmission Control Protocol`项，选择`Expand All`展开全部项，然后选择`Copy-All Visible Selected Tree Items`复制TCP报文中的全部内容，结果如下：
```c 
Transmission Control Protocol, Src Port: 51567 (51567), Dst Port: sd (9876), Seq: 1536975978, Len: 0
    Source Port: 51567 (51567)
    Destination Port: sd (9876)
    [Stream index: 0]
    [TCP Segment Len: 0]
    Sequence number: 1536975978
    [Next sequence number: 1536975978]
    Acknowledgment number: 0
    1011 .... = Header Length: 44 bytes (11)
    Flags: 0x002 (SYN)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Nonce: Not set
        .... 0... .... = Congestion Window Reduced (CWR): Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...0 .... = Acknowledgment: Not set
        .... .... 0... = Push: Not set
        .... .... .0.. = Reset: Not set
        .... .... ..1. = Syn: Set
            [Expert Info (Chat/Sequence): Connection establish request (SYN): server port 9876]
                [Connection establish request (SYN): server port 9876]
                [Severity level: Chat]
                [Group: Sequence]
        .... .... ...0 = Fin: Not set
        [TCP Flags: ··········S·]
    Window size value: 65535
    [Calculated window size: 65535]
    Checksum: 0xfe34 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (24 bytes), Maximum segment size, No-Operation (NOP), Window scale, No-Operation (NOP), No-Operation (NOP), Timestamps, SACK permitted, End of Option List (EOL)
        TCP Option - Maximum segment size: 16344 bytes
            Kind: Maximum Segment Size (2)
            Length: 4
            MSS Value: 16344
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Window scale: 6 (multiply by 64)
            Kind: Window Scale (3)
            Length: 3
            Shift count: 6
            [Multiplier: 64]
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps: TSval 901898369, TSecr 0
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 901898369
            Timestamp echo reply: 0
        TCP Option - SACK permitted
            Kind: SACK Permitted (4)
            Length: 2
        TCP Option - End of Option List (EOL)
            Kind: End of Option List (0)
```

分析TCP报文内容：
- Source Port 起始端口，当前是51576表示是客户端端口
- Destination Port 目的端口，9876表示是服务器端口
- Sequence number 顺序号码1536975978
- Acknowledgment number 确认号码0
- Flags 各种标志位，其中`SYN`是1

到此，我们就知道TCP三次握手的第一次握手内容：
> 客户端对服务器发起连接，发送SYN=1，以及Seq_Num=X

#### 第二次握手
同样选择**序号2**，然后查看TCP报文内容。
- Source Port 起始端口，9876表示从服务器出发
- Destination Port 目的端口，51576
- Sequence number: 3764405294
- Acknowledgment number: 1536975979
- Flags `SYN`和`ACK`是1

> 服务器发送数据给客户端，发送SYN=1，ACK=1，并且收到从客户端传来的Seq_Num=X，发送Ack_Num=X+1用于确认，并且发送自身的Seq_Num=Y

#### 第三次握手
- Source Port 51576
- Destination Port 9876
- Sequence number: 1536975979
- Acknowledgment number: 3764405295
- Flags `ACK`是1

> 客户端发送数据到服务器，发送ACK=1，收到从服务器传来的Seq_Num=Y，发送Ack_Num=Y+1确认，发送自身的当前顺序号码，第一次握手顺序号码是X，所以这一次的顺序号码Seq Num=X+1


序号为4的内容为`TCP Window Update`，这个用于窗口更新，这里不做赘述。
序号为5的内容如图：
![tcp_3_way_handshake_2](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/tcp_3_way_handshake_2.png)

可以看到，序号5中服务器对客户端发送了数据，数据内容是当时的服务器时间，这里开始就是TCP互相传输内容的部分。

可以用一张图来表现TCP三次握手的过程：
![tcp_3_way_handshake_3](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/tcp_3_way_handshake_3.png)


### TCP为什么是三次握手

我们已经知道建立连接需要三次握手，那么为什么是三次握手而不是两次？

为了实现可靠的传输，TCP协议的通信双方，都必需维护一个序列号，以标识发送出去的数据包中，哪些是已经被对方收到的。三次握手的过程即时通信双方相互告诉序列号起始值，并确认对方已经收到了序列号起始值的必经步骤。

如果只是两次握手，至多只有连接发起方的起始序列号能被确认，另一方的序列号则得不到确认。譬如发起请求遇到类似这样的情况：客户端发出去的第一个连接请求由于某些原因在网络节点中滞留了导致延迟，直到连接释放的某个时间点才到达服务端，这是一个早已失效的报文，但是此时服务端仍然认为这是客户端的建立连接请求第一次握手，于是服务端回应了客户端，第二次握手。

所以，为了保证服务端能接受到客户端的信息并能作出正确的应答而进行第一次和第二次握手；为了保证客户端能够收到服务端的信息并能作出正确的应答而进行第二次和第三次握手。

三次握手成功才表明双方已经建立的可靠的传输。

### 参考

- [https://notfalse.net/7/three-way-handshake](https://notfalse.net/7/three-way-handshake)
- [https://www.cnblogs.com/cy568searchx/p/3711670.html](https://www.cnblogs.com/cy568searchx/p/3711670.html)


##### 附录
- [day_time_tcp_cli.cc](https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_cli.cc)
- [day_time_tcp_serv.cc](https://github.com/vitahlin/UNPv1/blob/master/src/chapter1/day_time_tcp_srv.cc)
- [Wireshark抓包内容](http://vitah-blog.oss-cn-beijing.aliyuncs.com/2018/tcp_3_way_handshark_wireshark.pcapng)