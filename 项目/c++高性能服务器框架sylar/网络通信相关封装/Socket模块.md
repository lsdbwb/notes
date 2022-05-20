# 概述
## socket类图
套接字类，表示一个套接字对象，同时封装了套接字常用的操作
![[Pasted image 20220519164948.png]]

##  成员变量
保存套接字fd的值、套接字的类型相关信息和套接字的连接状态等等
```c++
	int m_sock;        // 套接字fd
    int m_family;      // 协议族
    int m_type;        // 套接字类型 TCP/UDP
	int m_protocol; 
    bool m_isConnected;  // 是否已连接
 
    Address::ptr m_localAddress;  // 本端地址
    Address::ptr m_remoteAddress; // 对端地址
```

## 具体方法
### 创建操作
提供了几个静态方法来创建Socket对象。
```c++
Socket::ptr Socket::CreateTCP(sylar::Address::ptr address) {
    Socket::ptr sock(new Socket(address->getFamily(), TCP, 0));
    return sock;
}

Socket::ptr Socket::CreateUDP(sylar::Address::ptr address) {
    Socket::ptr sock(new Socket(address->getFamily(), UDP, 0));
    return sock;
}
```
在Socket的构造函数中并没有实际使用socket系统调用去创建套接字
```c++
// 构造函数只是初始化了一些信息
Socket::Socket(int family, int type, int protocol)
    :m_sock(-1)
    ,m_family(family)
    ,m_type(type)
    ,m_protocol(protocol)
    ,m_isConnected(false) {
}
```

###  bind和connect是真正调用socket函数的地方
  在调用bind和connect函数时才去真正创建系统套接字。因为调用bind和connect时才是我们实际想去使用套接字进行通信的时机。
 - bind的实现：
```c++
bool Socket::bind(const Address::ptr addr) {
	// 套接字fd尚未创建
    if(!isValid()) {
	    // 真正调用socket系统调用创建套接字
	    // 并设置 m_fd为套接字的文件描述符
        newSock();
        if(SYLAR_UNLICKLY(!isValid())) {
            return false;
        }
    }

    if(SYLAR_UNLICKLY(addr->getFamily() != m_family)) {
        SYLAR_LOG_ERROR(g_logger) << "bind sock.family("
            << m_family << ") addr.family(" << addr->getFamily()
            << ") not equal, addr=" << addr->toString();
        return false;
    }
	// 调用bind绑定地址
    if(::bind(m_sock, addr->getAddr(), addr->getAddrLen())) {
        SYLAR_LOG_ERROR(g_logger) << "bind error errrno=" << errno
            << " errstr=" << strerror(errno);
        return false;
    }
    // 设置本地地址为绑定的地址
    getLocalAddress();
    return true;
}
```

bind使用的函数：
```c++
void Socket::initSock() {
    int val = 1; 
    // 设置套接字选项 SO_REUSEADDR
    setOption(SOL_SOCKET, SO_REUSEADDR, val);
    if(m_type == SOCK_STREAM) {
        setOption(IPPROTO_TCP, TCP_NODELAY, val);
    }
}

void Socket::newSock() {
	// 创建socket
    m_sock = socket(m_family, m_type, m_protocol);
    if(SYLAR_LICKLY(m_sock != -1)) {
        initSock();
    } else {
        SYLAR_LOG_ERROR(g_logger) << "socket(" << m_family
            << ", " << m_type << ", " << m_protocol << ") errno="
            << errno << " errstr=" << strerror(errno);
    }
}
```

- connect的实现
```c++
bool Socket::connect(const Address::ptr addr, uint64_t timeout_ms) {
    if(!isValid()) {
    // 真正调用socket系统调用创建套接字
	    // 并设置 m_fd为套接字的文件描述符
        newSock();
        if(SYLAR_UNLICKLY(!isValid())) {
            return false;
        }
    }

    if(SYLAR_UNLICKLY(addr->getFamily() != m_family)) {
        SYLAR_LOG_ERROR(g_logger) << "connect sock.family("
            << m_family << ") addr.family(" << addr->getFamily()
            << ") not equal, addr=" << addr->toString();
        return false;
    }

	// 没设超时时间
    if(timeout_ms == (uint64_t)-1) {
        if(::connect(m_sock, addr->getAddr(), addr->getAddrLen())) {
            SYLAR_LOG_ERROR(g_logger) << "sock=" << m_sock << " connect(" << addr->toString()
                << ") error errno=" << errno << " errstr=" << strerror(errno);
            close();
            return false;
        }
    } else {  // 设置了超时时间
        if(::connect_with_timeout(m_sock, addr->getAddr(), addr->getAddrLen(), timeout_ms)) {
            SYLAR_LOG_ERROR(g_logger) << "sock=" << m_sock << " connect(" << addr->toString()
                << ") timeout=" << timeout_ms << " error errno="
                << errno << " errstr=" << strerror(errno);
            close();
            return false;
        }
    }
    m_isConnected = true;
    // 获得对端的地址并保存
    getRemoteAddress();  // 调用getpeername
    // 保存本端地址
    getLocalAddress();   // 调用getsocketname
    return true;
}
```

### 其余操作
- accept
```c++
Socket::ptr Socket::accept() {
    Socket::ptr sock(new Socket(m_family, m_type, m_protocol));
    // 监听并返回新的套接字fd
    int newsock = ::accept(m_sock, nullptr, nullptr);
    if(newsock == -1) {
        SYLAR_LOG_ERROR(g_logger) << "accept(" << m_sock << ") errno="
            << errno << " errstr=" << strerror(errno);
        return nullptr;
    }
    // 设置新Socket对象的相关信息
    if(sock->init(newsock)) {
        return sock;
    }
    return nullptr;
}

bool Socket::init(int sock) {
    FdCtx::ptr ctx = FdMgr::GetInstance()->get(sock);
    if(ctx && ctx->isSocket() && !ctx->isClose()) {
        m_sock = sock;
        m_isConnected = true;
        initSock();
        getLocalAddress();
        getRemoteAddress();
        return true;
    }
    return false;
}
```

- 其余read/write等IO操作都是对底层系统调用的简单封装
