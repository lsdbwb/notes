# ECDHE算法
TLS 1.2使用ECDHE算法的握手过程，在Client Hello和Server Hello里用到了ECDHE算法做密钥交换，参数完全公开，但却能够防止黑客攻击，算出只有通信双方才能知道的秘密Pre-Master。

先从ECDHE算法的名字说起。ECDHE就是“短暂-椭圆曲线-迪菲-赫尔曼”算法（ephemeral Elliptic Curve Diffie–Hellman），里面的关键字是“短暂”“椭圆曲线”和“迪菲-赫尔曼”，我先来讲“迪菲-赫尔曼”，也就是DH算法。

