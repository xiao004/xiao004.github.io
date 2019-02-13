---
layout: post
title: "i/o function about network programming"
data: 2019-02-11 10:30:00 +0800
categories: network
---

和网络变成相关的 i/o 函数大致可以分文三类

* 用于创建文件描述符的函数，包括 pipe、dup/dup2 函数

* 用于读写数据的函数，包括 readv/writev、sendfile、mmap/munmap、splice 和 tee 函数

* 用于控制 i/o 行为和属性的函数，包括 fcntl 函数


##### pipe 函数
pipe 函数可用于创建一个管道，以实现进程间通信
``` c++
#include <unistd.h>
int pipe(int fd[2]);
// 成功时返回 0，并将一对打开的文件描述符值填入其参数指向的数组。如果失败则返回 -1 并设置 errno
```

通过 pipe 函数创建的两个文件描述符 fd[0] 和 fd[1] 分别构成管道的两端，fd[1] 写入的数据可以从 fd[0] 读出。且，fd[0] 端只能用于从管道读出数据，fd[1] 则只能用于往管道写入数据，不能反过来用。如果要用 pipe 实现双向，则只能使用两个管道。默认情况下 pipe 创建的管道是阻塞的。管道内传输的是字节流。管道本身存在容量限制，自 linux 2.6.11 内核起，管道的默认容量是 65536 字节。可以通过 fcntl 函数来修改管道容量

此外，socket 的基础 api 中有一个 socketpair 函数，能够创建双向管道
``` c++
#include <sys/types.h>
#include <sys/socket.h>

int socketpair(int domain, int type, int protocol, int fd[2]);
// 前三个参数含义和 socket 系统调用中完全相同，但 domain 只能使用 UNIX 本地域协议族 AF_UNIX。因为只能在本地使用这个双向管道。最后一个参数则和 pipe 系统调用一致，但 socketpair 创建的这对文件描述符两端都是既可读又可写的
// socketpair 成功时返回 0，失败返回 -1 并设置 errno
```


##### dup 函数和 dup2 函数
有时希望把标准输入重定向到一个文件，或者把标准输出重定向到一个网络连接(如 cgi 编程)。这可以通过用于复制文件描述符的 dup 或 dup2 函数来实现
``` c++
#include <unistd.h>
int dup(int file_descriptor);
int dup2(int file_descriptor_one, int file_descriptor_two);
// dup 函数创建一个新的且和 file_descriptor 指向相同文件、管道或网络连接的文件描述符。且 dup 返回的文件描述符总是取系统当前可用的最小整数值
//  dup2 和 dup 类似，不过它将返回第一个不小于 file_descriptor_two 的可用文件描述符
// dup 和 dup2 系统调用失败时都返回 -1 并设置 errno
// dup 和 dup2 创建的文件描述符并不继承原文件描述符的属性
```

用 dup 函数实现一个基本的 cgi 服务器
```c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>

int main(int argc, char const *argv[]) {
	if(argc < 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(sock, 5);
	assert(ret != -1);

	struct sockaddr_in client;
	socklen_t client_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);

	if(connfd < 0) {
		printf("errno is: %s\n", gai_strerror(errno));
	} else {
		close(STDOUT_FILENO); // 关闭标准输出文件描述符
		dup(connfd); // 复制 sock 文件描述符
		printf("acbd\n");
		close(connfd);
	}

	close(sock);

	return 0;
}
```
在上面的代码中，先关闭标准输出文件描述符 STDOUT_FILENO(值为 1)，然后复制 socket 文件描述符 connfd。因为 dup 总是返回系统中最小的可用文件描述符，所以这里的返回值实际上是 1，即之前关闭的标准输出文件描述符的值。因此，服务器输出到标准输出的内容就会直接发送到与客户端连接对应的 socket 上，因此 printf 调用输出将被客户端获得。这就是 cgi 服务器的基本工作原理


##### readv 函数和 writev 函数
read 函数将数据从文件描述符读到分散的内存块中，即分散读。writev 函数则将多块分散的内存数据一并写入文件描述符中，即集中写
``` c++
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec* vector, int count);
ssize_t writev(int fd, const struct iovec* vector, int count);
// fd　参数是被操作的文件描述符。vector 参数的类型是 iovec 结构数组 ([在 sendmsg/recvmsg 中有用到过 iovec](https://xiao004.github.io/network/2019/02/01/basic-api-of-network-programming.html))。count 参数是 vector 数组的长度，即有多少块内存数据需要从 fd 读出或写到 fd
// 成功时返回读出/写入 fd 的字节数，失败则返回 -1 并设置 errno
// 相当于简化版的 recvmsg 和 sendmsg 函数
```

通常 http 应答包含 1 个状态行、多个头部字段、1 个空行和文档内容和文档内容。其中前 3 部分的内容可能被 web 服务器放置在一块内存中，而文档的内容则通常被读入到另一块单独的内存中(通过 read 函数或 mmap 函数)。再通过 writev 函数将它们同时写出
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
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <netdb.h>
#include <sys/uio.h>

#define BUFFER_SIZE 1024

// 定义两种 http 状态码和信息状态
static const char *status_line[2] = {"200 OK", "500 Internal server error"};

int main(int argc, char const *argv[]) {
	if(argc <= 3) {
		printf("usage: %s ip_address port_number filename\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);
	const char *file_name = argv[3];

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(sock, 5);
	assert(ret != 1);

	struct sockaddr_in client;
	socklen_t client_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
	if(connfd < 0) {
		printf("error is: %s\n", gai_strerror(errno));
	} else {
		// 用于保存 http 应答的状态行、头部字段和一个空行的缓存区
		char header_buf[BUFFER_SIZE];
		memset(header_buf, '\0', BUFFER_SIZE);
		char *file_buf; //用于存放目标文件内容的应用程序缓存
		struct stat file_stat; //用于获取目标文件的属性，如是否为目录，文件大小等
		bool valid = true; //标记目标文件是否为有效文件
		int len = 0; //缓存区 header_buf 目前已经使用了多少字节空间

		if(stat(file_name, &file_stat) < 0) {// 目标文件不存在
			valid = false;
		} else {
			if(S_ISDIR(file_stat.st_mode)) {// 目标文件是一个目录
				valid = false;
			} else if(file_stat.st_mode & S_IROTH) {//当前用户有读取目标文件的权限
				int fd = open(file_name, O_RDONLY);
				file_buf = new char[file_stat.st_size + 1];
				memset(file_buf, '\0', file_stat.st_size + 1);
				
				// 将文件描述符中的内容读取到 file_buf 中
				if(read(fd, file_buf, file_stat.st_size) < 0) {
					valid = false;
				}
			} else {
				valid = false;
			}
		}

		if(valid) {//目标文件有效，发送正常的 http 应答
			//将 http 报文首行格式化为字符串并拷贝到 header_buf 中
			ret = snprintf(header_buf, BUFFER_SIZE - 1, "%s %s\r\n", "HTTP/1.1", status_line[0]);
			len += ret;//计算偏移量
			//将 http 头部字段和 crlf 拷贝到 header_buf 中
			ret = snprintf(header_buf + len, BUFFER_SIZE - 1 - len, "Content-Length: %ld\r\n", file_stat.st_size);

			//将 header_buf 和 file_buf 分别放入两个内存块中
			struct iovec iv[2];
			iv[0].iov_base = header_buf;
			iv[0].iov_len = strlen(header_buf);
			iv[1].iov_base = file_buf;
			iv[1].iov_len = file_stat.st_size;
			//通过 writev 将两个内存块中的内容集中写出
			ret = writev(connfd, iv, 2);
		} else {//目标文件无效的情况
			//将首行写入 header_buf 中
			ret = snprintf(header_buf, BUFFER_SIZE - 1, "%s %s\r\n", "HTTP/1.1", status_line[1]);
			len += ret;
			//将 crlf 写入 header_buf 中
			ret = snprintf(header_buf + len, BUFFER_SIZE - 1 - len, "%s", "\r\n");
			send(connfd, header_buf, strlen(header_buf), 0);
		}
		close(connfd);
		delete []file_buf;
	}

	close(sock);

	return 0;
}
```


##### sendfile 函数
sendfile 函数在两个文件描述符之间直接传递数据(完全在内核中操作)，从而避免了内核缓冲区和用户缓冲区之间的数据拷贝，即零拷贝，效率很高
``` c++
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
// in_fd 参数是待读出内容的文件描述符，out_fd 参数是待写入内容的文件描述符。offset 参数指定从读入文件描述符哪个位置开始读，如果为空则使用读入文件流默认的起始位置。count 参数指定在文件描述符 in_fd 和 out_fd 之间传输的字节数
// sendfile 成功时返回传输的字节数，失败则返回 -1 并设置 errno
// in_fd 必须是一个支持类似 mmap 函数的文件描述符，即它必须指向真实的文件，不能是 socket 和管道。而 out_fd 则必须是一个 socket。显然，sendfile 几乎是专门为在网络上传输文件而设计的
```

一个通过 sendfile 传输文件的 server
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
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/sendfile.h>
#include <netdb.h>

int main(int argc, char const *argv[]) {
	if(argc <= 3) {
		printf("usage: %s ip_address port_number filename\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);
	const char *file_name = argv[3];

	int filefd = open(file_name, O_RDONLY);
	assert(filefd > 0);

	struct stat stat_buf;
	fstat(filefd, &stat_buf);

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
	assert(ret != -1);

	ret = listen(sock, 5);
	assert(ret != -1);

	struct sockaddr_in client;
	socklen_t client_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);

	if(connfd < 0) {
		printf("errno is:%s\n", gai_strerror(errno));
	} else {
		sendfile(connfd, filefd, NULL, stat_buf.st_size);
		close(connfd);
	}

	close(sock);

	return 0;
}
```
相比 writev，sendfile 没有为目标文件分配任何用户空间的缓存，也没有执行读取文件的操作，但同样实现了文件的发送，且效率显然要高的多


##### mmap 函数和 munmap 函数
mmap 函数用于申请一段内存空间。可以将这段内存作为进程间通信的共享内存，也可以将文件映射到其中。munmap 函数则释放由 mmap 创建的这段内存空间
``` c++
#include <sys/mman.h>
void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *start, size_t length);
// start 参数允许用户使用某个特定的地址作为这段内存的起始地址。如果它被设置为 NULL，则系统自动分配一个地址。length 参数指定内存段的长度。prot 参数用来设置内存段的访问权限。它可以取以下几个值的按位或
// PROT_READ 内存段可读
// PROT_WRITE 内存段可写
// PROT_EXEC 内存段可执行
// PROT_NONE 内存段不能被访问
// flags 参数控制内存段内容被修改后程序的行为，具体值见下表(其中 MAP_SHARED 和 MAP_PROVATE 互斥，不能同时指定)
// fd 参数是被映射文件对应的文件描述符。一般通过 open 系统调用获得。offset 参数设置从文件的何处开始映射
// mmap 函数成功时返回指向目标内存区域的指针，失败则返回 MAP_FAILED((void*)-1) 并设置 errno。munmap 函数成功时返回 0，失败则返回 -1 并设置 errno
```
mmap flags 参数取值及其含义

<img src="/images/mmap_flags.png" width="600" height="220" />


##### splice 函数
splice 函数用于在两个文件描述符之间移动数据，也是零拷贝操
``` c++
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out, size_t len, unsigned int flags);
// fd_in 参数是待输入数据的文件描述符。如果 fd_in 是一个管道文件描述符，那么 off_in 参数必须被设置为 NULL。如果 fd_in 不是是一个管道文件描述符(如 socket)，那么 off_in 标书从输入数据流的何处开始读取数据。此时若 off_in 被设置为 NULL 则表示从输入数据流的当前偏移位置读入，若 off_in 不为 NULL，则它将指出具体的偏移位置
// fd_out/off_out 参数的含义与 fd_in/off_in 相同，不过其用于输出数据流
// len 参数指定移动数据的长度，flags 参数则控制数据如何移动，它可以被设置为下表中某些值按位或
// splice 函数成功时返回移动字节的数量，失败则返回 -1 并设置 errno
```

splice flags 参数的常用值及其含义

<img src="/images/splice_flags.png" width="600" height="150" />

使用 splice 函数时，fd_in 和 fd_out 必须至少有一个是管道文件描述符

用 splice 实现一个零拷贝回射服务器(将客户端发送的数据原样返回给客户端)
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
#include <fcntl.h>
#include <netdb.h>

int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sock = socket(PF_INET, SOCK_STREAM, 0);
	assert(sock >= 0);

	int ret = bind(sock, (struct sockaddr*)&address, sizeof(sockaddr));
	assert(ret != -1);

	ret = listen(sock, 5);
	assert(ret != -1);


	struct sockaddr_in client;
	socklen_t client_addrlength = sizeof(client);
	int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);

	if(connfd < 0) {
		printf("errno is: %s\n", gai_strerror(errno));
	} else {
		int pipefd[2];

		ret = pipe(pipefd);//创建一个管道，pipefd[1] 写入，pipefd[0] 写出
		assert(ret != -1);
		
		//将从 connfd 写入的数据读到管道中
		ret = splice(connfd, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
		assert(ret != -1);
		
		//将管道中的数据发送回 sock 连接中
		ret = splice(pipefd[0], NULL, connfd, NULL, 32768, SPLICE_F_MOVE | SPLICE_F_MORE);
		assert(ret != -1);
		close(connfd);
	}

	close(sock);

	return 0;
}
```
实现反射服务的原理为，先将客户端的内容读入到 pipefd[1] 中，然后再使用 splice 函数从 pipefd[0] 中读出该内容到客户端。整个过程未执行 recv/send 操作，因此并未涉及到用户空间和内核空间之间的数据拷贝


##### tee 函数
tee 函数在两个管道之间复制数据，同样是零拷贝操作。且它不消耗数据(非破坏性读出)。因此源文件描述符上的数据仍可以用于后续的读操作
``` c++
#include <fcntl.h>
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
// 其参数含义和 splice 中相同。但 fd_in 和 fd_out 都必须是管道文件描述符
// 成功时返回在两个文件描述符之间复制的数据字节数。失败则返回 -1 并设置 errno
```

用 tee 函数和 splice 函数实现 linux 下的 tee 程序(同时输出数据到终端和文件的程序)的基本功能

``` c++
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char const *argv[]) {
	if(argc != 2) {
		printf("usage: %s <file>\n", argv[0]);
		return 1;
	}

	int filefd = open(argv[1], O_CREAT | O_WRONLY | O_TRUNC, 0666);
	assert(filefd > 0);

	int pipefd_stdout[2];
	int ret = pipe(pipefd_stdout);
	assert(ret != -1);

	int pipefd_file[2];
	ret = pipe(pipefd_file);
	assert(ret != -1);
	
	//将标准输入内容输入管道 pipefd_stdout
	ret = splice(STDIN_FILENO, NULL, pipefd_stdout[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
	assert(ret != -1);
	
	// 通过 tee 将 pipefd_stdout 的输出内容复制(非破坏性读出)到 pipefd_file 的输入端
	ret = tee(pipefd_stdout[0], pipefd_file[1], 32768, SPLICE_F_NONBLOCK);
	assert(ret != -1);
	
	// 将管道 pipefd_file 的输出定向到文件描述符 filefd 上，从而将标准输入的内容写入文件
	ret = splice(pipefd_file[0], NULL, filefd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
	assert(ret != -1);
	
	// 将 pipefd_stdout 的输出定向到标准输出，其内容和写入文件的内容完全一致
	ret = splice(pipefd_stdout[0], NULL, STDIN_FILENO, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
	assert(ret != -1);

	close(filefd);
	close(pipefd_stdout[0]);
	close(pipefd_stdout[1]);
	close(pipefd_file[0]);
	close(pipefd_file[1]);

	return 0;
}
```


##### fcntl 函数
fcntl 提供了对文件描述符的各种控制操作(ioctl 也能够执行各种文件描述符操作，且比 fcntl 能执行更多的操作。但 fcntl 函数是有 posix 规范指定的首选方法)
``` c++
#include <fcntl.h>
int fcntl(int fd, int cmd, ...);
// fd 参数是被指定的文件描述符，cmd 参数指定执行何种类型的操作。根据操作类型的不同，该函数可能还需要第三个可选参数 arg
```

fcntl 函数支持的常用操作及其含义

<img src="/images/fcntl_1.png" width="600" height="450" />

<img src="/images/fcntl_2.png" width="600" height="130" />

在网络编程中，fcntl 函数通常用来将一个文件描述符设置为非阻塞的
``` c++
int setnoblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL); // 获取文件描述符旧的状态标志
    int new_option = old_option | O_NONBLOCK;//在原有状态上加上非阻塞标志
    fcntl(fd, F_SETFL, new_option);//设置非阻塞标志
    return old_option; //返回文件描述符旧的状态标志，以便需要恢复该状态时使用
}
```

