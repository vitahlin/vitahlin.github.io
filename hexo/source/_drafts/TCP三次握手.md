---
title: TCP三次握手
tags: Network
categories: Network
comments: true
copyright: true
abbrlink: TCP三次握手
date: 2017-08-03 20:29:33
updated: 2017-08-03 20:29:33
---

TCP标志位，有6种标示：
`SYN`：建立连接，synchronous
`ACK`：确认，acknowledgement
`PSH`：传送，push
`FIN`：结束，finish
`RST`：重置，reset
`URG`：紧急，urgent
`Sequence number`：顺序号码
`Acknowledge number`：确认号码

我们通过`Wireshark`抓包对`TCP`三次握手进行分析。

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

