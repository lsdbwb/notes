# 协程调度模块的缺陷
- 当线程池某个线程正在执行的协程进行IO事件时，该线程会阻塞
- 在没有任务可做时，线程池里的线程会进入忙等状态空占CPU

# IO协程调度模块的设计
- IO协程调度模块就是在协程调度模块的基础上加上事件通知机制，这样能够解决上面的线程空占CPU的问题
- 如下图所示，线程池中的工作线程在无任务可做时不会空占CPU，而是阻塞在epoll上，当有事件发生时，epoll负责唤醒这些线程继续工作
- IO协程调度模块要管理IO(socket网络IO)事件，协程调度结合IO多路复用机制来管理IO
![[IO协程调度模块.excalidraw]]

## IOManager的实现
1. IOManager主要进行网络IO任务的调度，也是一个调度器，因此要继承Scheduler类
```c++
class IOManager : public Scheduler
```
2. 需要使用IO多路复用机制，因此需要epoll；同时要管理所有的IO任务；因此IOManager的成员变量如下：
```c++
	// epoll的fd
    int m_epfd = 0;
    // 用来通知线程有任务到来的管道
    int m_tickleFds[2];
    /// 当前等待执行的IO事件数量
    std::atomic<size_t> m_pendingEventCount = {0};
    RWMutexType m_mutex;
    // 存放所有的IO事件
    std::vector<FdContext*> m_fdContexts;
```
- FdContext用来表示一个IO事件:主要是该事件关联哪个句柄以及事件发生时要进行的操作(回调函数)
	所有的事件都抽象为读事件或者写事件
```c++
    struct FdContext {
        typedef Mutex MutexType;
        // 事件的上下文
        struct EventContext {
            Scheduler* scheduler = nullptr; //事件执行的scheduler
            Fiber::ptr fiber;               //事件协程
            std::function<void()> cb;       //事件的回调函数
        };
		// 根据事件类型获取上下文
        EventContext& getContext(Event event);
        // 重设上下文
        void resetContext(EventContext& ctx);
        void triggerEvent(Event event);
		//读事件和写事件各有一个上下文
        EventContext read;      //读事件
        EventContext write;     //写事件
        int fd = 0;             //事件关联的句柄
        Event events = NONE;    //已经注册的事件
        MutexType mutex;
    };
```
   事件发生时会调用triggerEvent
```c++
void IOManager::FdContext::triggerEvent(IOManager::Event event) {
    SYLAR_ASSERT(events & event);
    // 每次触发事件后，会将该事件剔除， 因此不会持续触发
    events = (Event)(events & ~event);
    // 获取触发事件的上下文
    EventContext& ctx = getContext(event);
    // 调用回调函数
    if(ctx.cb) {
        ctx.scheduler->schedule(&ctx.cb);
    } else {
        ctx.scheduler->schedule(&ctx.fiber);
    }
    ctx.scheduler = nullptr;
    return;
}
```

3. IOManager的构造函数应该要初始化epoll和调度器
```c++
IOManager::IOManager(size_t threads, bool use_caller, const std::string& name)
    :Scheduler(threads, use_caller, name) //先调用父类Scheduler的构造函数
{
	// 创建epoll fd ，最多监听5000个事件
    m_epfd = epoll_create(5000);
    SYLAR_ASSERT(m_epfd > 0);
	// 创建通知空闲线程的管道
    int rt = pipe(m_tickleFds);
    SYLAR_ASSERT(!rt);
	// 管道的读端设置读事件
    epoll_event event;
    memset(&event, 0, sizeof(epoll_event));
    event.events = EPOLLIN | EPOLLET;
    event.data.fd = m_tickleFds[0];

    rt = fcntl(m_tickleFds[0], F_SETFL, O_NONBLOCK);
    SYLAR_ASSERT(!rt);
	// 将管道读端的读事件加入epoll监听
    rt = epoll_ctl(m_epfd, EPOLL_CTL_ADD, m_tickleFds[0], &event);
    SYLAR_ASSERT(!rt);

    contextResize(32);
	// 调度器开始调度
    start();
}
```

4. 线程池中的主协程平时应该阻塞在epoll上，因此IOManager要重写Scheduler的idle函数
- idle函数的处理流程
![[Pasted image 20220516160008.png]]
- 等idle协程yield退出后，调度协程即开始从任务队列取任务并执行

5. IOManager调度IO事件，需要提供基于epoll添加、取消、删除IO事件的接口
- 添加事件
	```c++
int IOManager::addEvent(int fd, Event event, std::function<void()> cb) {
    // 找到fd对应的FdContext，如果不存在就分配一个
    FdContext* fd_ctx = nullptr;
    RWMutexType::ReadLock lock(m_mutex);
    if((int)m_fdContexts.size() > fd) {
        fd_ctx = m_fdContexts[fd];
        lock.unlock();
    } else {
        lock.unlock();
        RWMutexType::WriteLock lock2(m_mutex);
        contextResize(fd * 1.5);
        fd_ctx = m_fdContexts[fd];
    }
	// 事件已存在则不重复添加
    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if(fd_ctx->events & event) {
        SYLAR_LOG_ERROR(g_logger) << "addEvent assert fd=" << fd
                    << " event=" << event
                    << " fd_ctx.event=" << fd_ctx->events;
        SYLAR_ASSERT(!(fd_ctx->events & event));
    }
	
    int op = fd_ctx->events ? EPOLL_CTL_MOD : EPOLL_CTL_ADD;
    epoll_event epevent;
    // 设置事件 设置为epoll边缘触发模式
    epevent.events = EPOLLET | fd_ctx->events | event;
    epevent.data.ptr = fd_ctx;
	// 添加事件进epoll
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if(rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
            << op << "," << fd << "," << epevent.events << "):"
            << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return -1;
    }
	// 待执行的IO事件数加一
    ++m_pendingEventCount;
    fd_ctx->events = (Event)(fd_ctx->events | event);
    // 找到该事件对应的EventContext
    FdContext::EventContext& event_ctx = fd_ctx->getContext(event);
    SYLAR_ASSERT(!event_ctx.scheduler
                && !event_ctx.fiber
                && !event_ctx.cb);

	// 对该eventcontext的调度器赋值
    event_ctx.scheduler = Scheduler::GetThis();
    // 对该eventcontext的回调函数赋值
    if(cb) {
        event_ctx.cb.swap(cb);
    } else {
	    // 如果没有显式设置回调函数，则继续当前协程执行
        event_ctx.fiber = Fiber::GetThis();
        SYLAR_ASSERT(event_ctx.fiber->getState() == Fiber::EXEC);
    }
    return 0;
}
```
- 删除事件（删除时不会触发事件）
```c++
bool IOManager::delEvent(int fd, Event event) {
    RWMutexType::ReadLock lock(m_mutex);
    if((int)m_fdContexts.size() <= fd) {
        return false;
    }
    // 根据fd找到该事件对应的fdcontext
    FdContext* fd_ctx = m_fdContexts[fd];
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if(!(fd_ctx->events & event)) {
        return false;
    }
	// 清除指定的事件
    Event new_events = (Event)(fd_ctx->events & ~event);
    int op = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;
	// 让epoll不再关心此事件
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if(rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
            << op << "," << fd << "," << epevent.events << "):"
            << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }
	// 待执行事件数减1
    --m_pendingEventCount;
    fd_ctx->events = new_events;
    FdContext::EventContext& event_ctx = fd_ctx->getContext(event);
    // 重置该fd对应的读或写事件的eventcontext
    fd_ctx->resetContext(event_ctx);
    return true;
}
```
- 取消事件（会触发一次事件回调然后再取消事件）
```c++
bool IOManager::cancelEvent(int fd, Event event) {
    RWMutexType::ReadLock lock(m_mutex);
    if((int)m_fdContexts.size() <= fd) {
        return false;
    }
    // 根据fd找到 FdContext
    FdContext* fd_ctx = m_fdContexts[fd];
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if(!(fd_ctx->events & event)) {
        return false;
    }
	// 清除指定的事件
    Event new_events = (Event)(fd_ctx->events & ~event);
    int op = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;
	// 让epoll不再关心此事件
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if(rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
            << op << "," << fd << "," << epevent.events << "):"
            << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }
	// 取消事件前触发一次回调
    fd_ctx->triggerEvent(event);
    // 待执行事件数减1
    --m_pendingEventCount;
    return true;
}
```

# 总结
