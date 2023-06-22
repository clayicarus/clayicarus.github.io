---
layout: post
title: Linux 套接字 API 简明调用框架
date: "2022-10-31 00:44:10"
tags: [linux, c, network programming]
abstract: "&emsp;&emsp;简单归纳了 Linux 环境下的 socket API 的调用通用流程，以及适用于**已连接套接字**的相关 API。"
comments: false
---

### 前置概念

&emsp;&emsp;**套接字**是网络数据传输用的软件设备，表示两台计算机之间的**连接（connection）**。其中套接字可再细分为**监听套接字**和**已连接套接字**。

&emsp;&emsp;**监听套接字**则用于监听欲连接的主机，对于一个到达的新连接，对其生成一个**已连接套接字**。已连接套接字则为处于 ESTABLISHED 状态的套接字，用于与已连接客户通信。

### 套接字的初始化及其配置

&emsp;&emsp;我们可以将从初始化套接字到建立套接字之间的连接的过程比作从安装电话到接通电话的过程。

&emsp;&emsp;`socket()` 函数可以用于**初始化**一个套接字，我将其看作安装电话这一步骤。其中该函数的用法如下。参数的含义留到后续的文章再作进一步的解释。

{% highlight c %}

// 返回**套接字**的**文件描述符（sockfd）**，失败则返回 -1。
int socket(int domain, int type, int protocol);

{% endhighlight %}

&emsp;&emsp;初始化一个套接字之后，我们还需要对套接字**绑定（bind）**合适的地址和端口。我将这个过程比作给电话分配号码的过程。这里需要注意的是，不调用 `bind()` 不会影响后续的 `connect()` 或是 `listen()` 的调用，若没有调用 `bind()` 则内核会自动选择 IP 地址和合适的端口（对于监听套接字同样适用，但是一般不会这么做）。

{% highlight c %}

// 成功返回 0，失败则 -1。
int bind(int sockfd, struct sockaddr *addr, socklen_t addrlen);

{% endhighlight %}

&emsp;&emsp;如果仅仅希望使用该套接字去**连接（connect）**到**监听套接字**，可以直接调用 `connect()` 函数而不需要调用 `bind()`，若成功调用，则该套接字就会转化为**已连接套接字**。

{% highlight c %}

// 成功返回 0，失败则返回 -1。
/* pserv_addr 目标服务器的 addr。 */
int connect(int csk, sockaddr *pserv_addr, sicklen_t addrlen);

{% endhighlight %}

&emsp;&emsp;再讨论监听套接字的情况。**绑定（bind）**套接字之后，自然就是将套接字转换为**监听套接字**。这里我将其比作给电话接电话线的过程。

&emsp;&emsp;调用 `listen()`，将套接字转换为**监听套接字**，并指定**排队（backlog）**的最大连接个数。这里需要注意的是 `backlog` 参数并不完全等于实际的最大排队连接个数，`backlog = 0` 的行为是未定义的。

{% highlight c %}

// 成功返回 0，失败则 -1。
int listen(int sockfd, int backlog);

{% endhighlight %}

&emsp;&emsp;经过一系列的操作，我们终于得到了**监听套接字**，此时已经可以接收到来自其他套接字的**连接（connect）**，对于到达的连接，我们需要调用 `accept()` 来建立连接。这里我将其比作接听电话的过程。

{% highlight c %}

// 成功则返回一个新的文件描述符，对应一个新的已连接套接字，失败则返回 -1。
int accept(int sockfd, struct sockaddr *paddr, socklen_t *addrlen);

{% endhighlight %}

### 建立连接后的操作

&emsp;&emsp;简单提一下建立连接后的数据交换、获取对端或本端的地址信息需要用到的 API，以及其中的一些特性。

**数据交换**

&emsp;&emsp;数据交换通常会用到 `read()` 和 `write()` 函数。以下是阻塞情况下 `read()` 函数的返回值。

| 接收内容            | 返回值       |
| ------------------- | ------------ |
| 数据或数据捎带的ACK | 数据的字节数 |
| FIN                 | EOF          |
| 调用出错            | -1           |

&emsp;&emsp;对于阻塞式 IO，未知需接收数据大小时，需自定义应用级 EOF 或利用 FIN 作为接收完毕的标志。

**关闭连接**

&emsp;&emsp;对已连接套接字调用 `close()` 以关闭该套接字对应的连接。调用后引用计数减 1，当引用计数为 0 时向对端发送 FIN。

**获取对端/本端的套接字协议地址信息**

&emsp;&emsp;可以通过调用 `getsockname()` 或 `getpeername()` 来获取某个套接字对应的 `sockaddr`。

{% highlight c %}

#include<sys/socket.h>
int getsockname(int fd, sockaddr *localaddr, socklen_t *addrlen);
int getpeername(int fd, sockaddr *peeraddr, socklen_t *addrlen);

{% endhighlight %}

&emsp;&emsp;调用 `getsockname()` 获取与该套接字相关联的本端协议地址。在未进行**绑定（bind）**的已连接套接字上获得内核赋予该连接（必须已经建立连接）的本地 IP 地址和本地端口。需在成功调用 `connect()` 或 `accept()` 后调用。调用 `getpeername()` 则可以获得对端的协议地址信息。

