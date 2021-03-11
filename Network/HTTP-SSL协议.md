https://cshihong.github.io/2019/05/09/SSL%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/

SSL协议：高层（握手协议 + 密码变化协议 + 告警协议） + 底层（记录协议）

最重要的是握手协议和记录协议：
- SSL记录协议：它建立在可靠传输（如TCP）之上，为高层协议提供数据封装，压缩，加密等功能。
- SSL握手协议：它建立在SSL记录协议之上，用于在实际的数据传输开始之前，通讯双方进行身份认证、协商加密算法、交换加密秘钥。

## SS建立阶段与IPSec VPN类比的话：

可以分为两个大阶段：
1）握手阶段：协商加密算法，认证服务器，建立用于加密和MAC(Message Authentication Code)用的秘钥.
该阶段类似于IPSec VPN IKE的作用

2）安全传输阶段：
在已建立的SSL连接里安全地传输数据。
该阶段类似于IPSec VPN ESP的作用。




SSL的建立过程总共有13个包，第一次建立至少需要9个包。
整体13个消息，四大步骤：
1. client hello: client生成的随机数Random，支持的加密套件Support Ciphers, SSL version, sessionID
2. server hello: server生成的随机数Random, 选择的加密套件, sessionID
3. Server Cetificate 第一次建立必须要有证书
4. Server key exchange(Optional)
5. Certificate Request(Optional)
6. Server hello done
7. client Certificate(Optional)
8. client key exchange
9. certificate verify(Optional)
10. change cipher spec
11. client finished
12. change cipher spec
13. server finished

### 第一阶段：client hello <===> server hello
1. client hello: client生成的随机数Random，支持的加密套件Support Ciphers, SSL version, sessionID
2. server hello: server生成的随机数Random, 选择的加密套件, sessionID

至此客户端和服务端都知道了：SSL版本、加密套件、随机数（Random1+ Random2，后续生成对称秘钥会用到)

### 第二阶段：server发送证书秘钥 ===> client
3. Server Cetificate：包含数字证书和到根CA整个链，使得客户端能用服务器证书中的服务器公钥认证服务器，第一次建立连接必须要有这个。
4. Server key exchange(Optional)：(DH需要，RSA不需要）作为秘钥生成的参数，视秘钥交换算法而定。
    > 在Diffie-Hellman中，客户端无法自行计算预主密钥pre-master-secret; 双方都有助于计算它，因此客户端需要从服务器获取Diffie-Hellman公钥。
    > 此时密钥交换也由签名保护。
5. Certificate Request(Optional)：服务器用来验证客户端。这一步是可选的，如果在对安全性要求高的常见可能用到（比如招行网上银行专业版，银企直连者插USB或动态秘钥）。
服务器端发出Certificate Request消息，要求客户端发他自己的证书过来进行验证。
该消息中包含服务器端支持的证书类型（RSA、DSA、ECDSA等）和服务器端所信任的所有证书发行机构的CA列表，客户端会用这些信息来筛选证书。
6. Server hello done：第二阶段的结束，第三阶段开始的信号

### 第三阶段：client发送秘钥 ===> server
7. client Certificate(Optional)：（要看前面第5个包）为了对服务器证明自身，客户要发送一个证书信息，这是可选的，在IIS中可以配置强制客户端证书认证。
（比如招行网上银行专业版，银企直连者插USB或动态秘钥）

----------------------------------------START 敲黑板，这里是关键--------------------------------------------------
最开始各自两个随机数都是明文传输的：client random, server random
只有这里pre-master是client公钥加密，server私钥解密，保证了隐私。

8. client key exchange：client用服务端给的公钥加密 预备主秘钥（Pre-master-secret），发给server.

根据之前从服务器端收到的随机数，按照不同的密钥交换算法，算出一个pre-master，发送给服务器。
服务器端收到pre-master算出main master。
而客户端当然也能自己通过pre-master算出main master。
如此以来双方就算出了对称密钥。

- 如果是RSA算法，会生成一个48字节的随机数，然后用server的公钥加密后再放入报文中。
- 如果是DH算法，这是发送的就是客户端的DH参数，之后服务器和客户端根据DH算法，各自计算出相同的pre-master secret.
----------------------------------------END 敲黑板，这里是关键--------------------------------------------------

9. certificate verify(Optional)：只有在客户端发送了自己证书到服务器端，这个消息才需要发送。
其中包含一个签名，对从第一条消息以来的所有握手消息的HMAC值（用master_secret）进行签名。

### 第四阶段：完成握手协议client finished <===> server finished
10. change cipher spec
11. client finished 客户端握手结束通知
 表示客户端的握手阶段已经结束。这一项同时也是前面发送的所有内容的hash值，用来供服务器校验（使用HMAC算法计算收到和发送的所有握手消息的摘要，然后通过RFC5246中定义的一个伪函数PRF计算出结果，加密后发送。此数据是为了在正式传输应用数据之前对刚刚握手建立起来的加解密通道进行验证。）


12. change cipher spec
13. server finished 服务端握手结束通知。
    - 1. 使用私钥解密加密的Pre-master数据，基于之前(Client Hello 和 Server Hello)交换的两个明文随机数 random_C 和 random_S，
    计算得到协商密钥:enc_key=Fuc(random_C, random_S, Pre-Master);
    - 2. 计算之前所有接收信息的 hash 值，然后解密客户端发送的 encrypted_handshake_message，验证数据和密钥正确性;
    - 3. 发送一个 ChangeCipherSpec（告知客户端已经切换到协商过的加密套件状态，准备使用加密套件和 Session Secret加密数据了）
    - 4. 服务端也会使用 Session Secret 加密一段 Finish 消息发送给客户端，以验证之前通过握手建立起来的加解密通道是否成功。
根据之前的握手信息，如果客户端和服务端都能对Finish信息进行正常加解密且消息正确的被验证，则说明握手通道已经建立成功，接下来，双方可以使用上面产生的Session Secret对数据进行加密传输了。

#### 消息验证代码（HMAC）和TLS数据完整性：
HMAC = Encrypt(Hash(messages), secretKey) 

当服务器或客户端使用主密钥加密数据时，它还会计算明文数据的校验和（哈希值），这个校验和称为消息验证代码（MAC）。然后在发送之前将MAC包含在加密数据中。
密钥用于从数据中生成MAC，以确保传输过程中攻击者无法从数据中生成相同的MAC，故而MAC被称为HMAC（哈希消息认证码）。
另一方面，在接收到消息时，解密器将MAC与明文分开，然后用它的密钥计算明文的校验和，并将其与接收到的MAC进行比较，如果匹配，那我们就可以得出结论：数据在传输过程中没有被篡改。



Client: 
- 客户端随机数
- 服务端随机数
- 服务端证书 + 公钥
- 客户端加密算法（服务器挑的）
- pre-master

Server: 
- 客户端随机数
- 服务端随机数
- 服务端证书 + 私钥
- 客户端加密算法（服务器挑的）
- pre-master ->算出 main master




#### 几个重要的secret key


#### SSL会话恢复： 
client hello <===> server hello, 
client finished <===>  server finished
SSL采用会话恢复的方式来减少SSL握手过程中造成的巨大开销。
会话恢复是指只要客户端和服务器已经通信过一次，它们就可以通过会话恢复的方式来跳过整个握手阶段二直接进行数据传输。


#### 应用数据传输：
在所有的握手阶段都完成之后，就可以开始传送应用数据了。
应用数据在传输之前，首先要附加上MAC secret，然后再对这个数据包使用write encryption key进行加密。
在服务端收到密文之后，使用Client write encryption key进行解密，
客户端收到服务端的数据之后使用Server write encryption key进行解密，
然后使用各自的write MAC key对数据的完整性包括是否被串改进行验证。

### 证书的吊销
https://hadyang.github.io/interview/docs/basic/net/https/
证书的吊销
CA 证书的吊销存在两种机制，一种是 在线检查（OCSP），客户端向 CA 机构发送请求检查公钥的靠谱性；第二种是客户端储存一份 CA 提供的 证书吊销列表（CRL），定期更新。前者要求查询服务器具备良好性能，后者要求每次更新提供下次更新的时间，一般时差在几天。安全性要求高的网站建议采用第一种方案。

大部分 CA 并不会提供吊销机制（CRL/OCSP），靠谱的方案是 为根证书提供中间证书，一旦中间证书的私钥泄漏或者证书过期，可以直接吊销中间证书并给用户颁发新的证书。中间证书还可以产生下一级中间证书，多级证书可以减少根证书的管理负担。


## 阿里云证书服务
https://help.aliyun.com/document_detail/98728.html

#### 问答时间
Q：为什么要用到随机数？
A：生成对称秘钥需要有参数，这个参数的随机性和动态性。

Q：服务器的私钥如何管理？
私钥放哪里不被窃取？


