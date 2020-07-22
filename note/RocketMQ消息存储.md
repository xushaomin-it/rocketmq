## 存储概要设计

#### 主要存储文件:
- CommitLog文件: 消息存储文件, 所有消息主题的消息都存储在CommitLog文件中
- ConsumeQueue文件: 消息消费队列, 消息到达CommitLog文件后,
  将异步转发到消息消费队列,供消息消费者消费
- IndexFile文件: 消息索引文件, 主要存储消息Key与Offset的对应关系

![RocketMQ物理部署图](picture/RocketMQ消息存储设计原理.png)


#### 消息存储
`org.apache.rocketmq.store.DefaultMessageStore` 消息存储实现类

#### 消息发送存储流程

`org.apache.rocketmq.store.DefaultMessageStore.putMessage` 消息存储入口
