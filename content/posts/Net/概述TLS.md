---
author: "李昌"
title: "详解TLS（译）"
date: "2022-07-28"
tags: ["TLS"]
categories: ["Net"]
ShowToc: true
TocOpen: true
---

## 1. TLS在哪些地方被使用

TLS(Transport Layer Security),是为计算机网络中提供安全通信的密码学协议。

- HTTPS = HTTP +　TLS
- SMTPS = SMTPS + TLS
- FTPS = FTP + TLS
- ...

## 2. TLS给我们带来了什么

1. 认证
   - TLS检查通信双方的身份
   - 借助于非对称加密，TLS保证我们访问的是“真网站”，不是假冒的。
2. 加密
   - TLS 通过使用对称加密算法对其进行加密来保护交换的数据免受未经授权的访问。
3. 校验
   - TLS 通过检查消息验证码来识别传输过程中的任何数据更改

## 3. TLS通信的工作的基本流程

![20220728103423](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728103423.png)

通常，TLS包含2个过程，或者说2个协议。

- Handshake Protocol
在这个阶段，客户端和服务端：
    - 协商协议版本
    - 选择加密算法
    - 通过非对称加密算法验证对方身份呢
    - 建立一个共享的对称加密密钥以应用于接下来的通信
- Record Protocol
在这个阶段：
    - 所有发出的信息都被上个阶段商定的对称密钥加密
    - 信息被发送到对面
    - 接收方验证信息是否受到篡改
    - 如果未被篡改，信息将被解密

## 4. 为什么TLS要同时使用对称加密和非对称加密

很明显，对称加密不能保证通信的安全性，通信双方共享同一个密钥，他们没法进行彼此验证，并且很难保证密钥不被泄露。

那么为什么不都使用非对称加密？非对称加密远远慢于对称加密，大概是100～10000量级，因此，只使用非对称加密也是不可取的。

## 5. 对称加密

![20220728104617](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728104617.png)

对称加密的流程大概是这样：

Alice有一个文本想要发送给Bob，但不想任何其他人读取到它。

因此她使用之前已经共享过的密钥对文本进行加密，然后她通过公共互联网将加密文本发送给Bob

接收到加密文本后，Bob使用相同的密钥解密。

加密和解密使用的是相同的密钥，存在某种意义上的“对称”，因此我们称为“对成加密”

现在有一个黑客Harry，他可以从网络中截取到Alice和Bob之间的通信信息。通信信息是加密的，Harry没法读取它。

但，Harry可以对其进行更改！

### 5.1 位翻转攻击 Bit-flipping attack

![20220728105058](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728105058.png)

位翻转攻击的工作方式类似如下所述：

Alice发送消息到银行，想要转账給某人100元，这个消息被密钥加密后通过网络发送给银行。

Harry捕获了加密信息，他不能进行解密，但他可以翻转其中的某些位，然后再将修改过的信息发送给银行。

这样银行在解密消息后，得到的是与原来不同的消息内容，转账100元变成了转账900元。

显然这非常危险，我们需要某种方法来确保消息在传输过程中没有被改变。

### 5.2 加密认证 Authenticated Encryption (AE)

一种检查消息未被篡改的方式是使用加密认证。这种方式并不是简单的加密，而是需要对消息进行认证。

![20220728162235](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728162235.png)

**第一步是加密。**

Alice的源文本通过一个对称加密算法被加密，如`AES-256-GCM`或`CHACHA20`.

这个加密算法需要一个共享的密钥和一个随机值（nonce），或是一个初始化向量`initialization vector (IV)`作为输入，返回加密消息。

**第二步是认证。**

加密消息、密钥、随机nonce值作为MAC算法的输入，例如，如果你用的是`AES-256-GCM`可以使用`GMAC`，如果用的是`CHACHA20`可以使用`POLY1305`.

这些MAC算法类似一个hash加密函数，其输出为一个`MAC，message authentication code`.

这个MAC值将被标记到加密消息上，然后整个消息将被发送到Bob。因此，我们有时称这个MAC为认证标签（authentication tag）。

**添加附加数据（Associated Data (AD)）**

![20220728163150](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728163150.png)

在TLS1.3版本中，除了加密消息，我们还希望认证一些额外的数据，比如地址、端口、协议版本，或是序列号。该信息是未加密的，通信双方都知道

因此AD同样是MAC算法的数据，正因如此，整个过程被称为`Authenticated Encryption with Associated Data`，简写为`AEAD`.

**解密和MAC验证**

现在来看Bob如何检查加密消息是否被篡改。

![20220728163506](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728163506.png)

这是一个简单的逆过程，首先我们将标签从加密消息中取下。

然后加密消息将和密钥，nonce一起被再次输入到MAC算法中，这里注意需要使用与Alice相同的nonce值。通常在发送给接收者之前，随机数被填充到加密消息中。

附加消息（AD）同样被添加到MAC算法中，MAC算法输出另一个hash值。

![20220728163918](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728163918.png)

然后Bob可以简单的对比两个MAC值，如果是相同的，代表消息未被篡改，他可以安心的解密消息并处理，否则消息是被篡改过的，应予以丢弃。

### 5.3 交换密钥

上面描述的过程存在一个问题：Alice和Bob如何在不将密钥泄露的情况下共享同一个密钥。

![20220728164034](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728164034.png)

答案是：他们需要使用非对称加密技术或公钥加密来达到这个目的。例如`Diffie-Hellman Ephemeral`,`Elliptic-Curve Diffie-Hellman Ephemeral`.

#### 5.3.1 Diffie-Hellman 密钥交换

![20220728164214](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728164214.png)

首先，Alice和Bob协商确认两个数字：基数`g`，和模数`p`。这两个数字是众所周知的（任何人都可以知道）。

然后他们都为自己选择一个私有的数字，例如Alice是`a`,Bob是`b`.

然后Alice计算出她的公钥并将其发送给Bob。
```
A = (g^a) mod p
```

同样的，Bob计算他的公钥并发送给Alice
```
B = (g^b) mod p
```

Alice收到了Bob的公钥B，Bob收到了Alice的公钥A。

重点来了！

Alice进行如下计算：
```
S = (B^a) mod p
```

Bob进行如下计算：
```
S = (A^b) mod p
```

他们得到了同样的数字`S`.

证明如下：
```
(B^a) mod p = (g^b)^a mod p = ( g^(b*a) ) mod p
(A^b) mod p = (g^a)^b mod p = ( g^(a*b) ) mod p
```

因此Alice和Bob在不泄露密钥的情况下拥有了同样的密钥`S`.

![20220728164812](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728164812.png)

#### 5.3.2 密钥导出函数 Key Derivation Function - KDF

不同的加密算法可能需要不同长度的密钥，因此要想获取密钥，Alice和Bob必须将S输入相同的密钥导出函数`KDF`，然后输出将是所需长度的共享密钥。

在TLS1.3中，使用基于HMAC的密钥导出函数，称为`HKDF`

![20220728191505](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728191505.png)

通常，KDF需要以下输入：
- input key material (`IKM`),这里IKM为`S`
- 所需密钥长度
- 一个加密hash函数，例如`HMAC-SHA256`
- 可选的一些信息
- 可选的盐值

#### 5.3.3 陷门函数 Trapdoor function

现在让我们回到`Diffie-Hellman`密钥交换

我们知道`p, g, A, B`是众所周知的，因此Harry，同样可以访问这些数字。

我们可能会问：这种密钥分享机制安全吗？给定`p, g, A, B`，Harry是否可以计算出`a, b, S`

![20220728192023](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728192023.png)

幸运的是，如果我们选定较好的`p, g, a, b`，这些函数将成为陷门函数。

例如：
- 选择一个2048位的素数为p
- 选择 g 作为原根模 p
- 选择256位的随机值作为a，b

![20220728192235](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728192235.png)

陷门函数的名字来源于其可轻易的从一个方向计算出来，但从另一个方向很难计算。就像陷阱，进去很容易，但出来很困难。在这种情况下：
- 给定`p, g, a`，很容易计算出A
- 但给定`p, g, A`，很难计算出a

我们可以以O(log(a))的时间复杂度计算出A，这是众所周知的` Modular exponentiation problem`

但计算出a非常困难，这是一个离散对数问题，在现有的计算能力下需要很长时间才能计算出。

因此我们至少目前是安全的，或在下一代量子计算机到来之前。

但目前来看，一个很长时间才能解决的问题，意味着无解。

#### 5.3.4 静态或临时密钥？

![20220728192841](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728192841.png)

如果Alice和Bob的每次通信都使用相同的密钥`a`和`b`,这样Harry就可以从他们第一次会话就开始计算`a`.

虽然计算需要花费很长时间，但经过N次会话，Harry可能会得到正确的`a`。那么他就可以计算出正确的密钥`A`，因此他可以解密所有他记录的信息。

如何解决？

![20220728193058](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728193058.png)

答案是使用临时密钥，在每次会话中使用不同的密钥。这样及时Harry已经计算出第一次会话的密钥`a`，但他依然不能将其用于其他会话。

这在TLS中被称为完美前向保密

现在我们可以理解什么是`Diffie-Hellman Ephemeral`，只是`Diffie-Hellman`算法加上临时，短期有效的key。

## 6. 椭圆曲线密码学

![20220728193408](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728193408.png)

椭圆曲线密码学（或 ECC）是一种非对称密码学方法，其算法相似，但使用了不同的陷门函数。

该陷门函数基于椭圆曲线的代数结构。这就是这个名字的原因。

椭圆曲线密码学的一个重点是：它可以更小的密钥提供相同的安全等级。在与RSA的比较中经常可以看到它。

美国国家安全局 (NSA) 过去使用 ECC 384 位密钥保护其最高机密，该密钥提供与 RSA-7680 位密钥相同的安全级别。

然而，椭圆曲线密码学更容易成为量子计算攻击的目标。与破解 RSA 相比，Shor 的算法可以在具有更少量子资源的假设量子计算机上破解 ECC。

## 7. 非对称加密

现在让我们回到非对称加密算法，这种令人惊叹的技术得到了非常广泛的应用。

![20220728193744](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728193744.png)

我们已经使用`Diffie-Hellman Ephemeral` 和 `Elliptic-Curve Diffie-Hellman Ephemeral` 解释了其中一个应用：对称密钥的交换，

事实上，RSA算法在过去也被用于密钥交换，但从TLS1.3起由于各种攻击和不具有前向保密能力其已被移除，

非对称加密还被用于保密系统。下面是一些非对称加密算法：

- RSA with optimal asymmetric encryption padding (RSA-OAEP).
- RSA with public key cryptography standard 1 (RSA-PKCS1) with the latest version 2.2
- Elgamal Encryption algorithm

最后，非对称加密的一个重要特性就是数字签名，TLS被广泛用于身份验证。

一些在TLS使用的流行的数字签名算法有：

- RSA with Probabilitic Signature Scheme.
- Elliptic-Curve Digital Signature Algorithm.
- Edwards-Curve Digital Signature Algorithm.

下面我们会介绍数字证书，但先让我们来看一下非对称加密系统是如何工作的。

### 7.1 非对称加密

![20220728194352](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220728194352.png)

TODO

## Reference

https://dev.to/techschoolguru/a-complete-overview-of-ssl-tls-and-its-cryptographic-system-36pd