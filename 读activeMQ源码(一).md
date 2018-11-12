title: 读activeMQ源码(一)
date: 2015-12-09 00:54:49
tags: [java,activemq]
categories: [java]
---
最近的学习到了消息的阶段，查了下网上的资料，并没有很多具体介绍某一个消息中间件的源码是如何实现，帮自己下了源码想分析下去，希望能坚持。

源码[GIT下载地址](https://github.com/apache/activemq.git)

下载完之后如果要导入eclipse 执行mvn eclipse:eclipse 或者导入IDEA mvn idea:idea

说下个人对rpc和消息的理解:

- 这2种方式从架构上就是不同的，rpc一般都是同步的，消息都是异步的，用在不同的场景上。
