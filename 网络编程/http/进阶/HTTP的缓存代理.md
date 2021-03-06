
客户端（浏览器）可以缓存服务器传来的资源，能够减少响应时间，节约带宽，提升客户体验

HTTP传输链路上，不仅仅客户端可以有缓存，中间经过的所有代理服务器也可以有缓存，代理服务器上的缓存是非常有用的，请求不必走完整个流程，可以就近获得响应结果

# 缓存代理服务
代理服务器在没有缓存时只是简单地转发客户端和服务端的报文，中间不会存储任何数据

加入缓存后代理服务器需要做的事：
1. 转发报文
2. 把报文存入自己的缓存
下一次再有相同的请求，代理服务器可以**直接发送304或者缓存数据**，不必再从源服务器获取。这就降低了客户端的响应时间，同时节约了源服务器的网络带宽。

缓存代理既是客户端也是服务器，所以既可以用客户端的缓存策略也可以用服务端的缓存策略（Cache-Control字段），但是代理并不是真正的客户端或者服务器，有其特殊性，因此还需要一些特殊的缓存控制字段

# 源服务器的缓存控制
客户端和代理是不一样的，客户端的缓存只能自己使用，而代理的缓存可以被其他客户端使用，因此需要用其他字段区分客户端缓存和代理缓存

- 使用**private和public**字段区分客户端缓存和代理缓存
	“private”表示缓存只能在客户端保存，是用户“私有”的，不能放在代理上与别人共享。而“public”的意思就是缓存完全开放，谁都可以存，谁都可以用
- 缓存失效后的重新验证（即使用条件请求if-modified和ETag）也要区分开。**must-revalidate**是要求必须请求到源服务器，**proxy-revalidate**则只要求代理的缓存过期后必须验证，客户端不必回源，客户端只要验证到代理服务器即可
- 代理的缓存的生命周期可以用新的**s-maxage**来表示，只限定资源在代理上能存多久，对于客户端仍然是使用**max-age**
- 还有一个代理专用的属性**no-transform**，代理有时候会对缓存下来的数据做一些优化，比如把图片生成png、webp等几种格式，方便今后的请求处理，而“no-transform”就会禁止这样做，不许“偷偷摸摸搞小动作

加上代理后源服务器的缓存控制策略：
![[09266657fa61d0d1a720ae3360fe9535.webp]]

# 客户端的缓存控制
策略：
![[47c1a69c800439e478c7a4ed40b8b992.webp]]

关于缓存的生存时间，多了两个新属性**max-stale**和**min-fresh**
- max-stale表示代理上的缓存过期了也能接受，但不能过期太久，超过x秒也会不要
- min-fresh表示缓存必须有效，并且在x秒后仍然有效

有的时候客户端还会发出一个特别的“**only-if-cached**”属性，表示只接受代理缓存的数据，不接受源服务器的响应。如果代理上没有缓存或者缓存过期，就应该给客户端返回一个504（Gateway Timeout）

# 其他问题
1. 第一个是“**Vary**”字段，它是内容协商的结果，相当于报文的一个版本标记。

同一个请求，经过内容协商后可能会有*不同的字符集、编码、浏览器等版本*。比如，“Vary: Accept-Encoding”“Vary: User-Agent”，缓存代理必须要存储这些不同的版本。

当再收到相同的请求时，代理就读取缓存里的“Vary”，对比请求头里相应的“ Accept-Encoding”“User-Agent”等字段，如果和上一个请求的*完全匹配*，比如都是“gzip”“Chrome”，就表示版本一致，可以返回缓存的数据。

2. 另一个问题是“**Purge**”，也就是“缓存清理”，它对于代理也是非常重要的功能，例如：

-   过期的数据应该及时淘汰，避免占用空间；
    
-   源站的资源有更新，需要删除旧版本，主动换成最新版（即刷新）；
    
-   有时候会缓存了一些本不该存储的信息，例如网络谣言或者危险链接，必须尽快把它们删除。
    

清理缓存的方法有很多，比较常用的一种做法是*使用自定义请求方法“PURGE”*，发给代理服务器，要求删除URI对应的缓存数据。