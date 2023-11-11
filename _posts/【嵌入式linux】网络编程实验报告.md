---
title: 【嵌入式linux】网络编程实验报告
date: 2023-11-07 11:16:01
tags: 
- linux
- 嵌入式
- 网络编程
- 作业
---
# linux作业 | 网络编程
## 作业实验环境
- 硬件：`树莓派3b+`
- 系统：`Raspbian GNU/Linux 11 (bullseye)`
- 编辑器与连接方式：`vscode远程ssh`
- 编译器：`gcc version 10.2.1 20210110 (Raspbian 10.2.1-6+rpi1) `
## 作业概述
通过实际运行tcp/udp实例代码，了解linux网络编程，熟悉linux网络编程基本过程。
本次作业报告分为两个部分：
- TCP回环测试
- UDP回环测试
将分别记录各自过程
## TCP回环实验测试
### 实验效果
![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/Pasted%20image%2020231029154045.png)
如上图所示，左侧为 `tcp_client` 端，右侧为 `tcp_server` 端，对应为各自所运行的终端。
- 先运行右侧 `server` 端，传入参数为 `127.0.0.1 55566` ，开始阻塞等待 `client` 端连接
- 再运行左侧 `client` 端，传入参数一致，此时右侧 `server` 端打印 `client` 端 `ip` 与 `port` ，结果为 `127.0.0.1:41292` ，可见 `client` 端程序为其套接字分配了一个随机的未被占用的端口
- 此时左侧 `client` 端输入消息，将会在右侧 `server` 端显示，并且收到 `server` 端的回传
- 左侧 `client` 输入 `quit` 后，将会将其发送给 `server` 后退出程序，同样，`server` 接收到 `quit` 后也会将自己关闭
- 另外，在 `server` 端输入的东西将会被存储，程序没有 `fget` ，因此在退出终端之后，终端将会读取之前输入的命令
下面逐步总结各自程序的过程
### tcp_client端过程
#### 定义变量
- `int sockfd;` - socket file descriptor - 套接字描述符
	- 用于存储后面创建的套接字
- `struct sockaddr_in serveraddr;` - 存储 `server` 端信息所用结构体
	- 存储地址与端口
	- `sockaddr_in` 源码注释： `/* Structure describing an Internet socket address.  */`
	- 后面暂时看不懂，手敲了一下好像有四个成员变量（ #need_to_search
	  ![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/Pasted%20image%2020231029160553.png))
- 地址长度和缓冲区略
#### 异常处理
- 传入参数合法与否
- 调用 `socket` 函数，创建套接字
	- `AF_INET` - Address Family_Internet - IPv4地址族
	- `SOCK_STREAM` - 面向流的TCP套接字（？这里的stream和NanoPB的stream是不是一个东西 #need_to_search
	- `0` - 表示自动选择协议，一般为TCP
	- 若创建失败返回 `-1`，并设置全局变量 `errno` 来指示发生了什么错误。通常情况下，如果连接失败，可以通过调用 `perror("socket error");` 来输出与连接失败相关的错误消息。
- 向 `serveraddr` 结构体中存储成员变量
	- `sin_family` - 用于指定地址族
	- `sin_port` 与 `sin_addr` - 端口和地址
		- ？为啥 `sin_addr` 还要 `.s_addr`
		> **拓展搜索**：`sin_addr.s_addr` 表示了 `struct sockaddr_in` 结构体中的 IP 地址字段。这样的结构设计是为了与网络编程中的网络字节序（大端Big-endian）相一致，同时也考虑了C语言的底层数据表示。
		- `inet_addr` - 点分十进制地址转换为32位整数
		- `atoi` 和 `htons` - ASCII to Integer 和 Host to Network Short
			- 一个是把阿斯克码转整形
			- 一个是把16位整数转网络字节序 （不是，那为啥port为啥不像addr一样分一个s_port啊
- 调用 `connect` 函数，根据 `套接字` + `地址信息` 向 `server` 端发送连接请求
	- `sockfd` - 存储的套接字
	- `(struct sockaddr *)&serveraddr` - 一个指向服务器地址信息的指针
		- 强制类型转换：`serveraddr` 是前面定义的 `struct sockaddr_in` 结构体，该函数需要的结构体类型为 `sockaddr` 
			- 拓展搜索：`serveraddr` | `serveraddr_in` | `serveraddr_in6` 之间的关系
				- 不管是`serveraddr_in` | `serveraddr_in6` ，都是`serveraddr` 的一个特殊形式，其中 `_in` 给IPv4用， `_in6` 给IPv6用
				- 这里的 `in` 不是输入的意思，而是 `Internet` 的意思
				- `sockaddr_in` 表示 "Socket Address Internet"，用于表示IPv4地址。
				- `sockaddr_in6` 表示 "Socket Address Internet version 6"，用于表示IPv6地址。
	- `addrlen` - 长度
	- 若连接失败返回 `-1`，并设置全局变量 `errno` 来指示发生了什么错误。通常情况下，如果连接失败，可以通过调用 `perror("connect error");` 来输出与连接失败相关的错误消息。
#### 主循环
- `fget` - 获取用户输入，丢到 `buf` 里面，删掉最后的换行符 `'\0'`
- 调用 `send` 函数，传入套接字 + 内容 + 内容大小 + 可选标志
	- 前面三个参数没啥说的，主要是可选标志：
		1. 0（默认）：通常情况下，将最后一个参数设置为0表示不使用特殊标志，即使用默认的发送行为。  
		2. `MSG_DONTWAIT`：如果将参数设置为 `MSG_DONTWAIT`，则 `send` 函数将以非阻塞方式发送数据。这意味着即使套接字的发送缓冲区已满，`send` 也会尽力发送尽可能多的数据，并立即返回，不会阻塞程序的执行。
		3. `MSG_NOSIGNAL`：这个标志通常在Linux系统上使用，它告诉操作系统不要发送 `SIGPIPE` 信号，即使套接字的远程端关闭了连接，也不会中断程序。这在某些情况下可以防止程序因为连接断开而崩溃。
		4. 其他特定标志：`send` 函数还可以接受其他特定于操作系统和套接字类型的标志参数，这些参数可以根据需要进行设置。这些标志通常用于高级用例，例如对套接字选项的设置或特殊的网络处理需求。
	- 若发送失败返回 `-1`，并设置全局变量 `errno` 来指示发生了什么错误。通常情况下，如果连接失败，可以通过调用 `perror("send error");` 来输出与连接失败相关的错误消息。
- `strncmp` 判断是否输入了 `quit` ，是则退出
- 调用 `recv` 函数，与 `send` 类似，略
- `printf` 在终端打印server回环的消息

没了
### tcp_server端过程
#### 定义变量
这部分和 `client` 端差不多，套接字描述符多了一个 `acceptfd` ，用于接受客户端用的，sockaddr_in 结构体也多了一个 `clientaddr` ，用于存储客户端的地址信息
`bzero` 函数将服务端和客户端的地址信息结构体初始化为零
#### 异常处理
- 传入参数合法与否
- 创建套接字，同上
- 向 `serveraddr` 结构体中存储成员变量，同上
- 调用 `bind` 函数，将套接字和 `serveraddr` 绑定，以便监听客户端的连接请求
- 调用 `listen` 函数，将套接字设置为监听模式，等待客户端的连接请求，`5` 表示等待连接队列的最大长度
- 调用 `accept` 函数，当有新的 `client` 连接请求后，该函数将会为这个特指的 `client` 返回一个新的套接字（？这里我的表述是不是有点问题）
	- `(struct sockaddr *)&clientaddr`：这是一个指向 `clientaddr` 结构体的指针，用于存储客户端的地址信息。当 `accept` 函数接受连接时，它会将客户端的地址信息填充到这个结构体中，以便后续使用。
	- `&addrlen` 是一个指向 `addrlen` 变量的指针，它用于指定 `clientaddr` 结构体的大小，以便 `accept` 函数知道要填充多少字节的地址信息。`addrlen` 在之前的代码中已经设置为 `sizeof(serveraddr)`，表示这个结构体的大小。
	- 这两个参数可以设置为 `NULL` 表示不关心 `client` 的信息

> **拓展搜索**：这个 `accept` 套接字和`sockfd` 有什么区别，为什么要设置两个套接字？
> - 首先明确，网络通信中一般服务器往往会处理很多个客户端的通信，因此需要将用于监听连接请求的套接字和用于数据传输的套接字分开，也就是说`accept` 套接字和 `sockfd` 具有不同的角色和用途，这是为了支持服务器端的并发连接处理。
> - 所以，`sockfd` 负责监听并接受多个客户端的连接请求，而 `acceptfd` 用于与单个客户端进行数据通信。这种分离允许服务器并发处理多个客户端的连接和通信，提高了服务器的性能和效率。当一个客户端连接时，服务器会为其创建一个独立的 `acceptfd`，这种模型被称为并发服务器模型，它允许多个客户端同时与服务器通信，而不会相互干扰。

- 没啥问题，上面的都跑完了就打出客户端的 `IP` 和 `port` 
#### 主循环
跟 `client` 端稍稍有点区别
- 接收消息还是调用 `recv` 函数，不过注意这里使用的套接字是 `acceptfd` ，原因上面有记录
	- 没东西就 `perror("no data");` 
	- 客户端叫 `quit` 就退出，`goto` 操作，关闭两个套接字。
	- 上面俩特例都没有，就正常打印客户端消息，并且 `strcat` 之后传回去给 `client` 一个响应

 没了
## UDP回环实验测试
### 实验效果
![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/Pasted%20image%2020231029174022.png)
实验效果跟tcp大差不差，只是右侧的 `server` 端的显示每一次接收到消息都会打印出地址和端口
### udp_client端过程
跟上面tcp的过程也差不多，主要留意几个区别的地方：
- UDP（User Datagram Protocol）套接字，与TCP（Transmission Control Protocol）套接字的创建有一些关键区别：
	- 套接字类型：
		- TCP套接字：在TCP通信中，使用 `SOCK_STREAM` 类型的套接字，它是一种**面向连接的、可靠的、基于流**的传输协议。TCP提供了数据的有序传输、错误检测和重传机制，适用于可靠性要求较高的应用。
		- UDP套接字：在UDP通信中，使用 `SOCK_DGRAM` 类型的套接字，它是一种**面向数据包的、不可靠的**传输协议。UDP不保证数据的可靠性、有序性或重传，适用于需要低延迟和较少开销的应用。
	- 连接性：
		- TCP套接字：TCP是面向连接的协议，它要求在通信前建立连接，有客户端和服务器之间的明确连接。使用 `socket`、`bind`、`listen`、`accept` 等函数来建立连接。
		- UDP套接字：UDP是非连接性的协议，通信不需要事先建立连接，可以直接发送数据包。UDP通信中的套接字不需要监听和接受连接请求。
	- 可靠性：
		- udp没啥可靠的（），主要就是快，怎么快怎么来，数据包可能会丢失，但是适用于实时要求高的应用

### udp_server端过程
跟tcp差不多，主要有这个区别：
- 只开了 `sockfd` 一个套接字
其他的略过
## 总结
在本次udp和tcp回环实验的过程中，发现之前学习的网络编程中有一个误解：
之前认为一个套接字绑定的是一个进程，以前一直以为这个“进程”指的是一个.c文件或者这个.c文件编译出的可执行文件，但是通过tcp的实验发现，貌似并不是这样。
