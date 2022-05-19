# 概述
## socket类图
套接字类，表示一个套接字对象，同时封装了套接字常用的操作
![[Pasted image 20220519164948.png]]

##  成员变量
```c++
	int m_sock;
    int m_family;
    int m_type;
    int m_protocol;
    bool m_isConnected;
 
    Address::ptr m_localAddress;  // 本端地址
    Address::ptr m_remoteAddress; // 对端地址
```

## 具体方法
### 创建操作
### 选项相关
### 连接
