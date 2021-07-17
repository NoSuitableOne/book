# nginx负载均衡5种策略
1. 默认策略Round Robin
  ```
  upstream backserver { 
    server 192.168.0.14; 
    server 192.168.0.15; 
  } 
  ```
2. 在默认基础上加权
  ```
  upstream backserver { 
    server 192.168.0.14 weight=8;
    server 192.168.0.15 weight=10; 
  } 
  ```
3. ip hash
  ```
  upstream backserver { 
    ip_hash; 
    server 192.168.0.14:88; 
    server 192.168.0.15:80; 
  } 
  ```
4. url hash （需要第三方插件）
  ```
  upstream backserver { 
    server squid1:3128; 
    server squid2:3128; 
    hash $request_uri; 
    hash_method crc32; 
  } 
  ```
  ```
  proxy_pass http://backserver/; 
  upstream backserver{ 
  ip_hash; 
  server 127.0.0.1:9090 down; (down 表示当前的server暂时不参与负载) 
  server 127.0.0.1:8080 weight=2; (weight 默认为1.weight越大，负载的权重就越大) 
  server 127.0.0.1:6060; 
  server 127.0.0.1:7070 backup; (其它所有的非backup机器down或者忙的时候，请求backup机器) 
  } 
  ```
5. fair （需要第三方）
  ```
  upstream backserver { 
    server server1; 
    server server2; 
    fair; 
  } 
  ```