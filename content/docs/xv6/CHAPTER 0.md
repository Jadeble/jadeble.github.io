---
title: "CHAPTER 0"
date: 2026-04-16
weight: 1
toc: true
---

# 操作系统接口
![操作系统](/image/1.png)

内核是向其他运行中程序提供服务的特殊程序

内核拥有实现保护机制的硬件权限

⭐在user/user.h中可以看到全部的系统调用函数，之后所有的实验实现全部基于这20个函数
![系统调用](/image/2.png)

# 进程和内存

用户内存空间（指令、数据、栈）；仅对内核课件的进程状态

进程通过`fork`创建子进程，`fork`在父进程和子进程中一次调用两次返回

对于父进程返回子进程的`pid`,对于子进程返回0

``` c
int pid;
pid = fork();
if(pid > 0){
    printf("parent: child=%d\n", pid);
    pid = wait();
    printf("child %d is done\n", pid);
} else if(pid == 0){
    printf("child: exiting\n");
    exit();
} else {
    printf("fork error\n");
}
```

⚠️对于这个代码在执行后输出混乱，是因为子进程和父进程交替进程输出导致的，在后面管道部分可以进行修复

`exit`会结束调用它的进程，并释放占用的资源

`exec`从文件中读取内存镜像，并将其替换到调用它的进程的内存空间

ℹ️使用`fork`创建子进程后，将子进程替换为目标进程，常见于`shell`开发中

``` c
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\n");
```

该段代码将调用从程序替换为`echo`

# I/O和文件描述符

**文件描述符**是一个整数，它代表了一个进程可以读写的被内核管理的对象。

| FD序号 | 含义      |
| ---- | ------- |
| 0    | 标准输入：键盘 |
| 1    | 标准输出：屏幕 |
| 2    | 标准错误：屏幕 |



`read(fd, buf, n)` 从 `fd` 读最多 n 个字节（`fd` 可能没有 n 个字节），将它们拷贝到 `buf` 中，然后返回读出的字节数。当没有数据可读时，`read` 就会返回0

⚠️每一个指向文件的文件描述符都和一个偏移关联。`read` 从当前文件偏移处读取数据，然后把偏移增加读出字节数。紧随其后的 `read` 会从新的起点开始读数据。

💡 `read` 可以记录该文件上次读取的位置

💡 参数n可以大于文件长度

`write`与`read`特性相同

`cat`的实现本质

```c
char buf[512];
int n;

for(;;){
    n = read(0, buf, sizeof buf);
    if(n == 0)
        break;
    if(n < 0){
        fprintf(2, "read error\n");
        exit();
    }
    if(write(1, buf, n) != n){
        fprintf(2, "write error\n");
        exit();
    }
}
```

系统调用 `close` 会释放一个文件描述符，使得它未来可以被 `open`, `pipe`, `dup` 等调用重用。

⚠️一个新分配的文件描述符永远都是当前进程的最小的未被使用的文件描述符。

🤓文件描述符和 `fork` 的交叉使用使得 I/O 重定向能够轻易实现。`fork` 会复制父进程的文件描述符和内存，所以子进程和父进程的文件描述符一模一样。`exec` 会替换调用它的进程的内存但是会保留它的文件描述符表。

💡 `fork` 一个进程，重新打开指定文件的文件描述符，然后执行新的程序。

简化版的 shell 执行 `cat<input.txt` 的代码:

```c
char *argv[2];
argv[0] = "cat";
### argv[1] = 0;
if(fork() == 0) {
    close(0);
    open("input.txt", O_RDONLY);
    exec("cat", argv);
}
```

程序实现流程详解
>**`if(fork() == 0) { ... }`**
>**动作**：Shell 调用 `fork()`。此时系统克隆了一个和父进程一模一样的子进程。
>**状态**：子进程的 FD 表里，0 依然指向键盘，1 指向屏幕。

>**`close(0);`**
>**动作**：子进程主动关闭了它的“标准输入”。
>**状态**：此时，子进程的 FD 表中，**0 位置空出来了**，它是当前最小的可用编号。

>**`open("input.txt", O_RDONLY);`**
>**动作**：子进程打开 `input.txt`。
>**关键点**：根据“最小可用”规则，操作系统发现 0 是空的，于是**把 `input.txt` 分配给了 FD 0**。
>**状态**：现在子进程以为它的“标准输入”就是这个文件。**这就是重定向发生的瞬间！**

>**`exec("cat", argv);`**
>**动作**：子进程把自己的“灵魂”换成了 cat 程序。
>**关键点**：exec 不会重置 FD 表。
>**结果**：cat 启动后，它的本职工作是从 **FD 0** 读取数据。它并不知道 FD 0 已经被换成了 `input.txt`，它依然像往常一样从 0 号位读数据。
>**现象**：cat 读出的就是文件的内容，而不是键盘输入。

⚠️ `fork`创建进程复制文件描述符，每个文件的偏移值在父子进程之间共享

```c
if(fork() == 0) {
    write(1, "hello ", 6);
    exit();
} else {
    wait();
    write(1, "world\n", 6);
}
```

`dup` 复制一个已有的文件描述符，返回一个指向同一个输入/输出对象的新描述符。这两个描述符共享一个文件偏移。正如被 `fork` 复制的文件描述符类似，只是并不会创建子进程。

```c
fd = dup(1);
write(1, "hello", 6);
write(fd, "world\n", 6);
```

💡两次`write` 的文件描述符不同，但是操作的是同一个文件

指令实现`ls existing-file non-exsiting-file > tmp1 2>&1`

ℹ️ 解释：`existing-file` 的名字和 `non-exsiting-file` 的错误输出都将出现在 `tmp1` 中

```c
// 假设是在 fork() 产生的子进程中
if(fork() == 0) {
    // 1. 先处理 "> tmp1" (重定向标准输出)
    close(1);               // 关闭当前标准输出（通常是屏幕）
    open("tmp1", O_WRONLY|O_CREATE|O_TRUNC); // 此时 FD 1 指向了 tmp1

    // 2. 再处理 "2>&1" (重定向标准错误到标准输出)
    close(2);               // 关闭当前标准错误
    dup(1);                 // 复制 FD 1。因为 2 是当前最小可用 FD，所以 FD 2 现在也指向 tmp1
                            // 并且 FD 2 和 FD 1 共享文件偏移量！

    // 3. 执行程序
    char *argv[] = {"ls", "existing-file", "non-existing-file", 0};
    exec("ls", argv);
}
```

# 管道

管道是一个小的内核缓冲区，它以文件描述符对的形式提供给进程

对于管道`p`，`p[0]`用于读取管道内容(也可以理解为管道向外输出)`p[1]`用于向管道内填入内容

```c
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if(fork() == 0) {
    close(0);    // 关掉键盘输入
    dup(p[0]);   // 把管子的读端复制到 0 号位置
    close(p[0]); // 原始的 p[0] 没用了，关掉
    close(p[1]); // ！！！重点：关掉子进程里的写端
    exec("/bin/wc", argv);
} else {
    write(p[1], "hello world\n", 12);
    close(p[0]); // 父进程不读，关掉
    close(p[1]); // 发完数据了，关掉写端
}
```

💡解析：

>子进程中，先关闭标准输入0，然后将FD 0连接到`p[0]`，此时子进程的输入变为读取管道
>然后依次关闭子进程中的**管道读**和**管道写**

⚠️ 使用`fork`之后，系统中共有四个描述符向这根管子（父读、父写、子读、子写）。

❓ 为什么要关闭子进程的管道读`p[0]`？

💡 **管道是引用计数的** 只要系统中还有一个进程的某个 FD 指向该管道，该管道就不会消失。前一步操作中，将原本的管道读使用`dup`复制到了`FD 0`，此时可以释放原本的管道读。不释放应该也没关系，只是可能不安全。

🔑 及时关闭冗余 FD 是为了防止**文件描述符泄漏**（每个进程的 FD 数量有限制），同时也是为了保持进程状态的整洁，避免误操作。

❓ 为什么要关闭子进程的管道写`p[1]`？

💡 此处涉及到read读管道时，什么时候返回 **`EOF`** 。当管道**无数据**且**无管道写**时，read会返回 **`EOF`** 。只要系统中还有一个进程的某个 FD 指向该管道的写端，内核就不会向读端发送 `EOF`。此时若不关闭子进程中的管道写，那么当父进程关闭管道写时，由于子进程本身还有一个管道写，read无法返回 **`EOF`**，导致`wc`无法结束循环。

>父进程中，使用管道写，向管道中加入数据，然后关闭管道读和管道写，确保read可以返回。

🔥 使用read在读管道时的特性，我们可以对前面的[进程和内存](#进程和内存)部分的代码进行修正：

> ```c
> int pid;
> int p[2]; // 管道文件描述符
> if(pipe(p) < 0){
> 	printf("pipe error\n");
> 	exit(0);
> }
> pid = fork();
> if(pid > 0){
> 	// 父进程
> 	printf("parent: child=%d\n", pid);
> 	// 打印完了，往管道写一个字节，通知子进程
> 	write(p[1], "x", 1);
> 	pid = wait(0);
> 	printf("child %d is done\n", pid);
> } else if(pid == 0){
> 	// 子进程
> 	char buf;
> 	// 阻塞在这里，直到从管道读到一个字节
> 	read(p[0], &buf, 1);
> 	printf("child: exiting\n");
> 	exit(0);
> } else {
> 	printf("fork error\n");
> 	exit(1);
> }
> exit(0);
> ```

在父进程中创建管道，使用fork产生子进程后，管道将父子进程联通。当子进程执行到read时，由于管道内没有数据，read会暂时阻塞程序进行，当管道中出现数据后，read会返回读到的数据数量，然后就可以执行后续的操作

⚠️ read并不一定要满足n个数据，而是有数据就会返回 

❓ 程序如何判断子进程还是父进程？为什么看似一份代码两者执行的不一样？

ℹ️ 子进程在fork处产生，子进程和父进程都会执行这一份代码。在父进程视角，fork返回了子进程的`pid`，因此不满足条件，进入else块；在子进程视角，fork返回0，满足条件，进入if块。子进程在最后使用exec替换了内存中的数据，因此不会进入属于父进程的else中。

# 文件系统

`xv6` 文件系统提供文件和目录，文件就是一个简单的字节数组，而目录包含指向文件和其他目录的引用。

👑 这个简单，不就是`cd`嘛

```c
chdir("/a");
chdir("b");
open("c", O_RDONLY);

open("/a/b/c", O_RDONLY);
```

🔑 三种调用

```c
mkdir("/dir");
fd = open("/dir/file", O_CREATE|O_WRONGLY);
close(fd);
mknod("/console", 1, 1);
```

`unix`中一切皆文件，`mknod`创建的就是设备文件，其中两个数据依次代表主设备号和辅设备号，理解为：类别和序号。当需要打开这类文件时，内核将读写转移到内核设备上实现。

`fstat`返回文件信息结构体

```c
#define T_DIR  1
#define T_FILE 2
#define T_DEV  3
// Directory
// File
// Device
     struct stat {
       short type;  // Type of file
       int dev;     // File system’s disk device
       uint ino;    // Inode number
       short nlink; // Number of links to file
       uint size;   // Size of file in bytes
};
```

同一个文件`inode`可以有多个名字，类似`软连接`，不同名字对应同一个文件
```c
open("a", O_CREATE|O_WRONGLY);
link("a", "b");
```

系统调用 `unlink` 从文件系统移除一个文件名。一个文件的 `inode` 和磁盘空间只有当它的链接数变为 0 的时候才会被清空，也就是没有一个文件再指向它。

⚠️ **核心区分：全局状态 vs 私有状态**

1. **全局持久化状态 (Global Persistent State)**
   
    - **范畴**：磁盘上的目录结构、文件内容。
      
    - **代表指令**：mkdir, rm, open(O_CREATE), write。
      
    - **本质**：操作的是**内核维护的全局资源（磁盘/文件系统）**。
      
    - **表现**：子进程操作，全系统生效。就像在公共黑板上写字，谁写大家都能看到。
    
2. **进程私有状态 (Per-process Private State)**
   
    - **范畴**：当前工作目录 (CWD)、进程内存（变量）、文件描述符表。
      
    - **代表指令**：cd, x = 1 (赋值), close。
      
    - **本质**：操作的是**进程控制块 (PCB) 或用户空间内存**。
      
    - **表现**：子进程操作，父进程无感。就像子进程在自己的笔记本上改了笔记，父进程的笔记本不会变。
    
3. **临时共享状态 (Transient Shared State)**
   
    - **范畴**：管道 (Pipe)、共享的文件偏移。
      
    - **本质**：存在于内核 RAM 中的缓冲区或对象。
      
    - **表现**：仅在有“血缘关系”的父子进程间通过继承 FD 来共享，生命周期随进程结束而结束。
