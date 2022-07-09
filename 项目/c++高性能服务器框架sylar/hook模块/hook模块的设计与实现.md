# 管理所有的fd
使用一个类FdCtx记录fd的上下文
```c++
class FdCtx : public std::enable_shared_from_this<FdCtx> {
public:
    typedef std::shared_ptr<FdCtx> ptr;
    FdCtx(int fd);
    ~FdCtx();

    bool init();
    bool isInit() const { return m_isInit;}
    bool isSocket() const { return m_isSocket;}
    bool isClose() const { return m_isClosed;}
    bool close();

    void setUserNonblock(bool v) { m_userNonblock = v;}
    bool getUserNonblock() const { return m_userNonblock;}

    void setSysNonblock(bool v) { m_sysNonblock = v;}
    bool getSysNonblock() const { return m_sysNonblock;}

    void setTimeout(int type, uint64_t v);
    uint64_t getTimeout(int type);
private:
    bool m_isInit: 1;
    bool m_isSocket: 1; 
    bool m_sysNonblock: 1; // 是否hook非阻塞
    bool m_userNonblock: 1; //用户是否显式设置为非阻塞
    bool m_isClosed: 1;
    int m_fd;  // 打开的文件描述符
    uint64_t m_recvTimeout; // 读超时时间
    uint64_t m_sendTimeout; // 写超时时间
};
```
使用一个类FdManager保存所有fd的上下文
```c++
class FdManager {
public:
    typedef RWMutex RWMutexType;
    FdManager();
	// 根据fd获取或创建FdCtx
    FdCtx::ptr get(int fd, bool auto_create = false);
    // 根据fd删除FdCtx
    void del(int fd);

private:
    RWMutexType m_mutex;
    // 以fd为下标
    std::vector<FdCtx::ptr> m_datas;
};
```

# hook
替换底层库函数或者系统调用为自己的实现
## hook需要注意的点
1. 被hook的函数需要100%模拟原函数的行为，函数参数，返回值，errno都应该一样
2. 对于socket，因为进行IO调度时，所有的socket默认被设置为非阻塞的，如果用户没有显式地设置socket是非阻塞的，那就要处理好fcntl，不要对用户暴露socket已经是非阻塞的事实
## hook的实现机制
1. 动态库的全局符号介入机制[[全局符号介入]]覆盖原来的系统调用
2. 使用dlsym[[动态链接相关api#dlsym、dlvsym函数]]找回被覆盖的系统调用的函数签名


# hook以线程为粒度开启
```c++
// 使用线程局部变量，开启hook时，该变量设置为true
static thread_local bool t_hook_enable = false;

bool is_hook_enable() {
    return t_hook_enable;
}

void set_hook_enable(bool flag) {
    t_hook_enable = flag;
}
```

每个被hook的函数在开始时都先判断t_hook_enable是否设置为true，再决定是否改变原函数的行为

# hook的函数
![[Pasted image 20220518095158.png]]
## sleep系列
- sleep系统调用是让线程休眠一段时间（以秒为单位）[[sleep系列#sleep函数]]
- hook后的sleep使用epoll来计时，向epoll中添加一个对应的超时事件，然后当前协程调用yield让出cpu
- 超时事件发生后，超时事件的回调函数将当前协程又重新加入IOManager的任务队列，等待后续调度执行

```c++
unsigned int sleep(unsigned int seconds) {
    if(!sylar::t_hook_enable) {
        return sleep_f(seconds);
    }
	
    sylar::Fiber::ptr fiber = sylar::Fiber::GetThis();
    sylar::IOManager* iom = sylar::IOManager::GetThis();
    iom->addTimer(seconds * 1000, std::bind((void(sylar::Scheduler::*)
            (sylar::Fiber::ptr, int thread))&sylar::IOManager::schedule
            ,iom, fiber, -1));
    sylar::Fiber::YieldToHold();
    return 0;
}
```

usleep和nanosleep的行为和sleep是一样的，只是休眠的时间的单位不同
- usleep是微秒
- nanosleep是纳秒
因为epoll定时的精度是毫秒，所以在hook之后，usleep和nanosleep都变成了毫秒的精度

## socket设置相关
### socket函数[[socket相关api#socket函数]]
```c++
int socket(int domain, int type, int protocol) {
    if(!sylar::t_hook_enable) {
        return socket_f(domain, type, protocol);
    }
    int fd = socket_f(domain, type, protocol);
    if(fd == -1) {
        return fd;
    }
	// 创建成功时将fd放入FDManager中进行管理
    sylar::FdMgr::GetInstance()->get(fd, true);
    return fd;
}
```

### connect函数[[socket相关api#connect函数]]
- 因为connect函数有默认的超时时间，因此在hook时直接改成带超时时间的connect即可
```c++
int connect_with_timeout(int fd, const struct sockaddr* addr, socklen_t addrlen, uint64_t timeout_ms)
```
- hook后的connect_with_timeout函数流程
1. 判断传入的fd是否是套接字，如果不是，则直接调用系统的connect函数并返回
2. 判断套接字是否被用户显式地设置为非阻塞模式，如果是则调用系统的connect函数并返回
3. 调用系统的connect函数，由于套接字默认被设置为非阻塞的，会直接返回EINPROGRESS错误
4. 如果超时参数有效，则会创建一个定时器，定时器的时间到后通过t->canceled设置超时标志，并且触发一次该fd的WRITE事件
5. 添加该fd的WRITE事件到epoll，然后调用yield让出CPU
6. 等待定时器超时或者套接字可写。如果套接字可写先发生，说明connect函数内部处理完毕，epoll通知唤醒该协程，该协程然后调用`getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len)`来获取connect的返回值；如果定时器超时先发生，则在回调函数中会设置ETIMEDOUT超时错误，然后取消epoll上注册的可写事件，最后再取消定时器并返回
- hook的connect函数
```c++
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen) {
    return connect_with_timeout(sockfd, addr, addrlen, sylar::s_connect_timeout);
}
```


**接下来的accept和/read/write/recv/send系统调用的hook是使用一个模板函数do_io来实现的，模板函数的流程和上面connect的处理流程大致相同，都是利用条件定时器和READ/WRITE事件以及IOManager来完成**

### close函数[[socket相关api#close函数]]
hook的close函数在关闭文件描述符前要先取消该fd上的IO事件，并且从FDManager中删除该fd
```c++
int close(int fd) {
    if(!sylar::t_hook_enable) {
        return close_f(fd);
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);
    if(ctx) {
        auto iom = sylar::IOManager::GetThis();
        if(iom) {
            iom->cancelAll(fd);
        }
        sylar::FdMgr::GetInstance()->del(fd);
    }
    return close_f(fd);
}
```

### ioctl[[socket相关api#ioctl函数]]
要特殊处理FIONBIO，处理好用户在应用层设置的非阻塞标志
```c++
int ioctl(int d, unsigned long int request, ...) {
    va_list va;
    va_start(va, request);
    void* arg = va_arg(va, void*);
    va_end(va);

    if(FIONBIO == request) {
        bool user_nonblock = !!*(int*)arg;
        sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(d);
        if(!ctx || ctx->isClose() || !ctx->isSocket()) {
            return ioctl_f(d, request, arg);
        }
        ctx->setUserNonblock(user_nonblock);
    }
    return ioctl_f(d, request, arg);
}
```
### fcntl[[socket相关api#fcntl函数]]
O_NONBLOCK标志要特殊处理，因为所有参与协程调度的fd都会被设置成非阻塞模式，所以要在应用层维护好用户设置的非阻塞标志。

### setsocketopt[[socket相关api#getsockopt和setsocketopt函数]]
如果optname为SO_RCVTIMEO和SO_SNDTIMEO等设置读写超时时间的选项，则在应用层也要记录超时时间，方便协程调度获取并处理
```c++
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen) {
    if(!sylar::t_hook_enable) {
        return setsockopt_f(sockfd, level, optname, optval, optlen);
    }
    if(level == SOL_SOCKET) {
        if(optname == SO_RCVTIMEO || optname == SO_SNDTIMEO) {
            sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(sockfd);
            if(ctx) {
                const timeval* v = (const timeval*)optval;
                ctx->setTimeout(optname, v->tv_sec * 1000 + v->tv_usec / 1000);
            }
        }
    }
    return setsockopt_f(sockfd, level, optname, optval, optlen);
}
```
