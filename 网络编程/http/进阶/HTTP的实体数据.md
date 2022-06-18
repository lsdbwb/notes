HTTP报文由header+body组成
下面详细介绍body


# 数据类型和编码
HTTP需要指定body里的内容是什么格式，这样对方才方便处理

HTTP借用了*多用途邮件扩展协议*定义的类型，叫做MIME type。MIME把数据分成了八大类，每个大类下再细分出多个子类，形式是“type/subtype”的字符串

**HTTP报文常见的body格式：**
1. text ： 文本格式的可读数据。
	text/html ： 超文本
	text/css : 层叠样式表
	text/plain : 纯文本

2. image :图像文件
	image/gif
	image/png
	image/jpg

3. audio/vedio : 音频、视频数据
	audio/mpeg
	vedio/mp4

4. application：数据格式不固定，可能是文本也可能是二进制，必须由上层应用程序来解释。常见的有application/json，application/javascript、application/pdf等，另外，如果实在是不知道数据是什么类型，像刚才说的“黑盒”，就会是*application/octet-stream，即不透明的二进制数据*


除了MIME type，还需要知道数据压缩方式（为了节省带宽，HTTP协议在传输时常常会对body数据进行压缩）

**HTTP协议使用Encoding-Type指定压缩方式**
1.  gzip：GNU zip压缩格式，也是互联网上最流行的压缩格式；
2.  deflate：zlib（deflate）压缩格式，流行程度仅次于gzip；
3.  br：一种专门为HTTP优化的新压缩算法（Brotli）。


# 数据类型和编码使用的头字段
有了MIME type和encoding-type，就可以正确的处理一个HTTP报文的body了

MIME type和encoding-type需要进行通信的双方进行协调，HTTP报文的header里应该有描述这两者类型的字段，用于客户端和服务端进行*body的内容协商*。

**客户端使用的是 Accept和Accept-Encoding字段**:
- **Accept**字段标记的是客户端可理解的MIME type，可以用“,”做分隔符列出多个类型，让服务器有更多的选择余地，例如下面的这个头：
`Accept: text/html,application/xml,image/webp,image/png`
- Accept-Encoding标记的是客户端支持的压缩格式，同样可以有多个
`Accept-Encoding: gzip, deflate, br`

**服务端使用的是Content-Type和Content-Encoding字段**:
- 服务器会在响应报文里用头字段*Content-Type*告诉客户端实体数据的真实类型，例如：`Content-Type: text/html`
- 服务器会在响应报文里用头字段*Content-Encoding*告诉客户端使用哪种压缩方式压缩的实体数据。例如`Content-Encoding: gzip`

**Accept-Encoding和Content-Encoding可以被省略**:
如果请求报文里没有Accept-Encoding字段，就表示客户端不支持压缩数据；如果响应报文里没有Content-Encoding字段，就表示响应数据没有被压缩。


# 语言类型与编码
语言是”国际化“问题，浏览器需要知道具体用哪种语言来显示网页

在计算机发展的早期，各个国家和地区的人们“各自为政”，发明了许多*字符编码方式*来处理文字，比如英语世界用的ASCII、汉语世界用的GBK、BIG5，日语世界用的Shift\_JIS等。同样的一段文字，用一种编码显示正常，换另一种编码后可能就会变得一团糟。

所以后来就出现了Unicode和UTF-8，把世界上所有的语言都容纳在一种编码方案里，遵循UTF-8字符编码方式的Unicode字符集也成为了互联网上的标准字符集。

## 语言类型使用的头字段
客户端使用Accept-Language字段
`Accept-Language: zh-CN, zh, en`

服务端使用Content-Language字段
`Content-Language: zh-CN`

## 编码类型使用的头字段
客户端使用的是Accept-Charset字段
`Accept-Charset: gbk, utf-8`

服务端没有专门表示编码类型的头字段，而是**Content-Type**字段的数据类型后面用“charset=xxx”来表示
`Content-Type: text/html; charset=utf-8`

# 内容协商的质量值
通信双方在使用以上字段进行内容协商时，可以*用一种特殊的“q”参数*指定每种格式的权值，权值越高，说明越希望对方使用此格式

权重的最大值是1，最小值是0.01，默认值是1，如果值是0就表示拒绝。具体的形式是在数据类型或语言代码后面加一个“;”，然后是“q=value”。

例子如下
```
客户端最希望接收html格式的报文，权值是1，其次是xml文件，权值是0.9；最后是任意类型的文件，权值是0.8
Accept: text/html,application/xml;q=0.9,*/*;q=0.8
```


# 内容协商的结果
内容协商的过程是不透明的，每个Web服务器使用的算法都不一样。但有的时候，服务器会在响应头里多加一个**Vary**字段，记录服务器在内容协商时参考的请求头字段，给出一点信息，例如：
```
服务端依据客户端发来报文中的以下三个字段决定响应报文的body的内容和格式
Vary: Accept-Encoding,User-Agent,Accept
```

Vary字段可以认为**是响应报文的一个特殊的“版本标记”**。每当Accept等请求头变化时，Vary也会随着响应报文一起变化。也就是说，同一个URI可能会有多个不同的“版本”，主要用在传输链路中间**的代理服务器实现缓存服务**


# 小结
![[Pasted image 20220618161525.png]]

1.  数据类型表示实体数据的内容是什么，使用的是MIME type，相关的头字段是Accept和Content-Type；
2.  数据编码表示实体数据的压缩方式，相关的头字段是Accept-Encoding和Content-Encoding；
3.  语言类型表示实体数据的自然语言，相关的头字段是Accept-Language和Content-Language；
4.  字符集表示实体数据的编码方式，相关的头字段是Accept-Charset和Content-Type；
5.  客户端需要在请求头里使用Accept等头字段与服务器进行“内容协商”，要求服务器返回最合适的数据；
6.  Accept等头字段可以用“,”顺序列出多个可能的选项，还可以用“;q=”参数来精确指定权重。