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