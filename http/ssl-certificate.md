# 2.5.4 升级 ECC 数字证书

SSL 层中的证书验证也是一个比较耗时的环节：服务器需要把自己的证书链全发给客户端，客户端接收后再逐一验证。证书环节我们关注两个方面优化：**证书校验** 、 **证书中非对称算法升级** 。

### 1. 证书校验优化

客户端在验证证书过程中，需要判断当前证书状态是否被撤销/过期等，需要再去访问 CA 下载 CRL（Certificate Revocation List，证书撤销列表）[^1]或者 OCSP（Online Certificate Status Protocol，在线证书状态协议）数据[^2]，这又会产生 DNS 查询、建立连接、收发数据等一系列网络通信，增加多个 RTT。

改进的方式是在服务端开启 OCSP Stapling，OCSP Stapling 原本需要客户端实时发起的 OCSP 请求转嫁给服务端，服务端通过预先访问 CA 获取 OCSP 响应，然后在握手时随着证书一起发给客户端，免去了客户端连接 CA 服务器查询的环节，解决了 OCSP 的隐私和性能问题。

1. 在 Nginx 中配置 OCSP Stapling 服务。
```nginx configuration
server {
    listen 443 ssl;
    server_name  thebyte.com.cn;
    index index.html;

    ssl_certificate         server.pem; #证书的.cer文件路径
    ssl_certificate_key     server-key.pem; #证书的.key文件

    # 开启 OCSP Stapling 当客户端访问时 NginX 将去指定的证书中查找 OCSP 服务的地址，获得响应内容后通过证书链下发给客户端。
    ssl_stapling on;
    ssl_stapling_verify on;# 启用OCSP响应验证，OCSP信息响应适用的证书
    ssl_trusted_certificate /path/to/xxx.pem;# 若 ssl_certificate 指令指定了完整的证书链，则 ssl_trusted_certificate 可省略。
    resolver 8.8.8.8 valid=60s;# 添加resolver解析OSCP响应服务器的主机名，valid表示缓存。
    resolver_timeout 2s;# resolver_timeout表示网络超时时间
}
```

2. 检查服务端是否已开启 OCSP Stapling。

```bash 
$ openssl s_client -connect thebyte.com.cn:443 -servername thebyte.com.cn -status -tlsextdebug < /dev/null 2>&1 | grep "OCSP" 
```
若结果中存在“successful”关键字，则表示已开启 OCSP Stapling 服务。
```plain
OCSP response:
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
```

### 2. 证书中非对称算法升级

HTTPS 常用的密钥交换算法有两种：
- 分别是 RSA 
- ECDHE（Elliptic Curve Diffie-Hellman Ephemeral，椭圆曲线 Diffie-Hellman 密钥交换）算法。

使用 ECDHE 算法的证书一般被称之为 ECC 证书，相应的 RSA 算法的证书就是 RSA 证书。

RSA 算法在密钥交换过程中需要进行大数的模幂运算，相对耗时。而 ECDHE 算法利用椭圆曲线密码学，可以用更小的密钥尺寸实现相同的安全级别。如图 2-19 所示，256 位 ECC Key 在安全性上等同于 3072 位 RSA Key。

:::center
  ![](../assets/ecc.png)<br/>
 图 2-19 ECC vs RSA
:::

所以，相比 RSA 证书，ECC 证书具有更高的安全性高以及处理速度更快的优点，尤其适合在移动设备上使用。


ECC 证书在 2019 年之前的缺点就是兼容性问题，古代的 XP 和 Android2.3 不支持这种加密方式。不过 2019 年 Nginx 发布了 1.11.0 版本，开始提供了对 RSA/ECC 双证书的支持，它可以在 TLS 握手的时候根据客户端支持的加密方法选择对应的证书，以向下兼容古代客户端。


在 Nginx 里可以用 ssl_ciphers、ssl_ecdh_curve 等指令配置服务器使用的密码套件和协议，把优先使用的放在前面，配置示例：

```plain
ssl_dyn_rec_enable on;
ssl_protocols TLSv1.3 TLSv1.2; // 优先使用 TLSv1.3 协议
ssl_ecdh_curve X25519:P-256;
ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:20m;
ssl_session_timeout 15m;
ssl_session_tickets off;
```
配置完成之后，使用 https://myssl.com/ 服务测试证书配置，如图 2-20 所示。

:::center
  ![](../assets/ssl-test.png)<br/>
 图 2-20 使用 myssl.com 测试证书
:::


[^1]: CRL（Certificate Revocation List）证书撤销列表，是由 CA 机构维护的一个列表，列表中包含已经被吊销的证书序列号和吊销时间
[^2]: OCSP（Online Certificate Status Protocol）在线证书状态协议，是一种改进的证书状态确认方法，用于减轻证书吊销检查的负载和提高数据传输的私密性，相比于 CRL ，OCSP提供了实时验证证书状态的能力。

