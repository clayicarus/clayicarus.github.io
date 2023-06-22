---
layout: post
title: Linux 套接字 API 简明调用框架
date: "2022-10-31 00:44:10"
tags: [linux, c, network]
abstract: "简单归纳了 Linux 环境的 socket API 的调用框架，并记录了一些重要的参数的含义及其传递方法。"
comments: false
---

套接字是网络数据传输用的软件设备，表连接——两台计算机之间的连接。

### 一些概念
- 监听套接字

    用于监听欲连接主机，生成已连接套接字。

- 已连接套接字

    处于ESTABLISHED状态的套接字，用于与已连接客户通信。

### 相关函数

- socket()

    安装电话。

    ```c
    //返回文件描述符(sockfd)，失败则-1。
    int socket(int domain,int type,int protocol);
    ```

- bind()

    给电话分配电话号码。

    bind()调用的有无不影响connect()或是listen()的调用。无调用bind()则内核选择IP地址和端口。

    ```c
    //成功返回0，失败则-1。
    int bind(int sockfd,struct sockaddr *paddr,socklen_t addrlen);
    ```

    TCP服务端

    | IP                     | 端口 | 结果                         |
    | ---------------------- | ---- | ---------------------------- |
    | INADDR_ANY（通配地址） | 0    | 内核选择IP和端口             |
    | INADDR_ANY（通配地址） | 非0  | 内核选择IP，进程指定端口     |
    | 本地IP地址             | 0    | 进程指定IP地址，内核选择端口 |
    | 本地IP地址             | 非0  | 进程指定IP和端口             |



- listen()

    将套接字转换为监听套接字，并指定套接字排队的最大连接个数。

    CLOSED状态转换为LISTEN状态。

    ```c
    //成功返回0，失败则-1。
    int listen(int sockfd,int backlog);
    ```

- - backlog参数

        未完成连接队列：处于SYN_RCVD状态的套接字队列。

        已完成连接队列：处于ESTABLISHED状态的套接字队列。

        上述两队列之和不超过backlog。

        - 不要把backlog定义为0。
        - 当队列满，SYN到达，则该SYN将被忽略，从而不发送RST（以防导致对端终止）。
        - 大于内核支持的最大backlog是合理的。

- connect()

    打电话。

- accept()

    听电话咯。

    ```c
    //成功返回文件描述符，失败则-1。
    int accept(int sockfd,struct sockaddr *paddr,socklen_t *addrlen);
    ```

- read()/write()

    数据交换。非阻塞情况。

- - 阻塞式I/O的函数特性

    - read()

    其余情况均阻塞。

    | 接收内容            | 返回值       |
    | ------------------- | ------------ |
    | 数据或数据捎带的ACK | 数据的字节数 |
    | FIN                 | 0（EOF）     |

    故在未知需接收数据大小时，需自定义应用级EOF或利用FIN作为接收完毕的标志。

    - write()

  

- close()

    引用计数减1。当引用次数为0时发送FIN，关闭套接字。

- getsockname()/getpeername()

    ```cpp
    #include<sys/socket.h>
    int getsockname(int fd, sockaddr *localaddr, socklen_t *addrlen);
    int getpeername(int fd, sockaddr *peeraddr, socklen_t *addrlen);
    ```

- - getsockname

    返回与该套接字相关联的本地协议地址。

    - 在未调用bind()的TCP客户端上获得内核赋予该连接（必须已经建立连接才可获得）的本地IP和本地端口。

        需在connect()成功返回后调用。

    - bind()通配地址的TCP服务端上获得内核分配的本地IP地址以及本地端口。

        需在accept()成功返回后对已连接套接字调用。

- - getpeername

    返回与该套接字相关联的外地协议地址（对端地址）。