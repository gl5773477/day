## [深入了解 gRPC：协议](https://zhuanlan.zhihu.com/p/27961684)
gRPC 通常有四种模式，unary，client streaming，server streaming 以及 bidirectional streaming，对于底层 HTTP/2 来说，它们都是 stream，并且仍然是一套 request + response 模型。

## service层
- http的心跳检测
- 创建监听套接字：lis, err := net.Listen("tcp", addr)；
- 创建服务端：grpc.NewServer()；
- 常用选项设置：grpc.ServerOption设置
    - interceptor
    - sercurity
- 注册该服务
    - 通过proto向grpcServer注册以备监听
    - 向etcd注册
- 启动服务grpcServer.Serve(lis)

## 流程
- 启动流程
    1. dial会启动进程内lb，即`Balancer`的`Start()接口`。lb机制默认为roundRoubin轮询。
    2. lb通过resolver初始化一个watcher，并开go程监听变化更新`rr.watchAddrUpdates`。
    3. watchAddrUpdates内堵塞调用watcher的`Next()接口`获取更新的地址并写入本地地址表。
- 每次请求流程
    1. proto.go调用call.go文件中的`grpc.Invoke()`函数
    2. Invoke函数调用clientconn.go文件的`getTransport`函数
    3. getTransport函数调用balancer的`Get()`接口
    4. Get()接口实现了lb策略，并获取最终请求的服务地址，实现负载均衡。

## proto文件的生成
1. 执行protoc命令所在的目录为protos，下面分各个子目录。
2. 必须满足条件1。引入其他目录下的proto文件： import "enum/enum.proto"

# 相关问题
1. 重连机制
2. 日志追踪trace
3. 健康检查，业务监控


grpc添加restful api接口，只需要在proto文件的rpc定义里添加即可，如：[restful](https://grpc.io/blog/coreos)
[](http://dockone.io/article/2836)

google api项目集合：https://github.com/googleapis/googleapis

```protos
service EchoService {
  rpc Echo(EchoMessage) returns (EchoMessage) {
    option (google.api.http) = {
      post: "/v1/echo"
      body: "*"
    };
  }
}
```