---
title: Https
date: {{ date }}
categories:
- 其他
---

## Https 握手

我们知道 HTTPS 其实就是 HTTP + SSL/TLS 的合体，它其实还是 HTTP 协议，只是在外面加了一层，SSL 是一种加密安全协议，引入 SSL 的目的是为了解决 HTTP 协议在不可信网络中使用明文传输数据导致的安全性问题。可以说，整个互联网的通信安全，都是建立在SSL/TLS 的安全性之上的。

学过计算机网络的同学肯定都还记得 TCP 在建立连接时的三次握手，之所以需要 TCP 三次握手，是因为网络中存在延迟的重复分组，可能会导致服务器重复建立连接造成不必要的开销。SSL/TLS 协议在建立连接时与此类似，也需要客户端和服务器之间进行握手，但是其目的却大相径庭，在 SSL/TLS 握手的过程中，客户端和服务器彼此交换并验证证书，并协商出一个 “对话密钥” ，后续的所有通信都使用这个 “对话密钥” 进行加密，保证通信安全。

整个 SSL/TLS 的握手和通信过程，简单来说，其实可以分成下面三个阶段：

1. 打招呼
   当用户通过浏览器访问 HTTPS 站点时，浏览器会向服务器打个招呼（ClientHello），服务器也会和浏览器打个招呼（ServerHello）。所谓的打招呼，实际上是告诉彼此各自的 SSL/TLS 版本号以及各自支持的加密算法等，让彼此有一个初步了解。
2. 表明身份、验证身份
   第二步是整个过程中最复杂的一步，也是 HTTPS 通信中的关键。为了保证通信的安全，首先要保证我正在通信的人确实就是那个我想与之通信的人，服务器会发送一个证书来表明自己的身份，浏览器根据证书里的信息进行核实（为什么通过证书就可以证明身份呢？怎么通过证书来验证对方的身份呢？这个后面再说）。如果是双向认证的话，浏览器也会向服务器发送客户端证书。
3. 通信
   双方的身份都验证没问题之后，浏览器会和服务器协商出一个 “对话密钥” ，要注意这个 “对话密钥” 不能直接发给对方，而是要用一种只有对方才能懂的方式发给他，这样才能保证密钥不被别人截获（或者就算被截获了也看不懂）。

至此，握手就结束了。双方开始聊天，并通过 “对话密钥” 加密通信的数据。

[![img](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221102191943969-583958421.png)](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221102191943969-583958421.png)



## 证书

简单来说，数字证书就好比介绍信上的公章，有了它，就可以证明这份介绍信确实是由某个公司发出的，而证书可以用来证明任何一个东西的身份，只要这个东西能出示一份证明自己身份的证书即可，譬如可以用来验证某个网站的身份，可以验证某个文件是否可信等等。[《数字证书及 CA 的扫盲介绍》](https://program-think.blogspot.com/2010/02/introduce-digital-certificate-and-ca.html) 和 [《数字证书原理》](http://www.cnblogs.com/JeffreySun/archive/2010/06/24/1627247.html) 这篇博客对数字证书进行了很通俗的介绍。

知道了证书是什么之后，我们往往更关心它的原理，在上面介绍 SSL/TLS 握手的时候留了两个问题：为什么通过证书就可以证明身份呢？怎么通过证书来验证对方的身份呢？

这就要用到非对称加密了，非对称加密的一个重要特点是：使用公钥加密的数据必须使用私钥才能解密，同样的，使用私钥加密的数据必须使用公钥解密。正是因为这个特点，网站就可以在自己的证书中公开自己的公钥，并使用自己的私钥将自己的身份信息进行加密一起公开出来，这段被私钥加密的信息就是证书的数字签名，浏览器在获取到证书之后，通过证书里的公钥对签名进行解密，如果能成功解密，则说明证书确实是由这个网站发布的，因为只有这个网站知道他自己的私钥（如果他的私钥没有泄露的话）。

当然，如果只是简单的对数字签名进行校验的话，还不能完全保证这个证书确实就是网站所有，黑客完全可以在中间进行劫持，使用自己的私钥对网站身份信息进行加密，并将证书中的公钥替换成自己的公钥，这样浏览器同样可以解密数字签名，签名中身份信息也是完全合法的。这就好比那些地摊上伪造公章的小贩，他们可以伪造出和真正的公章完全一样的出来以假乱真。为了解决这个问题，信息安全的专家们引入了 CA 这个东西，所谓 CA ，全称为 Certificate Authority ，翻译成中文就是证书授权中心，它是专门负责管理和签发证书的第三方机构。因为证书颁发机构关系到所有互联网通信的身份安全，因此一定要是一个非常权威的机构，像 GeoTrust、GlobalSign 等等，这里有一份常见的 CA 清单。如果一个网站需要支持 HTTPS ，它就要一份证书来证明自己的身份，而这个证书必须从 CA 机构申请，大多数情况下申请数字证书的价格都不菲，不过也有一些免费的证书供个人使用，像最近比较火的 Let's Encrypt 。从安全性的角度来说，免费的和收费的证书没有任何区别，都可以为你的网站提供足够高的安全性，唯一的区别在于如果你从权威机构购买了付费的证书，一旦由于证书安全问题导致经济损失，可以获得一笔巨额的赔偿。

如果用户想得到一份属于自己的证书，他应先向 CA 提出申请。在 CA 判明申请者的身份后，便为他分配一个公钥，并且 CA 将该公钥与申请者的身份信息绑在一起，并为之签字后，便形成证书发给申请者。如果一个用户想鉴别另一个证书的真伪，他就用 CA 的公钥对那个证书上的签字进行验证，一旦验证通过，该证书就被认为是有效的。通过这种方式，黑客就不能简单的修改证书中的公钥了，因为现在公钥有了 CA 的数字签名，由 CA 来证明公钥的有效性，不能轻易被篡改，而黑客自己的公钥是很难被 CA 认可的，所以我们无需担心证书被篡改的问题了。

证书的申请流程：

[![img](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221103140938551-804246813.png)](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221103140938551-804246813.png)

CA 证书可以具有层级结构，它建立了自上而下的信任链，下级 CA 信任上级 CA ，下级 CA 由上级 CA 颁发证书并认证。 譬如 Google 的证书链如下图所示：

[![img](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221103141022495-238574050.png)](https://img2022.cnblogs.com/blog/2029699/202211/2029699-20221103141022495-238574050.png)

可以看出：google.com.hk 的 SSL 证书由 Google Internet Authority G2 这个 CA 来验证，而 Google Internet Authority G2 由 GeoTrust Global CA 来验证，GeoTrust Global CA 由 Equifax Secure Certificate Authority 来验证。这个最顶部的证书，我们称之为根证书（root certificate），那么谁来验证根证书呢？答案是它自己，根证书自己证明自己，换句话来说也就是根证书是不需要证明的。浏览器在验证证书时，从根证书开始，沿着证书链的路径依次向下验证，根证书是整个证书链的安全之本，如果根证书被篡改，整个证书体系的安全将受到威胁。所以不要轻易的相信根证书，当下次你访问某个网站遇到提示说，请安装我们的根证书，它可以让你访问我们网站的体验更流畅通信更安全时，最好留个心眼。在安装之前，不妨看看这几篇博客：[《12306的证书问题》](https://www.jayxon.com/12306-certificate/)、[《在线买火车票为什么要安装根证书？》](http://www.xieyidian.com/3213)。

最后总结一下，其实上面说的这些，什么非对称加密，数字签名，CA 机构，根证书等等，其实都是 PKI 的核心概念。PKI（Public Key Infrastructure）中文称作公钥基础设施，它提供公钥加密和数字签名服务的系统或平台，方便管理密钥和证书，从而建立起一个安全的网络环境。而数字证书最常见的格式是 X.509 ，所以这种公钥基础设施又称之为 PKIX 。

至此，我们大致弄懂了上面的异常信息，sun.security.validator.ValidatorException: PKIX path building failed，也就是在沿着证书链的路径验证证书时出现异常，验证失败了。



## Java中的证书

### Java证书概述

Java 在 JRE 的安装目录下也保存了一份默认可信的证书列表，这个列表一般是保存在 `$JRE/lib/security/cacerts` 文件中。要查看这个文件，可以使用类似 KeyStore Explorer 这样的软件，当然也可以使用 JRE 自带的 keytool 工具（后面再介绍），cacerts 文件的默认密码为 `changeit`。

可以进入自己所安装的java文件下去查看证书

```sh
# path: java运行环境下的 $JRE/lib/security/cacerts
# 同时会提示要求填写密码，默认密码为 changeit。
keytool -list -keystore {path}
```

我们知道，证书有很多种不同的存储格式，譬如 CA 在发布证书时，常常使用 PEM 格式，这种格式的好处是纯文本，内容是 BASE64 编码的，证书中使用 "-----BEGIN CERTIFICATE-----" 和 "-----END CERTIFICATE-----" 来标识。另外还有比较常用的二进制 DER 格式，在 Windows 平台上较常使用的 PKCS#12 格式等等。当然，不同格式的证书之间是可以相互转换的，我们可以使用 openssl 这个命令行工具来转换，参考 SSL Converter ，另外，想了解更多证书格式的，可以参考这里： [Various SSL/TLS Certificate File Types/Extensions](https://blogs.msdn.microsoft.com/kaushal/2010/11/04/various-ssltls-certificate-file-typesextensions/) 。

在 Java 平台下，证书常常被存储在 KeyStore 文件中，上面说的 cacerts 文件就是一个 KeyStore 文件，KeyStore 不仅可以存储数字证书，还可以存储密钥，存储在 KeyStore 文件中的对象有三种类型：Certificate、PrivateKey 和 SecretKey 。Certificate 就是证书，PrivateKey 是非对称加密中的私钥，SecretKey 用于对称加密，是对称加密中的密钥。KeyStore 文件根据用途，也有很多种不同的格式：JKS、JCEKS、PKCS12、DKS 等等，PixelsTech 上有一系列文章对 KeyStore 有深入的介绍，可以学习下：[Different types of keystore in Java](http://www.pixelstech.net/article/1408345768-Different-types-of-keystore-in-Java----Overview) 。

到目前为止，我们所说的 KeyStore 其实只是一种文件格式而已，实际上在 Java 的世界里 KeyStore 文件分成两种：KeyStore 和 TrustStore，这是两个比较容易混淆的概念，不过这两个东西从文件格式来看其实是一样的。KeyStore 保存私钥，用来加解密或者为别人做签名；TrustStore 保存一些可信任的证书，访问 HTTPS 时对被访问者进行认证，以确保它是可信任的。所以准确来说，上面的 cacerts 文件应该叫做 TrustStore 而不是 KeyStore，只是它的文件格式是 KeyStore 文件格式罢了。

### Java证书导入

1. 安装OpenSSL

   进入OpenSSL官网 https://slproweb.com/products/Win32OpenSSL.html 

   下载一个Win客户端直接点击安装

2. 配置环境变量

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/6130900a9c724b708ce4ed87a57b1e93.png)

3. 命令行获取指定域名的证书（www.googleapis.com:443 换成你想获取证书的域名）

```sh
openssl s_client -showcerts -connect www.googleapis.com:443
```

4. 将红框中的字符（即----Begin CERTIFICATE------ 和 ----END CERTIFICATE------ 之间的字符串）拷贝到一个txt文件，然后改名xxx.crt,后缀名改成crt就行，命名自己取，这样一个证书就拿到了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/86a6a6198e704b7fb74d5057c35ab68d.png)

5. 使用 `keytool` 导入证书

```sh
# name: 证书别名
# cacertpath: 证书库路径 $JRE/lib/security/cacerts
# path: 证书路径
keytool -import -alias {name} -keystore {cacertspath} -file {path}
```

6. 使用 `keytool` 列出证书（Optional）

```sh
# 在目录 $JRE/lib/security 下
keytool -list -keystore cacerts
```

