# http报文
通用首部
Cache-Control：控制缓存行为
Connection：逐跳首部、连接的管理
Date：创建报文的日期时间
Pragma：报文指令
Transfer-Encoding：指定报文传输主体的编码方式
Upgrade：升级为其他协议
Via：代理服务器的相关信息
Warning：错误通知
请求首部
Accept：用户代理可以处理的媒体类型
Accept-Charset：优先的字符集
Accept-Encoding：优先的内容编码
Authorization：Web认证信息
Except：期待服务器的特定行为
Host：请求资源所在的服务器
if-Match：比较实体标记（ETag）
if-Modified-Since：比较资源的更新时间
Range：实体的字节范围请求
Refer：请求页面来源
TE：传输编码的优先级
User-Agent：HTTP客户端程序的信息
响应首部：
Accept-Ranges：是否接受字节范围请求
Age：推算资源创建经过的时间
ETag：资源的匹配信息
Location：令客户端重定向至指定UPI
Proxy-Authenticate：代理服务器对客户端的认证信息
WWW-Authenticate：服务器对客户端的认证信息
Server：HTTP服务器的安装信息
Vary：代理服务器的管理信息
实体首部：
Allow：资源可支持的HTTP方法
Content-Encoding：实体主体适用的编码方式
Content-Language：实体主体的自然语言
Content-Length：实体主体的大小
Content-Location：替代对应资源的URI
Content-MD5：实体主体的报文摘要
Content-Range：实体主体的位置范围
Content-Type：实体主体的媒体类型
EXpires：实体主体过期的日期时间
Last-Modified：资源的最后修改日期时间

# https
http是不安全的，相当于裸奔，报文传输过程中可能被别人嗅探、截获，解决方案就是加密报文。
加密方式分为两种，对称加密和非对称加密。
对称加密和非对称加密的区别：对称加密的加解密密钥是同一个，非对称加密的加解密密钥不同。对称加密和非对称加密在同样的加密密级下，非对称加密速度慢很多。
https整体思路是所有通信使用对称加密，但是对称加密双方需事先约定一个密钥且需保证密钥的安全性，所以密钥交换阶段要先使用非对称加密。
第一步，客户端发起请求，会自带一个随机数、客户端支持的加密协议
第二步，服务端把自己的加密证书发给客户端，同时还带一个随机数。所谓加密证书就是包含公钥、数字签名、校验码等一系列数据的内容，核心是传给客户端公钥，其余的内容都是用来校验公钥的安全性的
第三步，客户端用服务器发过来的公钥加密一个pre-master secret，发给服务器。
第四部，服务器接收到pre-master secret就明白后面可以开始对称加密的通信。

问题，为啥前面客户端和服务端都要发一个随机数？最后对称加密通信使用的是pre-master secret吗？
实际上，前两个随机数需要和pre-master secret进行数学运算，运算结果才是加密的密钥。同时由于随机数别人无法知道，如果发生了篡改，客户端和服务端都能第一时间发现，中断通信。

附：数字证书的意义
1. 如果数字证书上的公钥被篡改了，那客户端可以通过校验签名发现错误
2. 如果数字证书上的签名也被一并替换掉了，那客户端会解密失败
3. 如果替换证书的人有一本一样的证书，客户端还可以从绑定的域名判断中间被人篡改过

参考：
[https://blog.csdn.net/qq_31442743/article/details/116199453](HTTPS通信的过程的三个随机数的作用)
