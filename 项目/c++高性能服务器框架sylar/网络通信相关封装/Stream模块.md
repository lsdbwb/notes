# Stream模块的作用
提供字节流读写接口

# Stream模块的类
![[Pasted image 20220526164942.png]]

## Stream类
虚基类，提供了读写的接口，子类都需要继承该类
## SocketStream类
继承Stream的一个实体类，负责从套接字上读写内容
### 成员
```c++
	Socket::ptr m_socket; // 要进行读写操作的套接字对象的智能指针
	bool m_owner;   //  //句柄全权管理标志位
```

### 方法
主要是对Socket的读写方法的封装
read/write方法都有两套接口，一套操作普通的buffer，一套操作ByteArray
```c++

int SocketStream::read(void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->recv(buffer, length);
}

int SocketStream::read(ByteArray::ptr ba, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    std::vector<iovec> iovs;
    // 操作ByteArray时搭配scatter IO
    ba->getWriteBuffers(iovs, length);
    int rt = m_socket->recv(&iovs[0], iovs.size());
    if(rt > 0) {
        ba->setPosition(ba->getPosition() + rt);
    }
    return rt;
}

int SocketStream::write(const void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->send(buffer, length);
}

int SocketStream::write(ByteArray::ptr ba, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    std::vector<iovec> iovs;
    ba->getReadBuffers(iovs, length);
    int rt = m_socket->send(&iovs[0], iovs.size());
    if(rt > 0) {
        ba->setPosition(ba->getPosition() + rt);
    }
    return rt;
}
```

## HttpSession
HttpSeesion继承SocketStream主要负责读写HTTP报文

### 接收套接字传来的数据并返回一条HTTP请求报文
```c++
HttpRequest::ptr HttpSession::recvRequest() {
    HttpRequestParser::ptr parser(new HttpRequestParser);
    uint64_t buff_size = HttpRequestParser::GetHttpRequestBufferSize();
    //uint64_t buff_size = 100;
    std::shared_ptr<char> buffer(
            new char[buff_size], [](char* ptr){
                delete[] ptr;
            });
    char* data = buffer.get();
    int offset = 0;
    // 读数据并解析出http header
    do {
        int len = read(data + offset, buff_size - offset);
        if(len <= 0) {
            close();
            return nullptr;
        }
        len += offset;
        // 读一段流数据后调用parser进行http报文解析
        size_t nparse = parser->execute(data, len);
        // 解析出错，直接关闭套接字
        if(parser->hasError()) {
            close();
            return nullptr;
        }
        offset = len - nparse;
        // http请求报文太大也不行
        if(offset == (int)buff_size) {
            close();
            return nullptr;
        }
        // 解析完成就退出循环
        if(parser->isFinished()) {
            break;
        }
    } while(true);
    // 拷贝http body的内容
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
        // body还未读完，继续读
        if(length > 0) {
            if(readFixSize(&body[len], length) <= 0) {
                close();
                return nullptr;
            }
        }
        // 读完设置body
        parser->getData()->setBody(body);
    }
    // 返回处理好的HttpRequest
    return parser->getData();
}
```

### 发送组装好的HTTP响应报文
```c++
int HttpSession::sendResponse(HttpResponse::ptr rsp) {
    std::stringstream ss;
    ss << *rsp;  // HTTP响应报文转换为字符串发送
    std::string data = ss.str();
    return writeFixSize(data.c_str(), data.size());
}
```