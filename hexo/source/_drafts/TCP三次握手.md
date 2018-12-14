---
title: TCP三次握手
tags: Network
categories: Network
comments: true
copyright: true
abbrlink: tcp_3_way_handshake
date: 2017-08-03 20:29:33
updated: 2017-08-03 20:29:33
---

### TCP报文一些内容介绍 
- `SYN`：建立连接，synchronous
- `ACK`：确认，acknowledgement
- `PSH`：传送，push
- `FIN`：结束，finish
- `RST`：重置，reset
- `URG`：紧急，urgent
- `Sequence number`：顺序号码，用来跟踪该端发送的数据量
- `Acknowledge number`：确认号码，用来通知发送端接收数据成功

我们通过`Wireshark`抓包对`TCP`三次握手进行分析。


### 三次握手流程抓包分析

编写一个示例代码，让客户端从服务器获取当前的时间。

服务器代码 `tcp_serv_1.cc`：
```c 
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <iostream>

using std::cin;
using std::cout;
using std::endl;
using std::string;

#define MAXLINE 1024

int main(int argc, char *argv[]) {
    int listen_fd = 0;
    int conn_fd;

    struct sockaddr_in serv_addr;
    char send_line[MAXLINE + 1];

    time_t ticks;

    if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        cout << "Socket error" << endl;
        exit(0);
    }

    // 将对应字节全部置0
    bzero(&serv_addr, sizeof(serv_addr));

    // 协议族
    serv_addr.sin_family = AF_INET;

    // INADDR_ANY 指定0.0.0.0的地址，表示本机所有IP
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    // 端口指定
    serv_addr.sin_port = htons(9876);

    // 对套接字进行地址和端口绑定
    if (bind(listen_fd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        cout << "Bind error" << endl;
        exit(0);
    }

    if (listen(listen_fd, 32) != 0) {
        cout << "Listen error" << endl;
        exit(0);
    }

    cout << "Server is running..." << endl;
    for (;;) {
        if ((conn_fd = accept(listen_fd, NULL, NULL)) < 0) {
            cout << "Accept error" << endl;
            exit(0);
        }

        ticks = time(NULL);
        snprintf(send_line, sizeof(send_line), "%.24s\r\n", ctime(&ticks));

        if (write(conn_fd, send_line, sizeof(send_line)) != sizeof(send_line)) {
            cout << "Write error" << endl;
            exit(0);
        }

        if (close(conn_fd) == -1) {
            cout << "Close error" << endl;
            exit(0);
        }
    }

    return 0;
}
```

客户端代码 `tcp_cli_1.cc`：
```c 
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <iostream>

using std::cin;
using std::cout;
using std::endl;
using std::string;

#define MAXLINE 1024

int main(int argc, char *argv[]) {
    int n;
    int sock_fd;
    char receive_line[MAXLINE + 1];

    struct sockaddr_in sock_addr;

    if (argc != 2) {
        cout << "Param error" << endl;
        exit(0);
    }

    if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        cout << "Socket error" << endl;
        exit(0);
    }

    bzero(&sock_addr, sizeof(sock_addr));
    sock_addr.sin_family = AF_INET;
    sock_addr.sin_port = htons(9876);

    // 将要连接的服务器IP地址进行转换，并赋值
    if (inet_pton(AF_INET, argv[1], &sock_addr.sin_addr) <= 0) {
        cout << "inet_pton error" << endl;
        exit(0);
    }

    if (connect(sock_fd, (struct sockaddr *)&sock_addr, sizeof(sock_addr)) <
        0) {
        cout << "connect error" << endl;
        exit(0);
    }

    while ((n = read(sock_fd, receive_line, MAXLINE)) > 0) {
        receive_line[n] = 0;
        if (fputs(receive_line, stdout) == EOF) {
            cout << "Fputs error" << endl;
        }
    }
    if (n < 0) {
        cout << "Read error" << endl;
        exit(0);
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
- Flags `ACK`是1

> 服务器发送数据给客户端，发送ACK=1，表示确认位，并且收到从客户端传来的Seq_Num=X，发送Ack_Num=X+1用于确认，并且发送自身的Seq_Num=Y

#### 第三次握手内容
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


### 三次握手过程可以携带数据吗

### TCP为什么是三次握手