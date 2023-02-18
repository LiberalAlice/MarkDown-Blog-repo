# Blog-MD-repo
# zookeeper分布式入门



## 为什么需要zookeeper

1. #### zookeeper是什么？

   > 官方说明，ZooKeeper: A Distributed Coordination Service for Distributed Applications

   分布式应用协调服务

2. #### 为什么分布式需要协调？

   简单来说就是在分布式的模式下，如果资源只存放在一台主机中，如果该主机宕机或故障，则其他服务无法使用资源，造成其他服务崩溃。包括比如服务地址发生变动，如何同步到集群中；协调客户端和服务端等。为了保证系统能符合BASE理论而引入的中间件。

   C:\Program Files\PicGo

   ##### CAP 理论

   - **一致性**：在分布式环境中，一致性是指数据在多个副本之间是否能够保持一致的特性，等同于所有节点访问同一份最新的数据副本。在一致性的需求下，当一个系统在数据一致的状态下执行更新操作后，应该保证系统的数据仍然处于一致的状态。
   - **可用性：**每次请求都能获取到正确的响应，但是不保证获取的数据为最新数据。
   - **分区容错性：**分布式系统在遇到任何网络分区故障的时候，仍然需要能够保证对外提供满足一致性和可用性的服务，除非是整个网络环境都发生了故障。

   一个分布式系统最多只能同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）这三项中的两项。

   在这三个基本需求中，最多只能同时满足其中的两项，P 是必须的，因此只能在 CP 和 AP 中选择，zookeeper 保证的是 CP，对比 spring cloud 系统中的注册中心 eruka 实现的是 AP。

   ### BASE 理论

   BASE 是 Basically Available(基本可用)、Soft-state(软状态) 和 Eventually Consistent(最终一致性) 三个短语的缩写。

   - **基本可用：**在分布式系统出现故障，允许损失部分可用性（服务降级、页面降级）。
   - **软状态：**允许分布式系统出现中间状态。而且中间状态不影响系统的可用性。这里的中间状态是指不同的 data replication（数据备份节点）之间的数据更新可以出现延时的最终一致性。
   - **最终一致性：**data replications 经过一段时间达到一致性。

   BASE 理论是对 CAP 中的一致性和可用性进行一个权衡的结果，理论的核心思想就是：我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性。

## zookeeper结构与底层数据模型

1. ### zookeeper的宏观结构和工作流程![14022952-05a16aaeded6dc06](https://s2.loli.net/2023/02/18/dnwfK5joAlvBMsX.png)

由图可以看到，zookeeper服务器是由多个follower server和一个leader server组成，zookeeper会维护他们的状态，数据，事物日志以及快照。客户端通过TCP连接，来发送请求，获取响应，监听事件等。如果一个server宕机，客户端的连接会被转发到其他的服务器，保证服务的可用性。

#### zookeeper的角色

Leader：zookeeper集群中写请求唯一的调度者和处理者，增删改的请求都交由leader处理，如果leader宕机，所有的server会进入选举的looking状态，由follower中投票出一个新的leader后，zookeeper才会继续响应写请求。

Follower：集群中负责查询（读请求）的处理者，对写请求会转发给leader，会参与到选举中。

Observer：观察最新的状态变化并同步，可以处理读请求，写请求仍然转发给leader，不参与选举只提供服务。

zookeeper中划分角色是为了实现一些特性，如原子性，全局数据一致性，顺序性

#### 选举机制与节点状态

为了保证主从角色和当leader宕机时的正常运行，使用选举机制来进行启动时或宕机时的leader初始化。

服务器具有四种状态：LOOKING、FOLLOWING、LEADING、OBSERVING。

执行选举情况（需2台以上的服务器）

+ zookeeper启动，无leader。

1. 每个服务器都发出投票，初始情况都投自己，初始时的Zxid相同，以myid和Zxid来表示投举的服务器。
2. 接受各服务器的投票后，先校验投票的有效性（是否是本次选举-根据节点的时间戳）
3. 处理投票
   + 优先Zxid，大的优先
   + Zxid相同，则比较myid，myid大的优先
4. 统计投票，判断是否有超过半数的服务器接受到相同的投票信息
5. 改变服务器状态，leader改为LEADING，其他的为FOLLOWING或OBSERVING，如果此时有服务器加入，有leader存在则状态直接由LOOKING改为LEADING

+ zookeeper运行时leader宕机，进入选举状态。过程大致如上
  + 所有服务器票都投自己，但运行时Zxid不同，因此选Zxid最大（Zxid最大表示数据越新）的优先为Leader。

2. ### Znode的底层结构和节点数据模型

![image-20230217173527077](https://s2.loli.net/2023/02/18/rXAqu5hF7gMIVLH.png)可zookeeper的结构和文件系统类似，以理解为n叉树的类型。每一个节点（Znode）都可以存放数据和子节点，路径的表示是 “/” 来分隔节点名。如  /app1/p1

#### 节点分类： 在zookeeper中节点分为永久、临时，顺序节点

+ 永久（persistent）节点，只有当手动地调用delete命令才会失效

+ 临时（ephemeral ）节点，当客户端断开连接或崩溃后就会被删除

+ 永久/临时  顺序（SEQUENTIAL）节点，如上，节点名会追加单调递增的十进制序号

  > 顺序节点会在分布式锁中使用到

#### 节点的数据结构

因为znode既能当路径使用又能存放数据，因此需要以下的结构

```java
public class DataNode implements Record {
    byte data[];      //业务数据              
    Long acl;         //访问权限              
    public StatPersisted stat;           //当前节点的状态    
    private Set<String> children = null; //子节点的引用
}
```

stat：存放当前节点的状态，包含有版本号（更新数据时对比），事务id（Zxid，判断锁和选举用），时间戳

为了保证高吞吐和低延迟，数据的一致性，znode的大小不能超过1m，一般小于1k比较合适。

## 监听机制

Zookeeper 允许客户端向服务端的某个Znode注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，服务端会向指定客户端发送一个事件通知来实现分布式的通知功能，然后客户端根据 Watcher通知状态和事件类型做出业务上的改变。

客户端注册 watcher 有三种方式，调用客户端 API 可以分别通过 getData、exists、getChildren 实现

详细流程和代码参考：[11.0 Zookeeper watcher 事件机制原理剖析 | 菜鸟教程 (runoob.com)](https://www.runoob.com/w3cnote/zookeeper-watcher.html)

## ZAB协议与数据同步

ZAB协议使用**消息广播**和**崩溃恢复**来实现数据一致性

+ 消息广播

  leader接受到事务请求（写）后，会将事务请求转换成提议，向follower和observer广播，当半数的follower回复正确的ack后，leader会执行事务请求，并对所有的follower发送提交提议的commit。此时follower则会同步请求执行后的数据。observer只同步leader的数据，不参与提议的过程。

  ![img](https://s2.loli.net/2023/02/18/jtNX39HJqh7uBRF.png)

+ 崩溃恢复

  如果leader宕机或失去了于过半follower的通信，那么就会进入崩溃恢复，即重新选举leader。此时会出现数据不一致的隐患，需要ZAB协议进行处理。

  1. Leader 服务器将消息 commit 发出后，立即崩溃。因为follower已经接收到数据改变即上次的事务请求执行成功，则此时只需选择Zxid最大的为leader即可。
  2. Leader 服务器刚提出 提议后，立即崩溃。因为此次的事务请求还未执行，因此选举出的新leader会处理事务日志中未提交的消息来保证数据的准确。

## zookeeper 的特性

基于以上zookeeper的介绍，我们可以总结出zookeeper的一些特性

+ 全局数据一致性（ZAB协议实现）：客户端连接任意的服务器，数据都是一致的。
+ 顺序一致性（节点的Zxid）：事务的请求按照顺序执行
+ 原子性（单leader处理）：事务操作只有成功和失败
+ 实时性（最终一致性，watch机制）：保证客户端在一定的时间内获取服务器更新的消息。
+ 可靠性：只要事务执行并响应请求，则所有更新都会保留。

## zookeeper的场景应用

我们知道zookeeper是分布式协调协调服务，所以使用场景都是分布式的情况，大部分都是管理资源。

+ 服务注册中心
+ 配置管理
+ 集群管理
+ 分布式锁（基于顺序节点和监听机制实现）

## 分布式锁原理

![image-20230218162104179](https://s2.loli.net/2023/02/18/YRqn59yM8KupeFg.png)

我们知道Znode是有临时顺序节点的，所有的锁请求都会在永久节点Lock下创建一个临时节点，通过判断节点的排序是否是最小的来判断当前锁的状态。如果是则获取锁，不是则等待并创建一个监听时间，来监听前一个临时节点是否存在。后续请求类似，无法获取锁则监听进入等待状态。

释放锁则是只能由客户端完成操作后显式地删除临时节点才能释放锁，因为zookeeper是单一leader执行，因此其他请求都只能等待而不会出现抢占的情况。

但锁释放后，会触发监听机制，后一个锁请求会接收到通知，判断是否顺序最小后获取锁。



#### 参考如下：

[ZooKeeper: Because Coordinating Distributed Systems is a Zoo (apache.org)](https://zookeeper.apache.org/doc/r3.5.5/zookeeperOver.html#Nodes+and+ephemeral+nodes)

[ZooKeeper的十二连问，你顶得了嘛？ - Jay_huaxiao - 博客园 (cnblogs.com)](https://www.cnblogs.com/jay-huaxiao/p/13599519.html)

[Zookeeper 教程_w3cschool](https://m.w3cschool.cn/zookeeper)

[1.0 Zookeeper 教程 | 菜鸟教程 (runoob.com)](https://www.runoob.com/w3cnote/zookeeper-tutorial.html)
