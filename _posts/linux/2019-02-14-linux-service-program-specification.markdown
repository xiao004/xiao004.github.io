---
layout: post
title: "linux service program specification"
date: 2019-02-14 19:10:00 +0800
categories: linux
---

常见的服务器程序规范

1. linux 服务程序一般以后台进程形式运行(没有控制终端，可以避免意外接受到用户输入)。后台进程又称守护进程，守护进程的父进程通常是 init 进程(PID 为 1 的进程)

2. linux 服务程序通常有一套日志系统，它至少能输出日志到文件，有的高级服务器还能输出日志到专门的 UDP 服务器。大部分后台进程都在 /var/log 目录下拥有自己的日志目录

3. linux 服务程序一般以某个专门的非 root 省份运行。如 mysqld、httpd、syslogd 等后台进程分别拥有自己的运行账户 mysql、apache 和 syslog

4. linux 服务程序通常是可配置的。可以通过命令或者配制文件来管理。大多数服务器程序的配制文件都存放在 /etc 目录下

5. linux 服务器进程通常会在启动的时候生成一个 PID 文件并存入 /var/run 目录中，以记录该后台进程的 PID

6. linux 服务器程序通常要考虑系统资源和限制，以预测自身能承受多大负荷，如进程可用文件描述符总数和内存总数等


##### 日志
**linux 系统日志**

linux 提供了一个守护进程来处理系统日志——syslogd(或者它的升级版 rsyslogd)

rsyslogd 守护进程既能接收用户进程输出的日志，又能接收内核日志。用户进程是通过 syslog 函数生成系统日志的。该函数将日志输出到一个 UNIX 本地域类型 socket 类型的文件　/dev/log 中，rsyslogd 则监听该文件以获取用户进程的输出。内核日志由 printk 等函数打印至内核的环状缓存中，环状缓存的内容直接映射到 /proc/kmsg 文件中。rsyslogd 则通过读取该文件获取内核日志(在老系统上内核日志是通过另外一个进程 rklogd 来管理的)

rsyslogd 守护进程在收到用户进程或内核输入的日志后，会把它们输出至某些特定的日志文件。默认情况下，调试信息会保存至 /var/log/debug 文件中。普通文件信息保存至 /var/log/messages 文件中。内核消息则保存至 /var/log/kern.log 文件中。不过日志信息具体如何分发可以在 rsyslogd 的配置文件中设置。rsyslogd 的主配制文件是 /etc/rsyslog.conf，其中的设置项主要有：内核日志输入路径、是否接受 UDP 日志及其监听端口(默认是 514，见 /etc/services 文件)、是否接收 TCP 日志及其监听端口、日志文件的权限以及包含哪些子配制文件

linux 系统日志体系

![rsyslogd](/images/rsyslogd.png)

**syslog 函数**

``` c++
#include <syslog.h>
void syslog(int priority, const char *message, ...);
// 采用可变参数来结构化输出
// priority 参数即设施值与日志级别的按位或。设施值的默认值是 LOG_USER
// 有下列日志级别
// LOG_EMERG    0   系统不可用
// LOG_ALERT    1   报警，需要立即采取行动
// LOG_CRIT     2   非常严重的情况
// LOG_ERR      3   错误
// LOG_WARNING  4   警告
// LOG_NOTICE   5   通知
// LOG_INFO     6   信息
// LOG_DEBUG    7   调试
```

可以通过 openlog 函数改变 syslog 的默认输出方式，进一步结构化日志内容
``` c++
#include <syslog.h>
void openlog(const char *ident, int logopt, int facility);
// ident 参数指定的字符串将被添加到日志消息的日期和时间之后，通常被设置为程序的名字
// logopt 参数对后续 syslog 调用的行为进程配制，可取下列值的按位或
// LOG_PID      0x01    在日志消息中包含程序 PID
// LOG_CONS     0x02    如果消息不能记录到日志文件，则打印至终端
// LOG_ODELAY   0x04    延迟打开日志功能直到第一次调用 syslog
// LOG_NDELAY   0x08    不延迟打开日志功能

// facility 参数可以用来修改 syslog 函数中默认设施值
```

**日志过滤**

程序在开发阶段可能要输出很多调试信息，而发布之后又需要将这些调试信息关闭。程序发布之后删除调试代码显然不是一个好的方案。实际上解决方法很简单，通过设置日志掩码，使日志级别大于日志掩码的日志信息被系统忽略
``` c++
#include <syslog.h>
int setlogmask(int maskpri);
// maskpri 参数指定日志掩码值
// 该函数会始终成功，返回进程先前的日志掩码值
```

关闭日志功能

void closelog();


##### 用户信息
**UID、EUID、GID 和 EGID**

用户信息对于服务器程序的安全性来说非常重要，大部分服务器必须以 root 身份启动，但不能以 root 身份运行。下列函数可以获取和设置当前进程的真实用户 ID(UID)、有效用户 ID(EUID)、真实组 ID(GID) 和有效组 ID(EGID)
``` c++
#include <sys/types.h>
#include <unistd.h>

uid_t getuid(); // 获取真实用户 ID
udi_t geteuid(); // 获取有效用户 ID
gid_t getgid(); // 获取真实组 ID
gid_t getegid(); // 获取有效组 ID

int setuid(uid_t uid); // 设置真实用户 ID
int seteuid(uid_t uid); // 设置有效用户 ID
int setgid(gid_t gid); // 设置真实组 ID
int setegid(gid_t gid); //设置有效组 ID
```

一个进程拥有两个用户 ID: UID 和 EUID。EUID 存在的目的是方便资源访问，它使得运行程序的用户拥有该程序的有效用户的权限。如 su 程序，任何用户都可以使用它来修改自己的账户信息，但修改账户时 su 程序访问 /etc/passwd 文件是需要 root 权限的。那普通用户启动的 su 程序如何能访问 /etc/passwd 文件呢？通过 ls 命令可以发现，su 程序的所有者是 root，并且它设置了 set-user-id 标志。该标志表示，任何普通用户运行 su 程序时有效用户就是该程序的所有者 root。根据有效用户的含义，任何运行 su 程序的普通用户都能够访问 /etc/passwd 文件。有效用户为 root 的进程称为特权进程。EGID 的含义与 EUID 类似，给运行目标程序的组用户提供有效组的权限

测试进程的 UID 和 EUID 的区别
``` c++
#include <unistd.h>
#include <stdio.h>

int main(int argc, char const *argv[]) {
	uid_t uid = getuid();
	uid_t euid = geteuid();

	printf("userid is %d, effective userid is: %d\n", uid, euid);

	return 0;
}

// 编译文件
// g++ diff_uid_euid.cpp -o diff_uid_euid
// 修改目标文件的所有者为 root
// sudo chown root:root diff_uid_euid
// 设置目标文件的 set-user-id 标志
// sudo chmod +s diff_uid_euid
// 运行结果
// userid is 1000, effective userid is: 0
```

显然，进程的 UID 是启动用户程序的用户的 ID，而 EUID 则是 root 账户的 ID

**切换用户**

以 root 身份启动的进程切换为以一个普通用户身份运行
``` c++
#include <unistd.h>
#include <iostream>
using namespace std;

void prin(void) {
	gid_t gid = getgid();
	uid_t uid = getuid();

	cout << gid << " " << uid << endl;
}

static bool switch_to_user(uid_t user_id, gid_t gp_id) {
	if((user_id == 0) && (gp_id == 0)) {
		return false;
	}

	gid_t gid = getgid();
	uid_t uid = getuid();

	if(((gid != 0) || (uid != 0)) && ((gid != gp_id) || (uid != user_id))){
		return false;
	}

	if(uid != 0) {
		return true;
	}

	if((setgid(gp_id) < 0) || (setuid(user_id) < 0)) {
		return false;
	}

	return true;
}



int main(int argc, char const *argv[]) {
	prin();
	switch_to_user(1000, 1000);
	prin();
	return 0;
}

//output
// 0 0
// 1000 1000
```


##### 进程间关系
**进程组**

linux 下每个进程都隶属于一个进程组，因此处了 PID 信息外，还有进程组 ID(PGID)。可以通过 getpgi 和 sepgid 来获取指定和设置进程的 PGID
``` c++
#include <unistd.h>
pid_t getpgid(pid_t pid);
// 成功时返回进程 pid 所属进程组的 PGID，失败返回 -1 并设置 errno

int setpgid(pid_t pid, pid_t pgid);
// 该函数将 PID 为 pid 的进程的 PGID 设置为 pgid。如果 pid 和 pgid 相同，则由 pid 指定的进程将被设置为进程组首领。如果 pid 为 0，则表示设置当前进程的 PGID 为 pgid。如果 pgid 为 0，则使用 pid 作为目标 PGID。setpgid 函数成功时返回 0，失败则返回 -1 并设置 errno
```

每个进程组都有一个首领进程，其 PGID 和 PID 相同。进程组将一直存在，直到其中所有进程都退出，或者加入到其他进程组

一个进程只能设置自己或者其子进程的 PGID。并且当子进程调用 exec 系列函数后，我们也不能再在父进程中对它设置 PGID

**会话**

一些有关联的进程组将形成一个会话(session)。可以用 setsid 函数创建一个会话
``` c++
#include <unistd.h>
pid_t setsid(void);
// 该函数不能由进程的首领进程调用，否则将产生一个错误。对于非首领的进程，调用该函数除了创建新会话还会产生下列效果
// 调用进程成为会话的首领，此时该进程是新会话的唯一成员
// 新建一个进程组，其 PGID 就是调用进程的 PID，调用进程成为该组的首领
// 调用进程将甩开终端(如果有的话)

// 调用成功时返回新的进程组的 PGID，失败则返回 -1 并设置 errno

```

SID(会话 ID)即会话首领所在进程组的 PGID，可以通过 getsid 函数获取

pid_t getsid(pid_t pid);


可以通过 ps 命令查看进程、进程组和会话之间的关系
``` bash
ps -o pid,ppid,pgid,sid,comm | less
# ppid 是父进程 ID
# 我们是在 bash shell 下执行 ps 和 less 命令的，所以 ps 和 less 命令的父进程是 bash 命令(这可以从 ppid 看出)。这三条命令创建了一个会话，他们的 sid 都相同，且和 bash 命令的 pgid 一致，显然 bash 命令是这个会话的首领。同时 bash 命令的 PID 和 PGID 相同，所以 bash 也是它所在组的首领
```

##### 系统资源限制
linux 上运行的程序都会收到资源限制的影响，如物理设备限制(cpu 数量、内存数量等)、系统策略限制(cpu 时间等)，以及具体实现的限制(如文件名的最大长度)。linux 系统资源限制可以通过　getrlimit 和 setrlimit 来读取和设置
``` c++
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, struct rlimit *rlim);

//rlimit 结构体定义
struct rlimit{
    rlim_t rlim_cur;
    rlim_t rlim_max;
};
// rlim_t 是一个整数类型，描述资源级别。rlim_cur 成员指定资源的软限制，rlim_max 成员指定资源的硬限制
// 软限制是一个建议性的、最好不要超越的限制，如果超越的话，系统可能向进程发送信号以终止其运行。如进程 cpu 时间超过其软限制时，系统将向进程发送 SIGXCPU 信号
// 硬限制一般是软限制的上限。普通程序可以减小硬限制，但只有以 root 身份运行的程序才能增加硬限制
// resource 参数指定了资源限制类型
// 成功时返回 0，失败返回 -1 并设置 errno
```

resource 对应的部分资源限制类型及其含义
![resource](/images/resource.png)

此外，还可以使用 ulimit 命令修改当前 shell 环境下的资源限制，这种修改将对该 shell 启动后的所有后续程序有效。也可以通过修改配制文件来改变系统资源限制，且这种修改是永久的


##### 改变工作目录和根目录
有些服务器程序还需要改变工作目录和根目录，如 web 服务器。web 服务器的逻辑根目录并非文件系统的根目录 "/"，而是站点的根目录(如 linux 下的 /var/www/)。可以通过 getcwd、chdir 和 chroot 函数获取、改变进程工作目录和改变进程根目录

``` c++
#include <unistd.h>
char *getcwd(char *buf, size_t size);
// buf 参数指向的内存用于存储进程当前工作目录的决对路径，其大小由 size 参数决定。如果当前工作目录的绝对路径长度超过了 size，则 getcwd 将返回 NULL，并设置 errno 为 ERANGE
// 如果 buf 为 NULL 且 size 非 0，则 getcwd 可能在内部使用 malloc 动态分配内存并将进程的当前工作目录存储在其中。但是需要我们自己来释放这块内存
// getcwd 函数成功时返回一个指向目标存储区的指针，失败则返回 NULL 并设置 errno

int chdir(const char *path);
// 将目录切换到 path
// 成功时返回 0，失败时返回 -1 并设置 errno

int chroot(const char *path);
// path 参数指定要切换到的根目录
// 成功时返回 0，失败返回 -1 并设置 errno
```

注意，只有特权进程才能改变根目录。此外 chroot 并不能改变进程的当前工作目录，所以调用 chroot 之后，仍然需要使用 chdir("/") 来将工作目录切换到新的根目录。改变根目录后程序可能无法访问类似 /dev 的文件(和目录)，因为这些文件并非处于新的根目录之前。不过调用 chroot 之后，进程原先打开的文件描述符仍然生效，所以可以利用这些先打开的文件描述符来访问 chroot 之后不能直接访问的文件


##### 服务器程序后台化
``` c++
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

bool daemonize() {
    // 创建子进程，关闭父进程，使程序在后台运行
	pid_t pid = fork();

	if(pid < 0) {
		return false;
	} else if(pid > 0) {
		exit(0);
	}

    // 设置文件权限掩码。当用 open(const char *pathname, int flags, mode_t mode) 系统调用创建文件时，文件的权限将是 mode & (~0) 即 mode & 0777
	umask(0);

    // 创建新会话，设置本进程为进程会话的首领
	pid_t sid = setsid();
	if(sid < 0) {
		return false;
	}
    
    // 切换工作目录
	if(chdir("/") < 0) {
		return false;
	}
    
    // 关闭标准输入设备、标准输出设备和标准错误输出设备
	close(STDIN_FILENO);
	close(STDOUT_FILENO);
	close(STDERR_FILENO);
    
    // 将标准输入、标准输出和标准错误输出都定向到 /dev/null 文件
	open("/dev/null", O_RDONLY);
	open("/dev/null", O_RDWR);
	open("/dev/null", O_RDWR);

	return true;
}

int main(int argc, char const *argv[]) {
	daemonize();
	return 0;
}
```

实际上，linux 提供了完成同样功能的库函数
``` c++
#include <unistd.h>
int daemon(int nochdir, int noclose);
// 其中 nochdir 参数用于指定是否改变工作目录，如果给它传递 0，则工作目录将被设置为 "/"，否则继续使用当前工作目录
// noclose 参数为 0 时，标准输入、标准输出和标准错误输出都被重定向到 /dev/null 文件，否则依然使用原来的设备
// 成功返回 0，失败返回 -1 并设置 errno
```

