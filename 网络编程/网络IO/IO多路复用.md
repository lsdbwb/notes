# IO多路复用流程
- IO多路复用模型[[网络IO模型#IO多路复用]]
- 流程

和阻塞IO的流程大致相同,只是此时一个进程可以同时等待多个套接字
![[Pasted image 20220529102436.png]]
1. 一个进程同时加入到了多个套接字上的等待队列上
2. 同样地,当数据拷贝到内存网卡缓冲区之后,网卡中断处理程序将数据拷贝到1个或多个套接字的输入缓冲区
3. 然后进程A被唤醒,变为就绪态加入工作队列
4. 进程A再次执行时就去扫描一遍有哪些套接字准备好了,并做相应处理

# 三个多路复用函数select,poll,epoll
三个函数的原理和处理流程大致都是相同的,只是根据实现方式的不同,效率上有所区别

## select
- 使用三个事件位图来表示所有套接字上的事件
- 关注的套接字的数量由nfds指定(**数量有限制 <=FD_SETSIZE**)
```c++
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```
**参数**
- nfds : 此次监听的描述符的数量
这三个参数既作为调用时传入的参数,在函数返回时,也作为存放结果的地方.因此调用完时被结果覆盖了,下次调用时又得重新设置
- readfds : 监听读事件的文件描述符集合
- writefds : 监听写事件的文件描述符集合
- exceptfds : 监听异常事件的文件描述符集合

- timeout : 此次监听的超时时间

**返回值**
成功时返回监听到的事件发生的套接字数量(即readfds, writefds, exceptfds三个set中位被置1的个数),失败时返回-1并设置errno

使用fd_set来存储所有套接字关心的事件
使用以下几个宏来操作fd_set
```c++
	void FD_CLR(int fd, fd_set *set);   // 清除set中和fd相关的事件
    int  FD_ISSET(int fd, fd_set *set); // 判断fd在set中是否设置事件
    void FD_SET(int fd, fd_set *set);  // 设置set中和fd相关的事件
    void FD_ZERO(fd_set *set);    // 清空该位图
```

**使用例子**
```text
int s = socket(AF_INET, SOCK_STREAM, 0);  
bind(s, ...);
listen(s, ...);
int fds[] =  存放需要监听的socket;

&rset = 根据fds构建出的位图; // 读监听
while(1){
    int n = select(..., &rset, ...)
    for(int i=0; i < fds.length; i++){
		// FD_ISSET是fd_set的宏，可以判断该位置上的bitmap是否为1
        if(FD_ISSET(fds[i], &rset)){
            // fds[i]的数据处理
			read(fds[i], buffer);
			doSomething(buffer);
        }
}}
```

**缺点**
1. 监听的套接字数量有限制
2. 每次调用都要**拷贝**文件描述符的事件集合(readfds, writefds, exceptfds从用户态拷贝到内核态,调用结束时又需要从内核态拷贝回用户态).
3. 每次都要进行**三次遍历**:
	1.  **监听时**:每次调用都要**遍历**一遍文件描述符集合,将进程挂载到对应的套接字任务队列上
	2. **就绪时**:每次在内核态被唤醒时要去遍历一遍所有被监听的套接字,将就绪的套接字的相应事件位置1
	3. **返回时**:每次从内核态返回时因为不知道具体哪些套接字就绪了,所以还需要遍历一遍readfds, writefds, exceptfds
5. 每次返回时都要清理,将之前在套接字任务队列上添加的等待项全部清除

## poll
- poll和select的实现机制是一样的,都是采用轮询.
- poll的相比于select的优点是没有监听套接字的数量限制,同时传入的事件和返回的事件分开来存,不用每次调用都重新设置关心的事件
```c++
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```
**参数**
- fds : pollfd结构体指针,传入的是关心的套接字的链表
	pollfd的结构:
	```
	        struct pollfd {
               int   fd;         /* file descriptor */
               short events;     /* requested events */
               short revents;    /* returned events */
           };
	```
- nfds : 链表的大小
- timeout : 超时时间

**返回值**
成功时返回有事件发生的套接字的数量(revents不为0的数量),失败时返回-1并设置errno

**缺点**
1. 和select一样采用轮询的方式,因此仍然需要三次遍历
2. 和select一样,**包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间**，而**不论**这些**文件描述符是否就绪**

## epoll
- epoll解决了上述select和poll存在的问题
- select和epoll存在的问题：
	1. 每次调用都需要拷贝文件描述符集合
	2. 每次调用需要至少遍历一次文件描述符集合
	3. 从schedule()返回后，无法得知具体是哪些文件描述符就绪，还需要再遍历一次文件描述符集合进行判断
	4. 调用返回前，必须进行清理，将之前添加的等待队列项全部移除




# 总结
![[Pasted image 20220529121621.png]]