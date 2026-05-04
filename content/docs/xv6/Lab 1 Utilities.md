---
title: "Lab 1 Utilities"
date: 2026-04-19
weight: 2
toc: true
---

# `xv6` 部署

```bash
git clone git://g.csail.mit.edu/xv6-labs-2023
cd xv6-labs-2023
git branch -a #查看分支是否在对应实验分支
make qemu
```

该仓库对应的课程：[点击打开](https://pdos.csail.mit.edu/6.S081/2023/schedule.html "MIT 6.1810 2023")

# Task 1 : Sleep

>Implement a user-level sleep program for `xv6`, along the lines of the UNIX sleep command. Your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the `xv6` kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c.

>为 `xv6` 实现一个用户级的 sleep 程序，类似于 UNIX 的 sleep 命令。你的 sleep 应该暂停用户指定的时钟滴答数。一个滴答是 `xv6` 内核定义的时间概念，即计时芯片两次中断之间的时间。你的解决方案应该放在文件 user/sleep.c 中。

代码实现
```c
#include"kernel/types.h"
#include"user/user.h"
int main(int argc, char *argv[]) {
	if (argc != 2) {
		fprintf(2, "Usage: sleep <ticks>\n");
		exit(1);
	}
	int ticks = atoi(argv[1]);
	sleep(ticks);
	exit(0);
}
```

- 解释：
	终端中执行sleep调用时，进入该调用的main函数。
	main函数的参数`argc`、`*argv` 分别代表传入参数的数量和指向变量的指针数组。
	
	以该任务为例，题目要求用户需要传入指定的`时钟滴答数`，当用户在终端中只执行 **`sleep`** 时`argc=1`，因此可以通过`argc`的值来判断用户输入了几个参数
	
	`*argv[]`中存储用户输入的指令和参数，当用户只输入 **`sleep`** 时，`argv`中只有`argv[0]="sleep"`，当用户输入 **`sleep 10`** 时，则会增加`argv[1]="10"`。
	
	由于此时`argv`中参数均为字符类型，因此需要使用`atoi`将其转为`int`型变量后传入到sleep中

- 该代码中存在的安全隐患：未对用户输入的参数进行检查其是否是数字
- 在程序开发中，应该注重对输入进行安全检查，非法的输入尽管不报错，但是在未知情况下可能会导致内核遭受攻击

输入检测版本代码
```c
#include "kernel/types.h"
#include "user/user.h"
// 辅助函数：检查字符串是否全是数字
int is_numeric(char *s) {
	if (*s == '\0') return 0;
	while (*s) {
		if (*s < '0' || *s > '9') return 0;
		s++;
	}
	return 1;
}

int main(int argc, char *argv[]) {
// 1. 参数个数检查（你已经做了，很好）
	if (argc != 2) {
		fprintf(2, "Usage: sleep <ticks>\n");
		exit(1); // 返回 1 表示错误
	}
	// 2. 检查输入是否为合法的正整数
	if (!is_numeric(argv[1])) {
		fprintf(2, "sleep: invalid time interval '%s', must be a positive integer\n", argv[1]);
		exit(1);
	}
	int ticks = atoi(argv[1]);
	// 3. 处理 ticks 为 0 的情况（虽然 atoi 可能返回 0，但显式检查更好）
	if (ticks <= 0) {
		// 如果输入是 0，可以直接退出，或者打印提示
		exit(0);
	}
	// 4. 执行系统调用
	if (sleep(ticks) < 0) {
		fprintf(2, "sleep: system call failed\n");
		exit(1);
	}
	// 5. 成功完成任务，返回 0
	exit(0);
}
```

使用仓库自带的测试工具测试代码
```bash
(base) robot@robot-MS-7E25:~/xv6-labs-2023$ ./grade-lab-util sleep
make: “kernel/kernel”已是最新。
== Test sleep, no arguments == sleep, no arguments: OK (1.2s) 
== Test sleep, returns == sleep, returns: OK (1.0s) 
== Test sleep, makes syscall == sleep, makes syscall: OK (1.1s)
```

# Task 2 : PingPong

>Write a user-level program that uses xv6 system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "`<pid>`: received ping", where `<pid>` is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "`<pid>`: received pong", and exit. Your solution should be in the file user/pingpong.c.

>编写一个用户级程序，使用 xv6 系统调用在两个进程之间通过一对管道进行字节“乒乓”传递，每个方向各用一个管道。父进程应向子进程发送一个字节；子进程应打印“`<pid>`: received ping”，其中 `<pid>` 是其进程 ID，将字节写回父进程的管道，然后退出；父进程应从子进程读取字节，打印“`<pid>`: received pong”，并退出。你的解决方案应保存在文件 user/pingpong.c 中。

该任务用到了 [CHAPTER 0](./CHAPTER 0) 中提到的管道的进程同步作用,代码实现如下

```c
#include"kernel/types.h"
#include"user/user.h"

int main(int argc, char *argv[]) {
	int pid;
	int pipe1[2], pipe2[2];
	// 创建两个管道，pipe1 用于父进程向子进程发送消息，pipe2 用于子进程向父进程发送消息
	pipe(pipe1);
	pipe(pipe2);
	char buf[] = {'a'};
	int ret = fork();
	if (ret == 0){
	// 子进程
		pid = getpid();
		close(pipe1[1]); // 关闭写端
		close(pipe2[0]); // 关闭读端
		read(pipe1[0], buf, 1); // 从 pipe1 读数据
		printf("child %d: received ping\n", pid);
		write(pipe2[1], "x", 1); // 向 pipe2 写数据
		close(pipe1[0]);
		close(pipe2[1]);
		exit(0);
	}
	else if (ret > 0){
		// 父进程
		pid = getpid();
		close(pipe1[0]); // 关闭读端
		close(pipe2[1]); // 关闭写端
		write(pipe1[1], "x", 1); // 向 pipe1 写数据
		read(pipe2[0], buf, 1); // 从 pipe2 读数据
		printf("parent %d: received pong\n", pid);
		close(pipe1[1]);
		close(pipe2[0]);
		wait(0); // 等待子进程结束
	}
	else {
		printf("fork error\n");
		exit(1);
	}
	exit(0);
}
```


# Task 3 : Primes

>Write a concurrent prime sieve program for xv6 using pipes and the design illustrated in the picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text. This idea is due to Doug McIlroy, inventor of Unix pipes. Your solution should be in the file user/primes.c.

>使用管道和本页中间图片以及周围文本所示的设计，为 xv6 编写一个并发素数筛程序。这个想法归功于 Unix 管道的发明者 Doug McIlroy。你的解决方案应位于文件 user/primes.c 中。

该任务中使用和核心算法是埃氏筛法，实现方法是使用pipe的多进程并发计算

简单的流程图如下：

![流程图](/image/3.png)

实现思路使用递归思想

main函数中的任务是，向第一个子进程中输入2到35的数字，然后从第一个子进程开始，之后的子进程将执行相同的操作：

将接收到的第一个数字默认的素数(当然算法原理上是合理的)，然后判断后续的数字能否被第一个数字整除，如果可以整除，则不是素数；如果不能整除，则可能是素数，将其发送到下一个子进程中进行判断。

实现代码如下：
```c
# include "kernel/types.h"
# include "user/user.h"

void sieve(int fd){
	int prime;
	//判断是否接收到上一个进程的数字，若未接收到，说明已经筛选结束
	if (read(fd, &prime, sizeof(int)) <= 0){
		exit(0);
	}
	//第一个数字一定为素数，进行打印
	printf("prime %d\n", prime);
	int pipes[2];
	pipe(pipes);
	
	if (fork() == 0){
		//子进程
		close(fd);//关闭旧的读端
		close(pipes[1]);//关闭新的写端
		sieve(pipes[0]);//进入递归
	}
	else{
		//当前进程进行筛选
		close(pipes[0]);//关闭新的读端
		int num;
		//循环读取来自父进程的数据
		while (read(fd, &num, sizeof(int)) > 0){
			//进行筛选，如果无法被素数整除，则发送到子进程
			if (num % prime != 0){
				write(pipes[1], &num, sizeof(int));
			}
		}
		close(fd);//关闭读端口
		close(pipes[1]);//关闭新的写端口
		wait(0);
		exit(0);
	}
}

int main(int argc, char *argv[]){
	int pipes[2];
	pipe(pipes);
	if (fork() == 0){
		//子进程
		close(pipes[1]);//关闭不用的写端点
		sieve(pipes[0]);//开始递归
	}
	else{
		close(pipes[0]);
		//向第一个子进程中写2到35的数字
		for (int i = 2; i <= 35; i++){
			write(pipes[1], &i, sizeof(int));
		}
		close(pipes[1]);//关闭写端
		wait(0);//等待子进程结束
	}
	exit(0);
}
```

# Task 4 : Find

>Write a simple version of the UNIX find program for xv6: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.

>为 xv6 编写一个简单版本的 UNIX find 程序：在目录树中查找具有特定名称的所有文件。你的解决方案应位于文件 user/find.c 中。

实现流程：判断用户输入是否足够 -> 获取文件属性 -> 判断是否是目录，不是目录则直接退出 -> 循环读取目录中内容，如果是文件则判断是否是目录文件，如果是目录则进行递归

具体实现代码：
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

void find(char *current_path, char *target){
	char buf[512], *p;
	int fd;
	struct dirent de;
	struct stat st;
	//打开指定路径
	if((fd = open(current_path, O_RDONLY)) < 0){
		fprintf(2, "find: cannot open %s\n", current_path);
		return;
	}
	//获取文件状态
	if(fstat(fd, &st) < 0){
		fprintf(2, "find: cannot stat %s\n", current_path);
		close(fd);
		return;
	}
	//检查提供的路径是否是目录
	if (st.type != T_DIR) {
		fprintf(2, "find: %s is not a directory\n", current_path);
		close(fd);
		return;
	}
	//检查路径长度是否超过缓冲区大小
	if(strlen(current_path) + 1 + DIRSIZ + 1 > sizeof buf){
		fprintf(2, "find: path too long\n");
		close(fd);
		return;
	}
	//累积路径
	strcpy(buf, current_path);//将当前路径复制到buf中
	p = buf + strlen(buf);//使指针p指向buf的末尾(字符串结束符/0位置)
	*p++ = '/';//添加/之后再向后一位
	//读取目录项
	while(read(fd, &de, sizeof(de)) == sizeof(de)){
		//判断目录是否为空或是否已被删除
		if(de.inum == 0)
			continue;
		//跳过当前目录和父目录
		if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
			continue;
		//构建完整路径
		memmove(p, de.name, DIRSIZ);
		//在文件名可能的最长位置写入0，若文件名长度超过DIRSIZ时可能末尾没有0
		p[DIRSIZ] = 0; 
		//获取目录项的状态
		if(stat(buf, &st) < 0){
			fprintf(2, "find: cannot stat %s\n", buf);
			continue;
		}
		if(!strcmp(de.name, target)){
			printf("%s\n", buf);
		}
		//如果是目录，递归查找
		if(st.type == T_DIR){
			find(buf, target);
		}
	}
	close(fd);
}

int main(int argc, char *argv[]){
	if(argc != 3){
		fprintf(2, "Usage: find <path> <filename>\n");
		exit(1);
	}
	find(argv[1], argv[2]);
	exit(0);
}
```

- 关键点：
	`struct dirent de`是目录项，记录该文件夹中有哪些文件
	
	`struct stat st`文件属性，记录文件是设备、目录、还是文件
	
	`p`在步骤`p[DIRSIZ] = 0`中不会后移，相对寻址，操作的是指针p后的`DIRSIZ`位置处的数值
	
	正因为指针p不会动，因此每次添加路径时反复覆盖p后的内容，在递归后，使用strlen重新计算写入长度，因此不用担心在`memmove(p, de.name, DIRSIZ)`中强制写如`DIRSIZ`长度导致长度溢出

# Task 5 : Xargs

>Write a simple version of the UNIX xargs program for xv6: its arguments describe a command to run, it reads lines from the standard input, and it runs the command for each line, appending the line to the command's arguments. Your solution should be in the file user/xargs.c.

>为 xv6 编写 UNIX xargs 程序的简化版本：它的参数描述要运行的命令，它从标准输入读取行，并对每一行运行该命令，将该行附加到命令的参数中。你的解决方案应位于文件 user/xargs.c 中。

`xargs`功能解释：**从标准输入（stdin）读取行，并为每一行运行一次指定的命令，同时将读取到的行作为该命令的参数。**

直观解释：

	输入`echo hello | xargs echo bye`
	
	1. echo hello 会向标准输出打印 hello。
	2. | 管道符将这个 hello 传给 xargs 的标准输入。
	3. xargs 接收到 hello。
	4. xargs 后面跟着的命令是 echo bye。
	5. xargs 将接收到的 hello 拼在后面，最终执行：echo bye hello。
	
	输入`find . b | xargs grep hello`
	
	如果 find 找到了三个文件 ./b, ./a/b, ./a/aa/b。 
	
	xargs 会分别执行三次命令：
	- grep hello ./b
	- grep hello ./a/b
	- grep hello ./a/aa/b

总结：将前面指令的输出通过管道拼接到xargs后的指令的最后一个参数位置。

根据以上得出xargs代码的整体逻辑：**循环读取输入 -> 拼接参数 -> fork + exec -> wait**。这个过程中只需要关注xargs后的指令，不需要考虑管道前的指令

需要使用到的是`fork`和`exec`，用法见见 [CHAPTER 0 中的进程和内存部分](CHAPTER 0.md#进程和内存)

代码实现：
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"
  
int main(int argc, char *argv[]){
	char buf[512];//存储管道前指令的输出内容
	char *cmd_args[MAXARG];//存储拼接后的指令
	int i;
	
	if(argc < 2) {
		fprintf(2, "Usage: xargs command [args...]\n");
		exit(1);
	}
	//将完整输入参数移除xargs后存入到新的参数数组
	for(i = 1; i < argc; i++){
		cmd_args[i-1] = argv[i];
	}
	
	while(1){
		int n = 0;
		char ch;
		//循环读取管道前指令的输出内容中的一行
		while(read(0, &ch, 1) == 1){
			if(ch == '\n') break;
			buf[n++] = ch;
		}
		//如果未读到数据则退出整个循环
		if(n == 0) break;
		buf[n] = 0;//手动为指令封口
		//echo bye | xargs echo hello
		//前面输出bye，拼接后变为echo hello bye
		//argc-1是因为传入搭配exec的指令中需要剔除xargs
		cmd_args[argc-1] = buf;//将前面指令的输出内容作为最后一个参数
		cmd_args[argc] = 0;//手动为指令封口
		if(fork() == 0){
			//子进程
			//cmd_args[0]为xargs后的第一个参数，也就是新进程应该执行的内容
			//cmd_args中为完整的参数
			exec(cmd_args[0], cmd_args);//进程内容替换
			//若未完成子进程内容的替换才会执行下面的内容
			fprintf(2, "xargs: exec %s failed\n", cmd_args[0]);
			exit(1);
		}else{
			wait(0);
		}
	}
	exit(0);
}
```