---
layout: post
title: "basic api of network programming"
date: 2019-02-01 15:00:00 +0800
categories: network
---

##### socket 地址 api

**主机字节序和网络字节序**

现代 cpu 每次都装载至少 4 字节(32位机器)。那么这 4 字节在内存中的排列顺序将影响它被累加器装载成的整数的值。即字节序问题。字节序分为大端字节序和小端字节序。大端字节序是指一个整数的高位字节(23~31bit)存储在内存的低地址，低位字节(0~7bit)存储在内存的高地址处。小端字节序则相反。可以用下面的代码来判断本机内存是大端序号还是小端序。本机内存为小端字节序(大部分 system 都是小端字节序)。64 位机器类似

``` c++
#include <stdio.h>

void byteorder() {
	union {
		short value;
		char union_byte[sizeof(short)];
	} test;

	test.value = 0x0102;

	// value 和 union_byte 共享内存，union_byte 的两个元素分别为 01 或 02
	// 由于 test 分配的栈内存，具有 fifo 的特性，下标小的元素位于低地址，下标大的元素位于高地址
	// 在整数 0102 中 01 为高位，02 为低位。因此若下标 0(低内存地址) 处为元素 1(整数高位)，下标 1(高内存地址) 处为元素 2(整数低位)，则为大端字节序。反之则为小端字节序 

	// printf("%d %d\n", test.union_byte[0], test.union_byte[1]);

	if(test.union_byte[0] == 1 && test.union_byte[1] == 2) {
		printf("big endian\n");
	} else if(test.union_byte[0] == 2 && test.union_byte[1] == 1) {
		printf("little endian\n");
	} else {
		printf("unknow\n");
	}
}

int main() {
	//64 bit operating system
	printf("%d\n", sizeof(char)); //1

	printf("%d\n", sizeof(short));//2

	printf("%d\n", sizeof(int));//4

	printf("%d\n", sizeof(long));//8

	printf("%d\n", sizeof(long long));//8

	printf("%d\n", sizeof(int*));//8

	byteorder();
}
```
对于两台主机或者用不同语言写的两个进程通信都可能需要进行字节序转换


**通用 socket 地址**

在 socket 网络编程中 ip + port 为通用 socket 地址，socket 网络编程接口中表示 socket 地址的数据结构是 sockaddr，其定义在 sys/socket.h 中

``` c++
struct sockaddr {
	sa_family_t sa_family; // sa_family_t 表地址族类型，与协议族类型对应且具有相同的值
	char sa_data[14]; // socket 地址值
};
```

地址族和协议族的关系

<img src="/images/sa_family_t.png" width="480" height="95" />

宏 PF× 和 AF× 都定义在 sys/socket.h 头文件中，且后者与前者有完全相同的值，可以混用

sa_data 成员用于存放 socket 地址的值。不同的地址族具有不同的含义和长度

<img src="/images/sa_data.png" width="480" height="130" />

显然，14 字节的 sa_data 无法容纳多数协议族的地址值。因此 linux 又定义了新的通用 socket 地址(同样定义在 sys/socket.h 中)

``` c++
struct sockaddr_storage {
	sa_family_t sa_family; // 2 byte
	unsigned long int __ss__align; // 8 byte
	char __ss_padding[128-sizeof(__ss__align)]; // 120 byte
}
```

**专用 socket 地址**

上面两个通用 socket 地址结构体显然不怎么好用，比如设置与获取 ip 和 port 就要执行繁琐的位操作。所以 linux 给各个协议族提供了专门的 socket 地址结构

unix 本机域协议族专用 socket 地址结构，定义在 sys/un.h 文件中
``` c++
struct sockaddr_un {
	sa_family_t sin_family; // AF_UNIX
	char sun_path[108]; // file path
};
```

ipv4 专用 socket 地址结构
``` c++
struct in_addr {
	u_int32_t s_addr; // ipv4 地址，要用网络字节序
};

struct sockaddr_in {
	sa_family_t sin_family; // AF_INET
	u_int16_t sin_port; // port, 要用网络字节序
	struct in_addr sin_addr; // ipv4 地址结构体
};
```

ipv6 专用 socket 地址结构
``` c++
struct in_addr {
	unsigned char sa_addr[16];//ipv6 地址，要用网络字节序
};

struct sockaddr_in6 {
	sa_family_t sin6_family; // AF_INET6
	u_int16_t sin6_port; // port, 要用网络字节序
	u_int32_t sin6_flowinfo; // 信息流，应设置为 0
	struct in_addr sin_addr; // ipv6 地址结构体
	u_int32_t sin6_scope_id; // scope id，处于试验阶段
};
```

**ip 地址转换函数**

在编程时需要将 ip 地址转化为二进制才能用，而记录日志时为了方便阅读，需要将 ip 地址转换为点分十进制的字符串。下列的是 ipv4 地址转换函数

``` c++
#include <arpa/inet.h>

//将点分十进制表示的 ipv4 地址转化为二进制的网络字节序 ipv4 地址，失败时返回 INADDR_NONE
in_addr_t inet_addr(const char* strptr);

//功能同上，但是将转化结果存储到 inp 指向的地址。成功返回 1 失败返回 0
int inet_aton(const char* cp, struct in_add* inp);

//将网络字节序的 ipv4 转换为点分十进制的字符串
//注意，inet_ntoa 内部使用的 static 变量保持转化后的结果，所以该函数就有不可重入性
char* inet_ntoa(struct in_addr in);
```

ipv4 和 ipv6 通用转换函数
``` c++
//将点分十进制的 ipv4 或 ipv6 地址转化为网络字节序的整数地址存储到 dst 指向的内存中
//其中，af 参数指定协议族类型，值为 AF_INET 或 AF_INET6
//成功返回 1 失败返回 0 并设置 errno
int inet_pton(int af, const char* src, void* dst);

//进行和 inet_pton 相反的转换
//最后一个参数指定目标存储单元的大小，为 INET_ADDRSTRLEN(16 byte ipv4) 或 INET6_ADDRSTRLEN(46 byte ivp6)
//成功返回目标存储单元地址，失败返回 NULL 并设置 errno
const char* inet_ntop(int af, const void* src, char* dst, socklen_t cnt);
```


##### 创建 socket
在 unix/linux 中，所有东西都是文件。socket 是一个可读、可写、可控制、可关闭的文件描述符。可以通过下面的 socket 系统调用创建一个 soccket

``` c++
#include <sys/types.h>
#include <sys/socket.h>

// domain 参数表示选择哪个底层协议族。PF_INET 表 ipv4，PF_INET6 表 ipv6，PF_UNIX 表本地域协议族
// type 表服务类型。主要有 SOCK_STREAM(流服务) 和 SOCK_UGRAM(数据报)。对 tcp/ip 协议族而言，前者表传输层用 tcp 协议，后者用 udp 协议。linux 2.6.17 版本起，type 还增加了 SOCK_NONBLOCK 和 SOCK_CLOEXEC 两个选项。分别表创建非阻塞的 socket，以及 fork 调用创建子进程时在子进程中关闭该 socket 
// protocol 参数是在前两个参数构成的协议集合下，再选一个具体的协议。几乎所有情况下这个参数都设置为 0，表使用默认协议
// socket 系统调用成功返回一个 socket 文件描述符，失败则返回 -1 并设置 errno
int socket(int domain, int type, int protocol);
```


##### 命名 socket
创建 socket 时并未绑定地址，服务端的 socket 程序需要通过 bind 函数给 socket 对象绑定一个 socket 地址。该过程即 socket 命名。客户端程序通常不用命名，采用操作系统自动分配 socket 地址

``` c++
#include <sys/types.h>
#include <sys/socket.h>

// bind 将 my_addr 所指的 socket 地址分配给 sockfd 文件描述符，addrlen 参数指该 socket 地址的长度
// bind 成功时返回 0，失败则返回 -1 并设置 errno 
int bind(int sockfd, const struct sockaddr* my_addr, socklen_t addrlen);
```


##### 监听 socket
socket 命名之后，还不能马上接受客户端连接，还需要 listen 系统调用创建一个监听队列用来存放待处理的客户端连接

``` c++
// sockfd 表被监听的 socket，socklog 表内核监听队列的最大长度
// 监听的队列长度如果超过 backlog，服务器将不受理新的客户端连接，客户端也将收到 ECONNREFUSED 错误信息
// linux 2.2 之前 socket 参数指处于 SYN_RECV + ESTABLISHED 状态的连接数，2.2 之后的 socket 参数指处于 ESTABLISHED 状态的连接数
// 成功时返回 0，失败返回 -1 并设置 errno
int listen(int sockfd, int backlog);
```

一个简单的 server 监听 socket 的实例，这里并没有 accept 连接
``` c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <string.h>

static bool stop = false;

// SIGTERM 信号的处理函数，触发时结束主程序中的循环
static void handle_term(int sig) {
	printf("will be closed");
	stop = true;
}

int main(int argc, char const *argv[])
{
	// 接受 kill 信号
	signal(SIGTERM, handle_term);

	if(argc <= 3) {
		printf("usage: %s ip_address port_number backblog\n", basename(argv[0]));
		return 1;
	}

	const char* ip = argv[1];
	int port = atoi(argv[2]); // atoi 将字符串类型转换为整型
	int backblog = atoi(argv[3]);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);


	struct sockaddr_in address; // ipv4 socket 地址专用类型
	bzero(&address, sizeof(address)); // 将 address 中前 sizeof(address) 字节设置为 0
	address.sin_family = AF_INET; //选择 tcp/ip4 协议族

	// 将点分十进制的 ipv4 或 ipv6 地址转化为网络字节序的整数地址
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	// socket 命名
	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	// 监听 socket
	ret = listen(sock, backblog);

	while(!stop) {
		sleep(1);
	}

	close(sock);

	return 0;
}
```


##### 接收连接
下面系统调用从 listen 监听的队列中接受一个连接(无论当前网络状态以及连接的状态)

``` c++
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// sockfd 参数是执行过 listen 系统调用的监听 socket。addr 参数用来获取被接受连接的远端 socket 地址，该 socket 地址长度由 addrlen 参数指出
// 成功时返回一个新的 socket，该 socket 唯一的标识了被接受的这个连接。失败时返回 -1 并设置 errno
```

一个简单的 server 端 accept 样例
``` c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <assert.h>

int main(int argc, char const *argv[]) {
	
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char* ip = argv[1];
	int port = atoi(argv[2]); //字符串转数字

	struct sockaddr_in address; //tcp/ipv4 协议族专用 socket 地址类型
	bzero(&address, sizeof(address)); //清0
	address.sin_family = AF_INET; //tcp/ipv4 协议族
	inet_pton(AF_INET, ip, &address.sin_addr); //点分十进制字符串转网络字节序整数
	address.sin_port = htons(port); //整型转网络字节序

	int sock = socket(PF_INET, SOCK_STREAM, 0); //创建 socket 对象
	assert(sock >= 0);
	
	// socket 命名
	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);
	
	// socket 监听
	ret = listen(sock, 5);
	assert(ret != -1);

	sleep(20);

	struct sockaddr_in client;
	socklen_t client_addrlength = sizeof(client);
	
	// 接收 socket 连接
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);

	if (connfd < 0) {
		printf("errno is: %d\n", errno);
	} else {
		char remote[INET_ADDRSTRLEN];
		printf("connected with ip: %s and port %d\n", inet_ntop(AF_INET, &client.sin_addr, remote, client_addrlength), ntohs(client.sin_port));
		close(connfd);
	}

	close(sock);

	return 0;
}
```

注意: accept 只是从 listen 队列中取出连接，而不论连接处于何种状态(established 之外的状态也行)，也不关心网络状况的变化(在 listen 后断网，listen 中的连接仍然能被 accept)


##### 发起连接
server 端通过 listen 系统调用来被动接受连接，客户端需要通过 connect 系统调用来主动与服务器建立连接

``` c++
int connect(int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen);
// sockfd 参数由 socket 系统调用返回一个 socket。serv_addr 参数是服务器监听的 socket 地址，addrlen 参数指定这个地址的长度
// connect 成功时返回 0。一旦成功建立连接，sockfd 就唯一标识了这个连接，客户端就可以通过读写 sockfd 来与服务器通信。失败则返回 -1 并设置 errno
```


##### 关闭连接
关闭一个连接即关闭对应的 socket。可以通过 close 或者 shutdown 系统调用完成

* close
``` c++
#include <unistd.h>
int close(int fd);
// fd 参数是待关闭的 socket 
```

close 系统调用并非总是立即关闭一个连接，而是将 fd 的引用计数减一。只有当 fd 的引用计数为 0 时，才真正关闭连接。多进程程序中，一次 fork 系统调用默认将使父进程中打开的 socket 的引用计数加一，因此必须在父进程和子进程中都对 socket 执行 close 系统调用才能将连接关闭

* shutdown
``` c++
#include <sys/socket.h>
int shutdown(int sockfd, int howto);
// sockfd 参数是待关闭的 socket
// howto 参数决定了 socket 的行为
```

howto 参数列表

<img src="/images/shut_down_howto.png" width="600" height="150" />

shutdown 成功返回 0，失败返回 -1 并设置 errno


##### 数据读写
对文件的读写操作 read 和 write 同样适用于 socket。但 socket 编程接口提供了专门用于 socket 数据读写的系统调用，它们增加了对数据读写的控制

**tcp 数据读写**

``` c++
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
// recv 读取 sockfd 上的数据，buf 和 len 参数分别指定读缓冲区的位置和大小，flag 参数通常设置为 0。recv 返回实际读取到的数据长度，可能小于 len。因此通常需要多次调用 recv 才能读取到完整的数据。返回 0 表对端已经关闭了通信。出错时返回 -1 并设置 errno

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
// send 往 sockfd 上写入数据，buf 和 len 分别指定写缓冲区的位置和大小。send 成功返回实际写入的数据大小。失败则返回 -1 并设置 errno
```

flags 参数

<img src="/images/socket_flags.png" width="600" height="300" />

**udp 数据读写**

``` c++
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr* src_addr, socklen_t *addrlen);
// recvfrom 读取 sockfd 上的数据，buf 和 len 分别指定读缓冲区的位置和大小。udp 没有连接的概念，所以每次读取数据都要获取发送端的 socket 地址，即参数 src_addr 所指的内容，addrlen 所指的长度

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
// 参数意义和 recvfrom 同理
```

flags 参数和 send/recv 系统调用一致

另外需要注意的是，recvfrom/sendto 系统调用也可以用于面向连接(STREAM) 的 socket 数据读写，只需要将后两个参数设置为 NULL 即可(accept 获取了 socket 地址)

**通用数据读写函数**

``` c++
#include <sys/socket.h>

ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr *msg, int flags);
// sockfd 参数指定被操作的 socket。msg 是 msghdr 类型的指针

// msghdr 结构体的定义
struct msghdr {
	void *msg_name; // socket 地址
	socklen_t msg_namelen; // socket 地址的长度
	struct iovec *msg_iov; // 分散的内存块
	int msg_iovlen; // 分散内存块的数量
	void *msg_control; // 指向辅助数据的起始位置
	socklen_t msg_controllen; //辅助数据库的大小
	int msg_flags; // 复制函数中的 flags，并在调用过程中不断更新
};

// 对于面向连接的 tcp 协议，msg_name 成员没有意义，必须被设置为 NULL

// iovec 结构体的定义
struct iovec {
	void *iov_base; // 内存块的起始地址
	size_t iov_len; // 这块内存的长度
};
```

iovec 结构体封装了一个内存块的起始位置和长度。msg_iovlen 指定这样的 iovec 结构体对象的数量。对于 recvmsg 而言，数据将被读取并存储在 msg_iovlen 快分散的内存中，这些内存的位置和长度由 msg_iov 指向的数组指定，即分散读。对于 sendmsg 而言，msg_iovlen 块内存中的数据将被一并发送，即分散写

msg_flags 成员无需设定，它会自动复制 recvmsg/sendmsg 中的 flags 参数以影响读写过程。recvmsg 还会在调用结束前，将某些更新后的标志设置到 msg_flags 中。该成员的含义和 recv/send 中的 flags 参数一致


##### 带外标记
内核程序通知应用程序带外数据到达的两种常见方式是：I/O 复用产生的异常事件和 SIGURG 信号。recv/send 中的 flags 参数值为 MSG_OOB 时表示接受/发送带外数据。但这要求我们在使用 recv/send 接受/发送数据之前就知道下一个被读到的数据是否是带外数据才能选择合适的 flags 参数值。可以通过 sockatmark 系统调用实现这点

``` c++
#include <sys/socket.h>
int sockatmark(int sockfd);
// 若接受到的下一个数据是带外数据，返回 1，否则返回 0
```


##### 地址信息函数
在某些情况下，我们需要知道一个连接的本端或者远端 socket 地址。可以使用下列函数
``` c++
#include <sys/socket.h>

int getsockname(int sockfd, struct sockaddr *address, sockelen_t *address_len);
// 获取 sockfd 对应的本端 socket 地址，并将其存储于 address 参数指定的内存中，该 socket 地址的长度则存储于 address_len 参数指向的变量中。如果实际 socket 地址的长度大于 address 所指内存区的大小，则该 socket 地址将被截断
// 成功时返回 0，失败返回 -1 并设置 errno

int getpeername(int sockfd, struct sockaddr *address, sockelen_t *address_len);
// 获取 sockfd 对应的远端 socket 地址。参数及返回值意义同上
```


##### socket 选项
getsockopt 和 sesockopt 系统调用专门用于读取和设置 socket 文件描述符属性的方法
``` c++
#include <sys/socket.h>

int getsockopt(int sockfd, int level, int option_name, void *option_value, sockelen_t *restrict option_len);
// 获取 socket 选项信息
// restrict 关键字指定 option_len 所指对象只能被指针变量 option_len 所修改，用于帮助编译器优化汇编代码
// sockfd 参数指定被操作的目标。level 指定要操作哪个协议的选项，如 ipv4, ipv6, tcp 等。option_name 参数指定选项的名字。option_value 和 option_len 分别指定被操作选项的值和长度。不同的选项具有不同类型的值
// 成功返回 0，失败返回 -1 并设置 errno

int setsockopt(int sockfd, int level, int option_name, const void *option_value, sockelen_t option_len);
// 设置 socket 选项信息。参数及返回值含义同上
```

level 和 option_name 的对应关系

<img src="/images/option_name_1.png" width="600" height="400" />
<img src="/images/option_name_2.png" width="600" height="200" />

需要注意的是，对 server 端而言，有部分 socket 选项只能在 listen 系统调用前设置才有效。因为连接 socket 只能由 accept 调用返回，而 accept 从 listen 监听队列中接受的连接至少已经完成了 tcp 三次握手的前两个步骤(至少进入了 syn_rcvd) 状态。这说明服务器已经往被接受连接上发送出了 tcp 同步报文。但有的 socket 选项却应该在 tcp 同步报文段中设置，比如 tcp 最大报文段选项。对于这种情况，可以在 listen 钱设置这些 socket 选项，那么 accept 返回的连接 socket 将自动继承这些选项。这些选项包括：SO_DEBUG、SO_DONTROUTE、SO_KEEPALIVE、SO_LINGER、SO_OOBINLINE、SO_RCVBUF、SO_RCVLOWAT、SO_SNDBUF、SO_SNDLOWAT、TCP_MAXSEG 和 TCP_NODELAY。而对客户端而言，这些 socket 选项则应该在调用 connect 函数之前设置，因为 connect 调用成功返回之后，tcp 三次握手已完成

**SO-REUSEADDR选项**

可以通过设置 socket 选项 SO_REUSEADDR 来强制使用处于 time_wait 状态的连接占用的 socket 地址
``` c++
#include <sys/socket.h>
#include <assert.h>
#include <netinet/in.h>
#include <string.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char const *argv[]){

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int reuse = 1;
	// 设置 sock 的 SO_REUSEADDR 选项
	setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

	char *ip = "127.0.0.1\0";
	int port = 12345;

	printf("%s\n", ip);

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int ret = bind(sock, (struct sockaddr*) &address, sizeof(address));

	close(sock);
	
	return 0;
}
```
经过 setsockopt 的设置之后，即使 sock 处于 time_wait 状态，与之绑定的 socket 地址也可以被立即重用。此外，还可以通过修改内核参数 /proc/sys/net/ipv4/tcp_tw_recycle 来快速回收被关闭的 socket

**SO-RCVBUF 和 SO-SNDBUF**

SO_RCVBUF 和 SO_SNDBUF 选项分别表示 tcp 接收缓冲区和发送缓冲区的大小。不过，当使用 setsockopt 来设置 tcp 的接收缓冲区和发送缓冲区大小时，系统都会将其值加倍(如发送缓冲区设置的 2000，可能被系统设置为 4000)，并且不得小于某个最小值。tcp 接收缓冲区的最小值是 256 字节，而发送缓冲区的最小值是 2048 字节(不同系统的默认值可能不同)。系统这样做的目的，主要是确保 tcp 连接拥有足够的空闲缓冲区来处理拥塞(如，CWND 的默认值可能是 4SMSS)。当然，可以通过直接修改 /proc/sys/net/ipv4/tcp_rmem 和 /proc/sys/net/ipv4/tcp_wmem 内核参数来强制 tcp 接收缓冲区和发送缓冲区的大小没有最小值限制

server 端样例
``` c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

#define BUFFER_SIZE 1024

int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number recv_buffer_size\n", basename(argv[0]));
		return 1;
	}

	const char* ip = argv[1];
	int port = atoi(argv[2]);

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int recvbuf = atoi(argv[3]);
	int len = sizeof(recvbuf);
	setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &recvbuf, sizeof(recvbuf));
	getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &recvbuf, (socklen_t*)&len);
	printf("the tcp receive buffer size after setting is %d\n", recvbuf);

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(sock, 5);
	assert(ret != -1);

	struct sockaddr_in client;
	socklen_t clinet_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &clinet_addrlength);
	if(connfd < 0) {
		printf("errno is: %d\n", errno);
	} else {
		char buffer[BUFFER_SIZE];
		memset(buffer, '\0', BUFFER_SIZE);
		while(recv(connfd, buffer, BUFFER_SIZE-1, 0)){}
		printf("recv: %s\n", buffer);
		close(connfd);
	}

	close(sock);

	return 0;
}
```

client 端样例
``` c++
#include <sys/socket.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

#define BUFFER_SIZE 512

int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number send_bufer_size\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	struct sockaddr_in server_address;
	bzero(&server_address, sizeof(server_address));
	server_address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &server_address.sin_addr);
	server_address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int sendbuf = atoi(argv[3]);
	int len = sizeof(sendbuf);
	setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sendbuf, sizeof(sendbuf));
	getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sendbuf, (socklen_t*)&len);
	printf("the tcp buffer size after setting is %d\n", sendbuf);

	if(connect(sock, (struct sockaddr*)&server_address, sizeof(server_address)) != -1) {
		char buffer[BUFFER_SIZE];
		memset(buffer, 'a', BUFFER_SIZE);
		send(sock, buffer, BUFFER_SIZE, 0);
	}

	close(sock);

	return 0;
}
```

**SO-RCBLOWAT 和 SO-SNDLOWAT 选项**

SO_RCVLOWAT 和 SO_SNDLOWAT 选项分别表示 tcp 接收缓冲区和发送缓冲区的低水位标记。一般被 I/O 复系统调用来判断 socket 是否可读或可写。当 tcp 接收缓冲区中可读数据的总数大于其低水位标记时，I/O 复用系统调用将通知应用程序可以从对应的 socket 上读取数据。当 tcp 发送缓冲区中的空闲空间大于其低水位标记时，I/O 复用系统调用将通知应用程序可以往对应的 socke 写入数据。SO_RCVLOWAT 和 SO_SNDLOWAT 的默认值都为 1

**SO-LINGER 选项**

SO_LINGER 选项用于控制 close 系统调用在关闭 tcp 连接时的行为。默认情况下，当我们使用 close 关闭一个 socket 时，close 将立即返回，tcp 模块负责把该 socket 对应的 tcp 发送缓冲区残留数据发送给对方

设置/获取 SO_LINGER 选项的值时，需要给 setsockopt/getsockopt 系统调用传递一个 linger 类型的结构体
``` c++
struct linger {
    int l_onoff; // 非0开启，0关闭
    int l_linger; // 滞留时间
};
```
* l_onoff 等于 0。此时 SO_LINGER 选项不起作用，close 用默认行为来关闭 socket

* l_onoff 不等于 0，l_linger 等于 0。此时 close 系统调用立即返回，tcp 模块将丢弃发送缓冲区中残留的数据，同时给对端发送一个复位报文段

* l_onoff 不为 0，l_linger 大于 0。此时 close 的行为取决于两个条件：一是被关闭的 socket 对应的 tcp 发送缓冲区中是否还有残留的数据，二是该 socket 是阻塞的还是非阻塞的。对于阻塞的 socket，close 将等待一段长为 l_linger 的时间，直到 tcp 模块发送完所有残留数据并得到对方确认。如果没有发送完残留数据并得到对端确认，那么 close 调用将返回 -1 并设置 errno 为 EWOULDBLOCK。如果 socket 是非阻塞的，close 将立即返回，此时我们需要根据其返回值和 errno 来判断残留数据是否已经发送完毕


##### 网络信息 api

**gethostbyname 和 gethostbyaddr**

gethostbyname 函数根据主机名称获取主机的完整信息，gethostbyaddr 根据 ip 获取主机的完整信息。gethostbyname 函数通常先在本地的 /etc/hosts 配制文件中查找主机，没找到再去访问 dns 服务器

``` c++
#include <netdb.h>
struct hostent* gethostbyname(const char* name);
struct hostent* gethostbyaddr(const void* addr, size_t len, int type);
// name 参数指定目标主机名，addr 参数指定目标主机 ip 地址，len 参数指定 addr 所指 ip 地址长度，type 参数指定 addr 所指 ip 地址类型，其合法取值包括 AF_INET 和 AF_INET6

// hosten 结构体定义
#include <netdb.h>
struct hostent {
    char *h_nae; //主机名
    char **h_aliases; //主机别名列表，可能有多个
    int h_addrtype; //地址类型(地址族)
    int h_length; //地址长度
    char **h_addr_list; //按网络字节序列出的主机 ip 地址列表
};
```

**getservbyname 和 getservbyport**

getservbyname 函数根据名称获取某个服务的完整信息，getservbyport 函数根据端口号获取某个服务的完整信息。它们实际上都是通过读取 /etc/services 文件来获取服务的信息的
``` c++
#include <netdb.h>
struct servent* getservbyname(const char* name, const char *proto);
struct servent* getservbyport(int port, const char* proto);
// name 参数指定目标服务的名字，port 参数指定目标服务对应的端口号。proto 参数指定服务类型，给它传递 "tcp" 表示获取流服务，"udp" 表示获取数据报服务，NULL 则表示获取所有类型的服务

// servent 定义
#include <netdb.h>
struct servent {
    char *s_name; //服务名称
    char **s_aliases; //服务的别名列表，可能有多个
    int s_port; //端口号
    char *s_proto; //服务类型，通常是 tcp 或 udp
};
```

注意，上面 4 个函数都是不可重入的，即非线程安全。netdb.h 头文件中给出了可重入版本。这些函数的命名是在原函数尾部加上下划线r

**getaddrinfo**

getaddrinfo 函数既能通过主机名获取 ip 地址(内部实现用的 gehostbyname 函数)，也能通过服务名获得端口号(内部实现用的 getservbyname 函数)。它是否可重入取决于其内部调用的 gethostbyname 和 getservbyname 函数是否是可重入版本
``` c++
int getaddrinfo(const char *hostname, const char *service, const struct addrinfo *hints, struct addrinfo **result);
// hostname 参数可以接收主机名，也可以接收字符串表示的 ip 地址。同样，service 参数可以接受服务名也可以接收字符串表示的十进制端口号。hints 参数是应用程序给 getaddrinfo 的一个提示，以对 getaddrinfo 的输出进行更精确的控制。hints 参数可以被设置为 NULL，表示允许 getaddrinfo 反馈任何可用的结果。result 参数指向一个链表，该链表用于存储 getaddrinfo 反馈的结果

// addrinfo 结构体定义
struct addrinfo {
    int ai_flags; 
    int ai_family; // 地址族
    int ai_socktype; // 服务类型，SOCK_STREAM 或 SOCK_DGRAM
    int ai_protocol; // 具体网络协议，含义和 socket 系统调用第三个参数一致。值通常为 0
    socklen_t ai_addrlen; //socket 地址 ai_addr 的长度
    char *ai_canonname; //主机的别名
    struct sockaddr *ai_addr; //指向 socket 地址
    struct addrinfo *ai_next; //指向下一个 sockinfo 结构的对象
}
```
ai_flags 参数可以去下列表中的值，取多个则按位或

<img src="/images/getaddrinfo_ai_flags.png" width="600" height="280" />

注意，使用 hints 参数时只能设置 ai_flags、ai_family、ai_socktype 和 ai_protocol 四个字段，其余的字段必须设置为 NULL

用 getaddrinfo 获取本机的 daytime 流服务信息
``` c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <stdio.h>
#include <unistd.h>
#include <assert.h>
#include <string.h>

int main(int argc, char const *argv[]) {
	struct addrinfo hints;
	struct addrinfo *res;

	bzero(&hints, sizeof(hints));
	hints.ai_socktype = SOCK_STREAM;

	getaddrinfo("127.0.0.1", "daytime", &hints, &res);

	while(res != nullptr) {
		printf("ai_flags: %d\n", res->ai_flags);
		printf("ai_family: %d\n", res->ai_family);
		printf("ai_socktype: %d\n", res->ai_socktype);
		printf("ai_protocol: %d\n", res->ai_protocol);
		printf("ai_addrlen: %d\n", res->ai_addrlen);
		printf("ai_canonname: %s\n", res->ai_canonname);
		printf("ai_addr: %s\n", (res->ai_addr)->sa_data);
		res = res->ai_next;

	}

    // getaddrinfo 将隐式地分配内存(可以通过 valgrind 等工具查看)，因为 res 指针原本是没有指向一个合法内存的，所以，getaddrinfo 调用结束后，必须通过 freeaddrinfo 来释放对应的内存
	freeaddrinfo(res);

	return 0;
}
```

**getnameinfo**

getnameinfo 函数能通过 socket 地址同时获得字符串形式的主机名(内部实现用的 gethostbyaddr 函数) 和服务名(内部实现用的 getservbyport 函数)。它是否可重入取决于其内部调用的 gethostbyaddr 和 getservbyport 函数是否为可重入版本

``` c++
#include <netdb.h>
int getnameinfo(const struct sockaddr* sockaddr, socklen_t addrlen, char *host, socklen_t hostlen, char *serv, socklen_t servlen, int flags);
// getnameinfo 将返回的主机名存储在 host 参数指向的缓存中，将服务名存储在 serv 参数指向的缓存中，hostlen 和 servlen 参数分别指定这两块缓存的长度。flags 参数控制 getnameinfo 的行为
```

flags 参数选项及对应含义

<img src="/images/getnameinfo_flags.png" width="600" height="200" />

getaddrinfo 和 getnameinfo 函数成功时返回 0，失败则返回错误码。能通过 gai_strerror 函数将其错误码转换为易读的字符串形式
``` c++
#include <netdb.h>
const char* gai_strerror(int error);
```
