# zookeeper

## zk是什么？
zookeeper是一个分布式协调服务。


## zk提供什么？
1. 文件系统
2. 通知机制


## zk文件系统
zk提供一个多层级的命名空间（节点称为znode）。zk为了保证高吞吐和低延迟，在内存中维护了这个树状的目录结构，这种特性使得zk不能用于存放大量数据，每个节点的存放数据上限为1M。


## 四种类型znode
1. PERSISTENT：持久化节点，客户端与zk断线后仍然存在。
2. PERSISITENT_SEQUENTAL：持久化顺序编号节点，客户端与zk断线后仍然存在，而且zk会给该节点进行顺序编号。
3. EPHEMRAL：临时目录节点，客户端与zk断开连接后，该节点被删除。
4. EPHEMERAL_SEQUENTIAL：临时顺序编号节点，客户端与zk断开连接后，该节点被删除，zk会给该节点顺序编号。


## zk通知机制
client对某个znode建立watcher事件，当该znode发生变化时，这些client会收到zk的通知，然后client可以根据znode变化来作出业务上的改变等。


## zk做了什么
1. 命名服务
2. 配置管理
3. 集群管理
4. 分布式锁
5. 队列管理


## zk的命名服务（文件系统）
命名服务是通过指定的名字来**获取资源**或者**服务的地址**，利用zk创建一个全局的路径，即是唯一的路径，这个路径可以作为一个名字，指向集群中的集群，提供服务的地址，或者一个远程的对象等。


## zk的配置管理（文件系统、通知机制）
程序分布式地部署在不同机器上，将程序的配置信息放在zk的znode下，当有配置发生改变时，也就是znode发生变化时，可以通过改变zk中某个目录节点的内容，利用watcher通知给各个客户端，从而更改配置。


## zk的集群管理（文件系统、通知机制）
集群管理主要工作：是否有机器推出和加入、选举master。对于第一点，所有机器约定在父目录下创建临时目录节点，然后监听父目录节点的子节点变化信息。一旦有机器挂掉，该机器与zk会断开，创建的临时节点也会被删除，通过通知机制通知所有其他机器。


## zk分布式锁
满足CP。


## zk队列管理（文件系统、通知机制）
两种类型的队列：
1. 同步队列，当一个队列的成员都聚齐时，这个队列才可用。
2. FIFO。

第一类，在约定目录创建临时目录节点，监听节点数目是否满足数量要求。

第二类，和分布式锁中的控制时序场景基本原理一致，入列有编号，出列按编号。


## zk数据复制
数据复制提供好处：
1. 容错：一个节点出错，不至于让整个系统停止工作，别的节点可以接管它的工作。
2. 提高系统的扩展能力：把负载分布到多个节点上，或者增加节点来提高系统的负载能力。
3. 提高性能：让客户端本地访问就近的节点，提高用户访问速度。

从客户端读写访问的透明度来看，数据复制集群系统分下面两种：
1. 写主：对数据的修改提交给指定的节点。读无此限制。即读写分离。
2. 写任意：对数据修改可提交给任意的节点，读也一样。

对于zk来说，采用的方式是**写任意**。通过增加机器，它的读吞吐能力和响应能力扩展性非常好，而写，随着机器的增多吞吐能力下降，而响应能力取决于具体实现，是延迟复制保持最终一致性还是立刻复制快速响应。

## zk工作原理
zk的核心是原子传播，这个机制保证各个server之间的同步。实现这个机制协议叫做zab协议（没有使用paxos，zab是自行实现的）。zab协议有两种模式：**恢复模式（选主）**和**广播模式（同步）**。当服务启动或者在领导者崩溃后，zab进入恢复模式，当领导者被选举出来，且大多数server完成了和leader的状态同步之后，恢复模式结束，进行广播模式。

## zk是如何保证事务顺序的一致性
采用**递增的事务id**来标识，所有的proposal（提议）都在被提出的时候加上zxid，zxid实际上是一个64位数字，高32位是epoch用来标识leader是否改变，如果有新的leader选出，epoch会自增。低32位用来递增计数，当新产生proposal的时候，会以来数据库的两阶段过程，首先会向其他server发出事务执行请求，如果超过半数机器都能执行且能成功，那么开始执行。


## zk下server工作状态
1. LOOKING：正在寻找leader。
2. LEADING：当前server即为选举出来的leader。
3. following：leader已经选举出来，当前server与之同步。


## zk是如何选取主leader
当leader崩溃或者lader失去大多数的follower，这时zk进入恢复模式，恢复模式需要重新选举出一个新的leader，让所有的server都恢复到一个正确的状态。zk选举算法有两种：一种是基于basic paxos；另一种是fast paxos（默认）。

### basic paxos


### fast paxos
在选举过程中，某server首先向所有server提议自己要成为leader，当其他server收到提议后，解决epoch和zxid的冲突，并接收对方的提议，然后向对方发送接收提议完成的信息，重复这个流程，最后一定能选出leader。


## zk的同步流程
1. leader等待server连接
2. follower连接leader，将最大的zxid发给leader。
3. leader根据follower的zxid确定同步点。
4. 完成同步后通知follower已经称为uptodate状态。
5. follwer收到uptodate消息后，又可以重新接受client的请求进行服务了。
