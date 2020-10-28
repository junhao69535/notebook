# kafka

## kafka是什么
kafka是分布式发布-订阅消息系统。Kafka是一个分布式，可划分的，冗余备份的持久性的日志服务，它主要用于处理流式数据。


## 为什么要用kafka
缓冲和削峰：上游数据时有突发流量，下游可能扛不住，或者下游没有足够多的机器来保证冗余，kafka在中间可以起到一个缓冲的作用，把消息暂存在kafka中，下游服务就可以按照自己的节奏进行慢慢处理。

解耦和扩展性：项目开始的时候，并不能确定具体需求。消息队列可以作为一个接口层，解耦重要的业务流程。只需要遵守约定，针对数据编程即可获取扩展能力。

冗余：可以采用一对多的方式，一个生产者发布消息，可以被多个订阅topic的服务消费到，供多个毫无关联的业务使用。

健壮性：消息队列可以堆积请求，所以消费端业务即使短时间死掉，也不会影响主要业务的正常进行。

异步通信：很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。


## kafka中的ISR、AR代表什么？ISR的伸缩指的是什么？

1. ISR（In-Sync Replicas）：副本同步队列
2. AR（Assigned Replicas）：所有副本

ISR由leader维护，follwer从leader同步数据有一些延迟。当某个follower同步超时会剔除ISR，进入OSR。


## kafka的broker是干什么的
broker是消息的代理，produces往brokers里面指定topic写信息，


## kafka中的zk起到什么作用，可以不用zk吗？
早期版本kafka用zk做meta信息，consumer的消费状态，group管理以及offset值，选举，检测broker是否存货等。

## kafka follower如何与leader同步数据
利用isr，磁盘顺序写，零拷贝。


## 什么情况下broker会从isr中踢出去。
eader会维护一个与其基本保持同步的Replica列表，该列表称为ISR(in-sync Replica)，每个Partition都会有一个ISR，而且是由leader动态维护 ，如果一个follower比一个leader落后太多，或者超过一定时间未发起数据复制请求，则leader将其重ISR中移除 


## 为什么kafka那么快
Cache Filesystem Cache PageCache缓存

顺序写 由于现代的操作系统提供了预读和写技术，磁盘的顺序写大多数情况下比随机写内存还要快。

Zero-copy 零拷技术减少拷贝次数

Batching of Messages 批量量处理。合并小的请求，然后以流的方式进行交互，直顶网络上限。

Pull 拉模式 使用拉模式进行消息的获取消费，与消费端处理能力相符。

## kafka producer 打数据，ack  为 0， 1， -1 的时候代表啥， 设置 -1 的时候，什么情况下，leader 会认为一条消息 commit了
