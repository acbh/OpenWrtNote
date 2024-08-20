### 网络测速

`iperf` 默认的数据包大小取决于使用的协议类型：

1. **TCP**：
   - 在 TCP 模式下，`iperf` 没有严格定义“数据包大小”的概念，而是依赖于 TCP 协议本身的流量控制机制。TCP 通过设置缓冲区大小来控制数据传输。默认的 TCP 缓冲区大小通常是 8 KB（8192 字节），但可以通过 `-w` 参数来调整这个值。

2. **UDP**：
   - 在 UDP 模式下，`iperf` 默认使用 1470 字节的 UDP 数据包大小。你可以使用 `-l` 参数（如 `-l 1200`）来自定义数据包的大小。

这些默认值可以影响测试结果，尤其是在不同网络条件下的表现。如果手动修改了这些参数，可能会导致测得的带宽与默认配置下的测试结果有所不同。

### UDP实现通信demo

#### 服务器端

```c
// socket udp server.c
#include <stdio.h>
#include <unistd.h>     // posix api close()
#include <sys/types.h>  // socket() bind() recvfrom()
#include <sys/socket.h> // socket() bind() recvfrom()
#include <arpa/inet.h>  // inet_addr() htons()

int main()
{
    // 创建socket套接字 AF_INET 表示使用IPV4地址 SOCK_DGRAM使用UDP协议
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    // 创建网络通信对象 设置服务器地址和端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(1324);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    // 绑定socket对象与通信链接
    int ret = bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
    if (ret < 0)
    {
        printf("bind\n");
        return -1;
    }
    struct sockaddr_in cli; // 存储客户端地址信息
    socklen_t len = sizeof(cli);

    while (1)
    {
        char buf = 0;
        recvfrom(sockfd, &buf, sizeof(buf), 0, (struct sockaddr *)&cli, &len); // 接收来自客户端的数据包 存储在cli中
        printf("recv num =%hhd\n", buf);

        buf = 66;
        sendto(sockfd, &buf, sizeof(buf), 0, (struct sockaddr *)&cli, len); // 向刚才通信的客户端发送数据包
    }
    close(sockfd);
}
```

代码实现了一个简单的 UDP 服务器，监听本地 IP 地址 `127.0.0.1` 上的端口 `1324`。它接收来自客户端的一个字节的数据并将其打印，然后回应客户端一个值为 `66` 的字节。服务器在无限循环中持续运行，直到程序被手动终止。

#### 客户端

```c
// socket udp client.c
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    // 创建socket对象
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    // 创建网络通信对象
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(1324);
    addr.sin_addr.s_addr = inet_addr("192.168.0.143");

    while (1)
    {
        printf("请输入一个数字：");
        char buf = 0;
        scanf("%hhd", &buf);
        sendto(sockfd, &buf,
               sizeof(buf), 0, (struct sockaddr *)&addr, sizeof(addr));

        socklen_t len = sizeof(addr);
        recvfrom(sockfd, &buf, sizeof(buf), 0, (struct sockaddr *)&addr, &len);

        if (66 == buf)
        {
            printf(" server 成功接受\n");
        }
        else
        {
            printf("server 数据丢失\n");
        }
    }
    close(sockfd);
}
```

这个 UDP 客户端程序的工作流程如下：

1. 客户端运行后，进入一个无限循环。
2. 用户输入一个数字，客户端将该数字通过 UDP 协议发送给指定 IP 地址（服务器）。
3. 服务器接收到该数字后，返回一个响应（在之前的服务器代码中，返回值为 `66`）。
4. 客户端接收服务器的响应并检查响应内容是否为 `66`，然后根据情况打印相应的消息。
5. 客户端程序会一直运行，直到手动关闭。