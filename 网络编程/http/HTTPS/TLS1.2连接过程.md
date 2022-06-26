# HTTPS建立连接
在HTTP协议里，建立TCP连接后，浏览器会立即发送请求报文

在HTTPS协议里，为了保证连接的安全，建立TCP连接后，还需要在TCP之上建立安全连接（TLS握手），然后再开始收发报文

# TLS协议的构成
TLS由多个子协议构成：
1. **记录协议**（Record Protocol）：规定了TLS收发数据的基本单位：Record。它有点像是TCP里的segment，所有的其他子协议都需要通过记录协议发出。但多个记录数据可以在一个TCP包里一次性发出，也并不需要像TCP那样返回ACK
2. **警报协议**（Alert Protocol）：警报协议的职责是向对方发出告警信息。例如protocol_version就是不支持旧版本，bad_certificate就是证书有问题，收到警报后另一方可以选择继续，也可以立即终止连接
3. **握手协议**（Handshake Protocol）：TLS里最复杂的协议，比TCP握手复杂的多。浏览器和服务器在握手过程中协商*TLS版本号、随机数、密码套件*等信息，然后交换*证书和密匙参数*，最终双方协商得到会话密匙，用于后续的混合加密系统
4. **变更密码规范协议**：（Change Cipher Spec Protocol），通知对方后续数据都使用密文传输，在此之前都是明文传输


# ECDHE握手过程
ECDHE是非对称加密算法，前向安全，并且需要的数据量少[[DHE和ECDHE算法的原理#ECDHE算法]]

![[9caba6d4b527052bbe7168ed4013011e.png]]
首先是建立TCP连接
开始TLS握手过程
1. 客户端向服务端发送**Client Hello**消息，里面有客户端的TLS版本号，支持的密码套件列表以及扩展列表等，还有一个**随机数**，用来生成会话密匙
```
// client hello消息的大致格式：
Handshake Protocol: Client Hello
    Version: TLS 1.2 (0x0303)
    Random: 1cbf803321fd2623408dfe…
    Cipher Suites (17 suites)
        Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
        Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)
```
2. 服务端收到客户端的client hello消息后，也会发送一个**Server Hello**消息，对比一下版本号，也给出一个**随机数**，同时选择一个本次通信使用的密码套件
```
Handshake Protocol: Server Hello
    Version: TLS 1.2 (0x0303)
    Random: 0e6320f21bae50842e96…
    Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)
```
3. 服务端为了证明自己的身份，将自己的证书发送给客户端
4. 因为选择了ECDHE算法，因此在发送证书后选择发送“**Server Key Exchange**”消息，里面是**椭圆曲线的公匙（Server Params）**,用来实现密匙交换算法。还要加上自己的数字证书签名（私匙 + 摘要算出的）以证明自己的身份。
```
Handshake Protocol: Server Key Exchange
    EC Diffie-Hellman Server Params
        Curve Type: named_curve (0x03)
        Named Curve: x25519 (0x001d)
        Pubkey: 3b39deaf00217894e...
        Signature Algorithm: rsa_pkcs1_sha512 (0x0601)
        Signature: 37141adac38ea4...
```
5. 然后是**Server Hello Done**消息，表明服务端的消息发送完毕

这样第一个消息往返就结束了（两个TCP包），结果是客户端和服务器通过明文共享了三个信息：**Client Random、Server Random和Server Params**。

6. 客户端收到服务端的证书和数字签名后，先要去验证证书是不是真实有效的，然后使用证书附带的公匙去验证服务端数字签名（私匙加密的）。经过这两步如果没有问题就验证了服务端的身份

7. 客户端按照密码套件的要求，也生成一个**椭圆曲线的公匙（Client Params）**，用client key exchange消息发送给服务器。
```
Handshake Protocol: Client Key Exchange
    EC Diffie-Hellman Client Params
        Pubkey: 8c674d0e08dc27b5eaa…
```

现在客户端和服务器手里都拿到了密钥交换算法的两个参数（Client Params、Server Params），就用ECDHE算法一阵算，算出了一个新的东西，叫“**Pre-Master**”，其实也是一个随机数。

现在客户端和服务器手里有了三个随机数：**Client Random、Server Random和Pre-Master**。用这三个作为原始材料，就可以生成用于加密会话的*主密钥*，叫“**Master Secret**”。黑客因为拿不到Pre-Master因此也就得不到主密匙。
```
master_secret = PRF(pre_master_secret, "master secret",
                    ClientHello.random + ServerHello.random)
“PRF”就是伪随机数函数，它基于密码套件里的最后一个参数，比如这次的SHA384，通过摘要算法来再一次强化“Master Secret”的随机性。
```

主密钥有48字节，但它也不是最终用于通信的会话密钥，还会再用PRF扩展出更多的密钥，比如客户端发送用的会话密钥（client_write_key）、服务器发送用的会话密钥（server_write_key）等等，避免只用一个密钥带来的安全隐患。

9. 最后客户端发送**Change Cipher Spec**，然后再发送一个**Finished**消息。把之前所有发送的数据做个摘要，再加密一下，让服务器做个验证。告诉服务器接下来都使用*对称算法加密通信*了，用的加密算法就是打招呼时沟通的AES，加密对不对还需要服务端做个验证。
10. 服务端收到也是同样的操作，发送**Change Cipher Spec**，然后再发送一个**Finished**消息，双方都验证加密解密OK，然后TLS握手就结束了。后面就是发送和接收加密后的HTTP报文了


# RSA握手过程
上面的ECDHE是主流的TLS握手过程，与传统的握手过程有以下不同

1. 使用的是ECHDA算法，而不是RSA算法，所以会在服务端发送**Server key exchange消息交换ECDHA参数**。
2. 因为使用了ECDHA，因此客户端可以不用等到服务器发回“Finished”确认握手完毕，立即就发出HTTP报文，省去了一个消息往返的时间浪费。这叫做**TLS false start**。


![[cb9a89055eadb452b7835ba8db7c3ad2.png]]
大体的流程没有变，只是“Pre-Master”不再需要用算法生成，而是*客户端直接生成随机数，然后用服务器的公钥加密*，通过“**Client Key Exchange**”消息发给服务器。服务器再用私钥解密，这样双方也实现了共享三个随机数，就可以生成主密钥。

# 双向认证
上面所说的握手都只是**单向认证过程**，只确认了服务器的身份，没有确认客户端的身份，通常单向认证通过后已经建立了安全通信，用账号、密码等简单的手段就能够确认用户的真实身份

但为了防止账号、密码被盗，有的时候（比如网上银行）还会使用U盾给用户颁发客户端证书，实现“**双向认证**”，这样会更加安全

双向认证的流程也没有太多变化，只是在“**Server Hello Done**”之后，“**Client Key Exchange**”之前，客户端要发送“**Client Certificate**”消息，服务器收到后也把证书链走一遍，验证客户端的身份


## 小结

今天我们学习了HTTPS/TLS的握手，内容比较多、比较难，不过记住下面四点就可以。

1.  HTTPS协议会先与服务器执行TCP握手，然后执行TLS握手，才能建立安全连接；
    
2.  握手的目标是安全地交换对称密钥，需要三个随机数，**第三个随机数“Pre-Master”必须加密传输**，绝对不能让黑客破解；
    
3.  “Hello”消息交换随机数，*“Key Exchange”消息交换“Pre-Master”*；
    
4.  “Change Cipher Spec”之前传输的都是明文，之后都是对称密钥加密的密文。