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

#### epoll多线程网络并发通信

epoll操作函数

```c
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

