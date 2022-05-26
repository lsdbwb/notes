# TcpServer的功能
1. 负责创建并管理所有的监听套接字，当有新连接到来时，负责创建新的socket
2. 管理所有的TCP连接
3. 使所有的TCP连接能够正常地收发数据

# TcpServer类
TcpServer是一个基类，用户可以继承并扩展功能，比如基于TcpServer实现一个HttpServer
## 类关系设计
![[Pasted image 20220526151222.png]]
1. 继承Noncopyable类，禁止拷贝
2. 继承std::enable_shared_from_this<>类，以在类的成员方法中使用TcpServer的智能指针
3. 使用Socket套接字类来管理套接字
4. 使用IOManager类来管理TCP连接的IO事件以及进行任务调度
5. 还需要使用Address类绑定ip地址到监听套接字上

## 具体实现
- 成员变量
```c++
	// 套接字数组存放所有的监听套接字
	//可能支持多种协议， 有多个网卡或者可能监听多个地址
    std::vector<Socket::ptr> m_socks;
    // 管理普通连接套接字的IOManager
    IOManager* m_worker;
    // 负责管理监听套接字的IOManager
    IOManager* m_acceptWorker;
    // 读取的超时时机，若一段时间未从某个连接获得消息，则关闭该连接，避免空占socket资源
    uint64_t m_recvTimeout;
    // server的名称
    std::string m_name;
    // server的状态：是否停止
    bool m_isStop;
```

- 重要接口
### bind函数[[socket相关api#bind函数]]
给所有的监听套接字绑定地址
```c++
bool TcpServer::bind(const std::vector<Address::ptr>& addrs
                        ,std::vector<Address::ptr>& fails) {
    for(auto& addr : addrs) {
	    // 创建TCP套接字
        Socket::ptr sock = Socket::CreateTCP(addr);
        // 调用Scoket::bind方法绑定地址
        if(!sock->bind(addr)) {
            SYLAR_LOG_ERROR(g_logger) << "bind fail errno="
                << errno << " errstr=" << strerror(errno)
                << " addr=[" << addr->toString() << "]";
            // 暂存绑定失败的地址
            fails.push_back(addr);
            continue;
        }
        // 调用Socket::listen方法进行监听
        if(!sock->listen()) {
            SYLAR_LOG_ERROR(g_logger) << "listen fail errno="
                << errno << " errstr=" << strerror(errno)
                << " addr=[" << addr->toString() << "]";
            // 暂存监听失败的地址
            fails.push_back(addr);
            continue;
        }
        // 存放监听套接字
        m_socks.push_back(sock);
    }
	// 必须要所有的地址都绑定成功
    if(!fails.empty()) {
        m_socks.clear();
        return false;
    }

    for(auto& i : m_socks) {
        SYLAR_LOG_INFO(g_logger) << "server bind success: " << *i;
    }
    return true;
}
```

### startAccept函数
[[socket相关api#accept函数]]
```c++
void TcpServer::startAccept(Socket::ptr sock) {
    while(!m_isStop) {
	    // 调用accept接收新的连接
	    // accept是被hook过的，不会阻塞，会向epoll添加事件
        Socket::ptr client = sock->accept();
        if(client) {
	        // 设置连接的读超时时间
            client->setRecvTimeout(m_recvTimeout);
            // 设置读事件到来时的回调函数
            m_worker->schedule(std::bind(&TcpServer::handleClient,
                        shared_from_this(), client));
        } else {
            SYLAR_LOG_ERROR(g_logger) << "accept errno=" << errno
                << " errstr=" << strerror(errno);
        }
    }
}
```

### start函数
启动TcpServer，让每个监听套接字调用startAccept函数启动监听
```c++
bool TcpServer::start() {
    if(!m_isStop) {
        return true;
    }
    m_isStop = false;
    for(auto& sock : m_socks) {
        m_acceptWorker->schedule(std::bind(&TcpServer::startAccept,
                    shared_from_this(), sock));
    }
    return true;
}
```

### handleClient函数
handleClient是真正负责服务端和客户端数据交互的处理函数，由用户自行定义，同时也是一个虚函数，用户在继承TcpServer类时可以重写该函数来实现自己的业务需求

### stop函数
关闭TcpServer服务器
```c++
void TcpServer::stop() {
    m_isStop = true;
    auto self = shared_from_this();
    // 向管理监听套接字的线程池添加一个任务
    m_acceptWorker->schedule([this, self]() {
	    // 取消监听套接字的所有事件并关闭
        for(auto& sock : m_socks) {
            sock->cancelAll();
            sock->close();
        }
        m_socks.clear();
    });
}
```