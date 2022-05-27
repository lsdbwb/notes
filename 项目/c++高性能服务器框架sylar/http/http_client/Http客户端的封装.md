# HTTP客户端的作用
- 封装HTTP请求报文
- 发送HTTP请求报文到服务端
- 接收服务端传回的HTTP响应报文
- 解析HTTP响应报文并做相应处理

# 设计与实现
- HTTP客户端和HTTP服务端只是请求和接收的报文反过来了，其余和套接字连接相关都是一样的，都是继承TcpServer即可
- HTTP服务端使用HttpSession来处理HTTP报文，HTTP客户端相应地需要处理报文的类

## 类关系设计
- HttpConnection类是HTTP客户端用来封装HTTP请求报文和解析HTTP响应报文的类
- HttpResult是解析出的HTTP响应报文的结果
- Uri类是在封装HTTP请求报文时需要用到的

![[Pasted image 20220527150722.png]]

## 具体类的实现
### Uri类
Uri是http请求header中的一部分[[URI]]

- Uri的组成
![[Pasted image 20220527154059.png]]
- 成员变量
	和Uri的各个组成部分一一对应
```c++
   /// schema
    std::string m_scheme;
    /// 端口
    int32_t m_port;
    /// 用户信息
    std::string m_userinfo;
    /// host
    std::string m_host;
    /// 路径
    std::string m_path;
    /// 查询参数
    std::string m_query;
    /// fragment
    std::string m_fragment;
```

重要方法：
- 将URI字符串解析为Uri对象
```c++
    /**
     * @brief 创建Uri对象
     * @param uri uri字符串
     * @return 解析成功返回Uri对象否则返回nullptr
     */
    static Uri::ptr Create(const std::string& uri);
```
- 先编写Uri字符串解析的状态机，然后使用Ragel生成c++代码。通过`ragel -C -G2 uri.rl -o uri.rl.cpp`最终生成我们所需的.cpp文件，会在调用状态机的地方自动生成C/C++的代码嵌入到源代码中。
- 对应的Create函数如下：
```c++
/**
 * @brief 通过字符串URI创建URI对象
 * @param[in] uri URI字符串 
 * @return Uri::ptr 
 */
static Uri::ptr Create(const std::string& uri);

Uri::ptr Uri::Create(const std::string& u)
{
    Uri::ptr uri(new Uri);

    /*cs mark状态机中用到的变量 提前定义*/
    int cs = 0;
    const char* mark = NULL;

    //初始化自动状态机
    %% write init;
    //指向u头指针
    const char *p = u.c_str();
    //指向u尾指针
    const char *pe = p + u.size();
    //ragel指定的一个尾指针的名字
    const char *eof = pe;
    //调用自动状态机
    %% write exec;

    //uri_parser_error 检查解析是否错误
    if(cs == uri_parser_error)
        return nullptr;
    else if(cs >= uri_parser_first_final) //解析完成后的状态码是否合法  
        return uri;

    return nullptr;

}
```



### HttpResult类
封装从HTTP服务端发送过来的响应报文
```c++
struct HttpResult
{
    ....
    ....
        
	/**
     * @brief 错误码枚举类型
     */
    enum class Error
    {
        //成功
        OK,
        //非法URL
        INVALID_URL,
        //非法地址
        INVALID_ADDR,
        //非法套接字
        INVALID_SOCK,
        //连接失败
        CONNECT_FAIL,
        //发送请求失败
        SEND_REQ_FAIL,
        //接收响应超时
        RECV_RSP_TIMEOUT,
        //连接池取出连接失败
        POOL_GET_CONNECT_FAIL,
        //连接池非法sokcet
        POOL_INVALID_SOCK,

    };

	//错误码
    int result;
    //HTTP响应报文对象智能指针
    HttpResponse::ptr response;
    //错误原因短语
    std::string error;
};
```


### HttpConnection类
**成员变量**
```c++
	uint64_t m_createTime = 0;
	uint64_t m_request = 0;
```
**重要方法**
分为两类，一类是封装HTTP请求报文并发送；一类是解析出HTTP响应报文并保存
- 封装HTTP请求报文并发送
多个方法共同完成此功能
![[Http客户端的封装 2022-05-27 16.19.33.excalidraw]]
DoGet()方法根据传入的Uri字符串解析出Uri类，然后创建HttpRequest对象，最终调用DoRequest发起连接，发送Http请求报文(请求方法为Get)。DoPost类似，只是请求方法为Post
```c++
HttpResult::ptr HttpConnection::DoGet(const std::string& url
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    // 根据uri字符串解析出Uri对象
    Uri::ptr uri = Uri::Create(url);
    if(!uri) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_URL
                , nullptr, "invalid url: " + url);
    }
    return DoGet(uri, timeout_ms, headers, body);
}

HttpResult::ptr HttpConnection::DoGet(Uri::ptr uri
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    // 调用DoRequest操作
    return DoRequest(HttpMethod::GET, uri, timeout_ms, headers, body);
}

```
DoRequest方法有三个重载
- 第一个DoRequest方法根据uri字符串创建Uri对象
```c++
HttpResult::ptr HttpConnection::DoRequest(HttpMethod method
                            , const std::string& url
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    // 从Uri字符串解析出Uri对象
    Uri::ptr uri = Uri::Create(url);
    if(!uri) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_URL
                , nullptr, "invalid url: " + url);
    }
    return DoRequest(method, uri, timeout_ms, headers, body);
}
```
- 第二个DoRequest方法主要是根据Uri对象构造HttpRequest对象
```c++
HttpResult::ptr HttpConnection::DoRequest(HttpMethod method
                            , Uri::ptr uri
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    // 构造HttpRequest对象
    HttpRequest::ptr req = std::make_shared<HttpRequest>();
    // 设置HttpRequest的各个成员
    req->setPath(uri->getPath());
    req->setQuery(uri->getQuery());
    req->setFragment(uri->getFragment());
    req->setMethod(method);
    bool has_host = false;
    for(auto& i : headers) {
        if(strcasecmp(i.first.c_str(), "connection") == 0) {
            if(strcasecmp(i.second.c_str(), "keep-alive") == 0) {
                req->setClose(false);
            }
            continue;
        }

        if(!has_host && strcasecmp(i.first.c_str(), "host") == 0) {
            has_host = !i.second.empty();
        }

        req->setHeader(i.first, i.second);
    }
    if(!has_host) {
        req->setHeader("Host", uri->getHost());
    }
    req->setBody(body);
    return DoRequest(req, uri, timeout_ms);
}

```
- 第三个DoRequest方法创建连接并发送HTTP请求报文
```c++
HttpResult::ptr HttpConnection::DoRequest(HttpRequest::ptr req
                            , Uri::ptr uri
                            , uint64_t timeout_ms) {
    // 根据Uri创建Address
    Address::ptr addr = uri->createAddress();
    if(!addr) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_HOST
                , nullptr, "invalid host: " + uri->getHost());
    }
    // 创建套接字
    Socket::ptr sock = Socket::CreateTCP(addr);
    if(!sock) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::CREATE_SOCKET_ERROR
                , nullptr, "create socket fail: " + addr->toString()
                        + " errno=" + std::to_string(errno)
                        + " errstr=" + std::string(strerror(errno)));
    }
    // 调用connect连接服务端
    if(!sock->connect(addr)) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::CONNECT_FAIL
                , nullptr, "connect fail: " + addr->toString());
    }
    // 设置读超时
    sock->setRecvTimeout(timeout_ms);
    HttpConnection::ptr conn = std::make_shared<HttpConnection>(sock);
    // 发送请求
    int rt = conn->sendRequest(req);
    if(rt == 0) {
    // 发送失败则返回错误结果
        return std::make_shared<HttpResult>((int)HttpResult::Error::SEND_CLOSE_BY_PEER
                , nullptr, "send request closed by peer: " + addr->toString());
    }
    if(rt < 0) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::SEND_SOCKET_ERROR
                    , nullptr, "send request socket error errno=" + std::to_string(errno)
                    + " errstr=" + std::string(strerror(errno)));
    }
    // 发送成功则接收并解析HTTP响应报文
    auto rsp = conn->recvResponse();
    if(!rsp) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::TIMEOUT
                    , nullptr, "recv response timeout: " + addr->toString()
                    + " timeout_ms:" + std::to_string(timeout_ms));
    }
    // 将响应存储到HttpResult对象
    return std::make_shared<HttpResult>((int)HttpResult::Error::OK, rsp, "ok");
}
```

- 解析出HTTP响应报文并保存
```c++
HttpResponse::ptr HttpConnection::recvResponse() {
    HttpResponseParser::ptr parser(new HttpResponseParser);
    uint64_t buff_size = HttpRequestParser::GetHttpRequestBufferSize();
    //uint64_t buff_size = 100;
    std::shared_ptr<char> buffer(
            new char[buff_size + 1], [](char* ptr){
                delete[] ptr;
            });
    char* data = buffer.get();
    int offset = 0;
    do {
        int len = read(data + offset, buff_size - offset);
        if(len <= 0) {
            close();
            return nullptr;
        }
        len += offset;
        data[len] = '\0';
        size_t nparse = parser->execute(data, len, false);
        if(parser->hasError()) {
            close();
            return nullptr;
        }
        offset = len - nparse;
        if(offset == (int)buff_size) {
            close();
            return nullptr;
        }
        if(parser->isFinished()) {
            break;
        }
    } while(true);
    auto& client_parser = parser->getParser();
    if(client_parser.chunked) {
        std::string body;
        int len = offset;
        do {
            do {
                int rt = read(data + len, buff_size - len);
                if(rt <= 0) {
                    close();
                    return nullptr;
                }
                len += rt;
                data[len] = '\0';
                size_t nparse = parser->execute(data, len, true);
                if(parser->hasError()) {
                    close();
                    return nullptr;
                }
                len -= nparse;
                if(len == (int)buff_size) {
                    close();
                    return nullptr;
                }
            } while(!parser->isFinished());
            len -= 2;
            
            SYLAR_LOG_INFO(g_logger) << "content_len=" << client_parser.content_len;
            if(client_parser.content_len <= len) {
                body.append(data, client_parser.content_len);
                memmove(data, data + client_parser.content_len
                        , len - client_parser.content_len);
                len -= client_parser.content_len;
            } else {
                body.append(data, len);
                int left = client_parser.content_len - len;
                while(left > 0) {
                    int rt = read(data, left > (int)buff_size ? (int)buff_size : left);
                    if(rt <= 0) {
                        close();
                        return nullptr;
                    }
                    body.append(data, rt);
                    left -= rt;
                }
                len = 0;
            }
        } while(!client_parser.chunks_done);
        parser->getData()->setBody(body);
    } else {
        int64_t length = parser->getContentLength();
        if(length > 0) {
            std::string body;
            body.resize(length);

            int len = 0;
            if(length >= offset) {
                memcpy(&body[0], data, offset);
                len = offset;
            } else {
                memcpy(&body[0], data, length);
                len = length;
            }
            length -= offset;
            if(length > 0) {
                if(readFixSize(&body[len], length) <= 0) {
                    close();
                    return nullptr;
                }
            }
            parser->getData()->setBody(body);
        }
    }
    return parser->getData();
}
```
### HttpConnectionPool类
**成员变量**
```c++
	
    std::string m_host;
    std::string m_vhost;
    uint32_t m_port;
    uint32_t m_maxSize;
    uint32_t m_maxAliveTime;
    uint32_t m_maxRequest;

    MutexType m_mutex;
    std::list<HttpConnection*> m_conns;
    std::atomic<int32_t> m_total = {0};
```