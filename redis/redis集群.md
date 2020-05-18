# redis集群

以下笔记参考：https://blog.csdn.net/weixin_42711549/article/details/83061052

## 主从模式
Redis主从同步。数据可以从主服务器向任意数量的从服务器上同步，从服务器可以是关联其他从服务器的主服务器。这使得Redis可执行单层树复制。存盘可以有意无意的对数据进行写操作。由于完全实现了发布/订阅机制，使得从数据库在任何地方同步树时，可订阅一个频道并接收主服务器完整的消息发布 记录。同步对读取操作的可扩展性和数据冗余很有帮助。


### 工作原理
Redis的主从结构可以采用一主多从或者级联结构，Redis主从复制可以根据是否是全量分为全量同步和增量同步。

#### 全量同步
Redis全量复制一般发生在Slave初始化阶段，这时Slave需要将Master上的所有数据都复制一份。具体步骤如下：
* 从服务器连接主服务器，发送SYNC命令；
* 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令；
* 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令；
* 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照；
* 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令；
* 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

![](_v_images/20200508111916856_4904.png =800x)

完成上面几个步骤后就完成了从服务器数据初始化的所有操作，从服务器此时可以接收来自用户的读请求。

#### 增量同步
Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。

增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令。

#### 主从同步策略
主从刚刚连接的时候，进行全量同步；全同步结束后，进行增量同步。当然，如果有需要，slave 在任何时候都可以发起全量同步。redis 策略是，无论如何，首先会尝试进行增量同步，如不成功，要求从机进行全量同步。

**注，如果多个Slave断线了，需要重启的时候，因为只要Slave启动，就会发送sync请求和主机全量同步，当多个同时出现的时候，可能会导致Master IO剧增宕机。**


### 实验环境

|  主机名  |     ip和端口      | 主从 |
| ------- | ---------------- | ---- |
| server1 | 10.91.4.20:6888  | 主   |
| server2 | 10.91.4.247:6888 | 从   |
| server3 | 10.91.4.248:6888 | 从   |

**注，如果开启了防火墙，所有机器放行6888端口。如开启了iptables，则所有机器执行`iptables -I INPUT 4 -p tcp --dport 6888 -j ACCEPT`。**

#### redis安装
参考官方文档

#### redis配置文件
主服务器和从服务器的配置的区别在于从服务器需要配置slaveof。所有的配置的文件都是复制官方的配置文件进行修改。

##### 主服务器

```
# vim /etc/redis_6888.conf

daemonize yes
pidfile /var/run/redis_6888.pid
port 6888
bind 0.0.0.0
unixsokcet /dev/shm/redis_6888.sock
unisocketperm 755
logfile /var/log/redis/redis_6888.log
```

##### 从服务器

```
# vim /etc/redis_6888.conf

daemonize yes
pidfile /var/run/redis_6888.pid
port 6888
bind 0.0.0.0
unixsokcet /dev/shm/redis_6888.sock
unisocketperm 755
logfile /var/log/redis/redis_6888.log
slaveof 10.91.4.20 6888  # 和主服务器的区别
```


## Redis Sentinel（哨兵）模式
Redis的主从复制下，一旦主节点由于故障不能提供服务，需要人工将从节点晋升为主节点，同时还要通知应用方更新主节点地址，对于很多应用场景这种故障处理的方法是无法接受的。但是Redis从2.8开始正式提供了Redis Sentinel（哨兵）架构来解决这个问题。

Redis Sentinel是一个分布式架构，其中包含若干个Sentinel节点和Redis数据节点，每个Sentinel节点会对数据节点和其余Sentinel节点进行监控，当它发现节点不可达时，会对节点做下线标识。如果被标识的是主节点，它还会和其他Sentinel节点进行“协商”，当大多数Sentinel节点都认为主节点不可达时，它们会选举出一个Sentinel节点来完成自动故障转移的工作，同时会将这个变化通知给Redis应用方。整个过程完全是自动的，不需要人工来介入，所以这套方案很有效地解决了Redis的高可用问题。

### 工作原理
#### 三个定时监控任务
* 每隔10秒，每个Sentinel节点会向主节点和从节点发送info命令获取最新的拓扑结构。
* 每隔2秒，每个Sentinel节点会向Redis数据节点的__sentinel__:hello频道上发送该Sentinel节点对于主节点的判断以及当前Sentinel节点的信息，同时每个Sentinel节点也会订阅该频道，来了解其他Sentinel节点以及它们对主节点的判断。
* 每隔一秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达。

#### 主观下线
因为每隔一秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，当这些节点超过down-after-milliseconds没有进行有效回复，Sentinel节点就会对该节点做失败判定，这个行为叫做主观下线。

#### 客观下线
当Sentinel主观下线的节点是主节点时，该Sentinel节点会向其他Sentinel节点询问对主节点的判断，当超过<quorum>个数，那么意味着大部分的Sentinel节点都对这个主节点的下线做了同意的判定，于是该Sentinel节点认为主节点确实有问题，这时该Sentinel节点会做出客观下线的决定。

#### 领导者Sentinel节点选举
Raft算法：假设s1(sentinel-1)最先完成客观下线，它会向其余Sentinel节点发送命令，请求成为领导者；收到命令的Sentinel节点如果没有同意过其他Sentinel节点的请求，那么就会同意s1的请求，否则拒绝；如果s1发现自己的票数已经大于等于某个值，那么它将成为领导者。

#### 故障转移
* 领导者Sentinel节点在从节点列表中选出一个节点作为新的主节点
* 上一步的选取规则是与主节点复制相似度最高的从节点
* 领导者Sentinel节点让剩余的从节点成为新的主节点的从节点
* Sentinel节点集合会将原来的主节点更新为从节点，并保持着对其关注，当其恢复后命令它去复制新的主节点


### 实验环境
|  主机名  |     ip和端口      | 主从 |
| ------- | ----------------- | ---- |
| server1 | 10.91.4.20:26379  | 主   |
| server2 | 10.91.4.247:26379 | 从   |
| server3 | 10.91.4.248:26379 | 从   |

**注，如果开启了防火墙，所有机器放行6888端口。如开启了iptables，则所有机器执行`iptables -I INPUT 4 -p tcp --dport 26379 -j ACCEPT`。**

#### sentinel配置文件
没有主从之分，所有配置文件内容都一样。

```
# 复制官方的配置文件
cp sentinel.conf /etc/sentinel.conf
vim /etc/sentinel.conf

port 26379
bind 0.0.0.0
daemonize yes
logfile "/var/log/redis/sentinel.log"
sentinel monitor mymaster 10.91.4.20 6888 2
```

启动sentinel有两种方式：

```
redis-server /etc/sentinel.conf --sentinel
redis-sentinel /etc/sentinel.conf
```

### 虚拟ip高可用
redis和sentinel的集群已经配置完成，最后一步是利用Sentinel配置里的client-reconfig-script参数和自定义脚本实现虚拟IP的绑定和删除。因为绑定VIP需要ROOT权限，所以还需要分别修改三台服务器的sudo配置文件，给redis用户授权。

```
cat /etc/sudoers.d/redis
redis ALL=(ALL) NOPASSWD:/sbin/ip,NOPASSWD:/sbin/arping
```

配置自定义脚本

```
vim /var/lib/redis/failover.sh
#!/bin/bash

MASTER_IP=${6}
MY_IP='10.91.4.20'
VIP='10.91.4.30'
NETMASK='24'
INTERFACE='mgt'

if [ ${MASTER_IP} = ${MY_IP} ]; then
    sudo /sbin/ip addr add ${VIP}/${NETMASK} dev ${INTERFACE}
    sudo /sbin/arping -q -c 3 -A ${VIP} -I ${INTERFACE}
    exit 0
else
    sudo /sbin/ip addr del ${VIP}/${NETMASK} dev ${INTERFACE}
    exit 0
fi
exit 1
```

MY_IP、VIP和INTERFACE根据实际情况修改。

修改sentinel配置，添加如下内容

sentinel client-reconfig-script master-6379 /var/lib/redis/failover.sh
重启sentinel。

### 注意点
负载均衡和读写分离需在应用端中实现，适合读多写少，且能容忍短暂数据不一致性。或者使用中间件实现。


## 集群模式
sentinel模式只有主节点能进行写，而集群模式每个节点都读写。集群模式至少有6个节点。每个主节点至少有一个或多个从节点，从节点不工作，只有其主节点宕机了，该从节点自动变成主节点继续工作。

### 工作原理
参考官方文档

### 实验环境
redis在3.0之后的版本才提供集群模式，在3.0到5.0之间需要使用redis-trib.rb实现集群模式，在5.0之后继承到redis-cli命令中。由于需要机器过多，改为在docker部署：

|  主机名  |     ip和端口     | 主从 |
| ------- | --------------- | ---- |
| server1 | 172.17.0.3:7001 | 主   |
| server2 | 172.17.0.2:7002 | 主   |
| server3 | 172.17.0.4:7003 | 主   |
| server4 | 172.17.0.5:7004 | 从   |
| server5 | 172.17.0.6:7005 | 从   |
| server6 | 172.17.0.7:7006 | 从   |

#### 获取镜像

```
docker pull redis
```

确保redis版本在5.0及以上

#### 配置文件
根据节点情况修改

```
vim redis_7001.conf

bind 0.0.0.0
port 7001
cluster-enabled yes

cluster-config-file nodes_7001.conf
cluster-node-timeout 5000
appendonly yes
daemonize no
```

#### 运行
启动6个容器

```
docker run -d --name redis1 -p 7001:7001 -v /root/redis_cluster/7001/redis.conf:/etc/redis_7001.conf redis redis-server /etc/redis_7001.conf
...
docker run -d --name redis6 -p 7006:7006 -v /root/redis_cluster/7006/redis.conf:/etc/redis_7006.conf redis redis-server /etc/redis_7006.conf
```

进入某一个容器，执行集群命令

```
docker exec -it redis1 /bin/bash


redis-cli --cluster create 172.17.0.3:7001 172.17.0.2:7002 172.17.0.4:7003 172.17.0.5:7004 172.17.0.6:7005 172.17.0.7:7006 --cluster-replicas 1
```

看到以下信息，集群搭建完成：

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.17.0.6:7005 to 172.17.0.3:7001
Adding replica 172.17.0.7:7006 to 172.17.0.2:7002
Adding replica 172.17.0.5:7004 to 172.17.0.4:7003
M: 4d1c6412220b686417c171d7c4d1db5bedd26a96 172.17.0.3:7001
   slots:[0-5460] (5461 slots) master
M: a114991ef299b0cab2751b9cf36024d7fa37634e 172.17.0.2:7002
   slots:[5461-10922] (5462 slots) master
M: 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0 172.17.0.4:7003
   slots:[10923-16383] (5461 slots) master
S: 8d8e7ef8e7916ab6fafd3b2f482d0084df7196db 172.17.0.5:7004
   replicates 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0
S: 3d95b5f71d27b99e140cea383697c138411928de 172.17.0.6:7005
   replicates 4d1c6412220b686417c171d7c4d1db5bedd26a96
S: 583e0756a299b60f2b6d664bc391de89f8799dd9 172.17.0.7:7006
   replicates a114991ef299b0cab2751b9cf36024d7fa37634e
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 172.17.0.3:7001)
M: 4d1c6412220b686417c171d7c4d1db5bedd26a96 172.17.0.3:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 583e0756a299b60f2b6d664bc391de89f8799dd9 172.17.0.7:7006
   slots: (0 slots) slave
   replicates a114991ef299b0cab2751b9cf36024d7fa37634e
M: 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0 172.17.0.4:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: a114991ef299b0cab2751b9cf36024d7fa37634e 172.17.0.2:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 8d8e7ef8e7916ab6fafd3b2f482d0084df7196db 172.17.0.5:7004
   slots: (0 slots) slave
   replicates 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0
S: 3d95b5f71d27b99e140cea383697c138411928de 172.17.0.6:7005
   slots: (0 slots) slave
   replicates 4d1c6412220b686417c171d7c4d1db5bedd26a96
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### 连接集群
随便连接集群中的任一个主节点，执行命令时会自动选择slot：

```
root@855706dc965c:/data# redis-cli -c -p 7001 -h 172.17.0.3
172.17.0.3:7001> keys *
(empty array)
172.17.0.3:7001> set first_name zhao
OK
172.17.0.3:7001> get first_name
"zhao"
172.17.0.3:7001> set last_name junhao
-> Redirected to slot [16109] located at 172.17.0.4:7003
OK
172.17.0.4:7003> get last_name
"junhao"
```

#### failover
查看集群状态

```
root@a9988887fc5e:/data# redis-cli --cluster check 172.17.0.3:7001
172.17.0.3:7001 (4d1c6412...) -> 1 keys | 5461 slots | 1 slaves.
172.17.0.4:7003 (9ef5480e...) -> 1 keys | 5461 slots | 1 slaves.
172.17.0.2:7002 (a114991e...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.17.0.3:7001)
M: 4d1c6412220b686417c171d7c4d1db5bedd26a96 172.17.0.3:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 583e0756a299b60f2b6d664bc391de89f8799dd9 172.17.0.7:7006
   slots: (0 slots) slave
   replicates a114991ef299b0cab2751b9cf36024d7fa37634e
M: 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0 172.17.0.4:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: a114991ef299b0cab2751b9cf36024d7fa37634e 172.17.0.2:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 8d8e7ef8e7916ab6fafd3b2f482d0084df7196db 172.17.0.5:7004
   slots: (0 slots) slave
   replicates 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0
S: 3d95b5f71d27b99e140cea383697c138411928de 172.17.0.6:7005
   slots: (0 slots) slave
   replicates 4d1c6412220b686417c171d7c4d1db5bedd26a96
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

现在关掉某一个主节点：

```
docker stop redis2
```

再查看集群状态，可以看出，172.17.0.7:7006已经成为了主节点了，而172.17.0.2:7002已经不存在了。

```
root@a9988887fc5e:/data# redis-cli --cluster check 172.17.0.3:7001
Could not connect to Redis at 172.17.0.2:7002: No route to host
172.17.0.3:7001 (4d1c6412...) -> 1 keys | 5461 slots | 1 slaves.
172.17.0.7:7006 (583e0756...) -> 0 keys | 5462 slots | 0 slaves.
172.17.0.4:7003 (9ef5480e...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.17.0.3:7001)
M: 4d1c6412220b686417c171d7c4d1db5bedd26a96 172.17.0.3:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 583e0756a299b60f2b6d664bc391de89f8799dd9 172.17.0.7:7006
   slots:[5461-10922] (5462 slots) master
M: 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0 172.17.0.4:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 8d8e7ef8e7916ab6fafd3b2f482d0084df7196db 172.17.0.5:7004
   slots: (0 slots) slave
   replicates 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0
S: 3d95b5f71d27b99e140cea383697c138411928de 172.17.0.6:7005
   slots: (0 slots) slave
   replicates 4d1c6412220b686417c171d7c4d1db5bedd26a96
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

恢复宕掉的主节点：

```
docker start redis2
```

再次查看集群状态，可以看到172.17.0.2:7002已经起来了，但它从主节点变成从节点了。

```
root@a9988887fc5e:/data# redis-cli --cluster check 172.17.0.3:7001
172.17.0.3:7001 (4d1c6412...) -> 1 keys | 5461 slots | 1 slaves.
172.17.0.7:7006 (583e0756...) -> 0 keys | 5462 slots | 1 slaves.
172.17.0.4:7003 (9ef5480e...) -> 1 keys | 5461 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.17.0.3:7001)
M: 4d1c6412220b686417c171d7c4d1db5bedd26a96 172.17.0.3:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 583e0756a299b60f2b6d664bc391de89f8799dd9 172.17.0.7:7006
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0 172.17.0.4:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a114991ef299b0cab2751b9cf36024d7fa37634e 172.17.0.2:7002
   slots: (0 slots) slave
   replicates 583e0756a299b60f2b6d664bc391de89f8799dd9
S: 8d8e7ef8e7916ab6fafd3b2f482d0084df7196db 172.17.0.5:7004
   slots: (0 slots) slave
   replicates 9ef5480ec93e082e2d50d36f9f65e76f3bf398f0
S: 3d95b5f71d27b99e140cea383697c138411928de 172.17.0.6:7005
   slots: (0 slots) slave
   replicates 4d1c6412220b686417c171d7c4d1db5bedd26a96
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```