---
layout: post
title: "multi-process programming"
date: 2019-03-05 09:50:00 +0800
categories: operating-system
---

linux 多进程编程主要包括

* 复制进程映像的 fork 系统调用和替换进程映像的 exec 系列系统调用

* 僵尸进程以及如何避免僵尸进程

* 进程间通信(IPC) 管道, 信号量, 消息队列和共享内存

* 进程间传递文件描述符的通用方法: 通过 UNIX 本地域 socket 传递特殊的辅助数据


#### fork 系统调用

linux 下创建新进程的系统调用是 fork

``` c++
#include <sys/types.h>
#include <unistd.h>

pip_t fork(void)
```

该函数每次调用都返回两次, 在父进程中返回的是子进程的 PID, 在子进程中则返回 0. 该返回值是后续代码判断当前进程是父进程还是子进程的依据. fork 调用失败时返回 -1 并设置 errno

fork 函数复制当前进程, 在内核表中创建一个新的进程表项. 新的进程表项中有很多属性和原进程相同, 比如堆指针, 栈指针和标志寄存器的值. 也有许多属性被赋予了新的值, 比如该进程的 PPID 被设置成原进程的 PID, 信号位图被清除(原进程设置的信号处理函数不再对新进程起作用)

子进程的代码与父进程完全相同, 同时它还会复制父进程的数据(堆数据, 栈数据和静态数据). 数据的复制采用的是写时复制(copy on writte), 即只有在任一进程(父进程或子进程)对数据执行了写操作时, 复制才会发生(先是缺页中断, 然后操作系统给子进程分配内存并复制父进程的数据). 即便如此, 如果在程序中分配了大量内存, 那么使用 fork 时应当十分谨慎, 尽量避免没有必要的内存分配和数据复制

创建子进程后, 父进程中打开的文件描述符默认在子进程中也是打开的, 且文件描述符的引用计数加 1. 另外, 父进程的用户根目录, 当前工作目录等变量的引用计数均会加 1


#### exec 系列系统调用
有时需要在子进程中执行其他程序, 即替换当前进程映像. 这需用使用到 exec 系列函数

``` c++
#include <unistd.h>
extern char **environ;

int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execve(const char *path, char * const argv[], char *const envp[]);

// path 参数指定可执行文件的完整路径
// file 参数可以接受文件名, 该文件的具体位置则在环境变量 PATH 中查找
// arg 接受可变参数, argv 则接受参数数组, 它们都会传递给新程序(path 或 file 指定的程序) 的 main 函数
// emvp 参数用于设置新程序的环境变量. 如果为设置, 则新程序将使用由全局变量 environ 指定的环境变量
```

一般情况下, exec 函数是不会返回的, 除非出错. 它出错时返回 -1 并设置 errno. 如果没有出错, 则源程序中 exec 调用之后的代码都不会执行, 因为此时原程序已经被 exec 的参数指定的程序完全替换(包括代码和数据)

exec 函数不会关闭原程序打开的文件描述符, 除非该文件描述符设置了类似 SOCK_CLOEXEC 的属性


#### 处理僵尸进程
对于多进程程序而言, 父进程一般需要跟踪子进程的退出状态. 因此, 当子进程结束运行时, 内核不会立即释放该进程的进程表表项, 以满足父进程后续对该子进程退出信息的查询(如果父进程还在运行). 在子进程结束运行之后, 父进程读取其退出状态之前, 即该子进程处于僵尸状态. 另一种使子进程进入僵尸态的情况是: 父进程结束或者异常终止, 而子进程继续运行. 此时子进程的 PPID 被操作系统设置为 1, 即 init 进程. init 进程接管了该子进程, 并等待它结束. 在父进程退出之后, 子进程退出之前, 该子进程处于僵尸态

可以预见, 如果父进程没有正确处理子进程的返回信息, 子进程都将停留在僵尸态, 并占据内核资源. 可以在父进程中调用 wait 或 waitpid 等待子进程结束并获取子进程的返回信息, 从而避免僵尸进程的产生或者使子进程的僵尸态立即结束

``` c++
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *stat_loc);
pid_t waitpid(pid_t pid, int *stat_loc, int options);
```

wait 函数将阻塞进程, 直到该进程的某个子进程结束运行为止. 它返回结束运行的子进程的 PID 并将该子进程的退出状态信息存储于 stat_loc 参数指向的内存中

waitpid 只等待由 pid 参数指定的子进程. 如果 pid 值为 -1, 那么它就和 wait 函数相同, 即等待任意一个子进程结束. stat_loc 参数的含义和 wait 函数的 stat_loc 参数相同. options 参数可以控制 waitpid 函数的行为. 当 options 的取值是 WNOHANG 时, waitpid 立即返回 0. 如果目标子进程正常退出了, 则 waitpid 返回该子进程的 PID. waitpid 调用失败时返回 -1 并设置 errno

当一个进程结束时, 它2将给父进程发送一个 SIGCHLD 信号. 因此可以在父进程中补货 SIGCHLD 信号, 并在信号处理函数中调用 waitpid 函数以 "彻底结束" 一个子进程

``` c++
static void handle)_child(int sig) {
	piud_t pid;
	int stat;
	while((pid = waitpid(-1, &stat, WNOHANG)) > 0) {
		// 对子进程进行善后处理
	}
}
```

#### 管道
管道既可以实现进程内部的通信, 也可以实现父进程和子进程间通信. 管道能在父, 子进程间传递数据, 利用的是 fork 调用之后两个管道文件描述符都保持打开. 一对这样的文件描述符只能保证父, 子进程间一个方向的数据传输, 父进程和子进程必须有一个关闭 fd[0] 另一个关闭 fd[1]. 如果要实现父, 子进程之间的双向数据传输就必须要用两个管道. 另外 socket 编程接口提供了一个创建全双工管道的系统调用: socketpair


#### 信号量

##### semget 系统调用
semget 系统调用创建一个新的信号量集, 或者获取一个已经存在的信号量集

``` c++
#include <sys/sem.h>
int semget(key_t key, int num_sems, int sem_flags);

// key 参数是一个键值, 用来标识一个全局唯一的信号量集
// num_sems 参数指定要创建/获取的信号量集中信号量的数目. 如果是创建信号量则该值必须被指定, 如果是获取已经存在的信号量则可以将其设置为 0
// sem_flags 参数指定一组标志. 它低端的 9 个比特是该信号量的权限, 其格式和含义都与系统调用 open 和 mode 参数相同
// semget 成功时返回一个正整数值, 它是信号量集的标识符. 失败则返回 -1 并设置 errno
```

如果 semget 用于创建信号量集，则与之关联的内核数据结构体 semid_ds 将被创建并初始化．semid_ds 结构体的定义

``` c++
#include <sys/sem.h>

// 该结构体用于描述 IPC 对象(信号量，共享内存和消息队列)的权限
struct ipc_perm {
	key_t key; // 键值
	uid_t uid; //所有者的有效用户 ID
	gid_t git; // 所有者的有效组 ID
	uid_t cuid; // 创建者的有效用户 ID
	gid_t cgid; // 创建者的有效组 ID
	mode_t mode; // 访问权限
	// 省略其他填充字段
}

struct semid_ds {
	struct ipc_perm sem_perm; //信号量的操作权限
	unsigned long int sem_nsems; //该信号量集中信号的数目
	time_t sem_otime; // 最后一次调用 semop 的时间
	time_t sem_ctime; // 最后一次调用 semctl 的时间
	// 省略其他填充字段
}
```

semget 对 semid_ds 结构体的初始化包括

* 将 ssem_perm.cuid 和 sem_perm.uid 设置为调用进程的有效用户 ID

* 将用户 sem_perm.cgid 和 sem_perm.gid 设置为调用进程的有效组 ID

* 将 sem_perm.mode 的最低 9 位设置为 sem_flags 参数的最低 9 位

* 将 sem_nsems 设置为 num_sems

* 将 sem_otime 设置为 0

* 将 sem_ctime 设置为当前系统时间

##### semop 系统调用
与每个信号量关联的一些重要的内核变量

``` c++
unsigned short semval; //信号量的值
unsigned short semzcnt; // 等待信号量值变为 0 的进程数量
unsigned short semncnt; // 等待信号量值增加的进程数量
pid_t sempid; // 最后一次执行 semop 操作的进程 ID
```

semop 对信号量的操作实际上就是对这些内核变量的操作
``` c++
#include <sys/sem.h>
int semop(int sem_id, struct sembuf *sem_ops, size_t num_sem_ops);

// sem_id 参数是由 semget 调用返回的信号量集标识符，可以指定被操作的目标信号量集
// sem_ops 参数指向一个 sembuf 结构体类型的数组
// num_sem_ops 指定要执行的操作个数，　即 sem_ops 数组中元素的个数．semop 对数组 sem_ops 中的每个成员按照数组顺序依次进行操作，并且该过程是原子操作，以避免个别的进程在同一时刻按照不同的顺序对该信号集中的信号量执行 semop 操作导致的竞争条件

// sembuf 结构体的定义
struct sembuf {
	unsigned short int sem_num; //信号量中信号量的编号，从 0 开始计数
	short int sem_op; //操作类型，且其行为受到 sem_flg 成员的影响
	short int sem_flg; //可选值是 IPC_NOWAIT 和 SEM_INDO．前者含义是无论信号量操作是否成功，semop 调用都将立即返回．后者的含义是当进程退出是取消正在进行的 semop 操作
}
// 具体行为参见书 Linux 高性能服务器编程.pdf p264
```

semop 成功是返回 0, 失败则返回 -1 并设置 errno. 失败的时候 sem_ops 数组中指定的所有操作都不执行

##### semctl 系统调用
semctl 系统调用允许调用者对信号量进行直接控制
``` c++
#include <sys/sem.h>
int semctl(int sem_id, int sem_num, int command, ...);

// sem_id 参数是由 semget 调用返回的信号量标识符, 可以指定被操作的信号量集
// sem_num 参数指定被操作的信号量在信号量集中的编号
// command 参数指定要执行的命令
// 有的命令需要调用者传第 4 个参数. 第 4 个参数由用户自己定义

// semctl 成功时返回值取决于 command 参数. semctl 失败时返回 -1 并设置 errno
```

semctl 的 command 参数
![semctl command](/images/semctl_command.png)

在这些操作中, GETNCNT, GETPID, GETVAL, GETZCNT 和 SETVAL 操作是单个信号量, 它是指由标识符 sem_id 指定的信号量集中第 sem_num 个信号. 而其他操作针对的是整个信号量集, 此时 semctl 的参数 sem_num 被忽略

##### 特殊键值 IPC_PRIVATE
semget 的调用者可以给 key 参数传递一个特殊的键值 IPC_PRIVATE(其值为 0), 这样无论该信号量是否已经存在, semget 都将创建一个新的信号量. 使用该键值创建的信号量可以被其他进程访问而非像名字声称那样是进程私有的

在父子进程间使用一个 IPC_PRIVATE 信号量来同步
``` c++
#include <sys/sem.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>


union semun {
	int val;
	struct semid_ds *buf;
	unsigned short int *array;
	struct seminfo *__buf;
};


// op 为 -1 时执行 p 操作, op 为 1 时执行 v 操作
void pv(int sem_id, int op) {
	struct sembuf sem_b;

	sem_b.sem_num = 0; //信号量集中信号的标识符
	sem_b.sem_op = op;
	sem_b.sem_flg = SEM_UNDO; //当进程退出后取消正在进行的 semop 操作

	semop(sem_id, &sem_b, 1);
}


int main(int argc, char const *argv[]) {
	// 创建一个新的信号量集, 其中只有一个信号量, 权限为 0666
	int sem_id = semget(IPC_PRIVATE, 1, 0666);

	union semun sem_un;
	sem_un.val = 1;

	// SETVAL 操作将信号量的 semval 值设置为 semun.val, 同时内核数据中的 semid_ds.sem_ctime 被更新
	semctl(sem_id, 0, SETVAL, sem_un);

	pid_t id = fork();
	
	if(id < 0) {
		return 1;

	} else if(id == 0) {
		printf("child try to get binary sem\n");

		// 在父子进程间共享 IPC_PRIVATE 信号量的关键在于二者都可以操作信号量的标识符 sem_id

		pv(sem_id, -1);

		printf("child get the sem and would release it after 5 seconds\n");
		sleep(5);

		pv(sem_id, 1);

		exit(0);

	} else {
		printf("parent try get binary sem\n");

		pv(sem_id, -1);

		printf("parent get the sem and would release it after 5 seconds\n");
		sleep(5);

		pv(sem_id, 1);
	}

	// 捕获到子进程的 SIGCHLD 信号后彻底结束子进程
	waitpid(id, NULL, 0);
	// 删除信号量
	semctl(sem_id, 0, IPC_RMID, sem_un);

	return 0;
}
```

#### 共享内存
共享内存是最高效的 IPC 机制, 因为它不涉及进程之间的任何数据传输. 同时, 必须用其他辅助手段来同步进程对共享内存的访问, 否则会产生竞态条件. 因此共享内存通常和其他进程间通信方式一起使用

##### shmget 系统调用
shmget 系统调用创建一段新的共享内存, 或者获取一段已经存在的共享内存

``` c++
#include <sys/shm.h>
int shmget(key_t key, size_t size, int shmflg);
// key 参数是一个键值, 用来标识一段全局唯一的共享内存
// size 参数指定共享内存的大小, 单位是字节, 如果是创建新的东西内存, 则 size 值必须被指定. 如果是获取已经存在的共享内存则可以将 size 设置为 0
// shmflg 参数的使用和含义和 semget 系统调用的 sem_flags 参数相同. 不过 shmget 支持两个额外的标志: SHM_HUGETLB(使用大页面来为共享内存分配空间) 和 SHM_NORESERVE(不为共享内存保留 swap 空间, 当物理内存不足时对该共享内存执行写操作将触发 SIGSEGV 信号)
// shmget 成功时返回一个正整数值即共享内存的标识符. shmget 失败时返回 -1 并设置 errno
```

如果 shmget 使用于创建共享内存, 则这段共享内存的所有字节都被初始化为 0, 与之关联的内核数据结构 shmid_ds 将被创建并初始化. shmid_ds 结构体定义如下
``` c++
struct shmid_ds {
	struct ipc_perm shm_perm; // 共享内存的操作权限
	size_t shm_segsz; // 共享内存大小, 单位是字节
	__time_t shm_atime; // 对这段内存最后一次调用 shmat 的时间
	__time_t shm_dtime; // 对这段内存最后一次调用 shmdt 的时间
	__time_t shm_ctime; // 对这段内存最后一次调用 shmctl 的时间
	__pid_t shm_cpid; // 创建者的 PID
	__pid_t shm_lpid; // 最后一次执行 shmat 或 shmdt 操作的进程 PID
	shmatt_t shm_nattach; // 目前关联到此共享内存的进程数量
	// 省略一些填充字段
}
```

shmget 对 shmid_ds 结构体初始化包括:

* 将 shm_perm.cuid 和 shm_perm.uid 设置为调用进程的有效用户 ID

* 将 shm_perm.cgid 和 shm_perm.gid 设置为屌用进程的有效组 ID

* 将 shm_perm.mode 的最低 9 位设置为 shmflg 参数的最低 9 位

* 将 shm_segze 设置为 size

* 将 shm_lpid, shm_nattach, shm_atime, shm_dtime 设置为 0

##### shmat 和 shmdt 系统调用
共享内存被创建/获取之后, 我们不能立即访问它, 而是需要先将它关联到进程的地址空间中. 使用完共享内存之后, 还需要将它从进程地址空间中分离. 这两项任务分别由如下两个系统调用实现

``` c++
#include <sys/shm.h>
void *shmat(int shm_id, const void *shm_addr, int shmflg);
int shmdt(const void *shm_addr);

// shm_id 参数是由 shmget 调用返回的共享内存标识符
// shm_addr 参数指定共享内存关联到进程的那块地址空间, 最终的效果还受到 shmflg 参数的可选标志 SHM_RND 的影响
// shm_addr 通常被设置为 NULL, 表被关联的地址由操作系统选择, 以确保代码的可移植性
// 更多选择参见 Linux 高性能服务器编程.pdf p270

// shmat 成功时返回共享内存被关联到的地址, 失败则返回 (void*)-1 并设置 errno. shmat 成功时将修改内核数据结构 shmid_ds 的部分字段
// shm_nattach + 1
// shm_lpid 设置为调用的进程 PID
// shm_atime 设置为当前时间

// shmdt 函数将关联到 shm_addr 处的共享内存从进程中分离. 成功时返回 0, 失败则返回 -1 并设置 errno. shmdt 在成功调用时将修改内核数据结构 shmid_ds 的部分字段
// shm_nattach - 1
// shm_lpid 设置为调用进程的 PID
// shm_dtime 设置为当前时间
```

##### shmctl 系统调用
shmctl 系统调用控制共享内存的某些属性
``` c++
#include <sys/shm.h>
int shmctl(int shm_id, int command, struct shmid_ds *buf);
// shm_id 参数是由 shmget 调用返回的共享内存标识符
// command 参数指定要执行的命令
// shmctl 成功时返回值取决于 command 参数, shmctl 失败时返回 -1
```

shmctl 支持的所有命令
![shmctl command](/images/shmctl_command.png)
