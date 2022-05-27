# Servlet简介
- servlet 是Server Applet的简称，就是服务端小程序的意思
- HTTP服务器向外提供服务，servlet就是用来处理客户端的请求，并作出相应处理的一个服务

# HTTP Servlet模块
## 类关系图
![[Pasted image 20220526191820.png]]

### Servlet类
是一个虚基类，提供接口
```c++
    virtual int32_t handle(sylar::http::HttpRequest::ptr request
                   , sylar::http::HttpResponse::ptr response
                   , sylar::http::HttpSession::ptr session) = 0;
```
所有的servlet服务子类需要重写该接口

### FunctionalServlet
- 继承了Servlet类
- 表示以回调函数的形式提供服务的Servlet
- 回调函数的签名和handle函数一样

### ServletDispatch
用来分发服务
- 继承了Servlet类
- 实现了精准匹配和模糊匹配两种模式

**成员变量**
```c++
   RWMutexType m_mutex;
    /// 精准匹配servlet MAP
    /// uri(/sylar/xxx) -> servlet
    std::unordered_map<std::string, Servlet::ptr> m_datas;
    /// 模糊匹配servlet 数组
    /// uri(/sylar/*) -> servlet
    std::vector<std::pair<std::string, Servlet::ptr> > m_globs;
    /// 默认servlet，所有路径都没匹配到时使用
    Servlet::ptr m_default;
```

方法：
- 重写了handle方法
```c++
int32_t ServletDispatch::handle(sylar::http::HttpRequest::ptr request
               , sylar::http::HttpResponse::ptr response
               , sylar::http::HttpSession::ptr session) {
    // 找到匹配的servlet
    auto slt = getMatchedServlet(request->getPath());
    if(slt) {
	    // 调用该servlet进行处理
        slt->handle(request, response, session);
    }
    return 0;
}
```
- 根据uri寻找对应匹配的servlet
	使用fnmatch[[字符串匹配#fnmatch函数]]
```c++
Servlet::ptr ServletDispatch::getMatchedServlet(const std::string& uri) {
    RWMutexType::ReadLock lock(m_mutex);
    // 先去精确匹配
    auto mit = m_datas.find(uri);
    if(mit != m_datas.end()) {
        return mit->second;
    }
    // 精确匹配未找到，则进行模糊匹配
    for(auto it = m_globs.begin();
            it != m_globs.end(); ++it) {
        if(!fnmatch(it->first.c_str(), uri.c_str(), 0)) {
            return it->second;
        }
    }
    // 模糊匹配也没找到，则返回default servlet
    return m_default;
}
```

