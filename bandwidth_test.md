## 一对多带宽测速程序

### 实现思路概要

#### 运行时切换模式UP/DOWN/double的方法：

#### 通过命令行信号触发模式

定义一个全局变量存储模式

```
volatile sig_atomic_t mode  = 0
```

编写信号处理函数

```c
void switch_mode(int sig) {
    if (sig == SIGUSR1) {
        mode = (mode + 1) % 3;
    }
}
```

在`main`函数添加信号捕捉

```
signal(SIGUSR1, switch_mode);
```

通过发送`SIGUSR1`信号，在运行时切换模式

```
kill -USR1 <进程ID>
```

#### 通过键盘输入实现模式切换

设置键盘输入为非阻塞

```
nodelay(stdscr, TRUE);
```

在`main`循环中添加键盘检测逻辑

```c
int ch;
while (1) {
    ch = getch();
    if (ch == 'm' || ch == 'M') {
        mode = (mode + 1) % 3;
        switch (mode) {
            case 0:
                mvwprintw(main_win, 2, 1, "Mode: UP			");
                break;
            case 1:
                mvwprintw(main_win, 2, 1, "Mode: DOWN		");
                break;
            case 2:
                mvwprintw(main_win, 2, 1, "Mode: DOUBLE		");
                break;
        }
        wrefresh(main_win);
    }

    // handle client ... 
}
```

#### 考虑在`main`函数中添加按键输入逻辑替代命令行参数的模式选择

移除命令行参数获取模式的逻辑

使用`ncurses`获取按键输入 动态更改模式

根据按键选择不同模式 启动相应的线程

```c
int main(int argc, char* argv[]) {
    int mode = 0; // 0: UP, 1: DOWN, 2: DOUBLE

    // 初始化ncurses
    initscr();
    cbreak();
    noecho();
    curs_set(0);
    keypad(stdscr, TRUE);  // 使得我们能够捕获功能键
    main_win = newwin(MAX_CLIENTS * 2 + 4, 80, 0, 0);
    box(main_win, 0, 0);
    mvwprintw(main_win,  1, 1, "Server listening on port %d...", SERVER_PORT);
    mvwprintw(main_win,  9, 1, "| RANK | IP\t\t|  PORT  | UP\t\t | DOWN\t\t |");
    mvwprintw(main_win, 10, 1, "----------------------------------------------------------------");
    wrefresh(main_win);

    // 显示初始模式
    mvwprintw(main_win, 2, 1, "Current mode: UP");
    wrefresh(main_win);

    // 设置定时器，每秒触发一次
    struct itimerval timer;
    signal(SIGALRM, handle_alarm);
    timer.it_value.tv_sec = 1;
    timer.it_value.tv_usec = 0;
    timer.it_interval.tv_sec = 1;
    timer.it_interval.tv_usec = 0;
    setitimer(ITIMER_REAL, &timer, NULL);

    int server_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    pthread_t threads[MAX_CLIENTS * 2];
    int thread_count = 0;

    memset(clients, 0, sizeof(clients)); // 初始化客户端信息数组

    // 创建 TCP 套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("socket creation failed");
        exit(EXIT_FAILURE);
    }

    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定套接字到地址
    if (bind(server_fd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 监听端口
    if (listen(server_fd, MAX_CLIENTS) < 0) {
        perror("listen failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 主循环等待客户端连接并处理按键输入
    while (1) {
        int ch = getch();  // 捕获按键输入
        if (ch == 'u') {
            mode = 0; // 切换到 UP 模式
            mvwprintw(main_win, 2, 1, "Current mode: UP   ");
            wrefresh(main_win);
        } else if (ch == 'd') {
            mode = 1; // 切换到 DOWN 模式
            mvwprintw(main_win, 2, 1, "Current mode: DOWN ");
            wrefresh(main_win);
        } else if (ch == 'b') {
            mode = 2; // 切换到 DOUBLE 模式
            mvwprintw(main_win, 2, 1, "Current mode: DOUBLE");
            wrefresh(main_win);
        }

        int* client_fd = malloc(sizeof(int));
        if (client_fd == NULL) {
            perror("malloc failed");
            close(server_fd);
            endwin();
            exit(EXIT_FAILURE);
        }

        *client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_addr_len); // 接收客户端请求
        if (*client_fd < 0) {
            perror("accept failed");
            free(client_fd);
            continue;
        }

        // 寻找空闲的客户端槽
        client_info_t* client = NULL;
        for (size_t i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].fd == 0) {
                client = &clients[i];
                client->fd = *client_fd;
                client->total_bytes_up = 0;
                client->total_bytes_down = 0;
                gettimeofday(&client->start, NULL); // 获取当前时间
                pthread_mutex_init(&client->lock, NULL);

                // 记录 IP 和端口
                inet_ntop(AF_INET, &(client_addr.sin_addr), client->ip, INET_ADDRSTRLEN);
                client->port = ntohs(client_addr.sin_port);
                break;
            }
        }

        free(client_fd); 

        if (client != NULL) {
            // 根据选择的模式，创建相应的线程
            if (mode == 0 || mode == 2) {  // UP 模式或 DOUBLE 模式都启动上传线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_upload, client) != 0) {
                    perror("pthread_create for upload failed");
                }
            }

            if (mode == 1 || mode == 2) {  // DOWN 模式或 DOUBLE 模式都启动下载线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_download, client) != 0) {
                    perror("pthread_create for download failed");
                }
            }

            // 防止线程数量超过上限
            if (thread_count >= MAX_CLIENTS * 2) {
                thread_count = 0;
                for (int i = 0; i < MAX_CLIENTS * 2; i++) {
                    pthread_join(threads[i], NULL);
                }
            }
        } else {
            printf("Max clients reached. Connection refused.\n");
        }
    }

    close(server_fd);
    endwin();
    return 0;
}
```

#### 服务端主动通知客户端测速模式变化思路

1、服务端模式改变时通知所有连接的客户端，通过一个专门的线程来监控模式变化，当模式变化时，该线程会通知所有客户端

2、客户端持续监听模式的变化，接收到新的模式时调整相应的收发行为

```c
// 通知所有客户端模式变化
void notify_clients_mode_change(int new_mode) {
    for (size_t i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].fd != 0) {  // 确保该客户端已连接
            if (send(clients[i].fd, &new_mode, sizeof(new_mode), 0) <= 0) {
                perror("send mode change failed");
                close(clients[i].fd);
                clients[i].fd = 0;  // 移除该客户端
            }
        }
    }
}
int main(int argc, char* argv[]) {
    int mode = 0; // 0: UP, 1: DOWN, 2: DOUBLE

    // 初始化ncurses
    initscr();
    cbreak();
    noecho();
    curs_set(0);
    keypad(stdscr, TRUE);  // 使得我们能够捕获功能键
    nodelay(stdscr, TRUE); // 设置非阻塞模式
    main_win = newwin(MAX_CLIENTS * 2 + 4, 80, 0, 0);
    box(main_win, 0, 0);
    mvwprintw(main_win,  1, 1, "Server listening on port %d...", SERVER_PORT);
    mvwprintw(main_win,  9, 1, "| RANK | IP\t\t|  PORT  | UP\t\t | DOWN\t\t |");
    mvwprintw(main_win, 10, 1, "----------------------------------------------------------------");
    wrefresh(main_win);

    // 设置定时器，每秒触发一次
    struct itimerval timer;
    signal(SIGALRM, handle_alarm);
    timer.it_value.tv_sec = 1;
    timer.it_value.tv_usec = 0;
    timer.it_interval.tv_sec = 1;
    timer.it_interval.tv_usec = 0;
    setitimer(ITIMER_REAL, &timer, NULL);

    int server_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    pthread_t threads[MAX_CLIENTS * 2];
    int thread_count = 0;

    memset(clients, 0, sizeof(clients)); // 初始化客户端信息数组

    // 创建 TCP 套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("socket creation failed");
        exit(EXIT_FAILURE);
    }

    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定套接字到地址
    if (bind(server_fd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 监听端口
    if (listen(server_fd, MAX_CLIENTS) < 0) {
        perror("listen failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 主循环等待客户端连接并处理按键输入
    while (1) {
        int ch = getch();  // 捕获按键输入
        if (ch == 'u') {
            mode = 0; // 切换到 UP 模式
            mvwprintw(main_win, 2, 1, "Current mode: UP   ");
            notify_clients_mode_change(mode);  // 通知所有客户端模式变化
            wrefresh(main_win);
        } else if (ch == 'd') {
            mode = 1; // 切换到 DOWN 模式
            mvwprintw(main_win, 2, 1, "Current mode: DOWN ");
            notify_clients_mode_change(mode);  // 通知所有客户端模式变化
            wrefresh(main_win);
        } else if (ch == 'b') {
            mode = 2; // 切换到 DOUBLE 模式
            mvwprintw(main_win, 2, 1, "Current mode: DOUBLE");
            notify_clients_mode_change(mode);  // 通知所有客户端模式变化
            wrefresh(main_win);
        }

        int* client_fd = malloc(sizeof(int));
        if (client_fd == NULL) {
            perror("malloc failed");
            close(server_fd);
            endwin();
            exit(EXIT_FAILURE);
        }

        *client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_addr_len); // 接收客户端请求
        if (*client_fd < 0) {
            perror("accept failed");
            free(client_fd);
            continue;
        }

        // 寻找空闲的客户端槽
        client_info_t* client = NULL;
        for (size_t i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].fd == 0) {
                client = &clients[i];
                client->fd = *client_fd;
                client->total_bytes_up = 0;
                client->total_bytes_down = 0;
                gettimeofday(&client->start, NULL); // 获取当前时间
                pthread_mutex_init(&client->lock, NULL);

                // 记录 IP 和端口
                inet_ntop(AF_INET, &(client_addr.sin_addr), client->ip, INET_ADDRSTRLEN);
                client->port = ntohs(client_addr.sin_port);
                break;
            }
        }

        free(client_fd); 

        if (client != NULL) {
            // 根据选择的模式，创建相应的线程
            if (mode == 0 || mode == 2) {  // UP 模式或 DOUBLE 模式都启动上传线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_upload, client) != 0) {
                    perror("pthread_create for upload failed");
                }
            }

            if (mode == 1 || mode == 2) {  // DOWN 模式或 DOUBLE 模式都启动下载线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_download, client) != 0) {
                    perror("pthread_create for download failed");
                }
            }

            // 防止线程数量超过上限
            if (thread_count >= MAX_CLIENTS * 2) {
                thread_count = 0;
                for (int i = 0; i < MAX_CLIENTS * 2; i++) {
                    pthread_join(threads[i], NULL);
                }
            }
        } else {
            printf("Max clients reached. Connection refused.\n");
        }
    }

    close(server_fd);
    endwin();
    return 0;
}
```

关键点

在检测到用户按键改变了模式时`notify_clients_mode_change()`函数遍历所有已连接的客户端，将新的模式发送给每个客户端

当有新的客户端连接时服务端会按照当前的模式为其启动相应的线程（UP / DOWN / double）



2024年9月9日04:28:54

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/time.h>
#include <signal.h>
#include <ncurses.h>

#define SERVER_PORT 5201
#define BUFFER_SIZE 1470
#define MAX_CLIENTS 10

WINDOW *main_win; // ncurses窗口指针

// 记录每个客户端的状态
typedef struct {
    int fd;                    // 客户端套接字
    long total_bytes_up;        // 累计上传字节数
    long total_bytes_down;      // 累计下载字节数
    struct timeval start;       // 统计开始时间
    pthread_mutex_t lock;       // 锁用于线程安全
    char ip[INET_ADDRSTRLEN];   // 客户端IP地址
    int port;                   // 客户端端口
    struct sockaddr_in client_addr;  // 客户端UDP地址
} client_info_t;

client_info_t clients[MAX_CLIENTS];
int is_tcp = 1; // 默认TCP

// 计算带宽并在ncurses窗口中显示
void handle_alarm(int sig) {
    double up_bandwidth_mbps, down_bandwidth_mbps;
    struct timeval now, elapsed;

    gettimeofday(&now, NULL);
    int rank = 1;  // 用于显示的排名

    // 遍历客户端信息数组
    for (size_t i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].fd != 0) { // 确保该槽位已经分配了客户端
            pthread_mutex_lock(&clients[i].lock);

            timersub(&now, &clients[i].start, &elapsed);  // 计算时间差
            double elapsed_time = elapsed.tv_sec + elapsed.tv_usec / 1000000.0;

            if (elapsed_time > 0) {  // 确保时间间隔不为0
                up_bandwidth_mbps = (clients[i].total_bytes_up * 8.0) / elapsed_time / 1e6;
                down_bandwidth_mbps = (clients[i].total_bytes_down * 8.0) / elapsed_time / 1e6;
            } else {
                up_bandwidth_mbps = 0;
                down_bandwidth_mbps = 0;
            }

            // 显示客户端的上传和下载带宽
            if (clients[i].total_bytes_up > 0 || clients[i].total_bytes_down > 0) {
                mvwprintw(main_win, rank + 10, 1, "| [%2d] | %s\t|  %d | %8.2f Mbps | %8.2f Mbps |",
                    rank, clients[i].ip, clients[i].port, up_bandwidth_mbps, down_bandwidth_mbps);
                rank++;
            }

            wrefresh(main_win); // 刷新窗口以显示更新的信息

            // 如果采样间隔太小，保留数据量而不重置
            if (elapsed_time >= 1.0) {
                clients[i].total_bytes_up = 0;
                clients[i].total_bytes_down = 0;
                gettimeofday(&clients[i].start, NULL);  // 重置起始时间
            }

            pthread_mutex_unlock(&clients[i].lock);
        }
    }

    // 清除多余的行（如果有客户端断开，行数可能会减少）
    for (int j = rank; j <= MAX_CLIENTS; j++) {
        mvwprintw(main_win, j + 10, 1, "|      | \t\t|        | \t\t | \t\t |"); // 清空行内容
    }
    wrefresh(main_win);
}

// 处理客户端上传数据
void* handle_client_upload(void* arg) {
    client_info_t* client = (client_info_t*)arg;
    char buffer[BUFFER_SIZE];
    ssize_t len;

    if (is_tcp) {
        // TCP 模式上传处理
        while ((len = recv(client->fd, buffer, BUFFER_SIZE, 0)) > 0) {
            pthread_mutex_lock(&client->lock);
            client->total_bytes_up += len;  // 记录上传的字节数
            pthread_mutex_unlock(&client->lock);
        }
    } else {
        // UDP 模式上传处理
        socklen_t addr_len = sizeof(client->client_addr);
        while ((len = recvfrom(client->fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client->client_addr, &addr_len)) > 0) {
            pthread_mutex_lock(&client->lock);
            client->total_bytes_up += len;  // 记录上传的字节数
            pthread_mutex_unlock(&client->lock);
        }
    }

    close(client->fd);
    pthread_exit(NULL);
}

// 处理客户端下载数据
void* handle_client_download(void* arg) {
    client_info_t* client = (client_info_t*)arg;
    char buffer[BUFFER_SIZE];
    memset(buffer, 'D', BUFFER_SIZE);  // 模拟下载数据
    ssize_t len;

    if (is_tcp) {
        // TCP 模式下载处理
        while (1) {
            len = send(client->fd, buffer, BUFFER_SIZE, 0);  // 向客户端发送数据
            if (len <= 0) {
                break;
            }

            pthread_mutex_lock(&client->lock);
            client->total_bytes_down += len;  // 记录下载的字节数
            pthread_mutex_unlock(&client->lock);
        }
    } else {
        // UDP 模式下载处理
        while (1) {
            len = sendto(client->fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client->client_addr, sizeof(client->client_addr));
            if (len <= 0) {
                break;
            }

            pthread_mutex_lock(&client->lock);
            client->total_bytes_down += len;  // 记录下载的字节数
            pthread_mutex_unlock(&client->lock);
        }
    }

    close(client->fd);
    pthread_exit(NULL);
}

int main(int argc, char* argv[]) {
    int mode = 0; // 0: UP, 1: DOWN, 2: DOUBLE

    if (argc < 3) {
        fprintf(stderr, "Usage: %s <tcp/udp> <up/down/double>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[1], "tcp") == 0) {
        is_tcp = 1; // TCP模式
    } else if (strcmp(argv[1], "udp") == 0) {
        is_tcp = 0; // UDP模式
    } else {
        fprintf(stderr, "Invalid protocol: %s. Use 'tcp' or 'udp'.\n", argv[1]);
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[2], "up") == 0) {
        mode = 0; // UP
    } else if (strcmp(argv[2], "down") == 0) {
        mode = 1; // DOWN
    } else if (strcmp(argv[2], "double") == 0) {
        mode = 2; // DOUBLE
    } else {
        fprintf(stderr, "Invalid mode: %s. Use 'up', 'down', or 'double'.\n", argv[2]);
        exit(EXIT_FAILURE);
    }

    int server_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    pthread_t threads[MAX_CLIENTS * 2];
    int thread_count = 0;

    memset(clients, 0, sizeof(clients)); // 初始化客户端信息数组

    // 创建 TCP 或 UDP 套接字
    if (is_tcp) {
        if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
            perror("socket creation failed");
            exit(EXIT_FAILURE);
        }
    } else {
        if ((server_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
            perror("UDP socket creation failed");
            exit(EXIT_FAILURE);
        }
    }

    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定套接字到地址
    if (bind(server_fd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if (is_tcp) {
        // TCP模式下监听端口
        if (listen(server_fd, MAX_CLIENTS) < 0) {
            perror("listen failed");
            close(server_fd);
            exit(EXIT_FAILURE);
        }
    }

    // 初始化ncurses
    initscr();
    cbreak();
    noecho();
    curs_set(0);
    main_win = newwin(MAX_CLIENTS * 2 + 4, 80, 0, 0);
    box(main_win, 0, 0);
    mvwprintw(main_win, 1, 1, "Server listening on port %d...", SERVER_PORT);
    mvwprintw(main_win, 8, 1, "| RANK | IP\t\t|  PORT  | UP\t\t | DOWN\t\t |");
    mvwprintw(main_win, 9, 1, "------------------------------------------------------------");
    wrefresh(main_win);

    // 设置定时器，每秒触发一次
    struct itimerval timer;
    signal(SIGALRM, handle_alarm);
    timer.it_value.tv_sec = 1;
    timer.it_value.tv_usec = 0;
    timer.it_interval.tv_sec = 1;
    timer.it_interval.tv_usec = 0;
    setitimer(ITIMER_REAL, &timer, NULL);

    while (1) {
        int* client_fd = malloc(sizeof(int));
        if (client_fd == NULL) {
            perror("malloc failed");
            close(server_fd);
            endwin();
            exit(EXIT_FAILURE);
        }

        if (is_tcp) {
            // TCP模式下接收客户端请求
            *client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_addr_len);
            if (*client_fd < 0) {
                perror("accept failed");
                free(client_fd);
                continue;
            }
        } else {
            // UDP模式下无需accept，直接使用recvfrom
            *client_fd = server_fd;
        }

        // 寻找空闲的客户端槽
        client_info_t* client = NULL;
        for (size_t i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].fd == 0) {
                client = &clients[i];
                client->fd = *client_fd;
                client->total_bytes_up = 0;
                client->total_bytes_down = 0;
                gettimeofday(&client->start, NULL); // 获取当前时间
                pthread_mutex_init(&client->lock, NULL);

                // 记录 IP 和端口
                inet_ntop(AF_INET, &(client_addr.sin_addr), client->ip, INET_ADDRSTRLEN);
                client->port = ntohs(client_addr.sin_port);
                client->client_addr = client_addr; // UDP模式下需要存储客户端地址
                break;
            }
        }

        free(client_fd);

        if (client != NULL) {
            // 根据选择的模式，创建相应的线程
            if (mode == 0 || mode == 2) {  // UP 模式或 DOUBLE 模式都启动上传线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_upload, client) != 0) {
                    perror("pthread_create for upload failed");
                }
            }

            if (mode == 1 || mode == 2) {  // DOWN 模式或 DOUBLE 模式都启动下载线程
                if (pthread_create(&threads[thread_count++], NULL, handle_client_download, client) != 0) {
                    perror("pthread_create for download failed");
                }
            }

            // 防止线程数量超过上限
            if (thread_count >= MAX_CLIENTS * 2) {
                thread_count = 0;
                for (int i = 0; i < MAX_CLIENTS * 2; i++) {
                    pthread_join(threads[i], NULL);
                }
            }
        } else {
            printf("Max clients reached. Connection refused.\n");
        }
    }

    close(server_fd);
    endwin();
    return 0;
}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <sys/time.h>

#define SERVER_PORT 5201
#define BUFFER_SIZE 1470

// 记录传输状态
typedef struct {
    int sockfd;
    long total_bytes_sent;
    long total_bytes_received;
    struct timeval start_time;
    struct sockaddr_in server_addr;  // 仅用于UDP
    int is_udp;  // 标志当前是否使用UDP
} transfer_info_t;

// 发送数据（上传）
void* send_data(void* arg) {
    transfer_info_t* info = (transfer_info_t*)arg;
    char buffer[BUFFER_SIZE];
    memset(buffer, 'A', sizeof(buffer));  // 填充缓冲区数据

    if (info->is_udp) {
        // UDP发送
        while (1) {
            ssize_t n = sendto(info->sockfd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&(info->server_addr), sizeof(info->server_addr));
            if (n <= 0) {
                break;
            }
            info->total_bytes_sent += n;  // 累积上传的字节数
        }
    } else {
        // TCP发送
        while (1) {
            ssize_t n = send(info->sockfd, buffer, BUFFER_SIZE, 0);  // 发送数据
            if (n <= 0) {
                break;
            }
            info->total_bytes_sent += n;  // 累积上传的字节数
        }
    }

    return NULL;
}

// 接收数据（下载）
void* receive_data(void* arg) {
    transfer_info_t* info = (transfer_info_t*)arg;
    char buffer[BUFFER_SIZE];
    socklen_t addr_len = sizeof(info->server_addr);

    if (info->is_udp) {
        // UDP接收
        while (1) {
            ssize_t n = recvfrom(info->sockfd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&(info->server_addr), &addr_len);
            if (n <= 0) {
                break;
            }
            info->total_bytes_received += n;  // 累积下载的字节数
        }
    } else {
        // TCP接收
        while (1) {
            ssize_t n = recv(info->sockfd, buffer, BUFFER_SIZE, 0);  // 接收数据
            if (n <= 0) {
                break;
            }
            info->total_bytes_received += n;  // 累积下载的字节数
        }
    }

    return NULL;
}

int main(int argc, char* argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <tcp/udp> <up/down/double>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    int mode = 0; // 0: UP, 1: DOWN, 2: DOUBLE
    int is_udp = 0;  // 默认TCP

    // 解析协议类型
    if (strcmp(argv[1], "tcp") == 0) {
        is_udp = 0;  // TCP
    } else if (strcmp(argv[1], "udp") == 0) {
        is_udp = 1;  // UDP
    } else {
        fprintf(stderr, "Invalid protocol: %s. Use 'tcp' or 'udp'.\n", argv[1]);
        exit(EXIT_FAILURE);
    }

    // 解析模式类型
    if (strcmp(argv[2], "up") == 0) {
        mode = 0;  // 上传模式
    } else if (strcmp(argv[2], "down") == 0) {
        mode = 1;  // 下载模式
    } else if (strcmp(argv[2], "double") == 0) {
        mode = 2;  // 双向模式
    } else {
        fprintf(stderr, "Invalid mode: %s. Use 'up', 'down', or 'double'.\n", argv[2]);
        exit(EXIT_FAILURE);
    }

    int sockfd;
    struct sockaddr_in server_addr;
    struct timeval end_time;
    pthread_t send_thread, receive_thread;

    // 创建套接字
    if (is_udp) {
        // UDP 套接字
        if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
            perror("UDP socket creation failed");
            exit(EXIT_FAILURE);
        }
    } else {
        // TCP 套接字
        if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
            perror("TCP socket creation failed");
            exit(EXIT_FAILURE);
        }
    }

    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (!is_udp) {
        // TCP连接
        if (connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
            perror("connect failed");
            close(sockfd);
            exit(EXIT_FAILURE);
        }
    }

    // 初始化传输信息
    transfer_info_t transfer_info = {
        .sockfd = sockfd,
        .total_bytes_sent = 0,
        .total_bytes_received = 0,
        .server_addr = server_addr,
        .is_udp = is_udp,
    };
    gettimeofday(&transfer_info.start_time, NULL);

    // 根据模式启动相应的线程
    if (mode == 0 || mode == 2) {  // 上传模式或双向模式，启动发送线程
        pthread_create(&send_thread, NULL, send_data, &transfer_info);
    }

    if (mode == 1 || mode == 2) {  // 下载模式或双向模式，启动接收线程
        pthread_create(&receive_thread, NULL, receive_data, &transfer_info);
    }

    // 等待用户按下回车键结束测试
    printf("Press Enter to stop the test...\n");
    getchar();

    // 关闭线程
    if (mode == 0 || mode == 2) {
        pthread_cancel(send_thread);
        pthread_join(send_thread, NULL);
    }
    
    if (mode == 1 || mode == 2) {
        pthread_cancel(receive_thread);
        pthread_join(receive_thread, NULL);
    }

    // 计算带宽
    gettimeofday(&end_time, NULL);
    double elapsed_time = (end_time.tv_sec - transfer_info.start_time.tv_sec) +
                          ((end_time.tv_usec - transfer_info.start_time.tv_usec) / 1000000.0);

    double upload_bandwidth = (transfer_info.total_bytes_sent * 8.0) / elapsed_time / 1e6;
    double download_bandwidth = (transfer_info.total_bytes_received * 8.0) / elapsed_time / 1e6;

    printf("Total data sent: %ld bytes\n", transfer_info.total_bytes_sent);
    printf("Total data received: %ld bytes\n", transfer_info.total_bytes_received);
    printf("Upload Bandwidth: %.2f Mbps\n", upload_bandwidth);
    printf("Download Bandwidth: %.2f Mbps\n", download_bandwidth);

    // 关闭套接字
    close(sockfd);
    return 0;
}
```

#### UDP广播发现服务器

服务器端：增加监听UDP广播消息功能 另起一个线程监听客户端广播消息 回复自己的IP地址

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>

#define SERVER_PORT 5201
#define BROADCAST_PORT 5202
#define BUFFER_SIZE 1470

// 处理UDP广播消息的线程
void* handle_broadcast(void* arg) {
    int broadcast_fd;
    struct sockaddr_in broadcast_addr, client_addr;
    char buffer[BUFFER_SIZE];
    socklen_t addr_len = sizeof(client_addr);

    // 创建UDP套接字用于接收广播
    if ((broadcast_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("Broadcast socket creation failed");
        pthread_exit(NULL);
    }

    // 配置广播选项
    int broadcastPermission = 1;
    if (setsockopt(broadcast_fd, SOL_SOCKET, SO_BROADCAST, (void*)&broadcastPermission, sizeof(broadcastPermission)) < 0) {
        perror("setsockopt failed");
        close(broadcast_fd);
        pthread_exit(NULL);
    }

    // 配置服务器地址
    memset(&broadcast_addr, 0, sizeof(broadcast_addr));
    broadcast_addr.sin_family = AF_INET;
    broadcast_addr.sin_port = htons(BROADCAST_PORT);
    broadcast_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定到广播端口
    if (bind(broadcast_fd, (struct sockaddr*)&broadcast_addr, sizeof(broadcast_addr)) < 0) {
        perror("Broadcast bind failed");
        close(broadcast_fd);
        pthread_exit(NULL);
    }

    printf("Broadcast server listening on port %d...\n", BROADCAST_PORT);

    // 循环接收广播请求并回复
    while (1) {
        int len = recvfrom(broadcast_fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client_addr, &addr_len);
        if (len > 0) {
            buffer[len] = '\0';  // 确保字符串末尾
            printf("Received broadcast message: %s\n", buffer);

            // 向客户端回复服务器的IP地址
            const char* response = "Server is here";
            sendto(broadcast_fd, response, strlen(response), 0, (struct sockaddr*)&client_addr, addr_len);
        }
    }

    close(broadcast_fd);
    pthread_exit(NULL);
}
```

客户端：通过UDP发送广播消息发现服务器 收到服务器回复的IP地址后 建立连接

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define BROADCAST_PORT 5202
#define BROADCAST_IP "255.255.255.255"
#define BUFFER_SIZE 1470

// 通过UDP广播发现服务器
int discover_server(char* server_ip) {
    int sockfd;
    struct sockaddr_in broadcast_addr;
    char buffer[BUFFER_SIZE];
    socklen_t addr_len = sizeof(broadcast_addr);

    // 创建UDP套接字
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("socket failed");
        return -1;
    }

    // 设置套接字为广播
    int broadcastPermission = 1;
    if (setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, (void*)&broadcastPermission, sizeof(broadcastPermission)) < 0) {
        perror("setsockopt failed");
        close(sockfd);
        return -1;
    }

    // 配置广播地址
    memset(&broadcast_addr, 0, sizeof(broadcast_addr));
    broadcast_addr.sin_family = AF_INET;
    broadcast_addr.sin_port = htons(BROADCAST_PORT);
    broadcast_addr.sin_addr.s_addr = inet_addr(BROADCAST_IP);

    // 发送广播消息
    const char* message = "Client searching for server";
    if (sendto(sockfd, message, strlen(message), 0, (struct sockaddr*)&broadcast_addr, sizeof(broadcast_addr)) < 0) {
        perror("sendto failed");
        close(sockfd);
        return -1;
    }

    // 接收服务器的回复
    int len = recvfrom(sockfd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&broadcast_addr, &addr_len);
    if (len > 0) {
        buffer[len] = '\0';
        printf("Discovered server response: %s\n", buffer);
        strcpy(server_ip, inet_ntoa(broadcast_addr.sin_addr));  // 保存服务器IP
        close(sockfd);
        return 0;
    }

    close(sockfd);
    return -1;
}
```

客户端启动时广播查找服务器，或者使用指定IP

在服务器主程序启动广播线程，监听客户端广播请求

#### 带宽限制功能

定义一个全局变量存储带宽上限

根据传输的字节数和时间来计算当前的发送速率，如果当前速率超过设定的上限，通过sleep()限制传输速率

用户通过ncurses界面动态输入新的带宽上限，更新全局变量

```c
double bandwidth_limit_mbps = 10.0;  // 默认限制带宽为 10 Mbps
pthread_mutex_t bandwidth_lock = PTHREAD_MUTEX_INITIALIZER;  // 保护带宽上限的锁
// 发送数据（上传）并进行带宽限速
void* send_data(void* arg) {
    transfer_info_t* info = (transfer_info_t*)arg;
    char buffer[BUFFER_SIZE];
    memset(buffer, 'A', sizeof(buffer));  // 填充缓冲区数据
    struct timeval start, end;
    ssize_t n;
    double elapsed_time, current_bandwidth_mbps;

    while (1) {
        gettimeofday(&start, NULL);

        if (info->is_udp) {
            n = sendto(info->sockfd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&(info->server_addr), sizeof(info->server_addr));
        } else {
            n = send(info->sockfd, buffer, BUFFER_SIZE, 0);  // 发送数据
        }

        if (n <= 0) break;

        info->total_bytes_sent += n;

        gettimeofday(&end, NULL);
        elapsed_time = (end.tv_sec - start.tv_sec) + (end.tv_usec - start.tv_usec) / 1000000.0;

        // 计算当前带宽 (Mbps)
        current_bandwidth_mbps = (n * 8.0) / (elapsed_time * 1e6);

        // 检查是否超出限速
        pthread_mutex_lock(&bandwidth_lock);
        if (current_bandwidth_mbps > bandwidth_limit_mbps) {
            double excess = current_bandwidth_mbps / bandwidth_limit_mbps;
            usleep((useconds_t)(excess * 1000000));  // 暂停一段时间来限速
        }
        pthread_mutex_unlock(&bandwidth_lock);
    }

    return NULL;
}
void* ncurses_interface(void* arg) {
    int ch;
    char input[10];
    int input_pos = 0;

    while (1) {
        ch = wgetch(main_win);
        if (ch == 'q') {
            break;  // 用户输入'q'退出
        } else if (ch == '\n') {
            input[input_pos] = '\0';
            double new_limit = atof(input);

            // 更新带宽上限
            pthread_mutex_lock(&bandwidth_lock);
            bandwidth_limit_mbps = new_limit;
            pthread_mutex_unlock(&bandwidth_lock);

            mvwprintw(main_win, 2, 1, "Bandwidth limit set to: %.2f Mbps", bandwidth_limit_mbps);
            wrefresh(main_win);
            input_pos = 0;  // 清空输入缓冲
            memset(input, 0, sizeof(input));
        } else if (isdigit(ch)) {
            if (input_pos < sizeof(input) - 1) {
                input[input_pos++] = ch;
                mvwprintw(main_win, 3, 1, "Enter new bandwidth limit: %s", input);
                wrefresh(main_win);
            }
        }
    }

    return NULL;
}
```

在main函数启动一个单独的线程来处理用户的输入

```
pthread_create(&ncurses_thread, NULL, ncurses_interface, NULL) != 0
```

#### 界面设计

```c
    mvwprintw(main_win, 1, 1, "Server    addr: \t192.168.18.125\t\tport: 8888");
    mvwprintw(main_win, 2, 1, "Broadcast addr: \t192.168.18.255\t\tport: 5005"); // 广播地址 为实现广播功能
    mvwprintw(main_win, 3, 1, "Test      time: \t500 sec        \t\tPakg size: 1470 bit");
    mvwprintw(main_win, 4, 1, "Mode:    double \t\t\t\tLimit: no limit");
    mvwprintw(main_win, 6, 1, "pre        next \tset-limit\t\texit(press 'q')");
```

最新效果图：

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1725861811270-7844ed85-fe41-41cb-b5a6-be1198310cae.png)

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1725861834897-cdf4d705-f315-4c90-b729-804c289a52b7.png)

#### 显示当前设备连接数

解决double模式下在handle_client_upload和handle_client_download中进行connected_clients减一 会导致减两次的问题

```c
void* handle_client_upload(void* arg) {
    client_info_t* client = (client_info_t*)arg;
    char buffer[BUFFER_SIZE];
    ssize_t len;

    if (is_tcp) {
        // TCP 模式上传处理
        while ((len = recv(client->fd, buffer, BUFFER_SIZE, 0)) > 0) {
            pthread_mutex_lock(&client->lock);
            client->total_bytes_up += len;  // 记录上传的字节数
            pthread_mutex_unlock(&client->lock);
        }
    } else {
        // UDP 模式上传处理
        socklen_t addr_len = sizeof(client->client_addr);
        while ((len = recvfrom(client->fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client->client_addr, &addr_len)) > 0) {
            pthread_mutex_lock(&client->lock);
            client->total_bytes_up += len;  // 记录上传的字节数
            pthread_mutex_unlock(&client->lock);
        }
    }

    close(client->fd);

    // 设备数减一 仅仅在设备活跃时执行
    pthread_mutex_lock(&client_count_lock);
    if (client->is_active) {
        connected_clients --;
        client->is_active = 0;
    }
    pthread_mutex_unlock(&client_count_lock);
    // 更新界面
    mvwprintw(main_win, 3, 1, "connected_clients: \t%d\t\t Run       time:     %d", connected_clients, run_time);
    wrefresh(main_win);

    pthread_exit(NULL);
}
```

这里（或者handle_client_download）如果不做判断直接在断开连接时`connected_clients --`会导致double模式下断开客户端会减两次

所以需要在客户端结构体定义一个`int is_active = 0;`客户端启动时该值设为1 当关闭时进行减操作之后设为0

每次关闭客户端只有`is_active = 1`时才进行减操作，这样每个客户端断开只会执行一次`connected_clients --`.

#### UDP测速模式逻辑调整

问题：

使用 `UDP` 模式时，服务端的 `recvfrom()` 调用会一直使用相同的 `server_fd`，而不会像 `TCP` 模式那样为每个客户端创建新的套接字。这就导致即使客户端还没有达到最大数量，服务端也误认为客户端数量已经满了，因为所有的 `recvfrom()` 调用都使用了同一个 `server_fd`，无法正确管理多个客户端。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/time.h>
#include <signal.h>
#include <ncurses.h>

#define SERVER_PORT 5201
#define BUFFER_SIZE 1470
#define MAX_CLIENTS 10

WINDOW *main_win; // ncurses窗口指针

// 记录每个客户端的状态
typedef struct {
    int fd;                    // 客户端套接字
    long total_bytes_up;        // 累计上传字节数
    long total_bytes_down;      // 累计下载字节数
    struct timeval start;       // 统计开始时间
    pthread_mutex_t lock;       // 锁用于线程安全
    char ip[INET_ADDRSTRLEN];   // 客户端IP地址
    int port;                   // 客户端端口
    struct sockaddr_in client_addr;  // 客户端UDP地址
    int is_active; // 判断当前客户端是否活跃 用于计数
} client_info_t;

client_info_t clients[MAX_CLIENTS];
int is_tcp = 1; // 默认TCP
char server_ip[INET_ADDRSTRLEN] = "127.0.0.1";
// int current_mode = 0;
double bandwidth_limit_mbps = 0.0;
pthread_mutex_t bandwidth_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t client_count_lock = PTHREAD_MUTEX_INITIALIZER;
int connected_clients = 0;
int run_time = 0;

// 计算带宽并在ncurses窗口中显示
void handle_alarm(int sig) {
    double up_bandwidth_mbps, down_bandwidth_mbps;
    struct timeval now, elapsed;

    gettimeofday(&now, NULL);
    int rank = 1;  // 用于显示的排名

    // 遍历客户端信息数组
    for (size_t i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].is_active) { // 确保该槽位已经分配了客户端
            pthread_mutex_lock(&clients[i].lock);

            timersub(&now, &clients[i].start, &elapsed);  // 计算时间差
            double elapsed_time = elapsed.tv_sec + elapsed.tv_usec / 1000000.0;

            if (elapsed_time > 0) {  // 确保时间间隔不为0
                up_bandwidth_mbps = (clients[i].total_bytes_up * 8.0) / elapsed_time / 1e6;
                down_bandwidth_mbps = (clients[i].total_bytes_down * 8.0) / elapsed_time / 1e6;
            } else {
                up_bandwidth_mbps = 0;
                down_bandwidth_mbps = 0;
            }

            // 显示客户端的上传和下载带宽
            if (clients[i].total_bytes_up > 0 || clients[i].total_bytes_down > 0) {
                mvwprintw(main_win, rank + 10, 1, "| [%2d] | %s\t|  %d | %8.2f Mbps | %8.2f Mbps |",
                    rank, clients[i].ip, clients[i].port, up_bandwidth_mbps, down_bandwidth_mbps);
                rank++;
            }

            wrefresh(main_win); // 刷新窗口以显示更新的信息

            // 如果采样间隔太小，保留数据量而不重置
            if (elapsed_time >= 1.0) {
                clients[i].total_bytes_up = 0;
                clients[i].total_bytes_down = 0;
                gettimeofday(&clients[i].start, NULL);  // 重置起始时间
            }

            pthread_mutex_unlock(&clients[i].lock);
        }
    }

    // 清除多余的行（如果有客户端断开，行数可能会减少）
    for (int j = rank; j <= MAX_CLIENTS; j++) {
        mvwprintw(main_win, j + 10, 1, "|      | \t\t|        | \t\t | \t\t |"); // 清空行内容
    }
    wrefresh(main_win);
}

// 处理客户端上传数据
void* handle_udp_client(void* arg) {
    int server_fd = *(int*)arg;
    char buffer[BUFFER_SIZE];
    ssize_t len;
    struct sockaddr_in client_addr;
    socklen_t addr_len = sizeof(client_addr);

    while (1) {
        // 接收UDP数据
        len = recvfrom(server_fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client_addr, &addr_len);
        if (len < 0) {
            perror("recvfrom failed");
            continue;
        }

        // 查找该客户端是否已经存在
        client_info_t* client = NULL;
        for (size_t i = 0; i < MAX_CLIENTS; i++) {
            if (clients[i].is_active && memcmp(&clients[i].client_addr, &client_addr, sizeof(client_addr)) == 0) {
                client = &clients[i];
                break;
            }
        }

        // 如果客户端不存在，分配一个新的槽位
        if (client == NULL) {
            for (size_t i = 0; i < MAX_CLIENTS; i++) {
                if (!clients[i].is_active) {
                    client = &clients[i];
                    client->total_bytes_up = 0;
                    client->total_bytes_down = 0;
                    client->is_active = 1;
                    client->client_addr = client_addr;

                    gettimeofday(&client->start, NULL);
                    pthread_mutex_init(&client->lock, NULL);

                    // 记录 IP 和端口
                    inet_ntop(AF_INET, &(client_addr.sin_addr), client->ip, INET_ADDRSTRLEN);
                    client->port = ntohs(client_addr.sin_port);

                    pthread_mutex_lock(&client_count_lock);
                    connected_clients++;
                    pthread_mutex_unlock(&client_count_lock);

                    // 更新界面
                    mvwprintw(main_win, 3, 1, "connected_clients: \t%d\t\t Run       time:     %d", connected_clients, run_time);
                    wrefresh(main_win);

                    break;
                }
            }
        }

        // 如果没有空闲槽，拒绝新的客户端
        if (client == NULL) {
            printf("Max clients reached. Connection refused.\n");
            continue;
        }

        // 处理客户端上传的数据
        pthread_mutex_lock(&client->lock);
        client->total_bytes_up += len;
        pthread_mutex_unlock(&client->lock);
    }
}

int main(int argc, char* argv[]) {
    int mode = 0; // 0: UP, 1: DOWN, 2: DOUBLE

    if (argc < 3) {
        fprintf(stderr, "Usage: %s <tcp/udp> <up/down/double>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[1], "tcp") == 0) {
        is_tcp = 1; // TCP模式
    } else if (strcmp(argv[1], "udp") == 0) {
        is_tcp = 0; // UDP模式
    } else {
        fprintf(stderr, "Invalid protocol: %s. Use 'tcp' or 'udp'.\n", argv[1]);
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[2], "up") == 0) {
        mode = 0; // UP
    } else if (strcmp(argv[2], "down") == 0) {
        mode = 1; // DOWN
    } else if (strcmp(argv[2], "double") == 0) {
        mode = 2; // DOUBLE
    } else {
        fprintf(stderr, "Invalid mode: %s. Use 'up', 'down', or 'double'.\n", argv[2]);
        exit(EXIT_FAILURE);
    }

    int server_fd;
    struct sockaddr_in server_addr;
    pthread_t udp_thread;

    memset(clients, 0, sizeof(clients)); // 初始化客户端信息数组

    // 创建 TCP 或 UDP 套接字
    if (is_tcp) {
        if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
            perror("socket creation failed");
            exit(EXIT_FAILURE);
        }
    } else {
        if ((server_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
            perror("UDP socket creation failed");
            exit(EXIT_FAILURE);
        }
    }

    // 配置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定套接字到地址
    if (bind(server_fd, (const struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if (is_tcp) {
        // TCP模式下监听端口
        if (listen(server_fd, MAX_CLIENTS) < 0) {
            perror("listen failed");
            close(server_fd);
            exit(EXIT_FAILURE);
        }
    } else {
        // 启动UDP处理线程
        if (pthread_create(&udp_thread, NULL, handle_udp_client, &server_fd) != 0) {
            perror("pthread_create for UDP failed");
            close(server_fd);
            endwin();
            exit(EXIT_FAILURE);
        }
        pthread_join(udp_thread, NULL);
    }

    return 0;
}
```

**TCP** 使用`accept`来处理每个tcp连接，分别为每个连接创建上传或者下载线程

**UDP** 只用使用一个线程来处理所有的UDP客户端，UDP是无连接，服务端必须先接收到客户端的数据包，获得客户端的ip和port，接下来服务器使用`sendto()`进行下行

所以客户端需要先发送启动信号，通知服务器，服务器接收到信号之后开始进行收发，在接收后根据`mode`来区分上行还是下行



在下行或者double测试时先向服务器发送自己的信息，但是服务器界面还是没有反应

原因：UDP每次通信都是独立的，服务器不需要维持一个持久的连接状态，不需要向TCP那样通过文件描述符fd来跟踪每个客户端的连接，而是使用同一个fd来接收和发送不同客户端的数据包，在UDP服务器中，区分不同的客户端通常通过**IP地址**和**端口号**，而不是通过`fd`。每个客户端发送的数据报文中都包含客户端的IP地址和端口号，服务器可以根据这些信息来识别不同的客户端。而在TCP中，`fd`会在客户端和服务器之间建立连接后分配，作为连接的唯一标识。

`handle_alarm`函数中，对于UDP连接，`fd`不会一直有效，因为UDP是无连接的协议，不像TCP一直保持连接状态，需要通过`is_active`标志位来判断

```c
// 计算带宽并在ncurses窗口中显示
void handle_alarm(int sig) {
    double up_bandwidth_mbps, down_bandwidth_mbps;
    struct timeval now, elapsed;

    gettimeofday(&now, NULL);
    int rank = 1;  // 用于显示的排名

    // 遍历客户端信息数组
    for (size_t i = 0; i < MAX_CLIENTS; i++) {
        // if (clients[i].fd != 0) { // 确保该槽位已经分配了客户端
        if (clients[i].is_active) {
            pthread_mutex_lock(&clients[i].lock);
```



#### 通过心跳包机制或者定时检测清理不活跃客户端

1客户端定期向服务器发送心跳包，如果服务器没有收到，就认为该客户端已经断开连接，将其标记为不活跃，从客户端列表中移除

2每一秒检测一次客户端列表，如果在一秒内该客户端没有发生数据的收发，那么执行`connected_clients --`

```c
void* monitor_clients(void* arg) {
    while (1) {
        for (size_t i = 0; i < MAX_CLIENTS; i ++) {
            if (clients[i].is_active) {
                pthread_mutex_lock(&clients[i].lock);

                if (clients[i].total_bytes_up == 0 && clients[i].total_bytes_down == 0) {
                    clients[i].is_active = 0;

                    pthread_mutex_lock(&client_count_lock);
                    connected_clients --;
                    pthread_mutex_unlock(&client_count_lock);

                    mvwprintw(main_win, 3, 1, "connected_clients: \t%d\t\t UdpRunningtime:     %d", connected_clients, run_time);
                    wrefresh(main_win);
                }

                clients[i].total_bytes_up = 0;
                clients[i].total_bytes_down = 0;

                pthread_mutex_unlock(&clients[i].lock);
            }
        }

        sleep(1);
    }
}
```

**两者的区别:**

| **特点**         | **基于数据传输量的检测**               | **心跳包机制**                           |
| ---------------- | -------------------------------------- | ---------------------------------------- |
| **检测方式**     | 被动监控数据传输量，判断是否活跃       | 主动发送心跳包，确保客户端仍然在线       |
| **适用场景**     | 高频率数据传输，短期连接               | 低频数据传输，长期保持连接               |
| **额外通信开销** | 无额外开销                             | 心跳包的定期发送需要额外的网络开销       |
| **断连判断**     | 无数据传输则断连                       | 一定时间内没有收到心跳包则判断断连       |
| **误判可能性**   | 长时间无数据但连接还在，可能被误判断连 | 定时发送心跳包，更精确判断客户端连接状态 |

- **基于数据传输的检测方法：** 适合当前的带宽测试场景，客户端频繁发送和接收数据，断开时数据流停止，可以快速检测到不活跃状态。
- **心跳包机制：** 适合长时间维持连接、但数据传输并不频繁的场景，例如监控系统、在线服务等。

### tcp模式下 服务端断开 客户端会立马断开 但是udp模式 即使服务器断开 但是客户端还会等待重连

### udp广播 设计

#### 服务器端

服务器监听广播请求

固定端口接收广播

动态获取IP地址

```c
// 处理客户端的广播请求，发送服务器IP地址给客户端
void* handle_broadcast_requests(void* arg) {
    int server_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t addr_len = sizeof(client_addr);
    char buffer[BUFFER_SIZE];

    // 创建UDP套接字
    if ((server_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("UDP socket creation failed");
        exit(EXIT_FAILURE);
    }

    // 设置服务器地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(5202); // 用于广播的端口
    server_addr.sin_addr.s_addr = INADDR_ANY;

    // 绑定地址
    if (bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    while (1) {
        // 接收客户端的广播请求
        ssize_t len = recvfrom(server_fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&client_addr, &addr_len);
        if (len > 0) {
            printf("Received broadcast request from client\n");

            // 回复服务器IP地址
            char server_ip[] = "Server IP Address"; // 这里可以自动获取服务器IP
            sendto(server_fd, server_ip, strlen(server_ip), 0, (struct sockaddr*)&client_addr, addr_len);
        }
    }

    close(server_fd);
    return NULL;
}
```

#### 客户端

发送广播请求

等待服务器回复

```c
// 广播请求获取服务器的IP地址
void get_server_ip(char* server_ip) {
    int client_fd;
    struct sockaddr_in broadcast_addr;
    socklen_t addr_len = sizeof(broadcast_addr);
    char buffer[BUFFER_SIZE];
    
    // 创建UDP套接字
    if ((client_fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("UDP socket creation failed");
        exit(EXIT_FAILURE);
    }

    // 设置广播选项
    int broadcast = 1;
    if (setsockopt(client_fd, SOL_SOCKET, SO_BROADCAST, &broadcast, sizeof(broadcast)) < 0) {
        perror("setsockopt failed");
        close(client_fd);
        exit(EXIT_FAILURE);
    }

    // 配置广播地址
    memset(&broadcast_addr, 0, sizeof(broadcast_addr));
    broadcast_addr.sin_family = AF_INET;
    broadcast_addr.sin_port = htons(5202); // 服务端监听广播的端口
    broadcast_addr.sin_addr.s_addr = inet_addr("255.255.255.255");

    // 发送广播请求
    char message[] = "DISCOVER_SERVER";
    sendto(client_fd, message, strlen(message), 0, (struct sockaddr*)&broadcast_addr, addr_len);

    // 接收服务器的响应
    ssize_t len = recvfrom(client_fd, buffer, BUFFER_SIZE, 0, (struct sockaddr*)&broadcast_addr, &addr_len);
    if (len > 0) {
        buffer[len] = '\0';
        strcpy(server_ip, buffer); // 保存服务器IP地址
        printf("Server IP: %s\n", server_ip);
    }

    close(client_fd);
}
```

本机配置

```bash
bhhh@bhcomputer:~/desktop/profile/bandwidth-test$ ifconfig
enp2s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.18.140  netmask 255.255.255.0  broadcast 192.168.18.255
        inet6 fe80::d195:3d5f:fc8:9e05  prefixlen 64  scopeid 0x20<link>
        ether 20:88:10:aa:7b:e1  txqueuelen 1000  (以太网)
        RX packets 950349  bytes 108446861 (108.4 MB)
        RX errors 0  dropped 23689  overruns 0  frame 0
        TX packets 32002  bytes 8264546 (8.2 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (本地环回)
        RX packets 1526325  bytes 30908188631 (30.9 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1526325  bytes 30908188631 (30.9 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1726021388754-83085a3d-203a-4da5-a2a6-e016ebbb784b.png)

本地环回测试 客户端通过局域网地址连接到服务器 模拟局域网连接 而不是回环地址

### 限速功能 设计

客户端界面输入，将输入的limit值发送给客户端

```bash
// 将输入的limit值发给客户端 
void send_bandwidth_limit_to_clients(double new_limit) {
    char limit_message[50];
    snprintf(limit_message, sizeof(limit_message), "BANDWIDTH_LIMIT:%.2f", new_limit);

    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].is_active) {
            if (is_tcp) {
                send(clients[i].fd, limit_message, strlen(limit_message), 0);
            } else {
                sendto(clients[i].fd, limit_message, strlen(limit_message), 0, (struct sockaddr*)&clients[i].client_addr, sizeof(clients[i].client_addr));
            }
        }
    }
}

void* listen_for_input(void* arg) {
    char input[10];
    while (1) {
        mvwprintw(main_win, 7, 1, "Enter new bandwidth limit (Mbps): ");
        wrefresh(main_win);

        echo();
        wgetnstr(main_win, input, 8); // 字符串输入
        noecho();

        double new_limit = atof(input);

        if (new_limit > 0) {
            pthread_mutex_lock(&bandwidth_lock);
            bandwidth_limit_mbps = new_limit;
            pthread_mutex_unlock(&bandwidth_lock);

            send_bandwidth_limit_to_clients(new_limit);

            mvwprintw(main_win, 8, 1, "bandwidth limit updated to %.2f Mbps", bandwidth_limit_mbps);
            wrefresh(main_win);
        } else {
            mvwprintw(main_win, 8, 1, "invalid input. please enter a positive number.");
            wrefresh(main_win);
        }

        sleep(1);
        mvwprintw(main_win, 8, 1, "                                                      ");
        wrefresh(main_win);
    }
}
```

实现思路：

实时输入带宽限制值，在服务端允许用户输入带宽限制值，并更新在界面中

在客户端发送和接收数据时限速，通过计算每秒发送和接收的数据量，如果达到或超过限速，则用 `usleep()` 来暂停传输，直到下一秒钟为止

根据当前的传输速度，动态控制每秒的传输数据量，如果限速是 10 Mbps，则每秒最多传输 `10 * 1e6 / 8` 的字节数据

**服务器只负责发送限速值，客户端接收后应用限速逻辑**



使用退出标志控制线程的终止，使用条件变量或线程的循环逻辑判断是否继续运行，而不是直接强制终止。改进退出逻辑： 定义一个退出标志变量并通过它来控制线程结束

```c
int stop_threads = 0;

void* send_data(void* arg) {
    transfer_info_t* info = (transfer_info_t*)arg;
    char buffer[BUFFER_SIZE];
    memset(buffer, 'A', sizeof(buffer));  // 填充缓冲区数据

    while (!stop_threads) {  // 用 stop_threads 控制线程退出
        // 发送数据逻辑
    }

    return NULL;
}
```

主线程在检测到用户按下回车时，设置 `stop_threads = 1`，并等待所有线程自行退出。



**当界面更新了带宽限制，**`**bandwidth_limit_mbps**` **的值被更新了，但是正在运行的上传或下载线程并没有及时读取新的值。**

### 实时切换测速模式

切换上下行模式思路：

创建一个控制线程，通过信道来监听来自主线程的模式切换信号，根据该信号改变客户端的收发行为。

通过`u、d、p`按键来切换`up down double` 模式，**同时为了不影响限速输入 检测输入 如果是数字 则获取限速值**

```c
// 监听输入函数，处理模式切换和限速设置
void* input_listener(void* arg) {
    char input_buffer[10] = {0};  // 用于保存限速输入
    int input_index = 0;  // 当前输入索引
    int ch;

    nodelay(main_win, TRUE);  // 设置窗口为非阻塞模式

    while (1) {
        ch = wgetch(main_win);  // 监听按键输入
        if (ch == ERR) {
            usleep(100000);  // 如果没有按键输入，等待100ms
            continue;
        }

        if (isdigit(ch)) {  // 如果输入的是数字，则处理限速
            if (input_index < sizeof(input_buffer) - 1) {
                input_buffer[input_index++] = ch;
                mvwprintw(main_win, 7, 1, "Enter new bandwidth limit (Mbps): %s    ", input_buffer);
                wrefresh(main_win);
            }
        } else if (ch == '\n') {  // 如果按下了回车键，处理限速值
            if (input_index > 0) {
                input_buffer[input_index] = '\0';
                double new_limit = atof(input_buffer);

                if (new_limit > 0) {
                    pthread_mutex_lock(&bandwidth_lock);
                    bandwidth_limit_mbps = new_limit;
                    pthread_mutex_unlock(&bandwidth_lock);

                    send_bandwidth_limit_to_clients(new_limit);

                    // 显示更新信息并保持1秒
                    mvwprintw(main_win, 8, 1, "bandwidth limit updated to %.2f Mbps", bandwidth_limit_mbps);
                    wrefresh(main_win);
                    sleep(1);  // 暂停1秒
                    mvwprintw(main_win, 8, 1, "                                        ");  // 清空该行
                    wrefresh(main_win);
                } else {
                    mvwprintw(main_win, 8, 1, "invalid input. please enter a positive number.");
                    wrefresh(main_win);
                    sleep(1);  // 暂停1秒
                    mvwprintw(main_win, 8, 1, "                                        ");  // 清空该行
                    wrefresh(main_win);
                }

                memset(input_buffer, 0, sizeof(input_buffer));  // 重置输入缓冲区
                input_index = 0;  // 重置索引
            }
        } else if (ch == 'u' || ch == 'd' || ch == 'p') {  // 如果输入的是模式切换字母
            pthread_mutex_lock(&mode_lock);
            if (ch == 'u') {
                mode = 0;  // 切换到上传模式
            } else if (ch == 'd') {
                mode = 1;  // 切换到下载模式
            } else if (ch == 'p') {
                mode = 2;  // 切换到双向模式
            }
            mvwprintw(main_win, 4, 1, "Current Mode: \t%s    ", mode == 0 ? "UP" : (mode == 1) ? "DOWN" : "DOUBLE");
            wrefresh(main_win);
            pthread_mutex_unlock(&mode_lock);
        }
    }
}
```

创建一个单独的线程来处理用户的输入，并实时更新模式

使用全局变量`mode`存储当前模式

服务器端在检测到模式切换以后，通过主信道将`mode_change`指令发送给所有客户端

客户端在主信道中通过`recv()`函数监听模式切换指令，根据指令调整收发行为

不额外创建信道 更加简洁高效

**每次执行切换mode指令之后，需要根据模式只显示相应的带宽信息**

```c
// 根据模式只显示相应的带宽信息
            if (mode == 0) {  // 仅 UP 模式
                mvwprintw(main_win, rank + 10, 1, "| [%2d] | %s |  %d | %8.2f Mbps |      N/A      |", rank, clients[i].ip, clients[i].port, up_bandwidth_mbps);
            } else if (mode == 1) {  // 仅 DOWN 模式
                mvwprintw(main_win, rank + 10, 1, "| [%2d] | %s |  %d |      N/A      | %8.2f Mbps |", rank, clients[i].ip, clients[i].port, down_bandwidth_mbps);
            } else if (mode == 2) {  // DOUBLE 模式
                mvwprintw(main_win, rank + 10, 1, "| [%2d] | %s |  %d | %8.2f Mbps | %8.2f Mbps |", rank, clients[i].ip, clients[i].port, up_bandwidth_mbps, down_bandwidth_mbps);
            }
```

切换模式时，重置所有客户端的字节计数，避免错误显示

```c
for (size_t i = 0; i < MAX_CLIENTS; i++) {
                if (clients[i].is_active) {
                    pthread_mutex_lock(&clients[i].lock);
                    clients[i].total_bytes_up = 0;
                    clients[i].total_bytes_down = 0;
                    gettimeofday(&clients[i].start, NULL);  // 重置起始时间
                    pthread_mutex_unlock(&clients[i].lock);
                }
            }
```

![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1726629982187-56ad412b-7b2a-482f-9e9e-0d2708c91cf4.png)

### 目前的一些问题：

#### 使用tcp连接时，监听带宽限制的线程会阻塞主线程中的数据接收

解决：配置控制信道`control_sockfd`，监听5203端口，建立控制信道的连接，在限制线程的`listen_for_bandwidth_limit`中监听控制信道，从而避免监听带宽限制线程阻塞主线程的数据接收

在服务器添加一个控制信道，负责处理带宽限制

这里好像直接在主信道（测速信道）收发限速值也没什么问题

可以使用信令将界面获取到的限速值和切换模式一起发给客户端，通知客户端进行对应的收发操作

#### 上行带宽能被正确限制到输入的限速值，而下行带宽没被限速

原因：服务器端的`send()`函数不知道客户端已经限速，由于`TCP`协议的缓冲机制和流量控制机制，数据在TCP的发送缓冲区中积累，`send()`函数会持续成功返回字节数，虽然客户端没有及时接收数据，但是服务器会不断累加已经发送的字节数。

解决：服务端监控发送的字节数和实际时间，根据设定的带宽限制来判断是否需要暂停发送，如果超出限制，则让服务器`sleep()`一段时间。

#### `Enter new bandwidth limit:`的光标输入位置在界面最后一行而不是冒号后面

由于定时器每秒刷新 计算带宽 重新刷新界面  导致光标总是被刷新到最后一行 这里每次刷新之后重定位光标位置即可

```c
// 恢复光标位置
    wmove(main_win, 7, 35);  // 恢复光标位置
    wrefresh(main_win);
```

#### 客户端启动时会默认限制到`10Mbps`

由于设置了初始默认值，可以将这个值默认设置10Gbps，这样带宽还是会正常跑满，也可以做一个判断，如果默认值是0，就不执行限速逻辑。



#### UDP `double`模式下，上下行带宽一模一样



#### 上下行带宽计算逻辑混乱

正常不限速模式下 带宽跑满几秒之后会突然减半，怀疑是测速逻辑的问题



#### 虚拟机测试需要改变有线接口名称和广播地址

```c
// 获取 有线网络接口 enp2s0 ip地址 // ens33 is the cy'virtual machine wired interface
    if (get_interface_ip("ens33", server_ip, sizeof(server_ip)) != 0) {
        fprintf(stderr, "Failed to get IP address for enp2s0\n");
        close(server_fd);
        return NULL;
    }
memset(&broadcast_addr, 0, sizeof(broadcast_addr));
    broadcast_addr.sin_family = AF_INET;
    broadcast_addr.sin_port = htons(5202); // 服务端监听广播的端口
    broadcast_addr.sin_addr.s_addr = inet_addr("192.168.92.255");
```