## fork函数
- 创建一个新进程,新进程复制调用进程
- 根据写时复制的机制,调用fork时仅仅需要**复制父进程的页表**以及**创建子进程的task struct**
`pid_t fork(void);`
**返回值**
成功时,子进程返回0;父进程返回子进程的进程id.失败时返回-1并设置错误码

**父进程的不会被子进程复制的东西**：
1. 子进程有自己独立且唯一的进程ID
2. 子进程不继承父进程的memory lock
3.  Process resource utilizations (getrusage(2)) and CPU time counters (times(2)) are reset to zero in the child.
4. 子进程的pending signals被初始化为空
5. 。。。
6. 子进程的终止信号是SIGCHLD

**fork函数的原理**
1. 当fork 函数被当前进程调用时，内核为新进程创建各种数据结构，并分配给它一个唯一的PID
2. 为了给这个新进程创建虚拟内存，它创建了当前进程的*mm_struct、区域结构和页表的原样副本*。它将两个进程中的每个页面都标记为只读，并将两个进程中的每个区域结构都*标记为私有的写时复制*。
3. 当fork 在新进程中返回时，新进程现在的虚拟内存刚好和调用fork 时存在的虚拟内存相同。
4. 当这两个进程中的任一个后来进行写操作时，*写时复制机制就会创建新页面，* 因此，也就为每个进程保持了*私有地址空间*的抽象概念

**可以通过画进程图来理解调用多个fork的过程**
![[Pasted image 20220616100625.png]]

## execve函数
在当前进程中加载filename指定的可执行文件，并且执行该可执行文件的程序
```c++
int execve(const char *filename, char *const argv[],
                  char *const envp[]);
```
**参数**
- filename : 可执行文件的名字
- argv[]:该程序需要的参数
- envp[]:该程序需要的环境变量
![[Pasted image 20220616152028.png]]

**返回值**
函数调用成功时不会返回，而是将新程序的正文段、数据段、栈替换原来的程序后继续执行

**execve的内部流程：**
1. 删除当前进程虚拟地址的用户部分中的已存在的区域结构。（虚拟地址的内核部分不需要变化）
2. 映射私有区域，为新程序的代码、数据、bss 和栈区域创建新的区域结构。所有这些新的区域都是私有的、写时复制的。代码和数据区域被映射为a.out 文件中的.text和.data 区。bss 区域是请求二进制零的，映射到匿名文件，其大小包含在a.out中。栈和堆区域也是请求二进制零的，初始长度为零。图9-31 概括了私有区域的不同映射
	![[Pasted image 20220611165615.png]]
3. 映射共享区域。如果该可执行文件需要和共享目标文件链接，比如说标准c库libc.so，那么这些对象都是动态链接这个程序，然后再映射到进程的共享内存区域
4. 设置程序计数器
	设置当前进程上下文的程序计数器，使其指向新程序的第一行执行代码

## exit函数
以status退出状态来终止进程(将status & 0377返回给父进程,父进程调用wait获取)
```c++
 #include <stdlib.h>

void exit(int status);
```

**可以修改exit函数的行为：**
可以使用atexit(3) or on_exit(3)来向exit注册函数，exit调用时按照函数注册的相反的顺序执行这些注册函数

## daemon函数
调用daemon使得当前进程与控制终端分离,作为系统守护进程运行在后台
```c++
int daemon(int nochdir, int noclose);
```
**参数**
- nochdir : 为0时,改变当前进程的工作目录为根目录"/";为1时,不改变当前工作目录
- noclose : 为0时,重定向标准输入,标准输出和标准错误到/dev/null;为1时不做任何改变
**返回值**
成功时返回0,失败时返回-1并设置errno

## getpid和getppid
```c++
   #include <sys/types.h>
   #include <unistd.h>

	pid_t getpid(void);
    pid_t getppid(void);
```

- getpid返回当前进程的进程id
- getppid返回当前进程的父进程的进程id(调用fork的,如果父进程已经结束,则返回初始进程的进程id)


## waitpid
默认情况下（当options=0 时）， waitpid 挂起调用进程的
执行，直到它的等待集合(wait set)中的一个子进程终止
```c++
#include <sys/types.h>
#include <sys/wait.h>
 pid_t waitpid(pid_t pid, int *wstatus, int options);
```
**参数**
- pid : 等待的子进程的PID
	若pid>0，则等待集合就是pid指定的这个单独的子进程
	若pid= -1,则等待集合就是该父进程的所有子进程
- wstatus: 存放子进程的结束状态
![[Pasted image 20220616111116.png]]
- options:通过设置options修改waitpid的默认行为
	- WNOHANG ： 如果等待集合中没有任何子进程终止了，那么调用进程不会陷入等待，会直接返回（返回值为0）
	- WUNTRACED：挂起调用进程的执行，直到等待集合中的一个进程变成已终止或者*被停止*
	- WCONTINUED：挂起调用进程的执行，直到等待集合中一个正在运行的进程终止或等待集合中一个被停止的进程收到SIGCONT 信号*重新开始执行*


**返回值**
成功时返回成功回收的子进程的ID
![[Pasted image 20220616111139.png]]

## wait
wait挂起调用进程，直到它的等待集合(wait set)中的一个子进程终止
```c++
 pid_t wait(int *wstatus);
```

等价于调用 `waitpid(-1, wstatus, 0)`

## sleep
将一个进程挂起一段时间
```c++
#include <unistd.h>
unsigned int sleep(unsigned int secs);
```
**返回值**
如果请求挂起的时间到了，sleep返回0
如果在挂起的中途被信号提前唤醒了，则返回还剩下的需要休眠的秒数

## pause
让调用进程休眠，直到收到一个信号，进程被信号终止，或者从信号处理函数返回
```c++
  #include <unistd.h>

	int pause(void);
```
**返回值**
返回-1并设置errno为EINTR