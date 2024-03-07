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
![Redis 单线程模型-第 2 页.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681479509492-2423bcc5-5605-49d3-a4ab-d4202e7436a2.png#averageHue=%23fcfaf6&clientId=u2a8aec93-a9cd-4&from=ui&id=WxtW9&originHeight=1394&originWidth=974&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=89319&status=done&style=none&taskId=ud3468b47-cb95-43d5-b5a7-180220d1fbb&title=)
Redis 的通过IO多路复用来处理客户端连接，io多路复用通过一个线程来监听文件描述符的状态，当有连接进来时，这个文件描述的状态会变成可读或是可写。
分析的内容主要在server.c 文件

1. Server加载配置，进行EventLoop的初始化

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681464016984-37204fab-e941-4fe6-9e47-368469b3b65c.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=608&id=ueec294a9&originHeight=760&originWidth=888&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=84033&status=done&style=none&taskId=ue4cdd104-49d5-4941-8f6c-56408ebd0e4&title=&width=710.4)

2. Redis 调用 listenPort 监听了服务器端口，在server 的ipfd会保存服务端的监听的fd

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681461993718-4ec7db42-1ca6-4100-97b0-705cf7ea7cf5.png#averageHue=%2321201f&clientId=u2a8aec93-a9cd-4&from=paste&height=110&id=u5330a3cc&originHeight=130&originWidth=760&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=17887&status=done&style=none&taskId=u52438ec7-4a8b-4937-a082-4cf4cd0d116&title=&width=642)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681462641506-c417eefc-5e8d-474a-a325-f1740c50def4.png#averageHue=%231e1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=418&id=uc134853b&originHeight=522&originWidth=821&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=43752&status=done&style=none&taskId=u30466ba9-507d-4577-a71a-b18c47c13f1&title=&width=656.8)

3. 给增加可读的标识,会注册Accept时的回调函数，并加入EventLoop列表中

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681463036426-d25070d5-4578-45aa-a3c0-7aff5997621d.png#averageHue=%2322201f&clientId=u2a8aec93-a9cd-4&from=paste&height=102&id=u10c677cb&originHeight=128&originWidth=856&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=20579&status=done&style=none&taskId=u67a8086e-8679-4ea1-b71c-7858a6cc80b&title=&width=684.8)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681463190933-636340c7-0703-43d9-8bd0-68c6b75a3809.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=255&id=u2bc677cb&originHeight=408&originWidth=1082&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=50572&status=done&style=none&taskId=u402768ae-0658-48c0-ab7e-43964eee6a5&title=&width=677)

4. 当创建完成listen之后，进入eaMain方法，会有一个死循环来检查EventLoop列表中是否有可读写的fd

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681464137613-93ba9592-98b9-42d1-80e7-5ac0f7c3d03d.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=218&id=udac47793&originHeight=272&originWidth=862&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=28243&status=done&style=none&taskId=ud524634a-0020-4f75-875e-92752248ea4&title=&width=689.6)

5. 调用 select 进行阻塞等待可用的fd，并更新到 eventLoop的fd 列表中

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681465375646-76ef4108-583a-45aa-b3ed-ddfd0cb65e57.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=578&id=aJxJC&originHeight=883&originWidth=1041&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=117534&status=done&style=none&taskId=u097b94c3-a77c-47e7-bf8f-10d66ffa239&title=&width=681)

6. 当Accept 的数据触发后，会调用第3步的Accept的回调，并会进行创建Client ,调用路径为acceptTcphandler->acceptCommonHandler -> createClient -> connSetReadHandler

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681475372681-2defeaad-2abf-405d-ac24-e5ebbd02fe02.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=470&id=u84a74166&originHeight=587&originWidth=919&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=53621&status=done&style=none&taskId=u3d1a14ca-bc33-47b0-ad4a-7b676f6f910&title=&width=735.2)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681475473240-d6a95d23-32e8-4e2a-8065-e21e0ab63d49.png#averageHue=%23201f1e&clientId=u2a8aec93-a9cd-4&from=paste&height=246&id=u39bcc25e&originHeight=307&originWidth=1068&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=45935&status=done&style=none&taskId=ubaed4c19-6c75-472d-a07a-82f6dd60d64&title=&width=854.4)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681475549933-6149cbc3-1bbb-4104-8692-98526a32826b.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=346&id=u111a2802&originHeight=433&originWidth=1103&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=61085&status=done&style=none&taskId=u5cc4b42d-82c2-45a2-8dbd-d646ac5a79a&title=&width=882.4)

7. 当客户端被接收后，就可以开始接受客户端的请求，并处理命令了,当有可读的数据时，会回调到readQueryFromClient -> processInputBuffer -> processCommandAndResetClient->processCommand -> call

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681475726243-f6c465d3-f673-42ce-b924-d4b19d4de825.png#averageHue=%231f1f1e&clientId=u2a8aec93-a9cd-4&from=paste&height=274&id=u1f01fb38&originHeight=342&originWidth=979&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=41702&status=done&style=none&taskId=u0fb6abc9-2980-4867-a63a-06d372cf463&title=&width=783.2)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681475942079-44b0b3a4-2fdb-49b1-aa63-73b268e21936.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=283&id=ua4607d92&originHeight=354&originWidth=1013&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=48391&status=done&style=none&taskId=ud6c80a8f-359c-435b-b5f5-7c210e469bb&title=&width=810.4)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681476117240-d8b9be85-0ce3-4408-8bc3-fb1f7602e985.png#averageHue=%23292321&clientId=u2a8aec93-a9cd-4&from=paste&height=278&id=udde91fd1&originHeight=347&originWidth=1644&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=130675&status=done&style=none&taskId=u93e05483-7bcd-408f-b356-29a97239d19&title=&width=1315.2)
获取到redisCommand对象后，后面再进行调用 call
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681476266730-eb48fcdf-aaff-48a7-aa70-e2586ebc0b2c.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=234&id=u1babc09f&originHeight=292&originWidth=880&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=27133&status=done&style=none&taskId=u99660acf-2098-4710-99bf-818b68b72c3&title=&width=704)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681476158719-0b576cd4-f012-477a-bc6b-0eec8e68d5c3.png#averageHue=%231f1f1e&clientId=u2a8aec93-a9cd-4&from=paste&height=109&id=uccbe7732&originHeight=136&originWidth=750&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=11071&status=done&style=none&taskId=ufa7a1296-fcf6-485d-801e-96f56128fc4&title=&width=600)

8. 当命令执行完成后，将数据写入缓存区，当再次执行EventLoop的循环时，会检查缓冲区，如果有的话，会调用SendReplyToClient 

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681479718663-11f4332c-776a-43a8-b471-8fe17594f158.png#averageHue=%231f1e1e&clientId=u2a8aec93-a9cd-4&from=paste&height=371&id=u111f6b87&originHeight=464&originWidth=1012&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=50588&status=done&style=none&taskId=uf4340434-7af9-4f73-9f00-a9c2f459ab3&title=&width=809.6)
最终将缓存区的数据响应给客户端
最后，以下面的一张图来总结
![Redis 单线程模型-第 1 页.png](https://cdn.nlark.com/yuque/0/2023/png/28109690/1681479811000-ed0f4835-e270-4295-9774-ff19381d4260.png#averageHue=%23f9f2ea&clientId=u2a8aec93-a9cd-4&from=ui&id=ue2fa6435&originHeight=251&originWidth=944&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=29679&status=done&style=none&taskId=uc040f166-3e45-4a46-b6c2-188a33e9aa5&title=)
