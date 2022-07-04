# socket本身
## socket函数
创建通信的endpoint，创建成功返回文件描述符，文件描述符的值是当前系统最小可用的值
`int socket(int domain, int type, int protocol);`
**参数**
- domain : 通信协议族/域，比如unix ipv4 ipv6等
- type : 套接字类型
	最常用到的SOCK_STREAM流类型和SOCK_DGRAM数据报类型
	从linux2.6.7开始新增了两个选项：
	1.  SOCK_NONBLOCK : 创建套接字时设置套接字为非阻塞的
	2. SOCK_CLOEXEC ： 创建套接字时设置套接字为CLOSE ON EXEC[[close on exec]]
- protocol ： 指定该套接字使用的具体的协议，默认为0
**返回值**
成功时返回文件描述符的值，失败返回-1并设置errno

## connect函数
使sockfd指代的套接字向addr指示的地址发起连接
```c++
int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
```
**参数**
- sockfd : 发起连接的套接字描述符
- addr : 要连接到的套接字的地址
- addrlen ： 地址长度
**返回值**
成功时返回0，失败返回-1并设置errno

**提示**
1. 通常基于连接的协议（如TCP）只能connect成功一次
2. 对于数据报协议（如UDP），addr指向数据报发送到的位置，可以connect多次，每一次connect就是更改数据报发送到的位置
3.  ECONNREFUSED ： connect返回此错误表明要连接的地址addr没有监听套接字监听
4. EINPROGRESS ：设置连接套接字为**非阻塞时**调用connect函数，如果返回-1，并且errno为该错误，并不是说连接失败了，而是表明连接正在进行中，后面可以使用select或者poll来判断socket是否可写，如果可写，说明连接成功
5. ETIMEDOUT ： 连接超时。系统有默认超时时间，connect会间隔几秒尝试几次，直到到达最大超时时间，如果仍未连接成功，则返回该错误
6. 如果connect失败，连接套接字的状态是不确定的，重新连接时应该关闭该套接字再重新创建一个

## bind函数
给套接字绑定一个地址
```c++
int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
```
**参数**
- sockfd : 发起连接的套接字描述符
- addr : 要连接到的套接字的地址
- addrlen ： 地址长度
**返回值**
成功时返回0，失败返回-1并设置errno

## listen函数
将sockfd指代的socket标记为监听套接字，该套接字后面便可以调用accept监听连接
```c++
int listen(int sockfd, int backlog);
```
**参数**
- sockfd : 套接字描述符
- backlog：该监听套接字连接队列的最大长度，如果客户端某个连接到达时连接队列已满，则客户可能会收到 ECONNREFUSED；如果底层协议支持重传，则会忽略此次连接，等待客户下次重连
**返回值**
成功时返回0，失败返回-1并设置errno

## accept函数
从监听套接字sockfd连接队列取下第一个连接，并创建一个新的连接套接字，返回该套接字的fd
```c++
 int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

int accept4(int sockfd, struct sockaddr *addr,
                   socklen_t *addrlen, int flags);
```
**参数**
- sockfd : 监听套接字描述符
- addr ：客户端地址
- addrlen ：客户端地址长度
**返回值**
成功时返回新套接字的fd，失败返回-1并设置errno

**notes**
1. 如果监听套接字未设置未non-blocking，并且此时连接队列上没有连接，accept会阻塞
2. 如果监听套接字设置为non-blocking，并且此时连接队列上没有连接，accept会失败并且返回EAGAIN or EWOULDBLOCK

accept4多了一个flags参数，可以设置监听套接字的属性，比如SOCK_NONBLOCK和SOCK_CLOEXEC 

## getsockopt和setsocketopt函数
管理sockfd指代的套接字相关的选项,选项可能是多个协议层相关的，通常是在sockets API层面（level = SOL_SOCKET）
```c++
int getsockopt(int sockfd, int level, int optname,
                      void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
                      const void *optval, socklen_t optlen);
```
**参数**
- sockfd : socket的fd
- level : 设置哪个协议层面的选项
- optname : 选项名
- optval : 选项值
- optlen : 选项值的长度

## shutdown函数
关闭双端连接的一半
```c++
#include <sys/socket.h>

int shutdown(int sockfd, int how);
```
**参数**:
- sockfd: 套接字fd
- how：以哪种方式关闭
	SHUT_RD ： 关闭读端
	SHUT_WR ： 关闭写端
	SHUT_RDWR : 关闭读写端

# 文件描述符相关
## close函数
- 关闭一个文件描述符fd,close后fd不再引用任何一个文件，因此在调用close后该fd就可以被重用了。close fd后和该fd关联的文件的record lock就被释放了
- 如果该fd是最后一个引用该文件的fd，则在关闭该fd后，该文件的打开资源都被释放
`int close(int fd);`

**notes**
- 因为操作系统内核缓冲区的存在，close成功并不保证文件的内容被刷入磁盘
- 如果fd设置了close-on-exec标志[[close on exec]]，则在execve被成功调用时，该fd会被自动关闭
- 多线程下可能会有线程安全问题。（一个线程调用close时另一个线程在系统调用中正在使用该fd）

## ioctl函数
维护修改指定文件的参数
```c++
int ioctl(int fd, unsigned long request, ...);
```

## fcntl函数
对打开的文件描述符进行一些操作
`int fcntl(int fd, int cmd, ... /* arg */ );`
**参数**
- fd : 打开的文件描述符
- cmd : 指示对该文件描述符做哪种操作
- arg : 做操作时可能需要的参数

**可能的操作**(cmd可能的值)
- 复制文件描述符
	- F_DUPFD : 复制一个文件描述符，文件描述符的值为 ： min(当前系统可用的最小值,arg指定的值)
	- F_DUPFD_CLOEXEC :
		在F_DUPFD的基础上对复制出的fd设置close-on-exec标志
		
- 文件描述符的标志flags
	- F_GETFD ： 返回该fd上设置的flags
	- F_SETFD ：设置该fd的flags为arg指定的值
	
- 文件描述符对应文件的状态标志status flag
	- F_GETFL ： 返回该fd对应文件上设置的status flag
	- F_SETFL ：设置该fd对应文件的status  flag为arg指定的值

- 记录锁（record lock）相关的操作

- Open file description locks

- 管理信号

- 租约(leases)相关