# 回调函数的概念
编程分为两类：系统编程（system programming）和应用编程（application programming）。所谓系统编程，简单来说，就是编写**库**；而应用编程就是利用写好的各种库来编写具某种功用的程序，也就是**应用**。系统程序员会给自己写的库留下一些接口，即API（application programming interface，应用编程接口），以供应用程序员使用。所以在抽象层的图示里，库位于应用的底下。

当程序跑起来时，一般情况下，应用程序（application program）会时常通过API调用库里所预先备好的函数。但是有些[库函数](https://www.zhihu.com/search?q=%E5%BA%93%E5%87%BD%E6%95%B0&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A27459821%7D)（library function）却要求应用先传给它一个函数，好在合适的时候调用，以完成目标任务。这个被传入的、后又被调用的函数就称为**回调函数**（callback function）。

![[Pasted image 20220709140619.png]]

C++中回调函数的形式一般有两种：
- 库里面的类定义一个虚函数并暴露给应用程序，应用程序使用一个子类继承该类，并且在子类中重写该虚函数，库通过动态绑定的机制调用到子类的函数
- 库里面存放函数指针（或者std::funciton对象），应用程序先注册一个函数并保存到函数指针里，库函数调用函数指针指向的函数就可以


# 回调函数的问题
1. 回调函数经常被用在网络库中，结合异步事件驱动来使用。通常每个TCP连接上可能发生的事件（例如读写套接字、连接建立和断开）都会有一个对应的回调函数来处理。
2. **在reactor模式下，回调函数中是不能有阻塞操作的**，因为这会使得整个reactor无法响应其他的事件。因此如果一个业务需要执行阻塞操作时，不能将整个业务用一个回调函数来编写，而是*将业务拆分为多个回调函数，并且保证每个回调函数都是非阻塞的*。
3. 如下图所示，下面是一个HTTP Proxy服务器在缓存失效时的业务逻辑，此时该代理服务器需要向服务端转发客户端的请求然后将服务端的响应转发给客户端，也可能需要使用DNS解析域名。整个业务逻辑包含多个网络请求和响应的逻辑，如果只用一个回调函数来完成整个业务逻辑，那么该回调函数可能阻塞在多个子业务中，比如DNS解析时要阻塞等待结果，转发HTTP请求时要等待响应。

	 上述例子说明当**使用一个回调函数**处理有很多网络请求响应的业务时，回调函数很可能被阻塞，这不符合reactor模式的要求。因此需要把可能阻塞的子业务拆分出来，也注册到reactor的事件循环中去，这样子业务也能使用事件通知机制了。这样做的缺点就是*整体的业务逻辑就会被拆分，并且分散到多个回调函数中，变得难以理解和维护*
 
	![[回调函数callback 2022-07-09 14.18.18.excalidraw]]



# 使用协程改造异步回调为同步操作
上述HTTP Proxy的例子使用伪代码描述如下：
```c++
// 一个回调函数对应一个子业务

//“DNS域名解析相关事件”发生后要调用的回调
void cb_DNSParse(){
	...
	注册transmit_to_server()的相关事件和回调到reactor
	...
}

// “转发客户端HTTP请求到服务端”相关事件发生后要调用的回调
void cb_transmit_to_server(){
	...
	注册transmit_to_client()的相关事件和回调到reactor
	...
}

// ”转发服务端HTTP响应到客户端“相关事件发生后要调用的回调
void cb_transmit_to_client(){
	...
}

//一个业务被拆分到多个回调函数中
void HttpProxyDo(){
	注册DNSParser相关事件和回调函数到reactor
}
```
可见一个业务逻辑被切割到多个回调函数，业务逻辑复杂时变得难以理解和维护

使用协程可以将上述基于异步回调的逻辑改造为同步的逻辑
- 代码可以使用同步的写法
- 本质上还是通过事件驱动+回调的模式，只不过内部通过*协程+hook+事件通知机制*使得可以用同步的代码进行异步回调的操作。

改造之后上述业务就可以在一个函数内完成了，如下所示：
```c++
// 代码逻辑是同步的，包括使用read/write等可能阻塞的操作时也不用担心，协程网络库内部会帮你完成事件注册
void HttpProxyDo(){
	doDNSParse();         
	doTransmitToServer();
	doTransmitToClient();
}
```
