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

### epoll内部数据结构
![[Pasted image 20220530113017.png]]
1. 调用epoll_create时会创建一个文件描述符,并返回epollfd.同时会创建eventpoll,存放在private_data成员中
2. eventpoll中包含三个重要的数据结构
	- rbr代表一颗红黑树,存放该epollfd监听的套接字的fd到对应epitem的映射
	- rdlist代表就绪链表,存放已经有事件就绪的套接字
	- wq:epoll_wait阻塞的进程的唤醒队列
3. epitem表示每一个被epoll等待的fd的管理结构,包括以下字段
	- ep，回指所属的eventpoll
	- ffd，对应的fd和struct file
	- event，用户关心的事件
	- rdlist，链表节点，在事件到达时用于加入eventpoll的rdllist
	- fllink，链表节点，用于加入file->f_ep_links链表，从而可以知道某个file被哪些epoll等待
	- pwqlist，该epitem产生的所有eppoll_entry形成的链表的链表头

### epoll提供的接口函数
#### epoll_create
创建一个epollfd,初始化其内部的数据结构
```c++
// 参数size 从linux 2.6.8开始为无效参数
int epoll_create(int size);

// flags可以设置为EPOLL_CLOEXEC,使得epollfd有colse-on-exec的属性
int epoll_create1(int flags);
```
**返回值**
成功时返回epollfd,失败时返回-1并设置errno

#### epoll_ctl
向epoll中增加,修改或者删除fd及IO事件
(内部向epollfd对应的eventpoll的红黑树中**增加,修改或者删除**epitem节点,将eventpoll放到对应文件的等待队列上)
```c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
**参数**
- epfd : epollfd
- op : 要做的操作类型
	1.  EPOLL_CTL_ADD
		将fd对应的文件加入epoll管理
	2.  EPOLL_CTL_MOD
		修改fd关心的事件
	3.  EPOLL_CTL_DEL
		使得epoll不再管理fd对应的文件
- fd :待管理的文件的文件描述符
- event
	```c++
		typedef union epoll_data {
            void        *ptr;
            int          fd;
            uint32_t     u32;
            uint64_t     u64;
        } epoll_data_t;

        struct epoll_event {
            uint32_t     events;      /* Epoll events */
            epoll_data_t data;        /* User data variable */
        };
```
**返回值**
成功时返回0,失败时返回-1并设置错误码

#### epoll_wait
当前线程阻塞在epfd指定的epoll上,等待事件发生.所有的就绪事件通过链表events返回,events最大长度由maxevents指定
(当前线程会被放入eventpoll的等待队列上)
```c++
int epoll_wait(int epfd, struct epoll_event *events,
	                      int maxevents, int timeout);
```
**参数**
- epfd : epollfd
- events : 就绪事件链表
- maxevents : 一次最多返回多少就绪事件
- 等待超时时间

**返回值**
成功等待到事件时返回就绪事件的数量,如果先发生超时则返回0,如果出错则返回-1并设置errno

### epoll使用例子
```c++
    #define MAX_EVENTS 10
    struct epoll_event ev, events[MAX_EVENTS];
    int listen_sock, conn_sock, nfds, epollfd;

    /* Code to set up listening socket, 'listen_sock',
    (socket(), bind(), listen()) omitted */

    epollfd = epoll_create1(0);
    if (epollfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    ev.events = EPOLLIN;
    ev.data.fd = listen_sock;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
               perror("epoll_ctl: listen_sock");
               exit(EXIT_FAILURE);
    }

	for (;;) {
        nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }

	    for (n = 0; n < nfds; ++n) {
            if (events[n].data.fd == listen_sock) {
                conn_sock = accept(listen_sock,
                            (struct sockaddr *) &addr, &addrlen);
                if (conn_sock == -1) {
                    perror("accept");
                    exit(EXIT_FAILURE);
                }
                setnonblocking(conn_sock);
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn_sock;
                if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                            &ev) == -1) {
                    perror("epoll_ctl: conn_sock");
                    exit(EXIT_FAILURE);
                }
            } else {
                do_use_fd(events[n].data.fd);
            }
        }
    }
```

### epoll的处理流程
1. 调用epoll_create创建epollfd以及初始化epoll的内核数据结构
2. 调用epoll_ctl向epoll中添加关注的事件(内部创建epitem并放入红黑树[[红黑树]]),同时将eppoll_entry挂载到各个文件描述符的等待队列上
3. 当某个文件描述符上有事件发生时,会唤醒等待队列上的eppoll_entry,调用eppoll_entry的回调函数,将就绪的epitem移动到就绪链表,并唤醒阻塞在epoll_wait上的线程
4. 将就绪链表从内核态拷贝一份到用户态,线程被唤醒后遍历该就绪链表并一一处理各个事件

### epoll核心原理
epoll的核心原理——对文件的等待队列的定制化
- wait的定制化——用eppoll_entry代替普通的wait_queue_entry插入被等待文件的唤醒队列，并且不修改进程的执行状态，也不调用schedule函数

- wake up的定制化——eppoll_entry里注册的回调函数定制化wake up的行为。唤醒时不是修改线程状态，而是将就绪的epitem移动到就绪链表，并唤醒阻塞在epoll_wait上的线程。

### epoll边缘触发与水平触发
- 水平触发
	**触发时机**:当输入缓冲区有数据时
	当某个套接字被设置为水平触发模式时,以读事件为例,当epoll监听的某个套接字的输入缓冲区有数据时,调用epoll_wait会一直返回该套接字的读就绪事件,直到输入缓冲区被用户读空

- 边缘触发
	**触发时机**:当内核缓冲区有新数据到来时
	当某个套接字被设置为边缘触发模式时,以读事件为例,当epoll监听的某个套接字的输入缓冲区有数据时,调用epoll_wait只会返回该套接字的读就绪事件**1次**,即使这一次用户只读了输入缓冲区的**部分数据**,下一次调用epoll_wait时也不会触发事件.只有下一次数据到来才会触发该事件
	
	**使用边缘触发的建议**:
	1. 将套接字设置为非阻塞的
	2. 仅仅当read或者write返回EAGAIN时才等待事件

	**使用边缘触发解决惊群问题**
	当有多个线程或者进程等待在同一个epoll时,在边缘触发模式下,每次有事件发生时只会有一个线程或者进程被唤醒

- ONESHOT触发
	事件被触发一次之后就被禁用.想再次触发时需要用户自己调用epoll_ctl重新修改该事件
	
#### 边沿触发、水平触发、ONESHOT在内核代码上的体现
- 水平触发的情况，epitem被从rdlist取下处理后又立即重新加入rdlist，因此下一次调用epoll_wait时会再次从rdlist中找到它们，调用对应文件的poll回调判断是否有对应的事件

- 边沿触发的情况，epitem被从rdlist取下后不会被立即重新加入rdlist，直到文件的等待队列的中对应的wait_queue_entry_t的回调函数再次被调用，该epitem才会被再次加入rdlist

- ONESHOT，epitem被从rdlist取下处理后，不会再被加入rdlist，并且epitem->event.events &= EP_PRIVATE_BITS，相当于禁用了相关事件，因此在重新启用事件前，该epitem不会再被加入rdlist

# 总结
![[Pasted image 20220529121621.png]]