# HTTP Server的设计与实现
- Http Server相比于TcpServer只是需要在读客户端传来的数据时要将数据解析为HTTP请求报文，发送数据给客户端时要将数据封装为HTTP响应报文。因此HTTP Server只需要继承TcpServer，并且使用HttpSession的收发HTTP报文的能力
- HTTP Server提供http服务，因此需要ServletDispatch类[[Http servlet模块]]作为成员

下图为HttpServer类的关系图
![[Pasted image 20220526174909.png]]

## HttpServer类的实现
成员变量
```c++
    /// 是否支持长连接
    bool m_isKeepalive;
    /// Servlet分发器
    ServletDispatch::ptr m_dispatch;
```

方法
重写handleClient方法
```c++
void HttpServer::handleClient(Socket::ptr client) {
    SYLAR_LOG_DEBUG(g_logger) << "handleClient " << *client;
    // 有数据到来时，使用HttpSession读消息并解析为HTTP请求报文
    HttpSession::ptr session(new HttpSession(client));
    do {
        auto req = session->recvRequest();
        // 解析出错就退出并关闭该连接
        if(!req) {
            SYLAR_LOG_DEBUG(g_logger) << "recv http request fail, errno="
                << errno << " errstr=" << strerror(errno)
                << " cliet:" << *client << " keep_alive=" << m_isKeepalive;
            break;
        }
		// 构建响应报文
        HttpResponse::ptr rsp(new HttpResponse(req->getVersion()
                            ,req->isClose() || !m_isKeepalive));
        rsp->setHeader("Server", getName());
        // 使用servelet处理,根据不同请求获取不同服务
        m_dispatch->handle(req, rsp, session);
        // 发送响应报文
        session->sendResponse(rsp);

        if(!m_isKeepalive || req->isClose()) {
            break;
        }
    } while(true);
    session->close();
}
```
