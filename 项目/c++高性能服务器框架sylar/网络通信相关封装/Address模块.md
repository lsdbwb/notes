# Address模块类图
网络地址相关的类
- Address类：
	所有网络地址的基类，没有成员，只有接口方法。对应**sockaddr类型**[[套接字地址#sockaddr]]。提供了网络地址查询的功能

- IPAddress类：
	IP地址的基类，继承了Address类，也是一个虚基类。在Address类的基础上，新增了和IP地址相关的接口：端口，子网掩码，广播地址，网段地址等操作
	
- IPv4Address类：
	IPv4地址的类，继承了IPAddress类。是一个实体类，成员为**sockaddr_in**[[套接字地址#sockaddr]]类型，对应ipv4地址。可以操作该成员的端口，子网掩码，广播地址，网段地址等
	
- IPv6Address类：
	IPv6地址的类，继承了IPAddress类。是一个实体类，成员为**sockaddr_in6**类型，对应ipv6地址。可以操作该成员的端口，子网掩码，广播地址，网段地址等
	
- UnixAddress类：
	Unix域套接字类，继承Address类。是一个实体类，成员为**sockaddr_un**类型
	
- UnknownAddress类：
	表示未知类型的套接字，继承Address类。是一个实体类，成员为sockaddr

![[Pasted image 20220519160307.png]]

