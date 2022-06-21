# 为什么需要单例模式
在一个系统中某个对象你需要并且只需要一个
需要全局访问这个对象的方法


# Meyers' Singleton
```c++
  
// Singleton.hpp 
class Singleton {
public: 
	static Singleton& Instance() { 
		static Singleton S; 
		return S; 
	} 
private: 
	Singleton(); 
	~Singleton(); 
};
```
（static静态局部变量在进入函数时才会被初始化）
- 保证只有一个对象
	当你第一次调用 `Singleton& s=Singleton::Instance()`时，对象S被初始化，接下来的每一次调用都会返回相同的对象S
	
- 保证线程安全
	在c++11这是线程安全的，当多个线程并发调用`Singleton& s=Singleton::Instance()`时，其他线程都会等待S第一次被初始化完成

- 保证对象会被安全析构（其他全局对象的析构函数不会影响这个对象）
	编译器会在main函数之后调用static对象的析构函数
	![[Pasted image 20220621115504.png]]