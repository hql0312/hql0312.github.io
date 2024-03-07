---
layout:     post
title:      "Redis启动分析"
subtitle:   "通过Redis的源码进行启动流程的分析"
date:       2024-03-07
author:     "hql0312"
tags:
    - Redis
---


Redis的启动的流程比较复杂，这里我们挑其中的一个流程来理解Redis单线程模型下的流程，即客户连接->执行命令->返回,对应的处理为三个函数acceptTcpHandler,readQueryFromClient和SendReplayToClient，通过阅读源码梳理流程如下：
![Redis 单线程模型-第 2 页.png](/img/redis/1.png)
Redis 的通过IO多路复用来处理客户端连接，io多路复用通过一个线程来监听文件描述符的状态，当有连接进来时，这个文件描述的状态会变成可读或是可写。
分析的内容主要在server.c 文件

1. Server加载配置，进行EventLoop的初始化

![image.png](/img/redis/2.png)

2. Redis 调用 listenPort 监听了服务器端口，在server 的ipfd会保存服务端的监听的fd

![image.png](/img/redis/3.png)
![image.png](/img/redis/4.png)

3. 给增加可读的标识,会注册Accept时的回调函数，并加入EventLoop列表中

![image.png](/img/redis/5.png)
![image.png](/img/redis/6.png)

4. 当创建完成listen之后，进入eaMain方法，会有一个死循环来检查EventLoop列表中是否有可读写的fd

![image.png](/img/redis/7.png)

5. 调用 select 进行阻塞等待可用的fd，并更新到 eventLoop的fd 列表中

![image.png](/img/redis/8.png)

6. 当Accept 的数据触发后，会调用第3步的Accept的回调，并会进行创建Client ,调用路径为acceptTcphandler->acceptCommonHandler -> createClient -> connSetReadHandler

![image.png](/img/redis/9.png)
![image.png](/img/redis/10.png)
![image.png](/img/redis/11.png)

7. 当客户端被接收后，就可以开始接受客户端的请求，并处理命令了,当有可读的数据时，会回调到readQueryFromClient -> processInputBuffer -> processCommandAndResetClient->processCommand -> call

![image.png](/img/redis/12.png)
![image.png](/img/redis/13.png)
![image.png](/img/redis/14.png)
获取到redisCommand对象后，后面再进行调用 call
![image.png](/img/redis/15.png)
![image.png](/img/redis/16.png)

8. 当命令执行完成后，将数据写入缓存区，当再次执行EventLoop的循环时，会检查缓冲区，如果有的话，会调用SendReplyToClient 

![image.png](/img/redis/17.png)
最终将缓存区的数据响应给客户端
最后，以下面的一张图来总结
![Redis 单线程模型-第 1 页.png](/img/redis/18.png)
