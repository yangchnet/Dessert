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

类似对称加密，Alice有一个源文本需要发送给Bob。

但这次，他们之间没有共享的密钥。Alice使用Bob的公钥加密消息，并发送密文給Bob。

Bob受到密文后，用他的私钥解密密文。

虽然公钥和私钥完全不同，但他们之间仍存在某些陷门函数的联系，就像我们之前在`Diffie-Hellman`中看到的那样。

这个机制这样运行：密钥成对出现，只有密钥对中的私钥可以解密被公钥加密的密文。

因此，即使Harry具有Alice发送的密文和Bob的公钥，他依然无法解密消息。

### 7.2 公钥共享

![20220729095544](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729095544.png)

公钥共享非常简单，Bob直接通过公共网络发送他的公钥给Alice而不用担心密钥可以解密任何消息。

这个密钥是公开的，因此任何人都可以用它进行加密消息，但只有Bob可以读取加密消息，即使他们从未交流过。

### 7.3 中间人攻击
![20220729095814](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729095814.png)

我们已经知道Harry不能通过Bob的公钥解密消息，但他依然可以影响公钥交换，例如使用自己的公钥替换Bob的公钥。

现在当ALice受到密钥，她依然认为这个Bob的公钥，但事实上是Harry的。因此当Alice使用该公钥加密消息，Harry就可以用他的私钥解密消息。

原因在于密钥只是一个数字，并没有身份标识其所有者。

面对这种问题，我们该怎么做？我们应该将身份信息添加到密钥中，这就是数字证书的做法。

### 7.4 数字证书

![20220729100157](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729100157.png)

Bob将他的密钥加入到证书中，证书中包含他的姓名或其他身份信息。数字证书就像现实世界的签证。

当我们怎么确认Bob确实是证书的所有者？如何组织Harry伪造一个具有Bob的身份信息却包含Harry密钥的证书。

就像现实世界，签证必须由权威机构在一系列身份认证后签发，在数字世界，数字证书也必须由某些证书颁发机构认可并签发。

这些证书颁发结构(`certificat authority CA`)和签证颁发结构被第三方认可，他们帮助我们阻止证书被伪造。

**证书签名**

![20220729100653](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729100653.png)

证书签名过程如下：

- Bob具有一对公私钥
- 第一步，他构造一个证书签名请求（`certificat signing request CSR`），这个CSR包含他的公钥和一些身份信息，例如他的姓名，组织和邮箱。
- 第二步，他使用他的私钥对CSR进行签名，并且将其发送给CA
- CA将验证Bob的身份，其可要求Bob提供更多证明身份的信息。
- 然后证书颁发机构使用证书中Bob的公钥校验Bob的签名，这将确认Bob确实拥有与证书中公钥相对应的私钥。
- 认证全部通过，CA将使用他们的私钥对证书进行签名，并将其发送回Bob

证书分享过程：
![20220729101306](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729101306.png)

现在Bob将把他的证书分享给Alice，其中包含他的公钥，而不再是只发送他的公钥

收到证书后，Alice可以很容易通过CA的公钥来验证Bob的证书。

因此，Harry不能将其中Bob的公钥换成他的公钥，因为Harry并没有CA的私钥用于重新对伪造的证书进行签名。

这其中的重点在于我们都信任CA，如果CA不可信，将其私钥泄露给Harry，那么我们将面临严重的安全问题。


### 7.5 CA-信任链

事实上，CA具有链式的结构。

![20220729101817](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729101817.png)

最高层是`根CA`，他们自己签发自己的证书，同样签发他们子机构的证书，也就是中间CA。

中间CA又可以签发其他中间CA的证书，或者他们可以签发`end-entity certificat`，也即叶子证书。

每个证书都将引用他们上层的证书，一直到root。

操作系统和浏览器存储了一系列已被信任的CA结构，通过这种方式，可有效的验证所有证书的真实性。

### 7.6 数字证书

![20220729102227](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729102227.png)

现在来看数字证书到底是如何工作的。

要对一个文档进行签名：
- 签名者首先需要对文档进行hash
- 然后hash值将被用签名者的私钥进行加密
- 加密结果叫数字签名`digital signature`
- 数字签名将被附加到原始文档中。

这是签名过程，那么如何验证签名是正确的。

![20220729102504](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729102504.png)

只需做一个反向操作：
- 首先将数字签名从文档中取出
- 使用signer的公钥进行解密得到hash值
- 然后使用同样的算法对原始文档进行hash
- 对比两个hash值
- 如果相同，则数字签名正确

## 8. TLS1.3 握手协议

现在我们有了上面的这些知识，可以来近距离看看TLS握手协议。

### 8.1 完整握手

![20220729103311](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729103311.png)

TLS1.3 完整握手从一个客户端发送给服务端的hello消息开始。事实上这个消息包含很多信息，但这里只列举一些较为重要的：

- 首先是客户端支持的协议版本号
- 然后是支持的 AEAD 对称密码套件列表。在这种情况下，有 2 个选项：AES-256-GCM 或 CHACHA20-POLY1305
- 接着是受支持的密钥交换组列表。例如，此客户端支持有限域 Diffie-Hellman Ephemeral 和 Elliptic-Curve Diffie-Hellman Ephemeral。
- 这就是为什么客户端还共享其 2 个公钥，1 个用于 Diffie-Hellman，另一个用于 Elliptic-Curve Diffie-Hellman。这样，无论选择何种算法，服务器都能够计算共享密钥。
- 客户端在此消息中发送的最后一个字段是它支持的签名算法列表。这是为了让服务器选择它应该使用哪种算法来签署整个握手。我们稍后会看到它是如何工作的。

![20220729103616](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729103616.png)

收到客户端的hello消息后，服务端同样返回一个hello消息，其中包含：

- 选择的协议版本： TLS1.3
- 选择的密码套件：AES-256-GCM
- 选择的密钥交换方法：Diffie-Hellman Ephemeral
- 所选方法的服务器公钥
- 下一个字段是对客户端证书的请求，它是可选的，只有在服务器想要通过其证书对客户端进行身份验证时才会发送。
- 通常在 HTTPS 网站上，只有服务器端需要将其证书发送给客户端。这将在此消息的下一个字段中发送。

![20220729103834](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729103834.png)

- 下一个字段是证书验证，实际上是到此为止的整个握手的签名。 以下是它的生成方式：从握手开始到证书请求的整个数据称为握手上下文。 我们将此上下文与服务器的证书连接，对其进行哈希处理，并使用客户端支持的一种签名算法使用服务器的私钥对哈希值进行签名。

- 以类似的方式，通过连接握手上下文、证书和证书来生成服务器完成，验证、散列它，并将散列值放入所选密码套件的 MAC 算法。 结果是整个握手的 MAC。

此处的服务器证书、证书验证和服务器完成称为身份验证消息，因为它们用于对服务器进行身份验证。 借助整个握手的签名和 MAC，TLS 1.3 可以安全地抵御多种类型的中间人降级攻击。

![20220729104017](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729104017.png)

现在客户端收到服务端的hello消息后，会以root权限验证服务端的证书，并检查整个握手过程的签名和MAC，确保没有被篡改。

如果一切正常，则客户端发送其完成消息，其中包含整个握手的 MAC，并且可以选择客户端的证书和证书验证以防服务器请求。

这就是整个TLS握手的流程。

### 8.2 带 PSK 恢复的简短握手

为了提高性能，客户端和服务器并不总是经过这个完整的握手。 有时，他们通过使用预共享密钥恢复来执行简短的握手。

思路是：在上一次握手之后，客户端和服务端已经相互认识，不需要再次认证。

因此服务器可能会向客户端发送一个或多个会话票据，该票据可以用作下次握手时的预共享密钥（pre-shared key  PSK）身份。 它与票证有效期以及其他一些信息一起使用。

![20220729104157](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729104157.png)

这样在下一次握手，客户端只需要发送一个简单的hello消息：

- 从上次握手中获得的 PSK 身份（或票证）列表
- 一种 PSK 密钥交换模式，可以是仅 PSK，也可以是带有 Diffie-Hellman 的 PSK。
- 如果使用具有 Diffie-Hellman 模式的 PSK，则客户端还需要共享其 Diffie-Hellman 公钥。 这将提供完美的前向保密，并允许服务器在需要时回退到完全握手。

服务收到客户端的hello消息后，将返回一个hello消息：

- 选定的预共享密钥身份
- 服务器的可选 Diffie-Hellman 公钥
- 和服务器完成就像在完整的握手中一样。

最后客户端发回它的 Finish，这就是 PSK 恢复的结束。

如您所见，在这个简短的握手中，客户端和服务器之间没有证书身份验证。

这也为零往返时间 (0-RTT) 数据提供了机会，这意味着客户端无需等待握手完成即可将其第一个应用程序数据发送到服务器。

### 8.3 零往返 （0-RTT）握手

![20220729104439](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729104439.png)

在 0-RTT 中，客户端将应用程序数据与客户端 hello 消息一起发送。 此数据使用从票证列表中的第一个 PSK 派生的密钥进行加密。

它还增加了 1 个字段：早期数据指示，告诉服务器有早期应用程序数据正在发送。

如果服务器接受了这个 0-RTT 请求，它将像正常的 PSK 恢复一样向服务器发送 hello，并且还可以选择一些应用程序数据。

客户端将以包含 MAC 的消息和早期数据结束指示符结束。 这就是 TLS 1.3 中 0 往返时间的工作原理。

它的优点是将延迟减少 1 个往返时间。 但缺点是打开了重放攻击的潜在威胁。 这意味着，黑客可以多次复制并发送相同的加密 0-RTT 请求到服务器。 为避免这种情况，服务器应用程序必须以一种对重复请求具有弹性的方式实现。

## 9. TLS1.2 VS TLS1.3

在文章结束之前，来做一个TLS1.3和TLS1.2的对比

![20220729104515](https://raw.githubusercontent.com/lich-Img/blogImg/master/img/20220729104515.png)

- TLS 1.3 具有更安全的密钥交换机制，其中易受攻击的 RSA 和其他静态密钥交换方法被移除，只留下短暂的 Diffie-Hellman 或 Elliptic-Curve Diffie-Hellman，因此实现了完美的前向保密。

- TLS 1.3 握手比 TLS 1.2 至少快 1 次往返。

- TLS 1.3 中的对称加密更加安全，因为 AEAD 密码套件是强制性的，并且它还从列表中删除了一些弱算法，例如块密码模式 (CBC)、RC4 或三重 DES。

- TLS 1.3 中的密码套件也更简单，因为它只包含 AEAD 算法和哈希算法。密钥交换和签名算法被移到单独的字段中。在 TLS 1.2 中，它们被合并到密码套件中。这使得推荐密码套件的数量变得太大，如果我没记错的话，TLS 1.2 中有 37 个选项。而在 TLS 1.3 中，只有 5 个。

- 接下来，TLS 1.3 还给了我们更强的签名，因为它对整个握手进行签名，而不仅仅是像 TLS 1.2 中那样覆盖其中的一部分。

- 最后但并非最不重要的一点是，椭圆曲线密码学在 TLS 1.3 中得到了极大的关注，并添加了一些更好的曲线算法，例如 Edward-curve 数字签名算法，它在不牺牲安全性的情况下速度更快。


**END**

## Origin Article

https://dev.to/techschoolguru/a-complete-overview-of-ssl-tls-and-its-cryptographic-system-36pd