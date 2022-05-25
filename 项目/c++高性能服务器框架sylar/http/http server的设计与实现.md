## HTTP报文解析流程
![[http server的设计与实现 2022-05-25 16.42.15.excalidraw]]

- 发送过来的HTTP报文使用HttpRequestParser或者HttpResponseParser进行解析，解析后存入HttpRequest或HttpResponse对象[[http协议封装]]
- HttpRequestParser或者HttpResponseParser内部使用Ragel进行HTTP报文的解析

# HTTP解析器具体实现
最上层HttpRequestParser和HttpResponseParser是对Ragel编译生成的解析器的简单封装，如下所示
### HttpRequestParser
```c++
class HttpRequestParser {
public:
    typedef std::shared_ptr<HttpRequestParser> ptr;
    HttpRequestParser();
    size_t execute(char* data, size_t len);
    int isFinished();
    int hasError(); 

    HttpRequest::ptr getData() const { return m_data;}
    void setError(int v) { m_error = v;}

    uint64_t getContentLength();
private:
    http_parser m_parser; // 使用ragel编译生成的parser
    HttpRequest::ptr m_data; // 解析好的HttpRequest
    //1000: invalid method
    //1001: invalid version
    //1002: invalid field
    int m_error;  // 解析过程中出现的错误
};
```

#### http_parser
http_parser是一个结构体，里面存放了HTTP报文解析过程中用到的状态信息和回调函数，解析完每一个字段后，就要调用对应的回调函数
```c++
typedef struct http_parser { 
  int cs;
  size_t body_start;
  int content_len;
  size_t nread;
  size_t mark;
  size_t field_start;
  size_t field_len;
  size_t query_start;
  int xml_sent;
  int json_sent;

  void *data;

  int uri_relaxed;
  // 回调函数，解析到对应字段后应该执行什么操作
  field_cb http_field;
  element_cb request_method;
  element_cb request_uri;
  element_cb fragment;
  element_cb request_path;
  element_cb query_string;
  element_cb http_version;
  element_cb header_done;
  
} http_parser;
```
- http_parser的回调函数在HttpRequestParser的构造函数中被赋值
```c++
HttpRequestParser::HttpRequestParser()
    :m_error(0) {
    m_data.reset(new sylar::http::HttpRequest);
    //初始化http_parser结构体
    http_parser_init(&m_parser);
    // 回调函数赋值
    m_parser.request_method = on_request_method;
    m_parser.request_uri = on_request_uri;
    m_parser.fragment = on_request_fragment;
    m_parser.request_path = on_request_path;
    m_parser.query_string = on_request_query;
    m_parser.http_version = on_request_version;
    m_parser.header_done = on_request_header_done;
    m_parser.http_field = on_request_http_field;
    m_parser.data = this;
}
```
- http_parser提供的方法
```c++
int http_parser_init(http_parser *parser);
int http_parser_finish(http_parser *parser);
// 真正执行解析
size_t http_parser_execute(http_parser *parser, const char *data, size_t len, size_t off);
// 解析发生的错误
int http_parser_has_error(http_parser *parser);
// 解析是否完成
int http_parser_is_finished(http_parser *parser);
```

#### 回调函数作用
- 将解析出的内容存到HttpRequest或HttpResponse对象中
- 如果解析出的字段是有错误，做相应处理
以其中一个回调函数为例
```c++
// 解析出HTTP请求的请求行的http version字段后做的操作
void on_request_version(void *data, const char *at, size_t length) {
    HttpRequestParser* parser = static_cast<HttpRequestParser*>(data);
    // 在内存中http版本使用一个字节存放
    uint8_t v = 0;
    if(strncmp(at, "HTTP/1.1", length) == 0) {
        v = 0x11;
    } else if(strncmp(at, "HTTP/1.0", length) == 0) {
        v = 0x10;
    } else {
	    // 解析出的版本不对，则设置error
        SYLAR_LOG_WARN(g_logger) << "invalid http request version: "
            << std::string(at, length);
        parser->setError(1001);
        return;
    }
    // 存到HttpRequest对象
    parser->getData()->setVersion(v);
}
```

#### HttpRequestParser的其他方法
都是调用http_parser_* 系列方法
```c++
size_t HttpRequestParser::execute(char* data, size_t len) {
    size_t offset = http_parser_execute(&m_parser, data, len, 0);
    memmove(data, data + offset, (len - offset));
    return offset;
}

int HttpRequestParser::isFinished() {
    return http_parser_finish(&m_parser);
}

int HttpRequestParser::hasError() {
    return m_error || http_parser_has_error(&m_parser);
}
```

### HttpResponseParser
```c++
class HttpResponseParser {
public:
    typedef std::shared_ptr<HttpResponseParser> ptr;
    HttpResponseParser();
    size_t execute(char* data, size_t len);
    int isFinished();
    int hasError(); 

    HttpResponse::ptr getData() const { return m_data;}
    void setError(int v) { m_error = v;}

    uint64_t getContentLength();
private:
    httpclient_parser m_parser; // 使用ragel编译生成的parser
    HttpResponse::ptr m_data; // 解析好的HttpResponse
    //1001: invalid version
    //1002: invalid field
    int m_error;
};```

#### httpclient_parser
和http_parser类似，只是http_parser用来解析HTTP请求报文,httpclient_parser用来解析HTTP响应报文

