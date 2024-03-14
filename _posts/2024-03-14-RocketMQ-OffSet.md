---
layout:     post
title:      "RocketMQ的偏移量分析"
subtitle:   "源码分析RocketMQ的偏移量更新逻辑"
date:       2024-03-14
author:     "hql0312"
tags:
    - RocketMQ
---

# 引言
RocketMQ的消息是存储于指定的MQ队列中，而消费端在消费消息时，也消费端在处理消息时，一个MQ队列，也只会被一个消费端订阅，同一个消费端可以处理同一个topic下的多个队列，当订阅的队列中有数据时，就会将获取到的数据提交到消费线程池进行处理，处理完成后，进行更新每个消费组对应的topic的偏移量，那在异步更新的逻辑中如何保证这个偏移量的值的顺序呢？
![消费线程.png](/img/rocketmq/rocketmq-offset.png)
# 分析
当前主要集中于源代码 `ConsumeRequest`中的 `run`方法，因为`ConsumeRequest`是对于消息消费的封装。在消息消费成功会直接调用如下的代码：
```ruby
ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
```
`processConsumeResult`: 主要功能就是处理当前消息的执行结果，这里会根据处理的消息结果是否成功来决定如何处理消息。当前只分析成功的情况，会执行以下的代码
```java
// 获取当前消息的偏移量
long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
// 当前的队列没有移除，则进行更新偏移量
if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
    // 更新的偏移量，会进行比较，以最大值写入，最后统一由具体的定时任务进行提交
    this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
}
```
执行到最后，会调用 `this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset`来更新偏移量，那具体的逻辑就集中于`updateOffset`方法，由 `RemoteBrokerOffsetStore`来实现：
```java
    public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
        if (mq != null) {
            AtomicLong offsetOld = this.offsetTable.get(mq);
            if (null == offsetOld) {
                offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
            }

            if (null != offsetOld) {
                if (increaseOnly) {
                    MixAll.compareAndIncreaseOnly(offsetOld, offset);
                } else {
                    offsetOld.set(offset);
                }
            }
        }
    }
```
`RemoteBrokerOffsetStore`维护了一个变量 `offsetTable`来处理每个MQ队列的偏移量，当处理完成一个消息队列的消息时，首先获取当前队列对应的MQ对应的值，如果没有则直接设置。否则确认是否根据`increaseOnly`字段来递增更新当前的值，从 `processConsumeResult`中知道，`increaseOnly` 为true, 则调用 `MixAll.compareAndIncreaseOnly`来更新值。
```java
    public static boolean compareAndIncreaseOnly(final AtomicLong target, final long value) {
        long prev = target.get();
        // 循环确认，当前的值是否大于已经存在的值
        while (value > prev) {
            boolean updated = target.compareAndSet(prev, value);
            if (updated)
                return true;

            prev = target.get();
        }

        return false;
    }
```
如此之后，就更新到对应的MQ的偏移量了。
上面的流程，仅是将数据更新到内存，并未进行持久化，那是如何触发的呢？
其实在 `MQClientInstance`实例启动的时候，会启动一个定时任务
```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            MQClientInstance.this.persistAllConsumerOffset();
        } catch (Exception e) {
            log.error("ScheduledTask persistAllConsumerOffset exception", e);
        }
    }
}, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

    private void persistAllConsumerOffset() {
        Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, MQConsumerInner> entry = it.next();
            MQConsumerInner impl = entry.getValue();
            
            impl.persistConsumerOffset();
        }
    }

```
该任务在启动后10，每5秒钟执行一次 `persistAllConsumerOffset`方法，同时会执行对应消费端方法 `persistConsumerOffset`,而实现类为 `DefaultMQPushConsumerImpl`.
```java
    public void persistConsumerOffset() {
        try {
            this.makeSureStateOK();
            Set<MessageQueue> mqs = new HashSet<MessageQueue>();
            Set<MessageQueue> allocateMq = this.rebalanceImpl.getProcessQueueTable().keySet();
            mqs.addAll(allocateMq);

            this.offsetStore.persistAll(mqs);
        } catch (Exception e) {
            log.error("group: " + this.defaultMQPushConsumer.getConsumerGroup() + " persistConsumerOffset exception", e);
        }
    }
```
该方法会将分配的每个队列的偏移量同步至MQ服务端。
# 注意点

1. 当一个队列因为`rebalance`，而不在当前的消费端时，有可能当前的偏移量不会被更新到服务端，导致该消息会被重新被新的消费端所消费。
2. 因为偏移量是先更新到内存，再通过定时任务更新至服务端，所以也有可能因为消费端的宕机或是重启，也有可能导致偏移量数据的更新失败，从而导致消息被重复消费。
