---
layout: post
title: "nonblock connect"
date: 2019-03-03 18:00:00 +0800
categories: network
---

**阻塞 connect**

客户端调用 connect 发起对服务端的 socket 连接，调用 connect 函数将激发 tcp 三次握手过程．如果客户端的 socket 描述符为阻塞模式(默认)，则 connect 会阻塞到连接建立成功或连接超时（linux内核中对 connect 的超时时间限制是 75s). 在某些情况下我们并不希望 connect 阻塞这么久，如在做端口扫描时. 这个问题可以通过非阻塞 connect + select(或者 poll/epoll) 设置超时解决

**非阻塞 connect**

如果为非阻塞模式，则调用 connect 后函数立即返回. 如果连接不能马上建立成功返回-1. 当 errno 被设置为 EINPROGRESS 时，表示连接建立未完成但仍在继续(tcp 三次握手仍在继续)。此时可以调用 select，epoll 等函数监听这个连接失败的 socket 上的可写事件．当 select, poll 等函数返回后再用 getsockopt 来读取错误码并清除该 socket 上的错误．如果错误码是 0，表示连接建立，否则连接失败

``` c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <time.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <string.h>

#define BUFFER_SIZE 1023


// 设置文件描述符 fd 为非阻塞
int setnonblocking(int fd) {
	int old_option = fcntl(fd, F_GETFL);
	int new_option = old_option | O_NONBLOCK;

	fcntl(fd, F_SETFL, new_option);

	return old_option;
}


// 超时连接函数, 参数分别是服务器 ip, 端口号和超时时间(毫秒)
// 成功返回已经处于连接状态的 socket, 失败返回 -1
int unblock_connect(const char *ip, int port, int time) {
	int ret = 0;
	struct sockaddr_in address;
	bzero(&address, sizeof(address));
	address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &address.sin_addr);
	address.sin_port = htons(port);

	int sockfd = socket(PF_INET, SOCK_STREAM, 0);

	int fdopt = setnonblocking(sockfd);

	ret = connect(sockfd, (struct sockaddr*)&address, sizeof(address));
	
	// 如果连接成功, 则恢复 sockfd 的属性并立即返回
	if(ret == 0) {
		printf("connect with server immediately\n");
		fcntl(sockfd, F_SETFL, fdopt);
		return sockfd;

	} else if (errno != EINPROGRESS) {
		// 如果连接没有建立, 则只有当 errno 是 EINPROGRESS 时才表示连接还在进行
		printf("unblock connect not support\n");
		return -1;
	}

	fd_set readfds;
	fd_set writefds;
	struct timeval timeout;

	FD_ZERO(&readfds);
	FD_SET(sockfd, &writefds);

	timeout.tv_sec = time;
	timeout.tv_usec = 0;

	ret = select(sockfd + 1, NULL, &writefds, NULL, &timeout);

	// 超时或者出错, 立即返回
	if(ret <= 0) {
		printf("connection time out\n");
		close(sockfd);
		return -1;
	}

	if(!FD_ISSET(sockfd, &writefds)) {
		printf("no events on sockfd found\n");
		close(sockfd);
		return -1;
	}

	int error = 0;
	socklen_t length = sizeof(error);
	// 调用 getsockopt 来获取并清除 sockfd 上的错误
	if(getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &length) < 0) {
		printf("get socket option failed\n");
		close(sockfd);
		return -1;
	}

	// 错误号不为 0 表示连接出错
	if(error != 0) {
		printf("connection failed after select with the error: %s\n", strerror(error));
		close(sockfd);
		return -1;
	}
	
	// 连接成功
	printf("connection ready after select with the socket: %d\n", sockfd);
	// 将 sockfd 设置回原来的属性即阻塞
	fcntl(sockfd, F_SETFL, fdopt);
	return sockfd;

}


int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	for(int i = 1; i + 1 <= argc; i += 2) {
		const char *ip = argv[i];
		int port = atoi(argv[i + 1]);

		int sockfd = unblock_connect(ip, port, 10);

		if(sockfd < 0) {
			return 1;
		}

		close(sockfd);

	}

	return 0;
}
```


