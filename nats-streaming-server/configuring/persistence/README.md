# 持久化

NATS Streaming Server默认使用内存来存储状态，这意味着一旦服务停止，所有的状态都丢失了。当服务重启时，因为没有可以用于恢复的连接信息，运行中的应用程序将会停止接收消息并且新发送的消息将会被拒绝并返回`invalid publish request` 错误。支持并且设置了`Connection Lost` handler \(详情查看关于[连接状态](https://github.com/nats-io/stan.go#connection-status)的文档\) 的客户端将会收到连接丢失的错误`client has been replaced or is no longer registered`。

话虽如此，这种级别的持久性已经允许应用重启后继续消费之前的消息,避免应用程序因为连接断开\(网络或者程序崩溃导致\)而丢失消息。



* [文件存储](file_store.md)
* [SQL 存储](sql_store.md)



