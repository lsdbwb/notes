# Reactor模式的概念
reactor模式 = nonblocking IO + IO multiplexing

reactor允许单个线程处理多个IO事件
![[Pasted image 20220709120044.png]]
如上图所示，一个主线程运行Reactor，使用epoll IO多路复用机制监听多个IO事件，自身管理多个句柄。当监听到事件发生时，通过dispatcher事件分发器分发事件到对应的句柄上去处理。

Reactor代码示例：
程序基本结构是一个事件循环(event-loop)，以事件驱动(event-driven)和事件回调的方式处理业务逻辑
![[Pasted image 20220709120442.png]]

# Reactor的优缺点
**优点：**
编程不难，效率也不错。适合于IO密集的任务。不仅可以用于读写socket，连接的建立(connect/accept)，甚至DNS的解析都可以通过非阻塞的方式完成，提高吞吐率。

**缺点：**
1. 要求事件回调函数本身*必须是非阻塞的*（因为是单线程，如果回调函数阻塞了，整个reactor就不能响应其他事件了）
2. 对于涉及网络IO请求响应式协议，容易割裂业务逻辑，使其分散于多个回调函数[[回调函数]]之中，相对不容易理解和维护