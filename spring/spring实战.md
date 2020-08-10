1）问题：使用RestTemplate，在请求第三方接口时，发生无法建立连接的情况

​      排查：请检查一下当前环境TCP连接情况，是否出现大量TCP连接处于CLOSE_WAIT状态

​      可能原因：未设置连接超时时间，导致使用了DefaultConnectionKeepAliveStrategy默认策略，如果响应头中未设置timeout，则返回-1，它表示永久有效

​      解决办法：设置connectTimeOut和readTimeout 

