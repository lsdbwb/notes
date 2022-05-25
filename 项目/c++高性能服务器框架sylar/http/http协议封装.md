具体的报文结构详见[[http报文格式]]
# HTTP请求报文
HttpRequest类
- 成员变量
```c++
	// 请求行
    HttpMethod m_method; // 请求方法
    uint8_t m_version;   // http协议版本
	// URI
    std::string m_path; // 资源路径
    std::string m_query; // 查询字符串
    std::string m_fragment; // 片段标识
    
    // body ： 报文主体
    std::string m_body;
    
	bool m_close; // 是否处于长连接
	// 首部字段
    MapType m_headers;
    // 参数字段
    MapType m_params;
    // cookie字段
    MapType m_cookies;
```
- 方法
1. 对各个成员变量进行查询的get方法以及设置的set方法
2. 模板方法,根据需求将头部字段的value转变为指定格式
```c++
template<class MapType, class T>
T getAs(const MapType& m, const std::string& key, const T& def = T()) {
    auto it = m.find(key);
    if(it == m.end()) {
        return def;
    }
    try {
	    // 转换成指定类型
        return boost::lexical_cast<T>(it->second);
    } catch (...) {
    }
    return def;
}
```
# HTTP响应报文
HttpResponse类
- 成员变量
```c++
	//状态行
    HttpStatus m_status; // 状态码
    uint8_t m_version;  // http版本
    std::string m_reason; // 状态原因
    bool m_close;
    // 响应消息体
    std::string m_body;
    // 响应头部字段
    MapType m_headers;
```
1. 对各个成员变量进行查询的get方法以及设置的set方法
2. 模板方法（同HttpRequest）,根据需求将头部字段的value转变为指定格式