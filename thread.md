### 进程

#### 进程与线程

**进程：** 一个在内存中运行的应用程序,例如`cpp.exe`，每个进程都有自己的独立内存空间，一个进程可以有多个线程。

**线程：** 进程的一个执行单元，一个进程至少有一个线程，多个线程可以共享进程的堆和方法区资源。

进程是资源分配的最小单位，线程是操作系统调度执行的最小单位。

#### 线程函数

```c
pthread_t pthread_self(void);	// 返回当前线程的线程ID

#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg);
// Compile and link with -pthread, 线程库的名字叫pthread, 全名: libpthread.so libptread.a
```

#### 创建线程

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

void *working(void *arg)
{
	printf("this is child_thread, thread id is %ld\n", pthread_self());
	for (int i = 0; i < 10; i++)
	{
		printf("child is running %d\n", i);
	}
	return NULL;
}

int main()
{
	pthread_t tid;
	pthread_create(&tid, NULL, working, NULL);
	printf("child thread created, thread id is %ld\n", tid);
	printf("main thread is running id is %ld\n", pthread_self());

	for (int i = 0; i < 3; i++)
	{
		printf("i = %d\n", i);
	}

	// sleep(1);
	return 0;
}
```

编译

```sh
gcc pthread_create.c -o pthread_create -lpthread
```

结果

```sh
bhhh@bhcomputer:~/network-program/thread$ ./pthread_create 
child thread created, thread id is 140308364510976
main thread is running id is 140308364515136
i = 0
i = 1
i = 2
```

分析：没有人为干预的情况下，子线程的生命周期与主线程一致，主线程退出虚拟地址空间被释放，子线程会被销毁。可以在主线程中挂起`sleep()`让子线程执行完毕主线程再退出。

#### 线程退出

```c
#include <pthread.h>
void pthread_exit(void *retval);
```

调用该函数，当前线程马上退出，不会影响其他线程正常运行

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

void *working(void *arg)
{
	printf("this is child_thread, thread id is %ld\n", pthread_self());
	for (int i = 0; i < 10; i++)
	{
		if (i == 6)
		{
			pthread_exit(NULL);
		}
		printf("child is running %d\n", i);
	}
	return NULL;
}

int main()
{
	pthread_t tid;
	pthread_create(&tid, NULL, working, NULL);
	printf("child thread created, thread id is %ld\n", tid);
	printf("main thread is running id is %ld\n", pthread_self());

	for (int i = 0; i < 3; i++)
	{
		printf("i = %d\n", i);
	}

	pthread_exit(NULL);
	return 0;
}
```

#### 线程回收

```c
#include <pthread.h>
// 这是一个阻塞函数, 子线程在运行这个函数就阻塞
// 子线程退出, 函数解除阻塞, 回收对应的子线程资源, 类似于回收进程使用的函数 wait()
int pthread_join(pthread_t thread, void **retval);
```

参数:
thread: 要被回收的子线程的线程ID

retval: 二级指针, 指向一级指针的地址, 是一个传出参数, 这个地址中存储了pthread_exit() 传递出的数据，如果不需要这个参数，可以指定为NULL

返回值：线程回收成功返回0，回收失败返回错误号。

**使用全局变量进行线程回收:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

// 定义结构
struct Persion
{
	int id;
	char name[36];
	int age;
};

struct Persion p; // 定义全局变量

// 子线程的处理代码
void *working(void *arg)
{
	printf("我是子线程, 线程ID: %ld\n", pthread_self());
	for (int i = 0; i < 9; ++i)
	{
		printf("child == i: = %d\n", i);
		if (i == 6)
		{
			// 使用全局变量
			p.age = 12;
			strcpy(p.name, "tom");
			p.id = 100;
			// 该函数的参数将这个地址传递给了主线程的pthread_join()
			pthread_exit(&p);
		}
	}
	return NULL;
}

int main()
{
	// 1. 创建一个子线程
	pthread_t tid;
	pthread_create(&tid, NULL, working, NULL);

	printf("子线程创建成功, 线程ID: %ld\n", tid);
	// 2. 子线程不会执行下边的代码, 主线程执行
	printf("我是主线程, 线程ID: %ld\n", pthread_self());
	for (int i = 0; i < 3; ++i)
	{
		printf("i = %d\n", i);
	}

	// 阻塞等待子线程退出
	void *ptr = NULL;
	// ptr是一个传出参数, 在函数内部让这个指针指向一块有效内存
	// 这个内存地址就是pthread_exit() 参数指向的内存
	pthread_join(tid, &ptr);
	// 打印信息
	struct Persion *pp = (struct Persion *)ptr;
	printf("name: %s, age: %d, id: %d\n", pp->name, pp->age, pp->id);
	printf("子线程资源被成功回收...\n");

	return 0;
}
```

#### 线程分离

```c
#include <pthread.h>
// 参数是子线程的线程ID, 主线程就可以和这个子线程分离了
int pthread_detach(pthread_t thread);
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <pthread.h>

// 子线程的处理代码
void* working(void* arg)
{
    printf("我是子线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<9; ++i)
    {
        printf("child == i: = %d\n", i);
    }
    return NULL;
}

int main()
{
    // 1. 创建一个子线程
    pthread_t tid;
    pthread_create(&tid, NULL, working, NULL);

    printf("子线程创建成功, 线程ID: %ld\n", tid);
    // 2. 子线程不会执行下边的代码, 主线程执行
    printf("我是主线程, 线程ID: %ld\n", pthread_self());
    for(int i=0; i<3; ++i)
    {
        printf("i = %d\n", i);
    }

    // 设置子线程和主线程分离
    pthread_detach(tid);

    // 让主线程自己退出即可
    pthread_exit(NULL);
    
    return 0;
}
```

#### 其他线程函数

线程取消

```c
#include <pthread.h>
// 参数是子线程的线程ID
int pthread_cancel(pthread_t thread);
```

线程ID比较

```c
#include <pthread.h>
int pthread_equal(pthread_t t1, pthread_t t2);
```

### IO多路复用

使用IO多路复用函数委托内核检测服务器端所有的文件描述符，如果检测到已就绪的文件描述符就阻塞解除，将这些已就绪的文件描述符传出
