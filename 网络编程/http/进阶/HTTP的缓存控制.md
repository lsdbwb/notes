# 为什么需要缓存
由于链路漫长，网络时延不可控，使用HTTP协议访问服务器资源的成本较高

将资源缓存起来复用，能够避免多次请求应答的通信成本，节约网络带宽，也可以加快响应速度

实际上，HTTP传输的*每一环节基本上都有缓存*

# 浏览器缓存资源
服务端缓存资源时更多和代理服务联系在一起

客户端即浏览器缓存资源时的HTTP请求流程：
1. 浏览器发现无缓存，于是向服务器发送HTTP报文请求资源
2. 服务器响应请求，返回资源，并标记资源的有效期
3. 浏览器收到响应报文后缓存资源，等待下次重用

## 服务器缓存控制
服务器使用以下几个头字段来进行缓存控制并且指示浏览器应该如何使用缓存：
1. **Cache-Control**：服务器标记资源有效期
```
Cache-Control:max-age=30
```
max-age=30标识资源的有效时期是30秒，即告诉浏览器这个页面只能缓存30秒，30秒后这个页面就过期了，需要重新去服务器获取
max-age是**生存时间**，时间的计算起点是*响应报文的创建时刻*（即Date字段，也就是离开服务器的时刻）

2. no-store：指示浏览器不允许缓存页面，适用于变化非常频繁的页面
3. no-cache：可以缓存，但在使用缓存前需要去服务器验证页面是否已经过期
4. must-revalidate：缓存如果没过期则可以使用，如果已经过期了还想使用则需要去服务端验证

以上字段结合起来的缓存控制机制：
![[Pasted image 20220621144923.png]]

## 客户端的缓存控制
客户端即使已经有了页面的缓存，也不一定能够使用，客户端也有相应的缓存控制机制，和服务端相互配合

浏览器也可以发送带**Cache-Control**字段的头
- 点刷新时：浏览器会在请求头里加一个“**Cache-Control: max-age=0**”，指明要一个最新的页面。浏览器的缓存的页面至少都保存了几秒钟，不符合条件，所以不会使用缓存页面。服务器看到max-age=0，也就会用一个最新生成的报文回应浏览器。
- Ctrl+F5的“强制刷新：其实是发了一个“**Cache-Control: no-cache**”，含义和“max-age=0”基本一样，就看后台的服务器怎么理解，通常两者的效果是相同的。

浏览器缓存何时生效：
进行”前进“、”后退“、”跳转“这些重定向动作时会使用浏览器的缓存
进行这些操作时浏览器只会发送普通的请求头，不会加上cache-control字段，所以就会检查缓存，直接利用之前的资源，不再进行网络通信。


# 条件请求
## 为什么需要条件请求
浏览器用“Cache-Control”做缓存控制只能是刷新数据，不能很好地利用缓存数据，又因为缓存会失效，使用前还必须要去服务器验证是否是最新版。

浏览器可以用*两个连续的请求组成“验证动作”*：先是一个HEAD，获取资源的修改时间等元信息，然后与缓存数据比较，如果没有改动就使用缓存，节省网络流量，否则就再发一个GET请求，获取最新的版本。

但是这样两个请求的网络通信成本太高了，因此HTTP协议就增加了*条件请求字段*，把两次请求放到一次请求里做(如果缓存有效，那么和上面的验证动作一样，都只需要一次请求。如果缓存失效，则服务端会直接将资源随着第一次响应报文传给浏览器)，把资源验证的责任交给服务器

## 条件请求过程
1. 第一次GET请求资源时，服务端需要在响应报文头加上缓存控制字段Cache-Control，还要提供“**Last-modified**”和“**ETag**”字段
	Last-modified : 文件的最后修改时间
	ETag：是“实体标签”（Entity Tag）的缩写，**是资源的一个唯一标识**，主要是用来解决修改时间无法准确区分文件变化的问题。
2. 后续浏览器请求资源时，请求报文加上条件请求字段以及缓存里的Last-modified和ETag原值，验证资源是否是最新的
3. 如果资源没有变，服务器就回应一个”**304 NOT MODIFIED**“，表明资源没有变化，浏览器于是可以更新一下有效期，放心地使用缓存了

![[Pasted image 20220621151632.png]]

共有5个，最常用的是两个字段
1. **if-modified-since:** ： 看文件的最新修改时间有没有变化
	将服务端传回来的报文头里面的**Last-modified**字段的值（文件的最新修改时间）发回给服务端进行对比
2. “**If-None-Match**” ： 看文件的内容是否有变化
	将服务端传回来的报文头里面的**ETag**字段的值发回给服务端进行对比

notes：
Etag还有强弱之分
- 强ETag要求资源在字节级别必须完全相符
- 弱ETag在值前有个“W/”标记，只要求资源在语义上没有变化，但内部可能会有部分发生了改变
