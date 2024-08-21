#### socket套接字

**创建socket**

socket创建在内核中，创建成功返回核文件描述表中的socket描述符

实际上就是结构体，封装有很多成员

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol)

// 返回：成功 返回描述符，出错 返回-1
```

**参数**

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
