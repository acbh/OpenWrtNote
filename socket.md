#### socket套接字

**创建socket**

socket创建在内核中，创建成功返回核文件描述表中的socket描述符

实际上就是结构体，封装有很多成员

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol)

// 返回：成功返回描述符，出错返回-1
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