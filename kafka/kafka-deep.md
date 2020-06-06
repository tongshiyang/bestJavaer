# 深入 Kafka

如果只是为了开发 Kafka 应用程序，或者只是在生产环境使用 Kafka，那么了解 Kafka 的内部工作原理不是必须的。不过，了解 Kafka 的内部工作原理有助于理解 Kafka 的行为，也利用快速诊断问题。下面我们来探讨一下这三个问题

* Kafka 是如何进行复制的
* Kafka 是如何处理来自生产者和消费者的请求的
* Kafka 的存储细节是怎样的

如果感兴趣的话，就请花费你一些时间，耐心看完这篇文章。

## 集群成员间的关系

我们知道，Kafka 是运行在 ZooKeeper 之上的，因为 ZooKeeper 是以集群形式出现的，所以 Kafka 也可以以集群形式出现。这也就涉及到多个生产者和多个消费者如何协调的问题，这个维护集群间的关系也是由 ZooKeeper 来完成的。如果你看过我之前的文章([真的，关于 Kafka 入门看这一篇就够了](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247484768&idx=1&sn=724ebf1ecbb2e9df677242dec1ab217b&chksm=fc45f893cb327185db43ea9363928d71c54c62b1d9048f759671de3737d62e0e2d265289b354&token=562128166&lang=zh_CN#rd))，你应该会知道，Kafka 集群间会有多个 `主机(broker)`，每个 broker 都会有一个 `broker.id`，每个 broker.id 都有一个唯一的标识符用来区分，这个标识符可以在配置文件里手动指定，也可以自动生成。

> Kafka 可以通过 broker.id.generation.enable 和 reserved.broker.max.id 来配合生成新的 broker.id。
>
> broker.id.generation.enable参数是用来配置是否开启自动生成 broker.id 的功能，默认情况下为true，即开启此功能。自动生成的broker.id有一个默认值，默认值为1000，也就是说默认情况下自动生成的 broker.id 从1001开始。

Kafka 在启动时会在 ZooKeeper 中 `/brokers/ids` 路径下注册一个与当前 broker 的 id 相同的临时节点。Kafka 的健康状态检查就依赖于此节点。当有 broker 加入集群或者退出集群时，这些组件就会获得通知。

* 如果你要启动另外一个具有相同 ID 的 broker，那么就会得到一个错误 —— 新的 broker 会试着进行注册，但不会成功，因为 ZooKeeper 里面已经有一个相同 ID 的 broker。
* 在 broker 停机、出现分区或者长时间垃圾回收停顿时，broker 会从 ZooKeeper 上断开连接，此时 broker 在启动时创建的临时节点会从 ZooKeeper 中移除。监听 broker 列表的 Kafka 组件会被告知该 broker 已移除。
* 在关闭 broker 时，它对应的节点也会消失，不过它的 ID 会继续存在其他数据结构中，例如主题的副本列表中，副本列表复制我们下面再说。在完全关闭一个 broker 之后，如果使用相同的 ID 启动另一个全新的 broker，它会立刻加入集群，并拥有一个与旧 broker 相同的分区和主题。

## Broker Controller 的作用

我们之前在讲 Kafka Rebalance 重平衡的时候，提过一个群组协调器，负责协调群组间的关系，那么 broker 之间也有一个**控制器组件（Controller），它是 Kafka 的核心组件。它的主要作用是在 ZooKeeper 的帮助下管理和协调整个 Kafka 集群**，集群中的每个 broker 都可以称为 controller，但是在 Kafka 集群启动后，只有一个 broker 会成为 Controller 。既然 Kafka 集群是依赖于 ZooKeeper 集群的，所以有必要先介绍一下 ZooKeeper 是什么，可以参考作者的这一篇文章([ZooKeeper不仅仅是注册中心，你还知道有哪些？](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247484551&idx=1&sn=17f99fa32ea50a22eef5bb6fbef37a51&chksm=fc45f974cb32706258aa9bca165c3f999e3d719cb58e918dd1fd552df803380a431719ba4a46&token=2051356665&lang=zh_CN#rd))详细了解，在这里就简单提一下 `znode` 节点的问题。

ZooKeeper 的数据是保存在节点上的，每个节点也被称为`znode`，znode 节点是一种树形的文件结构，它很像 Linux 操作系统的文件路径，ZooKeeper 的根节点是 `/`。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123727723-706474702.png)

znode 根据数据的持久化方式可分为临时节点和持久性节点。持久性节点不会因为 ZooKeeper 状态的变化而消失，但是临时节点会随着 ZooKeeper 的重启而自动消失。

znode 节点有一个 `Watcher` 机制：当数据发生变化的时候， ZooKeeper 会产生一个 Watcher 事件，并且会发送到客户端。Watcher 监听机制是 Zookeeper 中非常重要的特性，我们基于 Zookeeper 上创建的节点，可以对这些节点绑定监听事件，比如可以监听节点数据变更、节点删除、子节点状态变更等事件，通过这个事件机制，可以基于 ZooKeeper 实现分布式锁、集群管理等功能。

### 控制器的选举

Kafka 当前选举控制器的规则是：Kafka 集群中第一个启动的 broker 通过在 ZooKeeper 里创建一个临时节点 `/controller` 让自己成为 controller 控制器。其他 broker 在启动时也会尝试创建这个节点，但是由于这个节点已存在，所以后面想要创建 /controller 节点时就会收到一个 **节点已存在** 的异常。然后其他 broker 会在这个控制器上注册一个 ZooKeeper 的 watch 对象，`/controller` 节点发生变化时，其他 broker 就会收到节点变更通知。这种方式可以确保只有一个控制器存在。那么只有单独的节点一定是有个问题的，那就是`单点问题`。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123753076-979835409.png)

如果控制器关闭或者与 ZooKeeper 断开链接，ZooKeeper 上的临时节点就会消失。集群中的其他节点收到 watch   对象发送控制器下线的消息后，其他 broker 节点都会尝试让自己去成为新的控制器。其他节点的创建规则和第一个节点的创建原则一致，都是第一个在 ZooKeeper 里成功创建控制器节点的 broker 会成为新的控制器，那么其他节点就会收到节点已存在的异常，然后在新的控制器节点上再次创建 watch 对象进行监听。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123804167-1834710583.png)

### 控制器的作用

那么说了这么多，控制是什么呢？控制器的作用是什么呢？或者说控制器的这么一个`组件`被设计用来干什么？别着急，接下来我们就要说一说。

Kafka 被设计为一种模拟状态机的多线程控制器，它可以作用有下面这几点

* 控制器相当于部门（集群）中的部门经理（broker controller），用于管理部门中的部门成员（broker）
* 控制器是所有 broker 的一个监视器，用于监控 broker 的上线和下线
* 在 broker 宕机后，控制器能够选举新的分区 Leader

* 控制器能够和 broker 新选取的 Leader 发送消息

再细分一下可以具体分为如下 5 点

* `主题管理` : Kafka Controller 可以帮助我们完成对 Kafka 主题创建、删除和增加分区的操作，简而言之就是对分区拥有最高行使权。

换句话说，当我们执行**kafka-topics 脚本**时，大部分的后台工作都是控制器来完成的。

* `分区重分配`: 分区重分配主要是指，**kafka-reassign-partitions 脚本**提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。
* `Prefered 领导者选举` : Preferred 领导者选举主要是 Kafka 为了避免部分 Broker 负载过重而提供的一种换 Leader 的方案。
* `集群成员管理`: 主要管理 新增 broker、broker 关闭、broker 宕机

* `数据服务`: 控制器的最后一大类工作，就是向其他 broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。这些数据我们会在下面讨论

当控制器发现一个 broker 离开集群（通过观察相关 ZooKeeper 路径），控制器会收到消息：这个 broker 所管理的那些分区需要一个新的 Leader。控制器会依次遍历每个分区，确定谁能够作为新的 Leader，然后向所有包含新 Leader 或现有 Follower 的分区发送消息，该请求消息包含谁是新的 Leader 以及谁是 Follower 的信息。随后，新的 Leader 开始处理来自生产者和消费者的请求，Follower 用于从新的 Leader 那里进行复制。

这就很像外包公司的一个部门，这个部门就是专门出差的，每个人在不同的地方办公，但是中央总部有一个部门经理，现在部门经理突然离职了。公司不打算外聘人员，决定从部门内部选一个能力强的人当领导，然后当上领导的人需要向自己的组员发送消息，这条消息就是任命消息和明确他管理了哪些人，大家都知道了，然后再各自给部门干活。

当控制器发现一个 broker 加入集群时，它会使用 broker ID 来检查新加入的 broker 是否包含现有分区的副本。如果有控制器就会把消息发送给新加入的 broker 和 现有的 broker。

上面这块关于分区复制的内容我们接下来会说到。

### broker controller 数据存储

上面我们介绍到 broker controller 会提供数据服务，用于保存大量的 Kafka 集群数据。如下图

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123820409-1552042907.png)

可以对上面保存信息归类，主要分为三类

* broker 上的所有信息，包括 broker 中的所有分区，broker 所有分区副本，当前都有哪些运行中的 broker，哪些正在关闭中的 broker 。
* 所有主题信息，包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
* 所有涉及运维任务的分区。包括当前正在进行 Preferred 领导者选举以及分区重分配的分区列表。

Kafka 是离不开 ZooKeeper的，所以这些数据信息在 ZooKeeper 中也保存了一份。每当控制器初始化时，它都会从 ZooKeeper 上读取对应的元数据并填充到自己的缓存中。

### broker controller 故障转移

我们在前面说过，第一个在 ZooKeeper 中的 `/brokers/ids`下创建节点的 broker 作为 broker controller，也就是说 broker controller 只有一个，那么必然会存在单点失效问题。kafka 为考虑到这种情况提供了`故障转移`功能，也就是 `Fail Over`。如下图

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123836924-1768848221.png)

最一开始，broker1 会抢先注册成功成为 controller，然后由于网络抖动或者其他原因致使 broker1 掉线，ZooKeeper 通过 Watch 机制觉察到 broker1 的掉线，之后所有存活的 brokers 开始竞争成为 controller，这时 broker3 抢先注册成功，此时 ZooKeeper 存储的 controller 信息由 broker1 -> broker3，之后，broker3 会从 ZooKeeper 中读取元数据信息，并初始化到自己的缓存中。

>注意：ZooKeeper 中存储的不是缓存信息，broker 中存储的才是缓存信息。

### broker controller 存在的问题

在 Kafka 0.11 版本之前，控制器的设计是相当繁琐的。我们上面提到过一句话：Kafka controller 被设计为一种模拟状态机的多线程控制器，这种设计其实是存在一些问题的

* controller 状态的更改由不同的监听器并罚执行，因此需要进行很复杂的同步，并且容易出错而且难以调试。
* 状态传播不同步，broker 可能在时间不确定的情况下出现多种状态，这会导致不必要的额外的数据丢失
* controller 控制器还会为主题删除创建额外的  I/O 线程，导致性能损耗
* controller 的多线程设计还会访问共享数据，我们知道，多线程访问共享数据是线程同步最麻烦的地方，为了保护数据安全性，控制器不得不在代码中大量使用**ReentrantLock 同步机制**，这就进一步拖慢了整个控制器的处理速度。

### broker controller 内部设计原理

在 Kafka 0.11 之后，Kafka controller 采用了新的设计，**把多线程的方案改成了单线程加事件队列的方案**。如下图所示

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123848297-1470791123.png)

主要所做的改变有下面这几点

第一个改进是增加了一个 `Event Executor Thread`，事件执行线程，从图中可以看出，不管是 Event Queue 事件队列还是 Controller context 控制器上下文都会交给事件执行线程进行处理。将原来执行的操作全部建模成一个个独立的事件，发送到专属的事件队列中，供此线程消费。

第二个改进是将之前同步的 ZooKeeper 全部改为`异步操作`。ZooKeeper API 提供了两种读写的方式：同步和异步。之前控制器操作 ZooKeeper 都是采用的同步方式，这次把同步方式改为异步，据测试，效率提升了10倍。

第三个改进是根据优先级处理请求，之前的设计是 broker 会公平性的处理所有 controller 发送的请求。什么意思呢？公平性难道还不好吗？在某些情况下是的，比如 broker 在排队处理 produce 请求，这时候 controller 发出了一个 **StopReplica** 的请求，你会怎么办？还在继续处理 produce 请求吗？这个 produce 请求还有用吗？此时最合理的处理顺序应该是，**赋予 StopReplica 请求更高的优先级，使它能够得到抢占式的处理。**

## 副本机制

复制功能是 Kafka 架构的核心功能，在 Kafka 文档里面 Kafka 把自己描述为 **一个分布式的、可分区的、可复制的提交日志服务**。复制之所以这么关键，是因为消息的持久存储非常重要，这能够保证在主节点宕机后依旧能够保证 Kafka 高可用。副本机制也可以称为`备份机制(Replication)`，通常指分布式系统在多台网络交互的机器上保存有相同的数据备份/拷贝。

Kafka 使用主题来组织数据，每个主题又被分为若干个分区，分区会部署在一到多个 broker 上，每个分区都会有多个副本，所以副本也会被保存在 broker 上，每个 broker 可能会保存成千上万个副本。下图是一个副本复制示意图

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123918587-2019612075.png)


如上图所示，为了简单我只画出了两个 broker ,每个 broker 指保存了一个 Topic 的消息，在 broker1 中分区0 是Leader，它负责进行分区的复制工作，把 broker1 中的分区0复制一个副本到 broker2 的主题 A 的分区0。同理，主题 A 的分区1也是一样的道理。

副本类型分为两种：一种是 `Leader(领导者)` 副本，一种是`Follower(跟随者)`副本。

### Leader 副本

Kafka 在创建分区的时候都要选举一个副本，这个选举出来的副本就是 Leader 领导者副本。

### Follower 副本

除了 Leader 副本以外的副本统称为 `Follower 副本`，Follower 不对外提供服务。下面是 Leader 副本的工作方式

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223123934038-102409288.png)

这幅图需要注意以下几点

* Kafka 中，Follower 副本也就是追随者副本是不对外提供服务的。这就是说，任何一个追随者副本都不能响应消费者和生产者的请求。所有的请求都是由领导者副本来处理。或者说，所有的请求都必须发送到 Leader 副本所在的 broker 中，Follower 副本只是用做数据拉取，采用`异步拉取`的方式，并写入到自己的提交日志中，从而实现与 Leader 的同步
* 当 Leader 副本所在的 broker 宕机后，Kafka 依托于 ZooKeeper 提供的监控功能能够实时感知到，并开启新一轮的选举，从追随者副本中选一个作为 Leader。如果宕机的 broker 重启完成后，该分区的副本会作为 Follower 重新加入。

首领的另一个任务是搞清楚哪个跟随者的状态与自己是一致的。跟随者为了保证与领导者的状态一致，在有新消息到达之前先尝试从领导者那里复制消息。为了与领导者保持一致，跟随者向领导者发起获取数据的请求，这种请求与消费者为了读取消息而发送的信息是一样的。

跟随者向领导者发送消息的过程是这样的，先请求消息1，然后再接收到消息1，在时候到请求1之后，发送请求2，在收到领导者给发送给跟随者之前，跟随者是不会继续发送消息的。这个过程如下

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124034253-1873750415.png)

跟随者副本在收到响应消息前，是不会继续发送消息，这一点很重要。通过查看每个跟随者请求的最新偏移量，首领就会知道每个跟随者复制的进度。如果跟随者在10s 内没有请求任何消息，或者虽然跟随者已经发送请求，但是在10s 内没有收到消息，就会被认为是`不同步`的。如果一个副本没有与领导者同步，那么在领导者掉线后，这个副本将不会称为领导者，因为这个副本的消息不是全部的。

与之相反的，如果跟随者同步的消息和领导者副本的消息一致，那么这个跟随者副本又被称为`同步的副本`。也就是说，如果领导者掉线，那么只有同步的副本能够称为领导者。

关于副本机制我们说了这么多，那么副本机制的好处是什么呢？

* 能够立刻看到写入的消息，就是你使用生产者 API 成功向分区写入消息后，马上使用消费者就能读取刚才写入的消息
* 能够实现消息的幂等性，啥意思呢？就是对于生产者产生的消息，在消费者进行消费的时候，它每次都会看到消息存在，并不会存在消息不存在的情况

### 同步复制和异步复制

我在学习副本机制的时候，有个疑问，既然领导者副本和跟随者副本是`发送 - 等待`机制的，这是一种同步的复制方式，那么为什么说跟随者副本同步领导者副本的时候是一种异步操作呢？

我认为是这样的，跟随者副本在同步领导者副本后会把消息保存在本地 log 中，这个时候跟随者会给领导者副本一个响应消息，告诉领导者自己已经保存成功了，同步复制的领导者会等待所有的跟随者副本都写入成功后，再返回给 producer 写入成功的消息。而异步复制是领导者副本不需要关心跟随者副本是否写入成功，只要领导者副本自己把消息保存到本地 log ，就会返回给 producer 写入成功的消息。下面是同步复制和异步复制的过程

**同步复制**

* producer 通知 ZooKeeper 识别领导者
* producer 向领导者写入消息
* 领导者收到消息后会把消息写入到本地 log
* 跟随者会从领导者那里拉取消息
* 跟随者向本地写入 log
* 跟随者向领导者发送写入成功的消息
* 领导者会收到所有的跟随者发送的消息
* 领导者向 producer 发送写入成功的消息

**异步复制**

和同步复制的区别在于，领导者在写入本地log之后，直接向客户端发送写入成功消息，不需要等待所有跟随者复制完成。

### ISR 

Kafka动态维护了一个同步状态的副本的集合`（a set of In-Sync Replicas），简称ISR`，ISR 也是一个很重要的概念，我们之前说过，追随者副本不提供服务，只是定期的异步拉取领导者副本的数据而已，拉取这个操作就相当于是复制，`ctrl-c + ctrl-v`大家肯定用的熟。那么是不是说 ISR 集合中的副本消息的数量都会与领导者副本消息数量一样呢？那也不一定，判断的依据是 broker 中参数 `replica.lag.time.max.ms` 的值，这个参数的含义就是跟随者副本能够落后领导者副本最长的时间间隔。

replica.lag.time.max.ms 参数默认的时间是 10秒，如果跟随者副本落后领导者副本的时间不超过 10秒，那么 Kafka 就认为领导者和跟随者是同步的。即使此时跟随者副本中存储的消息要小于领导者副本。如果跟随者副本要落后于领导者副本 10秒以上的话，跟随者副本就会从 ISR 被剔除。倘若该副本后面慢慢地追上了领导者的进度，那么它是能够重新被加回 ISR 的。这也表明，ISR 是一个动态调整的集合，而非静态不变的。

### Unclean 领导者选举

既然 ISR 是可以动态调整的，那么必然会出现 ISR 集合中为空的情况，由于领导者副本是一定出现在 ISR 集合中的，那么 ISR 集合为空必然说明领导者副本也挂了，所以此时 Kafka 需要重新选举一个新的领导者，那么该如何选举呢？现在你需要转变一下思路，我们上面说 ISR 集合中一定是与领导者同步的副本，那么不再 ISR 集合中的副本一定是不与领导者同步的副本了，也就是不再 ISR 列表中的跟随者副本会丢失一些消息。如果你开启 broker 端参数 `unclean.leader.election.enable`的话，下一个领导者就会在这些非同步的副本中选举。这种选举也叫做`Unclean 领导者选举`。

如果你接触过分布式项目的话你一定知道 CAP 理论，那么这种 Unclean 领导者选举其实是牺牲了数据一致性，保证了 Kafka 的高可用性。

你可以根据你的实际业务场景决定是否开启 Unclean 领导者选举，一般不建议开启这个参数，因为数据的一致性要比可用性重要的多。

## Kafka 请求处理流程

broker 的大部分工作是处理客户端、分区副本和控制器发送给分区领导者的请求。这种请求一般都是`请求/响应`式的，我猜测你接触最早的请求/响应的方式应该就是 HTTP 请求了。事实上，HTTP 请求可以是同步可以是异步的。一般正常的 HTTP 请求都是同步的，同步方式最大的一个特点是**提交请求->等待服务器处理->处理完毕返回 这个期间客户端浏览器不能做任何事**。而异步方式最大的特点是 **请求通过事件触发->服务器处理（这是浏览器仍然可以作其他事情）-> 处理完毕**。

那么我也可以说同步请求就是顺序处理的，而异步请求的执行方式则不确定，因为异步需要创建多个执行线程，而每个线程的执行顺序不同。

>这里需要注意一点，我们只是使用 HTTP 请求来举例子，而 Kafka 采用的是 TCP 基于 Socket 的方式进行通讯

那么这两种方式有什么缺点呢？

我相信聪明的你应该能马上想到，同步的方式最大的缺点就是`吞吐量太差`，资源利用率极低，由于只能顺序处理请求，因此，每个请求都必须等待前一个请求处理完毕才能得到处理。这种方式只适用于`请求发送非常不频繁的系统`。

异步的方式的缺点就是为每个请求都创建线程的做法开销极大，在某些场景下甚至会压垮整个服务。

### 响应式模型

说了这么半天，Kafka 采用同步还是异步的呢？都不是，Kafka 采用的是一种 `响应式(Reactor)模型`，那么什么是响应式模型呢？简单的说，**Reactor 模式是事件驱动架构的一种实现方式，特别适合应用于处理多个客户端并发向服务器端发送请求的场景**，如下图所示

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124053219-338314143.png)

Kafka 的 broker 端有个 SocketServer组件，类似于处理器，SocketServer 是基于 TCP 的 Socket 连接的，它用于接受客户端请求，所有的请求消息都包含一个消息头，消息头中都包含如下信息

* Request type （也就是 API Key）
* Request version（broker 可以处理不同版本的客户端请求，并根据客户版本做出不同的响应）
* Correlation ID --- 一个具有唯一性的数字，用于标示请求消息，同时也会出现在响应消息和错误日志中（用于诊断问题）
* Client ID --- 用于标示发送请求的客户端

broker 会在它所监听的每一个端口上运行一个 `Acceptor` 线程，这个线程会创建一个连接，并把它交给 `Processor(网络线程池)`， Processor 的数量可以使用 `num.network.threads` 进行配置，其默认值是3，表示每台 broker 启动时会创建3个线程，专门处理客户端发送的请求。

Acceptor 线程会采用`轮询`的方式将入栈请求公平的发送至网络线程池中，因此，在实际使用过程中，这些线程通常具有相同的机率被分配到待处理`请求队列`中，然后从`响应队列`获取响应消息，把它们发送给客户端。Processor 网络线程池中的请求 - 响应的处理还是比较复杂的，下面是网络线程池中的处理流程图

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124102313-2029698012.png)

Processor 网络线程池接收到客户和其他 broker 发送来的消息后，网络线程池会把消息放到请求队列中，注意这个是`共享请求队列`，因为网络线程池是多线程机制的，所以请求队列的消息是多线程共享的区域，然后由 IO 线程池进行处理，根据消息的种类判断做何处理，比如 `PRODUCE` 请求，就会将消息写入到 log 日志中，如果是`FETCH`请求，则从磁盘或者页缓存中读取消息。也就是说，IO线程池是真正做判断，处理请求的一个组件。在IO 线程池处理完毕后，就会判断是放入`响应队列`中还是 `Purgatory` 中，Purgatory 是什么我们下面再说，现在先说一下响应队列，响应队列是每个线程所独有的，因为响应式模型中不会关心请求发往何处，因此把响应回传的事情就交给每个线程了，所以也就不必共享了。

>注意：IO 线程池可以通过 broker 端参数 `num.io.threads` 来配置，默认的线程数是8，表示每台 broker 启动后自动创建 8 个IO 处理线程。

### 请求类型

下面是几种常见的请求类型

**生产请求**

我在 [真的，关于 Kafka 入门看这一篇就够了](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247484768&idx=1&sn=724ebf1ecbb2e9df677242dec1ab217b&chksm=fc45f893cb327185db43ea9363928d71c54c62b1d9048f759671de3737d62e0e2d265289b354&token=1263191457&lang=zh_CN#rd) 文章中提到过 `acks` 这个配置项的含义

简单来讲就是不同的配置对写入成功的界定是不同的，如果 acks = 1，那么只要领导者收到消息就表示写入成功，如果acks = 0，表示只要领导者发送消息就表示写入成功，根本不用考虑返回值的影响。如果 acks = all，就表示领导者需要收到所有副本的消息后才表示写入成功。

在消息被写入分区的首领后，如果 acks 配置的值是 `all`，那么这些请求会被保存在 `炼狱(Purgatory)`的缓冲区中，直到领导者副本发现跟随者副本都复制了消息，响应才会发送给客户端。

**获取请求**

broker 获取请求的方式与处理生产请求的方式类似，客户端发送请求，向 broker 请求主题分区中特定偏移量的消息，如果偏移量存在，Kafka 会采用 `零复制` 技术向客户端发送消息，Kafka 会直接把消息从文件中发送到网络通道中，而不需要经过任何的缓冲区，从而获得更好的性能。

客户端可以设置获取请求数据的上限和下限，`上限`指的是客户端为接受足够消息分配的内存空间，这个限制比较重要，如果上限太大的话，很有可能直接耗尽客户端内存。`下限`可以理解为攒足了数据包再发送的意思，这就相当于项目经理给程序员分配了 10 个bug，程序员每次改一个 bug 就会向项目经理汇报一下，有的时候改好了有的时候可能还没改好，这样就增加了沟通成本和时间成本，所以下限值得就是程序员你改完10个 bug 再向我汇报！！！如下图所示

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124116577-1821275060.png)

如图你可以看到，在`拉取消息` ---> `消息` 之间是有一个等待消息积累这么一个过程的，这个消息积累你可以把它想象成超时时间，不过超时会跑出异常，消息积累超时后会响应回执。延迟时间可以通过 `replica.lag.time.max.ms` 来配置，它指定了副本在复制消息时可被允许的最大延迟时间。

**元数据请求**

生产请求和响应请求都必须发送给领导者副本，如果 broker 收到一个针对某个特定分区的请求，而该请求的首领在另外一个 broker 中，那么发送请求的客户端会收到`非分区首领`的错误响应；如果针对某个分区的请求被发送到不含有领导者的 broker 上，也会出现同样的错误。Kafka 客户端需要把请求和响应发送到正确的 broker 上。这不是废话么？我怎么知道要往哪发送？

事实上，客户端会使用一种 `元数据请求` ，这种请求会包含客户端感兴趣的主题列表，服务端的响应消息指明了主题的分区，领导者副本和跟随者副本。元数据请求可以发送给任意一个 broker，因为所有的 broker 都会缓存这些信息。

一般情况下，客户端会把这些信息缓存，并直接向目标 broker 发送生产请求和相应请求，这些缓存需要隔一段时间就进行刷新，使用`metadata.max.age.ms` 参数来配置，从而知道元数据是否发生了变更。比如，新的 broker 加入后，会触发重平衡，部分副本会移动到新的 broker 上。这时候，如果客户端收到 `不是首领`的错误，客户端在发送请求之前刷新元数据缓存。

## Kafka 重平衡流程

我在  [真的，关于 Kafka 入门看这一篇就够了](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247484768&idx=1&sn=724ebf1ecbb2e9df677242dec1ab217b&chksm=fc45f893cb327185db43ea9363928d71c54c62b1d9048f759671de3737d62e0e2d265289b354&token=1263191457&lang=zh_CN#rd)  中关于消费者描述的时候大致说了一下消费者组和重平衡之间的关系，实际上，归纳为一点就是让组内所有的消费者实例就消费哪些主题分区达成一致。

我们知道，一个消费者组中是要有一个`群组协调者(Coordinator)`的，而重平衡的流程就是由 Coordinator 的帮助下来完成的。

这里需要先声明一下重平衡发生的条件

* 消费者订阅的任何主题发生变化
* 消费者数量发生变化
* 分区数量发生变化
* 如果你订阅了一个还尚未创建的主题，那么重平衡在该主题创建时发生。如果你订阅的主题发生删除那么也会发生重平衡
* 消费者被群组协调器认为是 `DEAD` 状态，这可能是由于消费者崩溃或者长时间处于运行状态下发生的，这意味着在配置合理时间的范围内，消费者没有向群组协调器发送任何心跳，这也会导致重平衡的发生。

### 在了解重平衡之前，你需要知道这两个角色

`群组协调器（Coordinator）`：群组协调器是一个能够从消费者群组中收到所有消费者发送心跳消息的 broker。在最早期的版本中，元数据信息是保存在 ZooKeeper 中的，但是目前元数据信息存储到了 broker 中。每个消费者组都应该和群组中的群组协调器同步。当所有的决策要在应用程序节点中进行时，群组协调器可以满足 `JoinGroup` 请求并提供有关消费者组的元数据信息，例如分配和偏移量。群组协调器还有权知道所有消费者的心跳，消费者群组中还有一个角色就是领导者，注意把它和领导者副本和 kafka controller 进行区分。领导者是群组中负责决策的角色，所以如果领导者掉线了，群组协调器有权把所有消费者踢出组。因此，消费者群组的一个很重要的行为是选举领导者，并与协调器读取和写入有关分配和分区的元数据信息。

`消费者领导者`： 每个消费者群组中都有一个领导者。如果消费者停止发送心跳了，协调者会触发重平衡。

### 在了解重平衡之前，你需要知道状态机是什么

Kafka 设计了一套`消费者组状态机(State Machine)` ，来帮助协调者完成整个重平衡流程。消费者状态机主要有五种状态它们分别是 **Empty、Dead、PreparingRebalance、CompletingRebalance 和 Stable**。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124211060-1576292762.png)

了解了这些状态的含义之后，下面我们用几条路径来表示一下消费者状态的轮转

消费者组一开始处于 `Empty` 状态，当重平衡开启后，它会被置于 `PreparingRebalance` 状态等待新消费者的加入，一旦有新的消费者加入后，消费者群组就会处于 `CompletingRebalance` 状态等待分配，只要有新的消费者加入群组或者离开，就会触发重平衡，消费者的状态处于 PreparingRebalance 状态。等待分配机制指定好后完成分配，那么它的流程图是这样的

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124214467-750018843.png)

在上图的基础上，当消费者群组都到达 `Stable` 状态后，一旦有新的**消费者加入/离开/心跳过期**，那么触发重平衡，消费者群组的状态重新处于 PreparingRebalance 状态。那么它的流程图是这样的。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124224577-1322757017.png)

在上图的基础上，消费者群组处于 PreparingRebalance 状态后，很不幸，没人玩儿了，所有消费者都离开了，这时候还可能会保留有消费者消费的位移数据，一旦位移数据过期或者被刷新，那么消费者群组就处于 `Dead` 状态了。它的流程图是这样的

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124233387-553976784.png)

在上图的基础上，我们分析了消费者的重平衡，在 `PreparingRebalance`或者 `CompletingRebalance`  或者 `Stable` 任意一种状态下发生位移主题分区 Leader 发生变更，群组会直接处于 Dead 状态，它的所有路径如下

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124248882-813348917.png)

>这里面需要注意两点：
>
>一般出现 **Required xx expired offsets in xxx milliseconds** 就表明Kafka 很可能就把该组的位移数据删除了
>
>只有 Empty 状态下的组，才会执行过期位移删除的操作。

### 重平衡流程

上面我们了解到了消费者群组状态的转化过程，下面我们真正开始介绍 `Rebalance` 的过程。重平衡过程可以从两个方面去看：消费者端和协调者端，首先我们先看一下消费者端

### 从消费者看重平衡

从消费者看重平衡有两个步骤：分别是 `消费者加入组` 和 `等待领导者分配方案`。这两个步骤后分别对应的请求是 `JoinGroup` 和 `SyncGroup`。

新的消费者加入群组时，这个消费者会向协调器发送 `JoinGroup` 请求。在该请求中，每个消费者成员都需要将自己消费的 topic 进行提交，我们上面描述群组协调器中说过，这么做的目的就是为了让协调器收集足够的元数据信息，来选取消费者组的领导者。通常情况下，第一个发送 JoinGroup 请求的消费者会自动称为领导者。**领导者的任务是收集所有成员的订阅信息，然后根据这些信息，制定具体的分区消费分配方案**。如图

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124259327-2003862454.png)

在所有的消费者都加入进来并把元数据信息提交给领导者后，领导者做出分配方案并发送 `SyncGroup `请求给协调者，协调者负责下发群组中的消费策略。下图描述了 SyncGroup 请求的过程

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124316597-1974807565.png)

当所有成员都成功接收到分配方案后，消费者组进入到 Stable 状态，即开始正常的消费工作。

### 从协调者来看重平衡

从协调者角度来看重平衡主要有下面这几种触发条件，

* 新成员加入组
* 组成员主动离开
* 组成员崩溃离开
* 组成员提交位移

我们分别来描述一下，先从新成员加入组开始

#### 新成员加入组

我们讨论的场景消费者集群状态处于`Stable` 等待分配的过程，这时候如果有新的成员加入组的话，重平衡的过程

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124345217-625961436.png)

从这个角度来看，协调者的过程和消费者类似，只是刚刚从消费者的角度去看，现在从领导者的角度去看

#### 组成员离开

组成员离开消费者群组指的是消费者实例调用 `close()` 方法主动通知协调者它要退出。这里又会有一个新的请求出现 `LeaveGroup()请求` 。如下图所示

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124353411-984684920.png)

#### 组成员崩溃

组成员崩溃是指消费者实例出现严重故障，宕机或者一段时间未响应，协调者接收不到消费者的心跳，就会被认为是`组成员崩溃`，崩溃离组是被动的，协调者通常需要等待一段时间才能感知到，这段时间一般是由消费者端参数 session.timeout.ms 控制的。如下图所示

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124403036-227820059.png)

#### 重平衡时提交位移

这个过程我们就不再用图形来表示了，大致描述一下就是 消费者发送 JoinGroup 请求后，群组中的消费者必须在指定的时间范围内提交各自的位移，然后再开启正常的 JoinGroup/SyncGroup 请求发送。

**如果大家认可我，请帮我点个赞，谢谢各位了。我们下篇技术文章见**。

![](https://img2018.cnblogs.com/blog/1515111/201912/1515111-20191223124455015-683851863.png)


文章参考：

《Kafka 权威指南》

https://blog.csdn.net/u013256816/article/details/80546337

https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch05.html#kafka_internals

https://www.cnblogs.com/kevingrace/p/9021508.html

https://www.cnblogs.com/huxi2b/p/6980045.html

《极客时间-Kafka核心技术与实战》

https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Controller+Redesign

https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Controller+Internals

[kafka 分区和副本以及kafaka 执行流程，以及消息的高可用](https://www.cnblogs.com/liyanbin/p/7815185.html)

[Http中的同步请求和异步请求](https://www.cnblogs.com/Black-YeJing/p/9131124.html)

[Reactor模式详解](https://www.cnblogs.com/winner-0715/p/8733787.html)

https://kafka.apache.org/documentation/

https://www.linkedin.com/pulse/partitions-rebalance-kafka-raghunandan-gupta

https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design