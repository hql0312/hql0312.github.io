---
layout:     post
title:      "RocketMQ��ƫ��������"
subtitle:   "Դ�����RocketMQ��ƫ���������߼�"
date:       2024-03-14
author:     "hql0312"
tags:
    - RocketMQ
---

# ����
RocketMQ����Ϣ�Ǵ洢��ָ����MQ�����У������Ѷ���������Ϣʱ��Ҳ���Ѷ��ڴ�����Ϣʱ��һ��MQ���У�Ҳֻ�ᱻһ�����Ѷ˶��ģ�ͬһ�����Ѷ˿��Դ���ͬһ��topic�µĶ�����У������ĵĶ�����������ʱ���ͻὫ��ȡ���������ύ�������̳߳ؽ��д���������ɺ󣬽��и���ÿ���������Ӧ��topic��ƫ�����������첽���µ��߼�����α�֤���ƫ������ֵ��˳���أ�
![�����߳�.png](/img/rocketmq/rocketmq-offset.png)
# ����
��ǰ��Ҫ������Դ���� `ConsumeRequest`�е� `run`��������Ϊ`ConsumeRequest`�Ƕ�����Ϣ���ѵķ�װ������Ϣ���ѳɹ���ֱ�ӵ������µĴ��룺
```ruby
ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
```
`processConsumeResult`: ��Ҫ���ܾ��Ǵ���ǰ��Ϣ��ִ�н�����������ݴ������Ϣ����Ƿ�ɹ���������δ�����Ϣ����ǰֻ�����ɹ����������ִ�����µĴ���
```java
// ��ȡ��ǰ��Ϣ��ƫ����
long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
// ��ǰ�Ķ���û���Ƴ�������и���ƫ����
if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
    // ���µ�ƫ����������бȽϣ������ֵд�룬���ͳһ�ɾ���Ķ�ʱ��������ύ
    this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
}
```
ִ�е���󣬻���� `this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset`������ƫ�������Ǿ�����߼��ͼ�����`updateOffset`�������� `RemoteBrokerOffsetStore`��ʵ�֣�
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
`RemoteBrokerOffsetStore`ά����һ������ `offsetTable`������ÿ��MQ���е�ƫ���������������һ����Ϣ���е���Ϣʱ�����Ȼ�ȡ��ǰ���ж�Ӧ��MQ��Ӧ��ֵ�����û����ֱ�����á�����ȷ���Ƿ����`increaseOnly`�ֶ����������µ�ǰ��ֵ���� `processConsumeResult`��֪����`increaseOnly` Ϊtrue, ����� `MixAll.compareAndIncreaseOnly`������ֵ��
```java
    public static boolean compareAndIncreaseOnly(final AtomicLong target, final long value) {
        long prev = target.get();
        // ѭ��ȷ�ϣ���ǰ��ֵ�Ƿ�����Ѿ����ڵ�ֵ
        while (value > prev) {
            boolean updated = target.compareAndSet(prev, value);
            if (updated)
                return true;

            prev = target.get();
        }

        return false;
    }
```
���֮�󣬾͸��µ���Ӧ��MQ��ƫ�����ˡ�
��������̣����ǽ����ݸ��µ��ڴ棬��δ���г־û���������δ������أ�
��ʵ�� `MQClientInstance`ʵ��������ʱ�򣬻�����һ����ʱ����
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
��������������10��ÿ5����ִ��һ�� `persistAllConsumerOffset`������ͬʱ��ִ�ж�Ӧ���Ѷ˷��� `persistConsumerOffset`,��ʵ����Ϊ `DefaultMQPushConsumerImpl`.
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
�÷����Ὣ�����ÿ�����е�ƫ����ͬ����MQ����ˡ�
# ע���

1. ��һ��������Ϊ`rebalance`�������ڵ�ǰ�����Ѷ�ʱ���п��ܵ�ǰ��ƫ�������ᱻ���µ�����ˣ����¸���Ϣ�ᱻ���±��µ����Ѷ������ѡ�
2. ��Ϊƫ�������ȸ��µ��ڴ棬��ͨ����ʱ�������������ˣ�����Ҳ�п�����Ϊ���Ѷ˵�崻�����������Ҳ�п��ܵ���ƫ�������ݵĸ���ʧ�ܣ��Ӷ�������Ϣ���ظ����ѡ�
