# TCP重传机制
TCP的重传机制是TCP解决“两将军问题”以及保证可靠传输重要的一个机制。因为网络通信是不可靠的，所以一端向另一端发送的报文可能在网络中丢失，为了确保对方收到了消息，就需要对丢失的信息进行重传

TCP有四种重传机制：
1. 超时重传
2. 快速重传
3. SACK
4. D-SACK


# 超时重传
发送数据时，设定一个定时器，如果在一段时间后没有收到对方的ACK确认应答报文，就会重新发送数据，也就是我们常说的超时重传

TCP会在以下两种情况下进行超时重传：
![[Pasted image 20220704160510.png]]

>超时重传时间（Retransmission Timeout，RTO）应该设置为多少？

**结论：超时时间RTO应该略大于往返时延（round-trip time，RTT）**
![[Pasted image 20220704161214.png]]

## RTT的概念
![[Pasted image 20220704161029.png]]

## RTO过大或者过小的弊端
![[Pasted image 20220704161101.png]]

## RTO的计算
1. RTO根据报文往返的RTT的值来计算
2. 实际上「报文往返 RTT 的值」是经常变化的，因为我们的网络也是时常变化的。也就因为「报文往返 RTT 的值」 是经常波动变化的，所以「超时重传时间 RTO 的值」应该是一个**动态变化的值**。

我们来看看 Linux 是如何计算 `RTO` 的呢？

估计往返时间，通常需要采样以下两个：

-   需要 TCP 通过采样 RTT 的时间，然后进行加权平均，算出一个平滑 RTT 的值，而且这个值还是要不断变化的，因为网络状况不断地变化。
-   除了采样 RTT，还要采样 RTT 的波动范围，这样就避免如果 RTT 有一个大的波动的话，很难被发现的情况。


如果超时重发的数据，再次超时的时候，又需要重传的时候，TCP 的策略是**超时间隔加倍。**

也就是**每当遇到一次超时重传的时候，都会将下一次超时时间间隔设为先前值的两倍。两次超时，就说明网络环境差，不宜频繁反复发送。**


# 快速重传（Fast Retransmit）
超时重传的缺点是重传的时间间隔可能较长，因此需要一种更为快速的重传机制

快速重传机制不以时间为驱动，而是以**数据为驱动**进行重传

如下图所示，当连续收到三次重复的ack时，会触发快速重传
![[Pasted image 20220704162136.png]]


# SACK
快速重传解决了重传时间的问题，但是没有解决具体重传哪些报文的问题
例如到底是重传之前的1个报文还是重传所有的报文。

比如对于上面的例子，是重传 Seq2 呢？还是重传 Seq2、Seq3、Seq4、Seq5 呢？因为发送端并不清楚这连续的三个 Ack 2 是谁传回来的。

> SACK : 选择性确认(Selective Acknowledgment)
> 使用SACK可以指定需要重传哪些报文

这种方式需要在 TCP 头部「选项」字段里加一个 `SACK` 的东西，它**可以将缓存的地图发送给发送方**，这样发送方就可以知道哪些数据收到了，哪些数据没收到，知道了这些信息，就可以**只重传丢失的数据**。
![[Pasted image 20220704165214.png]]


# D-SACK
> D-SACK : duplicated SACK
> 使用SACK来指示哪些报文被重复接受了

如下图所示：
- 发送方的发送的两个报文接收方其实已经收到了，但是接收方发送的ACK消息在网络中丢失了。
- 发送方因为没有收到ACK所以触发超时重传机制，重新发送3000~3499的包，接收方发现收到重复数据，于是发送一个ACK=4000以及SACK=3000~3500的响应报文告诉发送方4000以前的数据我都收到了。
- 这样「发送方」就知道了，数据没有丢，是「接收方」的 ACK 确认报文丢了
![[Pasted image 20220704165318.png]]

**D-SACK的好处**
1. 可以让「发送方」知道，是发出去的包丢了，还是接收方回应的 ACK 包丢了;
2. 可以知道是不是「发送方」的数据包被网络延迟了;
3. 可以知道网络中是不是把「发送方」的数据包给复制了


