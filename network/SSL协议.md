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

整体13个消息，四大步骤：
1. client hello: client生成的随机数Random，支持的加密套件Support Ciphers, SSL version, sessionID
2. server hello: server生成的随机数Random, 选择的加密套件, sessionID
3. Server Cetificate(Optional)
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


### client hello: 
client生成的随机数Random，支持的加密套件Support Ciphers, SSL version, sessionID

### server hello
server生成的随机数Random, 选择的加密套件, sessionID

### server发送证书秘钥
3. Server Cetificate：包含数字证书和到根CA整个链，使得客户端能用服务器证书中的服务器公钥认证服务器，第一次建立连接必须要有这个。
4. Server key exchange(Optional)：(DH需要，RSA不需要）作为秘钥生成的参数，视秘钥交换算法而定。
5. Certificate Request(Optional)：服务端可能会要求客户端自身进行验证，可以是单向的身份认证，也可以双向认证。
6. Server hello done：第二阶段的结束，第三阶段开始的信号

### client发送秘钥
7. client Certificate(Optional)：对服务器证明自身，可选。
8. client key exchange：client用服务端给的公钥加密 预备主秘钥（Pre-master-secret），发给server
9. certificate verify(Optional)：只有在客户端发送了自己证书到服务器端，这个消息才需要发送。其中包含一个签名，对从第一条消息以来的所有握手消息的HMAC值（用master_secret）进行签名。

### 
10. change cipher spec
11. client finished
12. change cipher spec
13. server finished
服务端握手结束通知。

使用私钥解密加密的Pre-master数据，基于之前(Client Hello 和 Server Hello)交换的两个明文随机数 random_C 和 random_S，计算得到协商密钥:enc_key=Fuc(random_C, random_S, Pre-Master);
计算之前所有接收信息的 hash 值，然后解密客户端发送的 encrypted_handshake_message，验证数据和密钥正确性;
发送一个 ChangeCipherSpec（告知客户端已经切换到协商过的加密套件状态，准备使用加密套件和 Session Secret加密数据了）
服务端也会使用 Session Secret 加密一段 Finish 消息发送给客户端，以验证之前通过握手建立起来的加解密通道是否成功。
根据之前的握手信息，如果客户端和服务端都能对Finish信息进行正常加解密且消息正确的被验证，则说明握手通道已经建立成功，接下来，双方可以使用上面产生的Session Secret对数据进行加密传输了。

消息验证代码（HMAC）和TLS数据完整性：
当服务器或客户端使用主密钥加密数据时，它还会计算明文数据的校验和（哈希值），这个校验和称为消息验证代码（MAC）。然后在发送之前将MAC包含在加密数据中。密钥用于从数据中生成MAC，以确保传输过程中攻击者无法从数据中生成相同的MAC，故而MAC被称为HMAC（哈希消息认证码）。另一方面，在接收到消息时，解密器将MAC与明文分开，然后用它的密钥计算明文的校验和，并将其与接收到的MAC进行比较，如果匹配，那我们就可以得出结论：数据在传输过程中没有被篡改。