#### socket套接字

**创建socket**

socket创建在内核中，创建成功返回核文件描述表中的socket描述符

实际上就是结构体，封装有很多成员

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol)
// 创建一个套接字
// 返回：成功 返回描述符可以操作内核的某一块内存，网络通信基于这个文件描述符来完成。出错 返回-1
    
// 将文件描述符和本地的IP与端口进行绑定   
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// 给监听的套接字设置监听
int listen(int sockfd, int backlog);

// 等待并接受客户端的连接请求, 建立新的连接, 会得到一个新的文件描述符(通信的)		
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// 接收数据
ssize_t read(int sockfd, void *buf, size_t size);
ssize_t recv(int sockfd, void *buf, size_t size, int flags);

// 发送数据的函数
ssize_t write(int fd, const void *buf, size_t len);
ssize_t send(int fd, const void *buf, size_t len, int flags);

// 成功连接服务器之后, 客户端会自动随机绑定一个端口
// 服务器端调用accept()的函数, 第二个参数存储的就是客户端的IP和端口信息
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**int socket参数**

* domain
  - AF_INET     IPv4
  - AF_INET6   IPv6
  - AF_UNIX     unix域
  - AF_UNSPEC 未指定

* protocol
  * 一般为 0，表示按给定的域和套接字类型选择默认协议

* type
  * SOCK_STREAM 	 TCP
  * SOCK_DGRAM  	 UDP
  * SOCK_RAW                IP\ICMP
  * SOCK_SEQPACKET   长度固定、有序、可靠的面向链接报文传递

**文件描述符对应的内存结构：**

一个文件文件描述符对应两块内存, 一块内存是读缓冲区, 一块内存是写缓冲区
读数据: 通过文件描述符将内存中的数据读出, 这块内存称之为读缓冲区
写数据: 通过文件描述符将数据写入到某块内存中, 这块内存称之为写缓冲区

**函数**

```c
// 这套api主要用于 网络通信过程中 IP 和 端口 的 转换
// 将一个短整形从主机字节序 -> 网络字节序
uint16_t htons(uint16_t hostshort);	
// 将一个整形从主机字节序 -> 网络字节序
uint32_t htonl(uint32_t hostlong);	
// 将一个短整形从网络字节序 -> 主机字节序
uint16_t ntohs(uint16_t netshort)
// 将一个整形从网络字节序 -> 主机字节序
uint32_t ntohl(uint32_t netlong);
```

**sockaddr数据结构**

```c
// 在写数据的时候不好用
struct sockaddr {
	sa_family_t sa_family;       // 地址族协议, ipv4
	char        sa_data[14];     // 端口(2字节) + IP地址(4字节) + 填充(8字节)
}

typedef unsigned short  uint16_t;
typedef unsigned int    uint32_t;
typedef uint16_t in_port_t;
typedef uint32_t in_addr_t;
typedef unsigned short int sa_family_t;
#define __SOCKADDR_COMMON_SIZE (sizeof (unsigned short int))

struct in_addr
{
    in_addr_t s_addr;
};  

// sizeof(struct sockaddr) == sizeof(struct sockaddr_in)
struct sockaddr_in
{
    sa_family_t sin_family;		/* 地址族协议: AF_INET */
    in_port_t sin_port;         /* 端口, 2字节-> 大端  */
    struct in_addr sin_addr;    /* IP地址, 4字节 -> 大端  */
    /* 填充 8字节 */
    unsigned char sin_zero[sizeof (struct sockaddr) - sizeof(sin_family) -
               sizeof (in_port_t) - sizeof (struct in_addr)];
}; 
```

#### TCP通信流程

面向连接、安全的流式传输协议

面向连接：是一个双向连接，通过三次握手完成，断开连接需要通过四次挥手完成。
安全：tcp通信过程中，会对发送的每一数据包都会进行校验, 如果发现数据丢失, 会自动重传
流式传输：发送端和接收端处理数据的速度，数据的量都可以不一致

**服务器端通信流程**

socket() -> bind() -> listen() -> accept() -> read()/write() -> close()

**在tcp的服务器端, 有两类文件描述符**

* 监听的文件描述符
  只需要有一个
  不负责和客户端通信, 负责检测客户端的连接请求, 检测到之后调用accept就可以建立新的连接

* 通信的文件描述符
  负责和建立连接的客户端通信
  如果有N个客户端和服务器建立了新的连接, 通信的文件描述符就有N个，每个客户端和服务器都对应一个通信的文件描述符

**客户端通信流程**

socket() -> connect() -> read()/recv() -> close()

#### epoll网络并发通信

epoll操作函数如下

```c
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

multiplexing 

```c
// 设置端口复用 SO_REUSEADDR 选项 允许多个进程或线程绑定到同一端口
	int opt = 1;
	setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```



文件描述符（File Descriptor, FD）是一个用于标识打开文件或网络连接的整数。监听的文件描述符和通信的文件描述符是两种不同的文件描述符，通常用于服务器端网络编程中。

#### 监听的文件描述符

监听的文件描述符是服务器程序用于接收新连接请求的文件描述符。它通常通过调用 `socket()` 函数创建，然后通过 `bind()` 函数绑定到一个特定的IP地址和端口，接着调用 `listen()` 函数开始监听连接请求。

- **作用**：监听文件描述符的作用是等待来自客户端的连接请求。
- **使用**：当有新的连接请求到达时，可以使用 `accept()` 函数接收连接请求并返回一个新的文件描述符，这个新的文件描述符就是通信的文件描述符。

#### 通信的文件描述符

通信的文件描述符是用于实际数据传输的文件描述符。每当 `accept()` 成功接受一个连接请求时，服务器就会生成一个新的通信文件描述符，这个描述符专门用于与特定客户端之间的数据交换。

- **作用**：通信文件描述符用于与客户端进行数据的收发操作。每个客户端与服务器的连接都会有一个独立的通信文件描述符。
- **使用**：通过这个文件描述符，可以使用 `send()`、`recv()` 等函数与客户端进行通信。

#### 示例：

```c
int listen_fd, conn_fd;

// 创建监听文件描述符
listen_fd = socket(AF_INET, SOCK_STREAM, 0);

// 绑定地址和端口
bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));

// 开始监听
listen(listen_fd, SOMAXCONN);

// 等待客户端连接
conn_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);

// 通过 conn_fd 与客户端通信
recv(conn_fd, buffer, sizeof(buffer), 0);
```

在这个示例中，`listen_fd` 是监听文件描述符，它用于监听新的连接请求。`conn_fd` 是通信文件描述符，用于与某个客户端进行实际的数据传输。
