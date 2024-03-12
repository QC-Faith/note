Mq相关问题及常用使用场景





kafka分区及主题的区别

topic无序，分区有序



kafka reblance的实现过程

1. **Coordinator 选举**：
   - Consumer Group 中的每个 Consumer 实例都会与 Kafka 集群中的一个 Coordinator 进行通信，Coordinator 负责协调 Consumer Group 中的各个 Consumer 实例。
   - 当 Consumer Group 中的某个 Consumer 实例需要加入或离开 Consumer Group 时，Coordinator 将负责协调相关的操作。
2. **Rebalance 触发**：
   - 当发生 Consumer 实例变更、Consumer Group 中某个 Consumer 实例失效、或者订阅的 Topic 的 partition 数量发生变化时，Coordinator 将触发 Rebalance 过程。
3. **Partition 分配**：
   - Coordinator 将根据一定的策略重新分配 Topic 的 partition 给 Consumer Group 中的各个 Consumer 实例。常见的策略包括 Round-Robin、Range、Sticky 等。
4. **Partition 重新分配通知**：
   - Coordinator 将向 Consumer Group 中的各个 Consumer 实例发送重新分配后的 partition 信息。每个 Consumer 实例将收到新分配的 partition 列表，并更新自己的 partition 分配情况。
5. **Rebalance 完成**：
   - 各个 Consumer 实例根据收到的新分配的 partition 列表重新开始消费对应的 partition 数据。一旦所有 Consumer 实例完成了分区的重新分配，Rebalance 过程即完成。





reblance是kafka实现高可用的重要手段

Coordinator 负责协调产生新的**Leader** 

再根据offset继续消费数据





kafka如何保证数据不丢失

针对于不丢失的问题统一回复死信队列，kafka手动提交位移选择同步提交位移非异步

通过任务消费私信队列数据实现数据的补偿机制





rocketMq

延时队列使用rocketMq,实现最简单，同setDelayTimeLevel秒数来控制多久后消费者收到数据

主要用于需要数据延时的情况，如触达，订单失效等场景





分布式事务最终一致性最佳方案就是mq

RabbitMQ ，ActiveMQ ，RocketMQ 均支持死信队列的数据消费

Kafka 需要单独配置



消息重试最大次数，消费超时，业务异常，消息过期，消费者不可用均进入死信队列

