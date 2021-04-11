---
title: GeekTime -《网络编程实战》笔记
date: 2019-07-26
tags:
- 笔记
- 极客时间
- 网络编程
categories:
- 极客时间
---

# 网络编程实战

### 02 网络编程模型

- 需要强调的是，无论是客户端，还是服务端，它们运行的单位都是进程，process，而不是机器。

- 在tcp/ip协议栈中，ip用来表示网络世界的地址。

- 端口号是一个16位的整数，最多为2^16，65536

- 客户端的端口是由操作系统内核临时分配的，称为临时端口；而服务端的端口通常是一个众所周知的端口

- 一个连接可以通过客户端-服务端的ip和端口号唯一确定，叫做套接字对，用下面的四元组表示：

  ```jsx
  (clientaddr:clientport,serveraddr:serverport)
  ```

  - TCP，又被叫做字节流套接字，Stream Socket；UDP也有类似叫法，数据报套接字，Datagram Socket
  - 一般分别以SOCK_STRAM与SOCK_DGRAM分别表示TCP和UDP套接字

  ### 03 套接字和地址

  - 网络编程中客户端和服务端的核心逻辑（重点掌握）

  ![Untitled.png](Untitled.png)

  - TCP三次握手，three-way handshake

  - 一旦连接建立，数据的传输就不再是单向的，而是双向的，这也是tcp的一个显著特性

  - socket是我们用来建立连接，传输数据的唯一途径

  - 套接字地址格式

    - 通用地址格式

    ```c
    1 /* POSIX.1g 规范规定了地址族为 2 字节的值.*/
    2 typedef unsigned short int sa_family_t; 
    3 /* 描述通用套接字地址 */ 
    4 struct sockaddr{ 
    5 sa_family_t sa_family; /* 地址族. 16-bit*/ 
    6 char sa_data[14]; /* 具体的地址值 112-bit */ 
    7 };
    ```

    - 地址族有以下几种：
      - AF_LOCAL，表示本地地址，一般用于本地socket通信，也可以写成，AFZ_UNIX,AF_FILE
      - AF_INET，因特网使用的ipv4地址
      - AF_INET6，因特网使用的ipv6地址
    - AF_表示，address family，地址族；PF_表示protocol family，协议族

### 04 怎么使用套接字格式建立连接？

- 创建套接字

```c
int socket(int domain, int type, int protocol)
```

- domain，指的是PF_INET,PF_INET6和PF_LOCAL等，表示什么样的套接字

- type可用值如下：

  - SOCK_STREAM，表示的是字节流，对应TCP
  - SOCKZ_DGRAM，表示的是数据报，对应UDP
  - SOCK_RAW，表示的是原始套接字

- protocol，基本废弃，一般写成0即可

- bind函数

  ```c
  bind(int fd, sockaddr * addr, socklen_t len)
  ```

  - 对于使用者，需要将IPv4，IPv6或本地套接字格式转化为通用套接字格式：

    ```c
    struct sockaddr_in name;
    bind (sock, (struct sockaddr *) &name, sizeof (name)
    ```

    - 对于实现者，可工具地址结构的前两个字节判断出是哪种地址
    - 通配地址的配置：

    ```c
    struct sockaddr_in name;
    name.sin_addr.s_addr = htonl (INADDR_ANY); /* IPV4 通配地址 */
    ```

- listen函数

  - 初始化创建的套接字，可以认为是一个”主动“套接字，其目的是之后主动发起请求（通过connect函数），通过listen函数可以转换为”被动“套接字

  ```c
  int listen (int socketfd, int backlog)
  ```

- accept函数

  - 作用是连接建立之后，操作系统内核和应用程序之间的桥梁：

  ```c
  int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen)
  ```

  - 完成了TCP三次握手之后，操作系统内核会为客户生成一个已连接套接字，应用程序使用这个已连接套接字和客户进行通信。而监听套接字一直都处于”监听“状态

- connect函数

  - 客户端与服务端的连接建立，是通过connect函数完成的：

  ```c
  int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)
  ```

  - 调用connect前不必非得调用bind函数，因为内核会确定源ip地址，并按照一定的算法选择一个临时端口作为源端口
  - 调用connect函数会激发TCP三次握手过程

  ![Untitled1.png](Untitled1.png)

### 05 使用套接字进行读写

- 发送数据
  - 常用的三个发送数据的函数

```c
ssize_t write (int socketfd, const void *buffer, size_t size)
ssize_t send (int socketfd, const void *buffer, size_t size, int flags)
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags)
```

- 发送缓存区

  - 当应用程序调用write函数时，实际做的事情是把数据从应用程序中拷贝到操作系统内核的发送缓冲区中，并不一定是把数据通过套接字写出去

- 读取数据

  ```c
  ssize_t read (int socketfd, void *buffer, size_t size)
  ```

  - read函数要求操作系统内核从套接字描述字socketfd读取最多多少个字节，并将结果存储到buffer中，返回值告诉我们实际读取的字节数目

- 在UNIX的世界里万物都是文件

- 阻塞式套接字最终发送返回的实际写入字节数和请求字节数是相等的

- 发送成功仅仅表示的是数据被拷贝到了发送缓冲区中，并不意味着连接对端已经收到所有的数据，至于什么时候发送到对端的接受缓冲区，或者更进一步说，什么时候被对方应用程序缓存所接收，对我们而言完全都是透明的

- 对于send来说，返回成功仅仅表示数据写到发送缓冲区成功，并不表示对端已经成功收到

- 对于read来说，需要循环读取数据，并且需要考虑EOF等异常条件

### 06 UDP编程

- recvfrom和sendto是UDP用来接收和发送报文的两个主要函数：

```c
#include <sys/socket.h>
 
ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags, 
　　　　　　　　　　struct sockaddr *from, socklen_t *addrlen); 
 
ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
                const struct sockaddr *to, socklen_t *addrlen);
```

- UDP是无连接的数据报程序，和TCP不同，不需要三次握手建立一条连接
- UDP程序通过recvfrom和sendto函数直接收发数据报报文

### 07 本地套接字

- 实现进程间通信的常用方法，如本地套接字，管道，共享消息队列等

### 08 工具

- ping，用来完成对网络连通性的探测

- ifconfig，用来显示当前系统中的所有网络设备，也就是网卡列表
- netstat，了解当前系统的网络连接状况
- lsof，找出在指定的IP地址hhuoz端口上打开套接字的进程
- tcpdump，抓包工具，用于查看报文，排查问题

### 10 TIME-WAIT

- TCP连接的四次挥手

![Untitled2.png](Untitled2.png)

- TIME_WAIT停留程序时间是固定的，是最长分节生命期MSL，maximum segment lifetime的两倍，一般称之为2MSL
- 只有发起连接终止的一方会进入TIME_WAIT状态（面试常问）
- TIME_WAIT的作用
  - 为了确保最后的ACK能让被动关闭方接收，从而帮助其正常关闭
  - 为了让旧连接的重复分节在网络中自然消失。经过2MSL时间，足以让两个方向上的分组都被丢弃，使得原来连接的分组在网络中都消失，再出现的分组一定都是新化身所产生的
- 2MSL的时间是从主机1接收到FIN后发送ACK开始计时的
- TIME_WAIT的危害
  - 内存资源占用
  - 对端口资源的占用，一个TCP连接至少消耗一个本地端口
- 优化TIME_WAIT
  - 调低TCP_TIMEWAIT_LEN，重新编译系统
  - SO_LINGER的设置，设置调用close或shutdown关闭连接时的行为
  - net.ipv4.tcp_tw_reuse，可以复用处于TIME_WAIT的套接字为新的连接所用，需要打开对TCP时间戳的支持，即net.ipv4.tcp_timestamps=1

### 11 关闭连接

- close函数

```c
int close(int sockfd)
```

- close函数会对套接字引用计数减一，引用计数到0，则会对套接字进行彻底释放，并且会关闭TCP两个方向的数据流
- shutdown函数

```c
int shutdown(int sockfd, int howto)
```

- shutdown函数可以关闭单方向的连接
- 期望关闭连接其中一个方向时，应该使用shutdown函数

### 12 使用Keep-Alive还是应用心跳来检测？

- TCP的保持活跃的机制，keep-alive
- 机制的原理，定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP保活机制会开始作用，每隔一个时间间隔，发送一个探测报文，报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的TCP连接已经死亡，系统内核将错误信息通知给上层应用程序
- 可以通过在应用程序设计一个ping-pong机制，来模拟TCP keep-alive机制，完成在应用层的连接探活

### 13 TCP协议中的动态数据传输

- 发送窗口用来控制发送和接收端的流量；阻塞窗口用来控制多条连接公平使用的有限带宽
- 小数据包加剧了网络带宽的浪费，为了解决这个问题，引入了如Nagle算法，延时ACK等机制
- 在程序设计层面，不要多次频繁地发送小报文，可以考虑使用writev批量发送

### 14 UDP也可以是“已连接”

- UDP套接字的connect操作和TCP的connect操作作用不同
- 如果UDP套接字不调用connect函数，则发送端不会报错，只会阻塞在recvfrom上，等待返回或超时
- UDP套接字调用connect方法，可以绑定本地地址和端口，为了让程序可以快速获取异步错误信息的通知，同时也可以获得一定性能上的提升

### 15 “地址已经被使用”？

- 出现`address already in use`一般是因为服务器端主动关闭连接，导致地址+端口的TCP连接处于TIME_WAIT的状态。服务器重启时继续绑定在原地址+端口上
- 最佳实践：在所有TCP服务器程序中，调用bind之前请设置SO_REUSEADDR套接字选项

### 16 如何理解TCP的“流”?

- readn函数

  - 读取报文预设大小的字节，readn调用会一直循环，尝试读取预设大小的字节，如果接收缓冲区数据为空，readn函数会阻塞在那里，直到有数据到达

- read_message函数

  - 调用多次readn函数，用于读取消息中每个固定长度的数据内容

- 两种报文格式

  - 发送端把要发送的报文长度预先通过报文告知给接收端
  - 通过一些特殊字符来进行边界的划分

- 显式编码报文长度

  ![Untitled3.png](Untitled3.png)

- 特殊字符作为边界

  ![Untitled4.png](Untitled4.png)

  - HTTP就是通过设置回车符，换行符作为HTTP报文协议的边界

### 17 TCP并不总是“可靠”的？

- 故障模式总结
  - 第一类，对端无FIN包发送出来的情况
    - 网络中断造成的对端无FIN包
    - 系统崩溃造成的对端无FIN包
  - 第二类，是对端有FIN包发送出来
    - read直接感知FIN包
    - 通过write产生RST，read调用感知RST
    - 向一个已关闭连接连续写，最终导致SUGPIPE

### 19 如何理解TCP的四次挥手？

- TCP协议栈会为FIN包插入一个文件结束符EOF到接收缓冲区中，EOF会被放在已排队等候的其他已接收的数据之后
- MSL是任何IP数据报能够在因特网存活的最长时间
- 每个数据报都包含一个被称为TTL，time to live的8位段，最大值255，每经过一跳就减一，减为0时所在路由器会将其丢弃

### 20 select

- I/O事件类型

  - 标准输入文件描述符准备好可以读
  - 监听套接字准备好，新的连接已经建立成功
  - 已连接套接字准备好可以写
  - I/O事件超时

- select函数

  ```c
  int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
   
  返回：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1
  ```

  - readset，读描述符集合，在指定描述符上检测数据可以读
  - writeset，写描述符集合，在指定描述符上检测数据可以写
  - exceptset，异常描述符集合，检测有异常发生
  - 宏
    - FD_ZERO，将描述符都设置为0
    - FD_SET，将a[fd]设置为1
    - FD_CLR，将a[fd]设置为0
    - FD_ISSET，判断a[fd]是0还是1
  - 0代表不需要处理，1代表需要处理
  - 内核通知套接字有数据可以读，使用read函数不会被阻塞
  - 内核通知套接字可以往里写，使用write函数不会阻塞
  - 套接字可写，完全是基于套接字本身的特性来说的
    - 套接字发送缓冲区足够大，进行write操作，不会阻塞，直接返回
    - 写的半边已关闭，继续写操作会产生SIGPIPE信号
    - 套接字上有错误待处理，进行write操作，不会阻塞，返回-1

### 21 poll

- select有一个缺点，就是所支持的文件描述符个数是有限的，在linux中select默认最大值为1024

- poll是除了select之外，另一种普遍使用的I/O多路复用技术，和select相比，它和内核交互的数据结构有所变化，另外也突破了文件描述符的个数限制

- poll函数

  ```c
  int poll(struct pollfd *fds, unsigned long nfds, int timeout); 
  　　　
  返回值：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1
  ```

- pollfd结构

  ```c
  struct pollfd {
      int    fd;       /* file descriptor */
      short  events;   /* events to look for */
      short  revents;  /* events returned */
   };
  ```

- events类型的事件可以分为两大类

  - events可以表示多个不同事件，可以通过使用二进制掩码位与操作来完成，如：

  ```c
  #define    POLLIN    0x0001    /* any readable data available */
  #define    POLLPRI   0x0002    /* OOB/Urgent readable data */
  #define    POLLOUT   0x0004    /* file descriptor is writeable */
  ```

  - 可读事件

    - 可读事件和select的readset基本一致，是系统内核通知应用程序有数据可以读，用read函数执行操作不会被阻塞，有以下几种：

      ```c
      #define POLLIN     0x0001    /* any readable data available */
      #define POLLPRI    0x0002    /* OOB/Urgent readable data */
      #define POLLRDNORM 0x0040    /* non-OOB/URG data available */
      #define POLLRDBAND 0x0080    /* OOB/Urgent readable data */
      ```

  - 可写事件

    - 可写事件和select的writeset基本一致，是系统内核通知套接字缓冲区已准备好，用write函数执行写操作不会被阻塞，以下几种：

      ```c
      #define POLLOUT    0x0004    /* file descriptor is writeable */
      #define POLLWRNORM POLLOUT   /* no write type differentiation */
      #define POLLWRBAND 0x0100    /* OOB/Urgent data can be written */
      ```

  - 错误事件

    - 只能通过revents加以检测，如下几种：

      ```c
      #define POLLERR    0x0008    /* 一些错误发送 */
      #define POLLHUP    0x0010    /* 描述符挂起 */
      #define POLLNVAL   0x0020    /* 请求的事件无效 */
      ```

  - 在poll函数中，如果不想对每个pollfd结构进行事件检测，就把对应pollfd结构的fd成员设置为一个负值

### 22 非阻塞I/O

- 非阻塞I/O

  - 读操作。非阻塞模式下read函数调用会立即返回，一般是EWOULDBLOCK或EAGAIN出错信息
  - 写操作。write函数返回当前函数调用有多少数据被拷贝到了发送缓冲区

  ![Untitled5.png](Untitled5.png)

- 一般非阻塞I/O下，使用轮询的方式引起CPU占用率高，所以一般将非阻塞I/O和I/O多路复用技术select，poll等倒赔使用，是linux下高性能网络编程的首选

### 23 epoll

- epoll是一种I/O多路复用技术，通过监控注册的多个描述字，来进行I/O事件的分发处理

- epoll_create函数

  ```c
  int epoll_create(int size);
  int epoll_create1(int flags);
          返回值: 若成功返回一个大于 0 的值，表示 epoll 实例；若返回 -1 表示出错
  ```

  - size参数已不再被需要，因为内核可以动态分配需要的内核数据结构，只需要设置为大于0的整数即可

- epoll_ctl函数

  ```c
  int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
          返回值: 若成功返回 0；若返回 -1 表示出错
  ```

  - 用于往这个epoll实例增加或删除监控的事件

- epoll_wait函数

  ```c
  int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
    返回值: 成功返回的是一个大于 0 的数，表示事件的个数；返回 0 表示的是超时时间到；若出错返回 -1.
  ```

- 对比

  - edge-triggered，边缘触发，只有第一次满足条件才触发，之后不再传递同样的事件
  - level-triggered，条件触发，只要满足事件的条件，就会一直不断把这个事件传递给用户
  - 一般认为，边缘触发的效率比条件触发的效率要高，是epoll的杀手锏之一

### 24 C10K问题

- C10K：如何在一台物理机上同时服务10000个用户，C表示并发，10K=10000
- 如何解决C10K问题
  - 应用程序如何和操作系统配合，调度处理上万个套接字上的处理I/O事件
  - 应用程序如何分配进程，线程资源来服务上万个连接
- 解决方案
  - 阻塞I/O+进程
  - 阻塞I/O+线程
  - 非阻塞I/O+readines notification + 单线程
  - 非阻塞I/O+readiness notification + 多线程
  - 异步I/O+多线程
- 在linux下，解决高性能问题的利器是非阻塞I/O加上epoll机制，再利用多线程能力

### 25 阻塞I/O和进程模型

- 进程是程序执行的最小单位

- 创建新的进程使用函数fork

  ```c
  pid_t fork(void)
  返回：在子进程中为 0，在父进程中为子进程 ID，若出错则为 -1
  ```

- 为每一个连接创建一个独立的子进程来进行服务，实现简单，但无法满足高性能程序的需求

### 26 阻塞I/O和线程模型

- 线程有自己的上下文context，有一个唯一标识线程的ID，thread ID，还有栈，程序计数器，寄存器等，同个进程内的线程共享改进程的整个虚拟地址空间，包括代码，数据，堆，共享库等
- 线程的上下文：程序计数器告诉CPU代码执行到哪，寄存器存了当前计算的一些中间值，内存内有一些当前用到的变量等
- 线程的上下文切换：从一个计算场景，切换到另外一个计算场景，程序计数器，寄存器等这些值重新载入新场景的值
- POSIX线程是现代UNIX系统提供的处理线程的标准接口，有60多个定义函数

### 27 I/O多路复用和线程模型

- 事件驱动模型，也被叫做反应堆模型reactor，或者是Event loop模型，核心有两点：
  - 存在一个无限循环的事件分发线程，或者叫做reactor线程，Event loop线程
  - 所有的I/O操作都可以抽象成事件，每个事件必须有回调函数来处理
- 任何的网络程序所作的事情都可以总结成以下几种：
  - read，从套接字收取数据
  - decode，解析，解码数据
  - compute，堆数据进行计算和处理
  - encode，将结果按照约定的格式进行编码
  - send，通过套接字把结果发送出去
- 支持多并发的网络编程技术
  - fork
  - pthread
  - single reactor thread
  - single reactor thread+worker threads

### 28 I/O多路复用进阶

- 将连接建立事件和已建立连接上的I/O事件分离，形成主-从reactor模式
  - 主reactor只负责分发Acceptor连接建立，已连接套接字上的I/O事件交给sub-reactor负责分发，而sub-reactor的数量可以根据CPU的核数来灵活配置
- 主-从reactor+worker threads模式

### 29 epoll和多线程模型

- 与poll相比，linux提供的epoll是一种更为高效的事件分发机制
- epoll的性能分析
  - 事件集合，epoll维护了一个全局的事件集合，通过epoll句柄操作整个事件集合
  - 就绪列表，epoll返回的是活的的事件列表，减少了大量的扫描时间

### 30 异步I/O

- 阻塞I/O

![Untitled6.png](Untitled6.png)

- 非阻塞I/O

![Untitled7.png](Untitled7.png)

- 多路复用

![Untitled8.png](Untitled8.png)

- 异步I/O

![Untitled9.png](Untitled9.png)

- 在read调用时，内核将数据从内核空间拷贝到应用程序空间，这个过程是在read函数中同步进行的，如果内核实现的拷贝效率很差，read调用就会在这个同步过程中消耗比较长的时间
- 在linux下的aio异步操作支持很少，故一般都是使用epoll这样的I/O分发技术

### 31 epoll源码深度剖析

- epoll中使用的数据结构
  - eventpoll
    - 调用epoll_create之后内核侧创建的一个句柄，表示一个epoll实例
    - epoll_ctl和epoll_wait都是对这个eventpoll实例数据进行操作
  - epitem
    - 每次调用epoll_ctl增加一个fd时，会创建一个新的epitem实例，加入到eventpoll结构体中的红黑树中，红黑树对应字段是rbr
    - 查找fd是否有就绪事件都是通过红黑树上的epitem来操作的
  - eppoll_entry
    - 每当一个fd关联到epoll实例，就会有一个eppoll_entry产生
- 基于epoll的事件分发能力，是linux下高性能网络编程的不二之选
- epoll维护了一棵红黑树来跟踪所有待检测的文件描述符，红黑树的使用减少了内核和用户空间之间大量的数据拷贝和内存分配，大大提高了性能