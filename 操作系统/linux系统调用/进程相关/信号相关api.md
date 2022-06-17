# kill函数
可以发送任何信号给任何进程或者进程组
```c++
   #include <sys/types.h>
    #include <signal.h>

    int kill(pid_t pid, int sig);
```
如果pid 大于零，那么kill 函数发送信号号码sig 给进程pid。
如果pid 等于零，那么kill 发送信号sig 给调用进程所在进程组中的每个进程，包括调用进程自己。
如果pid小于零，kill发送信号sig 给进程组|pid|（pid 的绝对值）中的每个进程

 On success (at least one signal was sent), zero is returned.  On error, -1 is returned, and errno is set appropriately.
 
# alarm函数
进程安排内核向自己发送SIGALARM信号
```c++
 #include <unistd.h>

unsigned int alarm(unsigned int seconds);
```
**参数**：
- seconds：alarm函数安排内核在seconds秒后向调用进程发送信号

如果seconds == 0，则不会调度安排新的闹钟

在任何情况下，对alarm 的调用都将取消任何待处理的（pending)闹钟，并且返回任何待处理的闹钟在被发送前还剩下的秒数。如果没有待处理的闹钟，则返回0
