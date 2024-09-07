### 0. **笔记**

#### 带宽

**网络带宽：**

​	网络带宽指网络连接的速率或数据传输速率，通常以比特每秒（bps，bits per second）为单位。常见的单位包括：

- Kbps（千比特每秒）
- Mbps（兆比特每秒）
- Gbps（吉比特每秒）

**带宽类型：**

* **对称带宽**：上行带宽和下行带宽相等。某些企业级网络连接会提供对称的100 Mbps上传和下载带宽。

* **非对称带宽**：上行带宽和下行带宽不相等。家用宽带通常具有较高的下载带宽，但上传带宽较低，例如100 Mbps下载，10 Mbps上传。

#### 计算

1 Mbps（兆比特每秒）表示数据传输速率，但要转换为MB（兆字节），需要理解比特（bit）与字节（byte）之间的关系。

**关系**：

- 1 Byte = 8 bits

因此，要将 Mbps 转换为 MBps（兆字节每秒），可以使用以下公式：
$$
\text{MBps} = \frac{\text{Mbps}}{8}
$$

**计算**：

$$
1 \text{Mbps} = \frac{1}{8} \text{MBps} = 0.125 \text{MBps}
$$

**结论**：
$$
1 \text{Mbps} = 0.125 \text{MBps}
$$
也就是说，1 Mbps 的数据传输速率相当于每秒传输 0.125 MB 的数据。

#### bmon工具

`bmon: Bandwidth Monitor`, 监控网络带宽和性能工具，方便查看查看网络接口的实时流量、带宽利用率、数据包统计等信息

安装：`sudo apt-get install bmon`, 使用：`bmon`

#### iperf3 测速

UDP

```shell
bhhh@bhcomputer:~$ iperf3 -u -c 127.0.0.1 -b 100G
Connecting to host 127.0.0.1, port 5201
[  5] local 127.0.0.1 port 51401 connected to 127.0.0.1 port 5201
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.00   sec  9.36 GBytes  80.4 Gbits/sec  306857
[  5]   1.00-2.00   sec  9.60 GBytes  82.5 Gbits/sec  314722
[  5]   2.00-3.00   sec  9.66 GBytes  82.9 Gbits/sec  316378
[  5]   3.00-4.00   sec  9.63 GBytes  82.8 Gbits/sec  315705
[  5]   4.00-5.00   sec  9.64 GBytes  82.8 Gbits/sec  315950
[  5]   5.00-6.00   sec  9.67 GBytes  83.0 Gbits/sec  316725
[  5]   6.00-7.00   sec  9.73 GBytes  83.6 Gbits/sec  318986
[  5]   7.00-8.00   sec  9.71 GBytes  83.4 Gbits/sec  318095
[  5]   8.00-9.00   sec  9.78 GBytes  84.0 Gbits/sec  320464
[  5]   9.00-10.00  sec  9.80 GBytes  84.2 Gbits/sec  321128
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  96.6 GBytes  83.0 Gbits/sec  0.000 ms  0/3165010 (0%)  sender
[  5]   0.00-10.04  sec  96.6 GBytes  82.6 Gbits/sec  0.001 ms  142/3165010 (0.0045%)  receiver

iperf Done.
```

TCP

```shell
bhhh@bhcomputer:~$ iperf3 -c 127.0.0.1
Connecting to host 127.0.0.1, port 5201
[  5] local 127.0.0.1 port 35176 connected to 127.0.0.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  11.2 GBytes  95.9 Gbits/sec    0   1.62 MBytes
[  5]   1.00-2.00   sec  11.4 GBytes  98.0 Gbits/sec    0   1.62 MBytes
[  5]   2.00-3.00   sec  11.5 GBytes  98.5 Gbits/sec    0   1.62 MBytes
[  5]   3.00-4.00   sec  11.4 GBytes  97.9 Gbits/sec    0   1.62 MBytes
[  5]   4.00-5.00   sec  11.3 GBytes  97.2 Gbits/sec    0   1.62 MBytes
[  5]   5.00-6.00   sec  10.5 GBytes  90.2 Gbits/sec    0   1.62 MBytes
[  5]   6.00-7.00   sec  10.4 GBytes  88.9 Gbits/sec    0   1.62 MBytes
[  5]   7.00-8.00   sec  10.8 GBytes  93.1 Gbits/sec    0   1.62 MBytes
[  5]   8.00-9.00   sec  10.3 GBytes  88.9 Gbits/sec    0   1.62 MBytes
[  5]   9.00-10.00  sec  10.3 GBytes  88.6 Gbits/sec    0   1.62 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   109 GBytes  93.7 Gbits/sec    0             sender
[  5]   0.00-10.04  sec   109 GBytes  93.3 Gbits/sec                  receiver

iperf Done.
```

### 1. **设计思路**

**了解多线程的方式，用`epoll`去写异步服务器端，客户端的话支持动态数据调整，要在服务器显示列表，发送指令进行上行下行，进行限速。扎实网络编程。传输的数据包大小最好是`MTU`的值，按字节统计**

- **服务器（Server）**：负责监听客户端的连接请求，并发送或接收数据以测试带宽。服务器会同时处理多个客户端的请求。
  - 初始化客户端信息 
  - 创建监听套接字
  - 绑定套接字 监听端口
  - 创建`epoll`实例
  - 添加监听套接字到`epoll`
  - 初始化`ncurses`界面
  - 等待事件，每秒更新显示
  - 创建线程处理客户端
  - 更新显示

- **客户端（Client）**：每个客户端连接到服务器并执行带宽测试，上传或下载数据。
  - 创建套接字
  - 设置`socket`为非阻塞模式
  - 配置服务器地址
  - 连续发送数据包

**通过命令行参数来选择协议和测试模式**

`./server -p tcp -m up`：选择使用TCP进行上行测试。

`./server -p udp -m down`：选择使用UDP进行下行测试。

`./server -p tcp -m double`：选择使用TCP进行双向测试。

如果没有指定协议参数，则默认使用TCP协议。

**总结需求**
服务端显示服务器地址和端口 广播地址以及端口 显示测速模式（上行、下行、上下行）支持实时设置限速 实时显示Max bandwidth\ Min bandwidth\Average bandwidth 以及各个客户端的信息 服务端支持接收命令行参数来选择测速协议和测速模式
客户端界面显示对应的服务器地址端口 测试时间 测速模式 是否限速 显示up和down的速率


### 2. **使用 `ncurses `实现实时显示**
   - **界面布局**：
     
     - 顶部显示服务器状态（如连接的客户端数量、当前测试类型等）。
     - 中间区域显示每个客户端的实时带宽信息，包括上传和下载速度。
     - 底部显示操作提示或日志信息。
     
- **窗口刷新**：
     - 使用 `ncurses` 的 `refresh()` 函数刷新整个窗口，确保显示内容实时更新。
     - 为每个客户端创建一个独立的窗口或区域（如 `WINDOW` 对象），用来显示各自的带宽信息。可以通过 `wrefresh()` 函数刷新这些窗口。

- **多线程处理**：
     - 由于服务器需要同时处理多个客户端的连接，可以使用多线程或多进程来管理每个客户端的带宽测试。
     - 主线程负责处理 `ncurses` 界面的更新和整体逻辑，而每个客户端的连接可以由独立的线程处理带宽测试。
     - 使用线程间通信机制（如共享变量、消息队列）将带宽测试的实时数据传递给主线程，以便在 `ncurses` 界面上显示。

- **界面概要设计**：

  ![image-20240829165441321](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240829165441321.png)
  
  ```
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │Server    addr:        192.168.18.125          port: 8888                     │
  │Broadcast addr:        192.168.18.255          port: 5005                     │
  │Test      time:        500 sec                 Pakg size: 1470 bit            │
  │Mode:    double                                Limit: no limit                │
  │                                                                              │
  │pre        next        set-limit               exit(press 'q')                │
  │                                                                              │
  │| RANK | IP            |  PORT  | UP            | DOWN          |             │
  │-----------------------------------------------------------------             │
  │|  [1] | 127.0.0.1     |  38482 | 10534.19 Mbps | 66666.66 Mbps |             │
  │|  [2] | 127.0.0.1     |  38490 | 12870.97 Mbps | 66666.66 Mbps |             │
  │|  [3] | 127.0.0.1     |  38494 | 10309.08 Mbps | 66666.66 Mbps |             │
  │|  [4] | 127.0.0.1     |  38502 | 11805.87 Mbps | 66666.66 Mbps |             │
  │|  [5] | 127.0.0.1     |  38504 | 15201.28 Mbps | 66666.66 Mbps |             │
  │|  [6] | 127.0.0.1     |  38510 | 13239.42 Mbps | 66666.66 Mbps |             │
  │|  [7] | 127.0.0.1     |  38526 | 13831.60 Mbps | 66666.66 Mbps |             │
  │|  [8] | 127.0.0.1     |  38534 | 15832.47 Mbps | 66666.66 Mbps |             │
  │|  [9] | 127.0.0.1     |  38536 | 10046.14 Mbps | 66666.66 Mbps |             │
  │                                                                              │
  │                                                                              │
  │                                                                              │
  │                                                                              │
  └──────────────────────────────────────────────────────────────────────────────┘
  ```
  
  连接列表 支持翻页
  
  共12个客户端，限速后实际速度与bmon监控的速度一致


### 3. **带宽测试逻辑**
   - **服务器**：
     - 监听特定端口，接受客户端连接。
     - 对每个连接，启动一个线程来处理数据传输并记录带宽信息。
     - 支持多种测试模式（如上传、下载、双向），通过命令或配置进行选择。
   - **客户端**：
     - 连接到服务器后，根据测试模式发送或接收数据包。
     - 实时计算传输速度，并将结果返回给服务器。

### 4. **实时更新与统计**
   - **实时带宽计算**：
     - 在每次数据传输时，计算传输的数据量并除以时间间隔，得出实时的带宽值。
     - 这些值会被频繁刷新，显示最新的网络状态。
   - **累积统计**：
     - 在每个连接结束后计算累计带宽、平均带宽等指标。
     - 在测试结束时，汇总所有客户端的测试结果并在 `ncurses` 界面上显示。

### 5. 代码

`server.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <errno.h>
#include <time.h>
#include <ncurses.h> // 引入 ncurses 库

#define PORT 8888		 // 服务器端口
#define MAX_EVENTS 10	 // 最大事件数
#define BUFFER_SIZE 1400 // 缓冲区大小
#define MAX_CLIENTS 1024 // 最大客户端数

typedef struct // 客户端信息
{
	int socket;
	double up_bw;
	double down_bw;
	time_t start_time;
	int num_packets;
} client_info;

client_info clients[MAX_CLIENTS];

void process_client_data(int client_fd) // 处理来自客户端的数据
{
	char buffer[BUFFER_SIZE];
	int bytes_read;

	while ((bytes_read = recv(client_fd, buffer, sizeof(buffer), 0)) > 0)
	{
		// 计算带宽
		for (int i = 0; i < MAX_CLIENTS; i++)
		{
			mvprintw(0,0,"test test test test \n");
			if (clients[i].socket == client_fd)
			{
				clients[i].num_packets++;
				time_t elapsed_time = time(NULL) - clients[i].start_time;

				if (elapsed_time > 0)
				{
					clients[i].up_bw = (clients[i].num_packets * sizeof(buffer) * 8.0) / (1024 * 1024 * elapsed_time); // 计算上行带宽，单位Mbps
					snprintf(buffer, sizeof(buffer), "Client %d: UP: %.2f Mbps", client_fd, clients[i].up_bw);
					send(client_fd, buffer, strlen(buffer), 0);
				}
				break;
			}
		}
	}

	if (bytes_read == 0)
	{
		// 客户端断开连接
		printf("Client %d disconnected\n", client_fd);
	}
	else if (bytes_read < 0)
	{
		perror("recv");
	}
}

void update_display()
{
	clear(); // 清屏

	mvprintw(0, 0, "Server listening on port %d", PORT);
	mvprintw(1, 0, "Connected clients: ");

	int row = 2; // 从第三行开始打印客户端信息

	for (int i = 0; i < MAX_CLIENTS; i++)
	{
		if (clients[i].socket != 0)
		{
			mvprintw(row++, 0, "Client %d: UP: %.2f Mbps", clients[i].socket, clients[i].up_bw);
		}
	}

	refresh(); // 刷新屏幕
}

int main()
{
	int server_fd, epoll_fd;
	struct sockaddr_in address;
	struct epoll_event ev, events[MAX_EVENTS];
	int addrlen = sizeof(address);

	// 初始化客户端信息
	memset(clients, 0, sizeof(clients));

	// 创建监听套接字
	if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0)
	{
		perror("socket failed");
		exit(EXIT_FAILURE);
	}

	int opt = 1;
	setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)); // 重用地址
	// setsockopt(server_fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt)); // 重用端口

	address.sin_family = AF_INET;
	address.sin_addr.s_addr = INADDR_ANY;
	address.sin_port = htons(PORT);

	// 绑定套接字
	if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0)
	{
		perror("bind failed");
		exit(EXIT_FAILURE);
	}

	// 监听端口
	if (listen(server_fd, MAX_CLIENTS) < 0)
	{
		perror("listen");
		exit(EXIT_FAILURE);
	}

	// 创建 epoll 实例
	epoll_fd = epoll_create1(0);
	if (epoll_fd == -1)
	{
		perror("epoll_create1");
		exit(EXIT_FAILURE);
	}

	// 添加监听套接字到 epoll
	ev.events = EPOLLIN;
	ev.data.fd = server_fd;
	if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1)
	{
		perror("epoll_ctl: server_fd");
		exit(EXIT_FAILURE);
	}

	// 初始化 ncurses
	initscr();
	cbreak();
	noecho();
	curs_set(FALSE);

	printf("Server is listening on port %d\n", PORT);

	while (1)
	{
		int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, 1000); // 等待事件，并每秒更新显示
		if (nfds == -1)
		{
			perror("epoll_wait");
			break;
		}

		for (int n = 0; n < nfds; n++)
		{
			if (events[n].data.fd == server_fd)
			{
				// 接受新连接
				int new_socket;
				if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen)) < 0)
				{
					perror("accept");
					continue;
				}

				// 添加新连接到 epoll 实例
				ev.events = EPOLLIN | EPOLLET; // 使用边缘触发（ET）模式
				ev.data.fd = new_socket;
				if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, new_socket, &ev) == -1)
				{
					perror("epoll_ctl: new_socket");
					close(new_socket);
					continue;
				}

				// 初始化新客户端的信息
				for (int i = 0; i < MAX_CLIENTS; i++)
				{
					if (clients[i].socket == 0)
					{
						clients[i].socket = new_socket;
						clients[i].start_time = time(NULL);
						clients[i].num_packets = 0;
						break;
					}
				}
			}
			else
			{
				// 处理来自客户端的数据
				int client_fd = events[n].data.fd;
				process_client_data(client_fd);

				// 从 epoll 实例中移除并关闭连接
				epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
				close(client_fd);

				// 清除客户端信息
				for (int i = 0; i < MAX_CLIENTS; i++)
				{
					if (clients[i].socket == client_fd)
					{
						clients[i].socket = 0;
						break;
					}
				}
			}
		}

		update_display(); // 更新显示
	}

	// 关闭 ncurses
	endwin();

	close(server_fd);
	close(epoll_fd);
	return 0;
}
```

`client.c`

```c
#include <ncurses.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <arpa/inet.h>
#include <time.h>
#include <fcntl.h>

void display_bandwidth_stats(double up, double down)
{
    mvprintw(10, 0, "UP: %.2f Mbps \t\tDOWN: %.2f Mbps", up, down);
    refresh();
}

int main()
{
    int sock = 0;
    struct sockaddr_in serv_addr;
    char buffer[1400];                   // 假设数据包大小为1400字节
    memset(buffer, 'A', sizeof(buffer)); // 用数据填充包

    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        printf("\n Socket creation error \n");
        return -1;
    }

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(8888);

    if (inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr) <= 0)
    {
        printf("\nInvalid address/ Address not supported \n");
        return -1;
    }

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0)
    {
        printf("\nConnection Failed \n");
        return -1;
    }

    // 设置 socket 为非阻塞模式
    fcntl(sock, F_SETFL, O_NONBLOCK);

    // 初始化 ncurses
    initscr();
    cbreak();
    noecho();
    curs_set(FALSE);

    double up_bw = 0.0, down_bw = 0.0;
    int num_packets = 0;
    time_t start_time = time(NULL);

    while (1)
    {
        int ch = getch();
        if (ch == 'q')
        {
            break;
        }

        send(sock, buffer, sizeof(buffer), 0);
        num_packets++;
        time_t elapsed_time = time(NULL) - start_time;

        if (elapsed_time > 0)
        {
            up_bw = (num_packets * sizeof(buffer) * 8.0) / (1024 * 1024 * elapsed_time); // 计算上行带宽，单位Mbps
            display_bandwidth_stats(up_bw, down_bw);                                     // 显示带宽统计信息
        }

        // 尝试接收服务器反馈数据（非阻塞）
        int n = recv(sock, buffer, sizeof(buffer) - 1, 0);
        if (n > 0)
        {
            buffer[n] = '\0'; // 确保字符串以空字符结束
            mvprintw(15, 0, "%s", buffer);
            refresh();
        }

        usleep(100000); // 100ms 延迟
    }

    endwin();
    close(sock);

    return 0;
}
```

编译 运行

```bash
gcc -o server server.c -lpthread -lncurses
gcc -o client client.c

./server
./client (打开多个)
```



