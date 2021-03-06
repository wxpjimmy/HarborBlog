title: GRPC系列－(1)： GRPC的历史以及设计理念
date: 2016-09-04 17:30:48
tags: [rpc, google]
---
最近听说GRPC 1.0发布了，这是google开源的一个rpc框架，本着“google出品，必属精品”的原则，决定来学习一下这个框架，看看它跟现有的一些框架相比有什么优缺点，第一篇先简单介绍一个grpc框架的由来以及设计理念。  

## GRPC 介绍
GRPC是由google开源的一个高性能、跨语言的RPC框架，面向移动和HTTP/2设计。目前提供 C、Java 和 Go 语言版本，分别是：[grpc](https://github.com/grpc/grpc), [grpc-java](https://github.com/grpc/grpc-java), [grpc-go](https://github.com/grpc/grpc-go). 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持.  
GRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。  

## GRPC 动机
十多年来，google一直使用一个叫做Stubby的通用RPC基础框架，用它来连接数据中心内和跨数据中心运行的大量微服务。google内部系统早就接受了如今越来越流行的微服务架构，拥有一个统一的、跨平台的RPC的基础框架，使得服务的首次发行在效率、安全性、可靠性和行为分析上得到全面提升，这是支撑这一时期谷歌快速增长的关键。﻿﻿  
Stubby有许多非常棒的特性，然而，它没有基于任何标准，而且与google内部的基础框架耦合得太紧密以至于被认为不适合公开发布。随着SPDY、HTTP/2和QUIC的到来，许多类似特性在公共标准中出现，并提供了Stubby不支持的其它功能。很明显，是时候利用这些标准来重写Stubby，并将其适用性扩展到移动、物联网和云场景。﻿﻿而这就是GRPC在google诞生的动机。

## GRPC 设计理念
按照google自己官方说法，grpc的设计理念主要包含以下几方面：  
* 服务而非对象、消息而非引用 —— 促进微服务的系统间粗粒度消息交互设计理念，同时避免分布式对象的陷阱和分布式计算的谬误。﻿  
* 普遍并且简单 —— 该基础框架应该在任何流行的开发平台上适用，并且易于被个人在自己的平台上构建。它在CPU和内存有限的设备上也应该切实可行。﻿  
* 免费并且开源 —— 所有人可免费使用基本特性。以友好的许可协议开源方式发布所有交付件。﻿  
* 互通性 —— 该数据传输协议(Wire Protocol)必须遵循普通互联网基础框架。﻿  
* 通用并且高性能 —— 该框架应该适用于绝大多数用例场景，相比针对特定用例的框架，该框架只会牺牲一点性能。﻿  
* 分层 —— 该框架的关键是必须能够独立演进。对数据传输格式(Wire Format)的修改不应该影响应用层。﻿  
* 负载无关的 —— 不同的服务需要使用不同的消息类型和编码，例如protocol buffers、JSON、XML和Thrift，协议上和实现上必须满足这样的诉求。类似地，对负载压缩的诉求也因应用场景和负载类型不同而不同，协议上应该支持可插拔的压缩机制。﻿  
* 流 —— 存储系统依赖于流和流控来传递大数据集。像语音转文本或股票代码等其它服务，依靠流表达时间相关的消息序列。﻿  
* 阻塞式和非阻塞式 —— 支持异步和同步处理在客户端和服务端间交互的消息序列。这是在某些平台上缩放和处理流的关键。﻿  
* 取消和超时 —— 有的操作可能会用时很长，客户端运行正常时，可以通过取消操作让服务端回收资源。当任务因果链被追踪时，取消可以级联。客户端可能会被告知调用超时，此时服务就可以根据客户端的需求来调整自己的行为。﻿
* Lameducking —— 服务端必须支持优雅关闭，优雅关闭时拒绝新请求，但继续处理正在运行中的请求。﻿  
* 流控 —— 在客户端和服务端之间，计算能力和网络容量往往是不平衡的。流控可以更好的缓冲管理，以及保护系统免受来自异常活跃对端的拒绝服务(DOS)攻击。﻿  
* 可插拔 —— 数据传输协议(Wire Protocol)只是功能完备API基础框架的一部分。大型分布式系统需要安全、健康检查、负载均衡和故障恢复、监控、跟踪、日志等。实现上应该提供扩展点，以允许插入这些特性和默认实现。﻿  
* API扩展 ——﻿ 可能的话，在服务间协作的扩展应该最好使用接口扩展，而不是协议扩展。这种类型的扩展可以包括健康检查、服务内省、负载监测和负载均衡分配。﻿  
* 元数据交换 —— 常见的横切关注点，如认证或跟踪，依赖数据交换，但这不是服务公共接口中的一部分。部署依赖于他们将这些特性以不同速度演进到服务暴露的个别API的能力。﻿  
* 标准化状态码 —— 客户端通常以有限的方式响应API调用返回的错误。应该限制状态代码名字空间，使得这些错误处理决定更清晰。如果需要更丰富的特定域的状态，可以使用元数据交换机制来提供。﻿﻿

grpc的理念是很赞的！很多都是目前的rpc框架所不能满足的（比如thrift不支持异步调用），接下来会参考源代码提供的示例具体看下grpc的使用情况。

