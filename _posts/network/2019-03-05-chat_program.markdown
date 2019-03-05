---
layout: post
title: "io multiplex application -- mini chat program"
date: 2019-03-05 11:10:00 +0800
categories: network
---

chat_server.cpp
``` c++
#define _GNU_SOURCE 1

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
#include <poll.h>

#define USER_LIMIT 5 // 最大用户数量
#define BUFFER_SIZE 64 // 读缓冲区大小
#define FD_LIMIT 65535 //文件描述符数量限制


// 客户端数据
struct client_data {
	sockaddr_in address; // 客户端 socket 地址
	char *write_buf; // 待写到客户端的数据的位置
	char buf[BUFFER_SIZE]; // 从客户端读入的数据
};


// 将 fd 设置为非阻塞的
int setnonblocking(int fd) {
	int old_option = fcntl(fd, F_GETFL);
	int new_option = old_option | O_NONBLOCK;

	fcntl(fd, F_SETFL, new_option);

	return old_option;
}


int main(int argc, char const *argv[]) {
	
	if(argc <= 2) {
		printf("uasge: %s ip_address port_number\n", basename(argv[0]));
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
	
	// socket 文件描述符的值映射到 client_data 的下标关联对应的客户数据
	client_data *users = new client_data[FD_LIMIT];
	pollfd fds[USER_LIMIT + 1];
	int user_counter = 0;

	for(int i = 1; i <= USER_LIMIT; ++i) {
		fds[i].fd = -1;
		fds[i].events = 0;
	}

	fds[0].fd = listenfd;
	fds[0].events = POLLIN | POLLERR;
	fds[0].revents = 0;

	while(1) {
		ret = poll(fds, user_counter + 1, -1);

		if(ret < 0) {
			printf("poll failure\n");
			break;
		}

		// 遍历当前所有的 socket 文件描述符, user_counter + 1 即加上监听 socket 的 listenfd
		for(int i = 0; i < user_counter + 1; ++i) {
			// 有新的 socket 连接加入
			if((fds[i].fd == listenfd) && (fds[i].revents & POLLIN)) {
				struct sockaddr_in client_address;
				socklen_t client_addrlength = sizeof(client_address);

				int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);

				if(connfd < 0) {
					printf("errno is: %s\n", strerror(errno));
					continue;
				}

				// 如果连接数超过 USER_LIMIT 个则关闭新到的连接
				if(user_counter >= USER_LIMIT) {
					const char *info = "too many users";
					printf("%s\n", info);

					send(connfd, info, strlen(info), 0);

					close(connfd);
					continue;
				}

				// 当前人数 +1
				++user_counter;
				// 将客户端的数据映射到 user[connfd]
				users[connfd].address = client_address;

				// 将当前 socket 设置为非阻塞的
				setnonblocking(connfd);

				// 设置 poll 监听 connfd 上的 POLLIN, POLLRDHUP 和 POLLERR 事件
				fds[user_counter].fd = connfd;
				fds[user_counter].events = POLLIN | POLLRDHUP | POLLERR;
				fds[user_counter].revents = 0;

				printf("come a new user, now have %d users\n", user_counter);

			} else if(fds[i].revents & POLLERR) {
				
				// 监听到异常事件
				printf("get an error from %d\n", fds[i].fd);

				char errors[100];
				memset(errors, '\0', 100);
				socklen_t length = sizeof(errors);
				
				// 尝试通过 getsockopt 来清除该错误码
				if(getsockopt(fds[i].fd, SOL_SOCKET, SO_ERROR, &errors, &length) < 0) {
					// 如果 getsockopt 失败则打印这条语句
					printf("get socket option failed\n");
				}

				continue;

			} else if(fds[i].revents & POLLRDHUP) {
				// 若客户端关闭连接则服务器端也关闭连接
				//users[fds[i].fd] = users[fds[user_counter].fd];
				close(fds[i].fd);
				
				// 将最后一个 socket 连接移至当前关闭 socket 连接所在的坑位
				fds[i] = fds[user_counter];
				--i;
				--user_counter;

				printf("a client left, now have %d users online\n", user_counter);

			} else if (fds[i].revents & POLLIN) {
				// 可读事件
				int connfd = fds[i].fd;
				memset(users[connfd].buf, '\0', BUFFER_SIZE);
				
				// 将从客户端接受到的数据放入 buf 中
				ret = recv(connfd, users[connfd].buf, BUFFER_SIZE - 1, 0);

				printf("get %d bytes of client data %s from %d\n", ret, users[connfd].buf, connfd);

				if(ret < 0) {
					// 若读取错误则关闭该连接
					if(errno != EAGAIN) {
						close(connfd);

						//users[fds[i].fd] = users[fds[user_counter].fd];
						// 将最后一个 socket 连接移至当前关闭 socket 连接所在的坑位
						fds[i] = fds[user_counter];
						--i;
						--user_counter;
					}
				} else if (ret == 0) {

				} else {
					// 如果接受到数据, 则通知其他客户端准备写数据
					for(int j = 1; j <= user_counter; ++j) {
						if(fds[j].fd == connfd) {
							continue;
						}
						
						fds[j].events |= ~POLLIN;
						fds[j].events |= POLLOUT;
						users[fds[j].fd].write_buf = users[connfd].buf;
					}
				}

			} else if(fds[i].revents & POLLOUT) {
				int connfd = fds[i].fd;
				
				if(!users[connfd].write_buf) {
					continue;
				}

				ret = send(connfd, users[connfd].write_buf, strlen(users[connfd].write_buf), 0);
				users[connfd].write_buf = NULL;

				fds[i].events |= ~POLLOUT;
				fds[i].events |= POLLIN;
			}
		}
	}

	delete []users;
	close(listenfd);

	return 0;
}
```

chat_client.cpp
``` c++
#define _GNU_SOURCE 1

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <poll.h>
#include <fcntl.h>

#define BUFFER_SIZE 64


int main(int argc, char const *argv[]) {
	if(argc <= 2) {
		printf("usage: %s ip_address port_number\n", basename(argv[0]));
		return 1;
	}

	const char *ip = argv[1];
	int port = atoi(argv[2]);

	struct sockaddr_in server_address;
	bzero(&server_address, sizeof(server_address));
	server_address.sin_family = AF_INET;
	inet_pton(AF_INET, ip, &server_address.sin_addr);
	server_address.sin_port = htons(port);

	int sockfd = socket(PF_INET, SOCK_STREAM, 0);
	assert(socket >= 0);

	if(connect(sockfd, (struct sockaddr*)&server_address, sizeof(server_address)) < 0) {
		printf("connection failed\n");
		close(sockfd);
		return 1;
	}

	pollfd fds[2];

	fds[0].fd = 0;
	fds[0].events = POLLIN;
	fds[0].revents = 0;

	fds[1].fd = sockfd;
	fds[1].events = POLLIN | POLLRDHUP;
	fds[1].revents = 0;

	char read_buf[BUFFER_SIZE];
	int pipefd[2];
	int ret = pipe(pipefd);
	assert(ret != -1);

	while(1) {
		ret = poll(fds, 2, -1);
		
		if(ret < 0) {
			printf("poll failure\n");
			break;
		}

		if(fds[1].revents & POLLRDHUP) {
			printf("server close the connection\n");
			break;

		} else if(fds[1].revents & POLLIN) {
			memset(read_buf, '\0', BUFFER_SIZE);

			recv(fds[1].fd, read_buf, BUFFER_SIZE - 1, 0);

			printf("%s\n", read_buf);
		}

		if(fds[0].revents & POLLIN) {
			// 通过 splice 0 拷贝将用户数据写到 sockfd 上			
			ret = splice(0, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);

			ret = splice(pipefd[0], NULL, sockfd, NULL, 32786, SPLICE_F_MORE | SPLICE_F_MOVE);
		}
	}

	close(sockfd);

	return 0;
}
```
