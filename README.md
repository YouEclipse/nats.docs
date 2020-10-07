---
description: NATS 中文文档由 @YouEclipse 业余时间维护，如有疑问或相关问题可以联系 you@golang.im
---

# 引言

## 消息系统的重要性

开发和部署分布式系统中的应用的通信系统会是一个比较大的挑战，目前的分布式系统有两种基本的通信模式，基于request/reply模式和基于RPC的事件流。随着技术的发展，一个现代的消息系统应该让这些变得简单，并且可伸缩，安全，能够跨地域和可观测的。

### 当今的分布式计算的要求

一个现代的消息系统应该能够支持多种通信模型，并且默认是安全的、支持多种QoS,并且提供多租户的真正贡献的基础设施。一个现代的消息系统应该包含:

* 对于微服务，边缘平台和设备默认是安全的
* 支持单一分布式通信技术中安全的多租户
* 透明的位置寻址和服务发现
* 基于系统的运行情况的弹性\(Resiliency\)
* 对于敏捷开发和大规模CI/CD和运维是简单的
* 内置负载均衡和动态自动扩展功能，高扩展性和高性能
* 从边缘设备到后端服务的一致身份和安全机制

### NATS

NATS旨在满足当今和未来的分布式计算需求。NATS 是一个为那些希望能够更多的聚焦于现代应用和服务而不必过多关心分布式通信系统的细节的开发和运维，而设计的足够简单但是安全的消息系统。

* 易于开发和运维使用
* 高性能
* 高可用
* 极度轻量
* 至多一次和至少一次投递
* 提供可观察性，弹性伸缩的服务和事件/数据流
* 超过30种的编程语言的支持的客户端
* 云原生，基于K8s并集成Prometheus的 CNCF 项目

### 使用场景

NATS可以运行在任何地方，从大型服务器到云服务器，到边缘网关甚至物联网设备，使用场景包含：

* Cloud Messaging
* 云消息系统
  * 服务\(微服务, 服务网格\)
  * 事件/数据流 \(可观察性, 数据分析, 机器学习/人工智能\)
* 命令与控制
  * 物联网和边缘计算
  * 遥测 / 传感器数据 / 命令和控制
* 增强或者替换现有消息系统

