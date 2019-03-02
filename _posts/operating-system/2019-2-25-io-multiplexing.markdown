---
layout: post
title: "io multiplexing"
date: 2019-02-25 21:30:00 +0800
categories: operating-system
---

i/o multiplexing 技术使得程序能同时监听多个文件描述符, 对提高程序的性能提高非常重要. 通常网络在下列情况下需要使用 i/o 复用技术

* 客户端程序要同时处理多个 socket. 如非阻塞 connect 技术

* 客户端程序要同时处理用户输入和网络链接. 如聊天室程序

* tcp 服务器要同时处理监听 socket 和链接 socket. 这是 i/o 复用使用最多的场合

* 服务器要同时处理 tcp 和 udp 请求. 如回射服务器

* 服务器要同时监听多个端口, 或者处理多种服务. 如 xinetd 服务器

**i/o 服务器虽然能同时监听多个文件描述符, 但它本事是阻塞的. 并且当多个文件描述符同时就绪时, 如果不采取额外的措施, 程序就只能按顺序依次处理其中的每个文件描述符. 如果要实现并发, 只能使用多进程或多线程等编程手段**

Linux 下实现 i/o 复用的系统调用主要有 select, poll 和 epoll


##### select 系统调用
select 系统调用的用途是, 在一段时间内, 监听用户感兴趣的文件描述符上的可读, 可写和异常等事件

``` c++
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
// nfds 参数指定被监听的文件描述符的总数. 通常被设定为 slect 监听的所有文件描述符中的最大值加 1, 因为文件描述符是从 0 开始计数的
// readfds, writefds 和 exceptfds 参数分别指向可读, 可写和异常等事件对应的文件描述符集合. 应用程序调用 select 函数时, 通过这 3 个参数传入自己感兴趣的文件描述符. select 调用返回时, 内核将修改它们来通知应用程序哪些文件描述符已经就绪
// timeout 参数用来设置 select 函数的超时时间. 它是一个 timeval 结构类型的指针. 内核将通过它告诉应用程序 select 等待了多久. 但是不能完全信任 select 调用后返回的 timeout 值, 因为调用失败时 timeout 值是不确定的

// fd_set 结构体定义
#include <bits/types.h>
#define __FD_SETSIZE 1024//限定 fd_set 能容纳的文件描述符数量

#include <sys/select.h>
#define FD_SETSIZE __FD_SETSIZE
typedef long int __fd_mask;
#undef __NFDBITS
#define __NFDBITS (8 * (int)sizeof(__fd_mask))

typedef struct {
#ifdef __USE_XOPEN
	__fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->fds_bits)
#else
	__fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->__fds_bits)
#endif
} fd_set;

//tiemval 结构体的定义
struct timeval {
    long tv_sec; //秒数
    long tv_usec; // 微秒数
}
// 若 tv_sec 和 tv_usec 都为 0, 则 select 将立即返回
// 若 timeout 为 NULL, 则 select 将一直阻塞直到某个文件描述符就绪
```

select 成功时返回就绪(可读, 可写和异常)文件描述符的总数. 如果在超时时间内没有任何文件描述符就绪, select 返回 0. select 失败返回 -1 并设置 errno. 如果在 select 等待期间, 程序接收到信号, 则 select 立即返回 -1 并设置 errno 为 EINTR

由于位操作过于繁琐，可以使用一些宏来访问 fd_set 结构体中的位
``` c++
#include <sys/select.h>
FD_ZERO(fd_set *fdset); // 清除 fdset 的所有位
FD_SET(int fd, fd_set *fdset); // 设置 fdset 的位 fd
FD_CLR(int fd, fd_set *fdset); // 清除 fdset 的位 fd
int FD_ISSET(int fd, fd_set *fdset); // 测试 fdset 的位 fd 是否被设置
```

**select 文件描述符就绪条件**

在网络编程中，下列情况下 socket 可读

* socket 内核接收缓存区中的字节数大于或等于其低水位标记 SO_RCVLOWAT. 此时可以无阻塞地读取该 socket, 并且读取操作返回的字节数大于 0

* socket 通信的对方关闭连接．此时对该 socket 的读操作将返回 0

* 监听 socket 上有新的连接请求

* socket 上有未处理的错误．此时可以通过 getsockopt 来读取和清除该错误

下列 socket 可写

* socket 内核发送缓存区中的可用字节数大于或等于其低水位标记 SO_SNDLOWAT

* socket 的写操作被关闭．对写操作被关闭的 socket 执行写操作将触发一个 SIGPIPE 信号

* socket 使用非阻塞 connect 连接成功或失败之后

* socket 上有未处理的错误

网络程序中，select 能处理的异常情况只有一种，socket 上接收到带外数据

**处理带外数据**

从上面可以发现，socket 上接收到普通数据和带外数据将使 select 返回，但 socket 处于不同的就绪状态．前者处于可读状态，后者处于异常状态．下面代码描述了 select 如何同时处理二者

``` c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <netdb.h>

int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address prot_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	int ret = 0;
	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int listenfd = socket(PF_INET, SOCK_STREAM, 0);
	assert(listenfd >= 0);

	ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(listenfd, 5);
	assert(ret != -1);

	struct sockaddr_in client_address;
	socklen_t client_addrlength = sizeof(client_address);
	//在 accept 前没有使用 io 复用，所以只能从 listen 内核队列中读取一条连接
	int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);

	if(connfd < 0) {
		printf("errno is: %s\n", gai_strerror(errno));
	}

	char buf[1024];
	fd_set read_fds;
	fd_set exception_fds;
	FD_ZERO(&read_fds);
	FD_ZERO(&exception_fds);

	while(1) {
		memset(buf, '\0', sizeof(buf));

		//read_fds 和 exception_fds 在 select 中会被内核重置．所以每次调用 select 前都要重新在 read_fds 和 exception_fds 中设置文件描述符 connfd
		FD_SET(connfd, &read_fds);
		FD_SET(connfd, &exception_fds);
		ret = select(connfd + 1, &read_fds, NULL, &exception_fds, NULL);

		if(ret < 0) {
			printf("selection failure\n");
			break;
		}

		//对于可读事件，采用普通的 recv 函数读取数据
		if(FD_ISSET(connfd, &read_fds)) {
			ret = recv(connfd, buf, sizeof(buf) - 1, 0);
			if(ret <= 0) {
				break;
			}
			printf("get %d bytes of normal data: %s\n", ret, buf);
		} else if (FD_ISSET(connfd, &exception_fds)) {
			//对于异常事件(带外数据)采用带 MSG_OOB 标志的 recv 函数读取数据
			ret = recv(connfd, buf, sizeof(buf) - 1, MSG_OOB);
			if(ret <= 0) {
				break;
			}
			printf("get %d bytes of oob data: %s\n", ret, buf);
		}
	}

	close(connfd);
	close(listenfd);

	return 0;
}
```


##### poll 系统调用
poll 系统调用和 select 类似，也是在指定时间内轮询一定数量的文件描述符，以测其中是否有就绪者
``` c++
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
// fds 参数是一个 pollfd 结构体类型的数组，指定文件描述符上发生的可读，可写和异常等时间

//pollfd 结构体
struct pollfd {
	int fd; //文件描述符
	short events; //注册的事件
	short revents; //实际发生的时间，由内核填充
}
// 其中，fd 成员指定文件描述符．events 成员告诉 poll 监听 fd 上的哪些事件，它是一系列时间的安位或．revents 成员由内核修改，以通知应用程序 fd 上实际发生了哪些事件

// nfds 参数指定被监听事件集合 fds 的大小
// typedef unsigned long int nfds_t;

// timeout 参数指定 poll 的超时值，单位是毫秒．当 timeout 为 -1 时，poll 将阻塞到某个事件发生，timeout 为 0 时 poll 调用立即返回

// poll 系统调用返回值的含义与 select 相同
```

poll 事件类型

![poll 事件类型](/images/poll_type_1.png)
![poll 事件类型](/images/poll_type_2.png)


##### epoll 系列系统调用
**内核事件表**

epoll 是 linux 特有的 I/O 复用函数, 在实现上与 select 和 poll 有很大差异

* epoll 使用一组函数来完成任务，而不是单个函数

* epoll 把用户关心的文件描述符上的事件放在内核里的一个事件表总，无需像 select 和 poll 那样每次调用都要重复传入文件描述符集或事件集

* epoll 需要使用一个额外的文件描述符来唯一标识内核中的这个事件表．这个文件描述符使用 ***int epoll_create(int size)*** 函数来创建．其中 size 参数用来提示内核事件表需要多大．该函数返回的文件描述符将作为其他所有 epoll 系统调用的第一个参数，以指定要访问的内核事件表

epoll_ctl 函数用来操作内核事件表
``` c++
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// fd 参数是要操作的文件描述符, op 参数则指定操作类型(3种)
// EPOLL_CTL_ADD 往事件表上注册 fd 上的事件
// EPOLL_CTL_MOD 修改 fd 上的注册事件
// EPOLL_CTL_DEL 删除 fd 上的注册事件

// event 参数指定事件，是 epoll_event 结构体指针类型．epoll_event 定义如下
strcut epoll_event {
	__uint32_t events; //epoll 事件类型, 和 poll 的事件类型基本相同，标示 epoll 事件的宏是在对应 poll 事件宏前加上 "E"
	epoll_data_t data; //用户数据
};
// epoll 有两个额外的事件类型, EPOLLET 和 EPOLLONESHOT

// data 成员用于存储用户数据，其类型 epoll_data_t 的定义如下
typedef union epoll_data{
	void *ptr;
	int fd;
	uint32_t u32;
	uint64_t u64;
}epoll_data_t;

//epoll_ctl 成功返回 0，失败返回 -1 并设置 errno
```

**epoll_wait 函数**

epoll_wait 在一段超时时间内等待一组文件描述符上的事件

``` c++
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
// 成功时返回就绪文件描述符的个数, 失败返回 -1 并设置 errno
// timeout 参数的含义和 poll 接口的 timeout 参数相同
// maxevents 参数指定最多监听多少个事件, 其值必须大于 0
// epoll_wait 函数如果检测到事件, 就将所有就绪的事件从内核事件表(由 epfd 参数指定) 中复制到它的第二个参数 events 指向的数组中
```

poll 和 epoll 在使用上的差别
``` c++
int ret = poll(fds, MAX_EVENT_NUMBER, -1);
int count = 0;

for(int i = 0; i < MAX_EVENT_NUMBER; ++i) {
	if(count >= ret) break;
	if(fds[i].revents & POLLIN) {
		++count;
		int sockfd = fds[i].fd;
		// 处理 sockfd
	}
}


int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);

for(int i = 0; i < ret; i++) {
	int sockfd = events[i].data.fd;
	// 处理 sockfd
}
```

**LT 和 ET 模式**

epoll 对文件描述符的操作有两种模式: lt(电频触发)模式和 et(边沿触发)模式. lt 是默认的工作模式, 这种模式下 epoll 相当于一个效率较高的 poll. 当往 epoll 内核事件表中注册一个文件描述符上的 EPOLLET 事件时, epoll 将以 et 模式来操作该文件描述符. et 模式是 epoll 的高效工作模式

对于采用 lt 工作模式的文件描述符, 当 epoll_wait 检测二到其上有事件发生并将此事件通知应用程序后, 应用程序可以不立即处理该事件. 当应用程序下一次调用 epoll_wait 时, epoll_wait 还会再次响应应用程序通告此事件, 直到该事件被处理. 而对于采用 et 工作模式的文件描述符, 当 epoll_wait 检测到其上有事件发生并将此事件通知应用程序后, 应用程序必须立即处理该事件, 因为后续的 epoll_wait 调用将不再通知该事件. 显然 et 模式在很大程度上降低了同一个 epoll 事件被重复触发的次数, 因此效率要比 lt 模式高

``` c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 10

// 打印客户端信息
void print_sockaddr_in(struct sockaddr_in *address) {
	printf("a new socket client add, the infomation of this client is:\n");
	printf("sin_addr : %s\n", inet_ntoa(address->sin_addr));
	printf("sin_port : %d\n", address->sin_port);
	printf("sin_family : %d\n", address->sin_family);
	printf("==========================================================\n");
}


// 将文件描述符 fd 设置为非阻塞的
int setnonblocking(int fd) {
	// 获取 fd 当前的属性
	int old_option = fcntl(fd, F_GETFL);
	// 新属性为 old 属性加上 O_NONBLOCK 属性
	int new_option = old_option | O_NONBLOCK;
	
	// 设置 fd 的属性为新属性
	fcntl(fd, F_SETFL, new_option);

	// 返回 old 属性
	return old_option;
}


// 将文件描述符 fd 上的 EPOLLIN 事件注册到内核事件表 epollfd 上. 参数 enable_et 指定是否对 fd 开启 et 模式
void addfd(int epollfd, int fd, bool enable_et) {
	// event 为要注册的事件
	epoll_event event;

	// event.data.fd 指定事件 event 所属的文件描述符
	event.data.fd = fd;
	// event.events 表事件类型, EPOLLIN 即 epoll 的可读事件
	event.events = EPOLLIN;

	// 开启 et 模式
	if(enable_et) {
		event.events |= EPOLLET;
	}
	
	// 往 epollfd 内核事件表上注册 fd 上的 event 事件
	epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
	// 设置 fd 为非阻塞的 (io multiplexing 上的单个事件是非阻塞的)
	setnonblocking(fd);
}


// 处理 lt 模式下的就绪事件
void lt(epoll_event *events, int number, int epollfd, int listenfd) {
	char buf[BUFFER_SIZE];

	for(int i = 0; i < number; i++) {
		// 就绪事件 event[i] 所属的 socket 文件描述符
		int sockfd = events[i].data.fd;

		// 若 sockfd 为原始的 socket 文件描述符(listen 调用返回的那个 socket 文件描述符), 即有新的客户端链入
		if(sockfd == listenfd) {
			struct sockaddr_in client_address;
			socklen_t client_addrlength = sizeof(client_address);

			// accept 一下新加入的 socket 连接
			int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
			// 将新加入的 socket 连接注册到 epollfd 内核事件表, 不开启 et 模式
			addfd(epollfd, connfd, false);
			
			// 打印新加入的 client 的信息
			print_sockaddr_in(&client_address);
			
		} else if(events[i].events & EPOLLIN) {
			// 只要 socket 读缓存中还有未读取的数据该段代码就会被触发
			printf("event trigger once--------------------------------------\n");
			memset(buf, '\0', BUFFER_SIZE);
			int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);

			if(ret <= 0) {
				close(sockfd);
				continue;
			}
			printf("get %d bytes of content: %s\n", ret, buf);
		} else {
			printf("something else happened\n");
		}
	}
}


// 处理 et 模式下的就绪事件
void et(epoll_event *events, int number, int epollfd, int listenfd) {
	char buf[BUFFER_SIZE];
	for(int i = 0; i < number; ++i) {
		// 就绪事件 event[i] 所属的 socket 文件描述符
		int sockfd = events[i].data.fd;

		// 若 sockfd 为原始的 socket 文件描述符(listen 调用返回的那个 socket 文件描述符), 即有新的客户端链入
		if(sockfd == listenfd) {
			struct sockaddr_in client_address;
			socklen_t client_addrlength = sizeof(client_address);
			
			// accept 一下新加入的 socket 连接
			int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
			// 将新加入的 socket 连接注册到 epollfd 内核事件表, 且开启 et 模式
			addfd(epollfd, connfd, true);
			
			// 打印新加入的 client 的信息
			print_sockaddr_in(&client_address);

		} else if(events[i].events & EPOLLIN) { // 当前 socket 连接上有可读事件
			printf("event trigger once--------------------------------------\n");
			// et 事件不会被重复触发, 需要循环读取数据, 确保把当前 socket 读缓存中的所有数据读出
			while(1) {
				memset(buf, '\0', BUFFER_SIZE);
				int ret = recv(sockfd, buf, BUFFER_SIZE-1, 0);
				if(ret < 0) {
					if((errno == EAGAIN) || (errno == EWOULDBLOCK)) {
						printf("raed later\n");
						break;
					}
					close(sockfd);
					break;
				} else if(ret == 0) {
					close(sockfd);
				} else {
					printf("get %d bytes of content: %s\n", ret, buf);
				}
			}
		} else {
			printf("something else happened\n");
		}
	}
}


int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	int ret = 0;
	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int listenfd = socket(PF_INET, SOCK_STREAM, 0);
	assert(listenfd >= 0);

	ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(listenfd, 5);
	assert(ret != -1);

	epoll_event events[MAX_EVENT_NUMBER];
	int epollfd = epoll_create(5); //创建内核事件注册表文件描述符
	assert(epollfd != -1);

	// 将 listenfd socket 文件描述符上的 EPOLLIN 事件注册到内核事件表 epollfd 中, 且对 listenfd 文件描述符使用 et 模式
	addfd(epollfd, listenfd, true);

	while(1) {
		// 阻塞到 epollfd 内核事件表上有就绪事件
		int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
		if(ret < 0) {
			printf("epoll failure\n");
			break;
		}
		// 处理 lt 模式下的就绪事件
		lt(events, ret, epollfd, listenfd);
		// 处理 et 模式下的就绪事件
		// et(events, ret, epollfd, listenfd);
	}

	close(listenfd);

	return 0;
}
```

通过上面代码很容易发现 et 模式下事件被触发的次数要比 lt 要少的多(输入规模要大于 10 byte)

**EPOLLONESHOT 事件**

即使使用 et 模式, 一个 socket 上的某个事件还是可能被触发多次. 这在并发程序中显然是不行的. 因为 et 模式需要循环读取，但是在读取过程中，如果有新的数据可读(EPOLLIN 再次被触发), 此时另一个线程被唤醒来处理这些数据. 出现了两个线程同时操作一个 socket 的局面. 这会出现资源竞争和处理事件时序问题, 显然不是我们期待的. 可以通过 epoll 的 EPOLLONESHOT 事件解决

对于注册了 EPOLLONESHOT 事件的文件描述符, 操作系统最多触发其上注册的一个可读, 可写或异常事件. 这样, 当一个线程处理某个 socket 时, 其他线程是没有机会操作该 socket 的 . 实际上是每次触发事件之后，就将事件注册从fd上清除了. 因此注册了 EPOLLONESHOT 事件的 socket 一旦被某个线程处理完毕, 该线程需要立即重置这个 socket 上的 EPOLLONESHOT 事件以确保这个 socket 下一次可读时, 其上的 EPOLLIN 事件能被其他进程处理

``` c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>

#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 1024


struct fds {
	int epollfd;
	int sockfd;
};


// 设置 fd 为非阻塞
int setnonblocking(int fd) {
	int old_option = fcntl(fd, F_GETFL);
	int new_option = old_option | O_NONBLOCK;
	
	fcntl(fd, F_SETFL, new_option);

	return old_option;
}


// 将 fd 上的 EPOLLIN 和 EPOLLET 事件注册到 epollfd 内核事件表中
// 参数 oneshot 指定是否注册 fd 上的 EPOLLONESHOT 事件
void addfd(int epollfd, int fd, bool oneshot) {
	epoll_event event;
	event.data.fd = fd;
	event.events = EPOLLIN | EPOLLET;
	if(oneshot) {
		event.events |= EPOLLONESHOT;
	}

	epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);

	setnonblocking(fd);
}


// 重置 socket 上的 EPOLLONESHOT 事件
void reset_oneshot(int epollfd, int fd) {
	epoll_event event;
	event.data.fd = fd;
	event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;

	epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}


void* worker(void *arg) {
	int sockfd = ((fds*)arg)->sockfd;
	int epollfd = ((fds*)arg)->epollfd;
	printf("start new thread to receive data on fd: %d\n", sockfd);

	char buf[BUFFER_SIZE];
	memset(buf, '\0', BUFFER_SIZE);

	while(1) {
		int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);

		if(ret == 0) {
			close(sockfd);
			printf("foreiner closed the connection\n");
			break;

		} else if (ret < 0) {
			if(errno == EAGAIN) {
				// 重置 sockfd 上的注册事件
				reset_oneshot(epollfd, sockfd);
				printf("read later\n");
				break;
			}

		} else {
			printf("get content: %s\n", buf);
			sleep(5);
		}
	}
	printf("end thread receiving data on fd: %d\n", sockfd);
}


int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	int ret = 0;
	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int listenfd = socket(PF_INET, SOCK_STREAM, 0);
	assert(listenfd >= 0);

	ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(listenfd, 5);
	assert(ret != -1);

	epoll_event events[MAX_EVENT_NUMBER];
	int epollfd = epoll_create(5);
	assert(epollfd != -1);
	
	// 监听 socket listenfd 上不能注册 EPOLLONESHOT 事件, 否则应用程序只能处理一个客户连接, 后续的客户连接请求都不再触发 listenfd 上的 EPOLLIN 事件
	addfd(epollfd, listenfd, false);

	while(1) {
		int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
		if(ret < 0) {
			printf("epoll failure\n");
			break;
		}

		for(int i = 0; i < ret; ++i) {
			int sockfd = events[i].data.fd;

			if(sockfd == listenfd) {
				struct sockaddr_in client_address;
				socklen_t client_addrlength = sizeof(client_address);

				int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);

				addfd(epollfd, connfd, true);

			} else if(events[i].events & EPOLLIN) {
				pthread_t thread;
				fds fds_for_new_worker;
				fds_for_new_worker.epollfd = epollfd;
				fds_for_new_worker.sockfd = sockfd;

				pthread_create(&thread, NULL, worker, (void*)&fds_for_new_worker);

			} else {
				printf("something else happened\n");
			}
		}
	}

	return 0;
}
```


###### 三组 i/o 复用函数的比较
![diff io multiplexing](/images/diff_io_multiplexing.png)

注意，当活动连接比较多的时候，epoll_wait 的效率未必比 select 和 poll 高，因为此时回调函数触发得过于频繁．所以 epoll_wait 适用于连接数量多，但活动连接较少的情况
