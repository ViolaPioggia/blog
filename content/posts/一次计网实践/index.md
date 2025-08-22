---
title: "一次计网实践"
date: "2025-08-05"
---

## 前言
> 客户端发送一个 1000B 的数据给应用层窗口大小为 800B 的服务端，服务端会怎么处理这个数据？

大概意思是这样，题目很简单，碰到这个问题是在面试的时候，之前没有遇到过这个问题也没有看到过对应的场景题，我给的答案是如果发送的是 TCP 字节流且超过了 MSS 大小则会进行分片，
分片后如果能够接收则一个分片一个分片地接收，如果依然大于服务端的窗口大小则直接丢弃。面试官听完后说下来可以自己实践一下，因此就有了这篇记录。

## TCP
TCP 是面向字节流的网络层协议，通过滑动窗口和流控制来确保数据的可靠传输，而不会丢失数据


tcp_server.c
```cgo
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUF_SIZE 800  // 设置缓冲区大小为 800 字节

void handle_client(int client_socket) {
    char buffer[BUF_SIZE];
    int bytes_received;

    // 接收数据
    while ((bytes_received = recv(client_socket, buffer, sizeof(buffer), 0)) > 0) {
        printf("Received %d bytes\n", bytes_received);
        
        // 模拟处理数据
        // 实际应用中这里可以做一些数据处理
        // 接收到数据后打印
        buffer[bytes_received] = '\0'; // 确保字符串结束
        printf("Data: %s\n", buffer);
        
        // 回送数据
        send(client_socket, buffer, bytes_received, 0);
    }

    if (bytes_received == 0) {
        printf("Client disconnected\n");
    } else if (bytes_received == -1) {
        perror("recv failed");
    }

    close(client_socket);
}

int main() {
    int server_socket, client_socket;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);

    // 创建套接字
    if ((server_socket = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("Socket failed");
        exit(1);
    }

    // 设置服务器地址
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    // 绑定套接字
    if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("Bind failed");
        close(server_socket);
        exit(1);
    }

    // 监听连接
    if (listen(server_socket, 5) == -1) {
        perror("Listen failed");
        close(server_socket);
        exit(1);
    }

    printf("Server listening on port %d\n", PORT);

    // 接受客户端连接
    if ((client_socket = accept(server_socket, (struct sockaddr *)&client_addr, &client_len)) == -1) {
        perror("Accept failed");
        close(server_socket);
        exit(1);
    }

    printf("Client connected\n");

    // 处理客户端请求
    handle_client(client_socket);

    // 关闭服务器套接字
    close(server_socket);
    return 0;
}

```

tcp_client.c
```cgo
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define PORT 8080
#define MESSAGE_SIZE 1000  // 客户端发送的消息大小（1000字节）

int main() {
    int client_socket;
    struct sockaddr_in server_addr;
    char message[MESSAGE_SIZE];
    int bytes_sent, bytes_received;

    // 填充数据发送
    memset(message, 'A', sizeof(message)); // 用字母 'A' 填充消息

    // 创建套接字
    if ((client_socket = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("Socket failed");
        exit(1);
    }

    // 设置服务器地址
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP);

    // 连接服务器
    if (connect(client_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("Connect failed");
        close(client_socket);
        exit(1);
    }

    // 发送数据
    bytes_sent = send(client_socket, message, sizeof(message), 0);
    if (bytes_sent == -1) {
        perror("Send failed");
        close(client_socket);
        exit(1);
    }
    printf("Sent %d bytes to server\n", bytes_sent);

    // 接收服务器回送的数据
    char buffer[BUF_SIZE];
    bytes_received = recv(client_socket, buffer, sizeof(buffer), 0);
    if (bytes_received == -1) {
        perror("Receive failed");
        close(client_socket);
        exit(1);
    }
    buffer[bytes_received] = '\0'; // Null-terminate received data
    printf("Received %d bytes: %s\n", bytes_received, buffer);

    // 关闭套接字
    close(client_socket);
    return 0;
}

```

服务端输出：
```shell
Server listening on port 8080
Client connected
Received 800 bytes
Data: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Received 200 bytes
Data: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Client disconnected
zsh: abort      ./tcp_server
```

由此可见，服务端一开始空闲的窗口大小是 800B，因此接收了 800B，待处理完得到新的窗口后接收了剩下的 200B

```shell
17:55:11.191337 IP localhost.52620 > localhost.http-alt: Flags [S], seq 1348506317, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 686522646 ecr 0,sackOK,eol], length 0
17:55:11.191351 IP localhost.52620 > localhost.http-alt: Flags [S], seq 1348506317, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 686522646 ecr 0,sackOK,eol], length 0
17:55:11.191506 IP localhost.http-alt > localhost.52620: Flags [S.], seq 2275530281, ack 1348506318, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 2158161082 ecr 686522646,sackOK,eol], length 0
17:55:11.191512 IP localhost.http-alt > localhost.52620: Flags [S.], seq 2275530281, ack 1348506318, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 2158161082 ecr 686522646,sackOK,eol], length 0
17:55:11.191527 IP localhost.52620 > localhost.http-alt: Flags [.], ack 1, win 6379, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191528 IP localhost.52620 > localhost.http-alt: Flags [.], ack 1, win 6379, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191538 IP localhost.http-alt > localhost.52620: Flags [.], ack 1, win 6379, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191539 IP localhost.http-alt > localhost.52620: Flags [.], ack 1, win 6379, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191653 IP localhost.52620 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 6379, options [nop,nop,TS val 686522646 ecr 2158161082], length 1000: HTTP
17:55:11.191664 IP localhost.52620 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 6379, options [nop,nop,TS val 686522646 ecr 2158161082], length 1000: HTTP
17:55:11.191681 IP localhost.http-alt > localhost.52620: Flags [.], ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191683 IP localhost.http-alt > localhost.52620: Flags [.], ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191817 IP localhost.http-alt > localhost.52620: Flags [P.], seq 1:801, ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 800: HTTP
17:55:11.191829 IP localhost.http-alt > localhost.52620: Flags [P.], seq 1:801, ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 800: HTTP
17:55:11.191836 IP localhost.52620 > localhost.http-alt: Flags [.], ack 801, win 6367, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191841 IP localhost.52620 > localhost.http-alt: Flags [.], ack 801, win 6367, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191948 IP localhost.52620 > localhost.http-alt: Flags [F.], seq 1001, ack 801, win 6367, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191968 IP localhost.http-alt > localhost.52620: Flags [P.], seq 801:1001, ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 200: HTTP
17:55:11.191970 IP localhost.52620 > localhost.http-alt: Flags [F.], seq 1001, ack 801, win 6367, options [nop,nop,TS val 686522646 ecr 2158161082], length 0
17:55:11.191977 IP localhost.http-alt > localhost.52620: Flags [.], ack 1002, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191978 IP localhost.http-alt > localhost.52620: Flags [P.], seq 801:1001, ack 1001, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 200: HTTP
17:55:11.191979 IP localhost.http-alt > localhost.52620: Flags [.], ack 1002, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.191997 IP localhost.http-alt > localhost.52620: Flags [F.], seq 1001, ack 1002, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.192006 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507318, win 0, length 0
17:55:11.192008 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507319, win 0, length 0
17:55:11.192009 IP localhost.http-alt > localhost.52620: Flags [F.], seq 1001, ack 1002, win 6364, options [nop,nop,TS val 2158161082 ecr 686522646], length 0
17:55:11.192010 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507318, win 0, length 0
17:55:11.192010 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507319, win 0, length 0
17:55:11.192012 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507319, win 0, length 0
17:55:11.192017 IP localhost.52620 > localhost.http-alt: Flags [R], seq 1348507319, win 0, length 0
```

根据抓包结果来看的确如此

但是如果我们给服务端窗口设置为 1B 呢？

```cgo
#define BUF_SIZE 1
```

```shell
Server listening on port 8080
Client connected
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
Received 1 bytes
Data: A
```

```shell
12:22:33.544594 IP localhost.62034 > localhost.http-alt: Flags [S], seq 111187871, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 353277601 ecr 0,sackOK,eol], length 0
12:22:33.544615 IP localhost.62034 > localhost.http-alt: Flags [S], seq 111187871, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 353277601 ecr 0,sackOK,eol], length 0
12:22:33.544994 IP localhost.http-alt > localhost.62034: Flags [S.], seq 3261677008, ack 111187872, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 4152554222 ecr 353277601,sackOK,eol], length 0
12:22:33.545007 IP localhost.http-alt > localhost.62034: Flags [S.], seq 3261677008, ack 111187872, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 4152554222 ecr 353277601,sackOK,eol], length 0
12:22:33.545040 IP localhost.62034 > localhost.http-alt: Flags [.], ack 1, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545042 IP localhost.62034 > localhost.http-alt: Flags [.], ack 1, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545063 IP localhost.http-alt > localhost.62034: Flags [.], ack 1, win 6379, options [nop,nop,TS val 4152554222 ecr 353277601], length 0
12:22:33.545069 IP localhost.http-alt > localhost.62034: Flags [.], ack 1, win 6379, options [nop,nop,TS val 4152554222 ecr 353277601], length 0
12:22:33.545073 IP localhost.62034 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 1000: HTTP
12:22:33.545084 IP localhost.62034 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 1000: HTTP
12:22:33.545096 IP localhost.http-alt > localhost.62034: Flags [.], ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 0
12:22:33.545100 IP localhost.http-alt > localhost.62034: Flags [.], ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 0
12:22:33.545150 IP localhost.http-alt > localhost.62034: Flags [P.], seq 1:2, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545171 IP localhost.http-alt > localhost.62034: Flags [P.], seq 1:2, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545197 IP localhost.62034 > localhost.http-alt: Flags [.], ack 2, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545209 IP localhost.62034 > localhost.http-alt: Flags [.], ack 2, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545218 IP localhost.http-alt > localhost.62034: Flags [P.], seq 2:3, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545223 IP localhost.http-alt > localhost.62034: Flags [P.], seq 2:3, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545230 IP localhost.62034 > localhost.http-alt: Flags [.], ack 3, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545233 IP localhost.62034 > localhost.http-alt: Flags [.], ack 3, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 0
12:22:33.545239 IP localhost.http-alt > localhost.62034: Flags [P.], seq 3:4, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545247 IP localhost.62034 > localhost.http-alt: Flags [R.], seq 1001, ack 3, win 6379, length 0
12:22:33.545284 IP localhost.http-alt > localhost.62034: Flags [P.], seq 3:4, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
12:22:33.545302 IP localhost.62034 > localhost.http-alt: Flags [R], seq 111188872, win 0, length 0
12:22:33.545307 IP localhost.62034 > localhost.http-alt: Flags [R.], seq 1001, ack 3, win 6379, length 0
12:22:33.545308 IP localhost.62034 > localhost.http-alt: Flags [R], seq 111188872, win 0, length 0
```

我们来简单分析一下抓包的结果：
前面三次握手就不看了，直接从客户端发送第一个包开始看

```shell
12:22:33.545084 IP localhost.62034 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 6379, options [nop,nop,TS val 353277601 ecr 4152554222], length 1000: HTTP
```
客户端（localhost.62034）向 localhost.http-alt (8080) 发送了 PSH-ACK 包，每次包含 1000 字节的数据。

seq 1:1001：表示客户端发送的数据的字节序列号为 1 到 1001。

ack 1：表示客户端确认已收到服务器的数据，ACK 包的确认号为 1。

length 1000：发送的数据大小是 1000 字节。

```shell
12:22:33.545150 IP localhost.http-alt > localhost.62034: Flags [P.], seq 1:2, ack 1001, win 6364, options [nop,nop,TS val 4152554222 ecr 353277601], length 1: HTTP
```
服务端（localhost.http-alt）向客户端（localhost.62034）发送了 PSH-ACK 包，每次包含 1 字节的数据。
seq 1:2：表示服务端发送的数据的字节序列号为 1 到 2。

ack 1001：表示服务端确认已收到客户端的数据，ACK 包的确认号为 1001。

length 1：发送的数据大小是 1 字节。

```shell
12:22:33.545247 IP localhost.62034 > localhost.http-alt: Flags [R.], seq 1001, ack 3, win 6379, length 0
```
客户端收到服务端发来的第一条数据后，发送一个 RST-ACK 包 close 掉连接


**因此可以得出结论即使是改到 1 字节，服务端依然能够正常接收数**

可以看到抓包结果有 win 6364 的字样，如果我们直接改 TCP 的窗口大小，这时候还能正常接收吗？

首先我们试图直接在代码里设置窗口大小，但是发现并不能生效，原因是过小会自动被设置为操作系统上的参数
```cgo
    int recv_buffer_size = 1;
    if (setsockopt(server_socket, SOL_SOCKET, SO_RCVBUF, &recv_buffer_size, sizeof(recv_buffer_size)) == -1) {
    perror("setsockopt SO_RCVBUF failed");
    }
```

通过 sysctl net.inet.tcp 命令可以看到接收和发送窗口大小都是 131072 Byte
```cgo
net.inet.tcp.mssdflt: 512
net.inet.tcp.keepidle: 7200000
net.inet.tcp.keepintvl: 75000
net.inet.tcp.sendspace: 131072
net.inet.tcp.recvspace: 131072
```

修改参数后发现接收窗口已经被修改为 1 Byte
```shell
sudo sysctl -w net.inet.tcp.recvspace=1
```

```shell
tcpdump: data link type PKTAP
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type PKTAP (Apple DLT_PKTAP), snapshot length 524288 bytes
10:58:52.298006 IP localhost.64394 > localhost.http-alt: Flags [S], seq 574499454, win 8193, options [mss 16344,nop,wscale 6,nop,nop,TS val 933892012 ecr 0,sackOK,eol], length 0
10:58:52.298029 IP localhost.64394 > localhost.http-alt: Flags [S], seq 574499454, win 8193, options [mss 16344,nop,wscale 6,nop,nop,TS val 933892012 ecr 0,sackOK,eol], length 0
10:58:52.298237 IP localhost.http-alt > localhost.64394: Flags [S.], seq 2855353296, ack 574499455, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 2325854558 ecr 933892012,sackOK,eol], length 0
10:58:52.298249 IP localhost.http-alt > localhost.64394: Flags [S.], seq 2855353296, ack 574499455, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 2325854558 ecr 933892012,sackOK,eol], length 0
10:58:52.298267 IP localhost.64394 > localhost.http-alt: Flags [.], ack 1, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298269 IP localhost.64394 > localhost.http-alt: Flags [.], ack 1, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298283 IP localhost.64394 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 1000: HTTP
10:58:52.298285 IP localhost.http-alt > localhost.64394: Flags [.], ack 1, win 5103, options [nop,nop,TS val 2325854558 ecr 933892012], length 0
10:58:52.298289 IP localhost.64394 > localhost.http-alt: Flags [P.], seq 1:1001, ack 1, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 1000: HTTP
10:58:52.298292 IP localhost.http-alt > localhost.64394: Flags [.], ack 1, win 5103, options [nop,nop,TS val 2325854558 ecr 933892012], length 0
10:58:52.298303 IP localhost.http-alt > localhost.64394: Flags [.], ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 0
10:58:52.298307 IP localhost.http-alt > localhost.64394: Flags [.], ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 0
10:58:52.298340 IP localhost.http-alt > localhost.64394: Flags [P.], seq 1:2, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 1: HTTP
10:58:52.298353 IP localhost.http-alt > localhost.64394: Flags [P.], seq 1:2, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 1: HTTP
10:58:52.298362 IP localhost.64394 > localhost.http-alt: Flags [.], ack 2, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298370 IP localhost.64394 > localhost.http-alt: Flags [.], ack 2, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298376 IP localhost.http-alt > localhost.64394: Flags [P.], seq 2:4, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 2: HTTP
10:58:52.298379 IP localhost.http-alt > localhost.64394: Flags [P.], seq 2:4, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 2: HTTP
10:58:52.298382 IP localhost.64394 > localhost.http-alt: Flags [.], ack 4, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298383 IP localhost.64394 > localhost.http-alt: Flags [.], ack 4, win 5103, options [nop,nop,TS val 933892012 ecr 2325854558], length 0
10:58:52.298391 IP localhost.http-alt > localhost.64394: Flags [P.], seq 4:5, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 1: HTTP
10:58:52.298394 IP localhost.64394 > localhost.http-alt: Flags [R.], seq 1001, ack 4, win 5103, length 0
10:58:52.298405 IP localhost.http-alt > localhost.64394: Flags [P.], seq 4:5, ack 1001, win 5088, options [nop,nop,TS val 2325854558 ecr 933892012], length 1: HTTP
10:58:52.298407 IP localhost.64394 > localhost.http-alt: Flags [R.], seq 1001, ack 4, win 5103, length 0
10:58:52.298415 IP localhost.64394 > localhost.http-alt: Flags [R], seq 574500455, win 0, length 0
10:58:52.298437 IP localhost.64394 > localhost.http-alt: Flags [R], seq 574500455, win 0, length 0
```

发现还是没有修改成功，原因可能是命中了操作系统的最小设置

到这里就有点慌，不敢再直接在操作系统里修改了，于是改换 Linux 开发机

查看窗口大小和窗口缩放，目前 TCP 接收缓冲区设置为了最小 4096 字节，默认 87380 字节，最大 33554432 字节
```shell
wanziyu@n37-020-116:~/tcp-demo$ cat /proc/sys/net/ipv4/tcp_rmem
4096    87380   33554432
wanziyu@n37-020-116:~/tcp-demo$ cat /proc/sys/net/ipv4/tcp_window_scaling
1
```

修改最小窗口值
```shell
echo "1 87380 33554432" | sudo tee /proc/sys/net/ipv4/tcp_rmem 
```

但还是未能修改成功，之后的内容下次有空再更新