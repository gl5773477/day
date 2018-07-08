# [浅谈Charles抓取HTTPS原理]https://www.jianshu.com/p/405f9d76f8c4

## https证书
- https是需要证书。
- 分为单向认证和双向认证

[https单向认证和双向认证](https://www.jianshu.com/p/257a939b08f6)

### 单向认证
- 服务端认证客户端证书。可以强制，也可以不强制。
- 客户端认证服务端证书，服务端会下发证书，客户端关闭校验即可。

### 双向认证