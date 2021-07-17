在http1.1协议下，我们可以通过如下几种方案来做：

1、压缩代码，去掉注释

2、对不依赖dom的js文件合理应用async和defer避免dom解析的阻塞

3、对css应用媒体查询，对某些特定场景的css避免加载。

4、合理调整文件的个数和大小，这里不能一味的合并所有css或者js，如果某个css或者js体积过大，同样影响效率，只能不断的调整测试。本质就是减少资源加载花费的RTT，并且不要超过浏览器对同一域名最大的并发数。

5、合理利用CDN。

6、应用域名分片技术。