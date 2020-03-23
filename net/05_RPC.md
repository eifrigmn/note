# RPC

RPC (Remote Procedure Call) 远程过程调用，是一种通过网络从远程计算机上请求服务，而不需要了解底层网络协议的技术。

RPC协议嘉定某些传输协议的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。

## 结构

RPC的基本框架包含下列五个部分：

+ User：客户端，负责导入 (import) 远程接口及调用
+ User-stub：负责将接口、方法和参数通过约定的协议规范进行编码并通过本地的RPCRuntime实例传输到远端的实例
+ RPCRuntime：负责两端stub的数据交互
+ Server-stub：接收到客户端请求后进行解码并发起本地端调用
+ Server：服务端，负责导出 (export)远程接口

![rpc-structure-2](../assets/rpc-structure-2.png)

## 工作原理

![rpc-work-principle](../assets/rpc-work-principle.png)

1. 客户端像调用本地服务一样调用远程服务
2. Client-stub接收到调用后，将方法、参数序列化
3. 客户端通过sockets将消息发送到服务端
4. Server-stub收到消息后进行解码
5. Server-stub根据解码结果调用本地服务
6. 本地服务执行对应的请求，并将结果返回Serer-stub
7. Server-stub将结果数据序列化，打包成消息准备发送个客户端
8. 服务端通过sockets将消息发送给客户端
9. Client-stub接收到结果消息，并进行解码
10. 客户端得到最终的结果

## 参考

[浅谈RPC](https://dubbo.apache.org/zh-cn/blog/rpc-introduction.html)

https://developer.51cto.com/art/201906/597963.htm