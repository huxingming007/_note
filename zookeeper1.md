### 前言

#### HTTP和RPC

虽然现在服务间的调用越来越多地使用了 RPC 和消息队列，但是 HTTP 依然有适合它的场景。

- RPC 的优势在于高效的网络传输模型(常使用 NIO 来实现)，以及针对服务调用场景专门设 计协议和高效的序列化技术。
- HTTP 的优势在于它的成熟稳定、使用实现简单、被广泛支持、兼容性良好、防火墙友好、 消息的可读性高。所以 http 协议在开放 API、跨平台的服务间调用、对性能要求不苛刻的场 景中有着广泛的使用。

#### 微服务通信带来的问题

1. 目标服务肯定会做扩容，扩容之后，客户端会带来一些变化
2. 客户端对于目标服务如何进行负载均衡
3. 客户端如何维护目标服务的地址信息
4. 服务端的服务状态变化，如何让客户端感知

![](http://ww2.sinaimg.cn/large/006y8mN6ly1g675ehiz69j312n0u0ju5.jpg)

#### 引入注册中心

服务注册中心主要用于实现服务的注册和服务的发现功能，在微服务架构中，它起到了非常大的作用。列如：dubbo体系中的zookeeper、springcloud的eureka和consul。

### 重新认识Zookeeper

#### Zookeeper的前世今生

Apache ZooKeeper 是一个高可靠的分布式协调中间件。它是 Google Chubby 的一个开源 实现，那么它主要是解决什么问题的呢？那就得先了解 Google Chubby，Google Chubby 是谷歌的一个用来解决分布式一致性问题的组件，同时，也是粗粒度的分布 式锁服务。

#### 分布式一致性问题

什么是分布式一致性问题呢?简单来说，就是在一个分布式系统中，有多个节点，每个节点 都会提出一个请求，但是在所有节点中只能确定一个请求被通过。而这个通过是需要所有节 点达成一致的结果，所以所谓的一致性就是在提出的所有请求中能够选出最终一个确定请求。 并且这个请求选出来以后，所有的节点都要知道。

这个就是典型的拜占庭将军问题

拜占庭将军问题说的是:拜占庭帝国军队的将军们必须通过投票达成一致来决定是否对某一 个国家发起进攻。但是这些将军在地里位置上是分开的，并且在将军中存在叛徒。叛徒可以 通过任意行动来达到自己的目标:
1. 欺骗某些将军采取进攻行动
2. 促使一个不是所有将军都统一的决定，比如将军们本意是不希望进攻，但是叛徒可以促成 进攻行动
3. 迷惑将军使得他们无法做出决定 如果叛徒达到了任意一个目标，那么这次行动必然失败。只有完全达成一致那么这次进攻才 可能胜利 拜占庭问题的本质是，由于网络通信存在不可靠的问题，也就是可能存在消息丢失，或者网 络延迟。如何在这样的背景下对某一个请求达成一致。

为了解决这个问题，很多人提出了各种协议，比如大名鼎鼎的 Paxos; 也就是说在不可信的 网络环境中，按照 paxos 这个协议就能够针对某个提议达成一致。 所以:分布式一致性的本质，就是在分布式系统中，多个节点就某一个提议如何达成一致。

➢ 这个和 Google Chubby 有什么关系呢？

在 Google 有一个 GFS(google file system)，他们有一个需求就是要从多个 gfs server 中选出 一个 master server。这个就是典型的一致性问题，5 个分布在不同节点的 server，需要确定 一个 master server，而他们要达成的一致性目标是:确定某一个节点为 master，并且所有节 点要同意。

而 GFS 就是使用 chubby 来解决这个问题的。
实现原理是:所有的 server 通过 Chubby 提供的通信协议到 Chubby server 上创建同一个文 件，当然，最终只有一个 server 能够获准创建这个文件，这个 server 就成为了 master，它 会在这个文件中写入自己 的地址，这样其它的 server 通过读取这个文件就能知道被选出的 master 的地址

#### 分布式锁

从另外一个层面来看，Chubby 提供了一种粗粒度的分布式锁服务，chubby 是通过创建文件 的形式来提供锁的功能，server 向 chubby 中创建文件其实就表示加锁操作，创建文件成功 表示抢占到了锁。
由于 Chubby 没有开源，所以雅虎公司基于 chubby 的思想，开发了一个类似的分布式协调 组件 Zookeeper，后来捐赠给了 Apache。
所以，大家一定要了解，zookeeper 并不是作为注册中心而设计，他是作为分布式锁的一种 设计。而注册中心只是他能够实现的一种功能而已。

### zookeeper的设计猜想

基于 Zookeeper 本身的一个设计目标，zookeeper 主要是解决分布式环境下的服务协调问题 而产生的，我们来猜想一下，如果我们要去设计一个 zookeeper，需要满足那些功能呢?

#### 防止单点故障

首先，在分布式架构中，任何的节点都不能以单点的方式存在，因此我们需要解决单点的问 题。常见的解决单点问题的方式就是集群。
大家来思考一下，这个集群需要满足那些功能?

1. 集群中要有主节点和从节点(也就是集群要有角色)
2. 集群要能做到数据同步，当主节点出现故障时，从节点能够顶替主节点继续工作，但是继 续工作的前提是数据必须要主节点保持一致
3. 主节点挂了以后，从节点如何接替成为主节点? 是人工干预?还是自动选举 所以基于这几个点，我们先来把 zookeeper 的集群节点画出来。

##### leader角色

Leader 服务器是整个 zookeeper 集群的核心，主要的工作任务有两项 

1. **事务**请求的唯一调度和处理者，保证集群事物处理的顺序性
2. 集群内部各服务器的调度者

##### follower角色

Follower 角色的主要职责是
1. 处理客户端非事物请求、转发事物请求给leader服务器
2. 参与事物请求 Proposal 的投票(需要半数以上服务器通过才能通知 leader commit 数据; Leader 发起的提案，要求 Follower 投票)
3. 参与 Leader 选举的投票

#### 数据同步

接着上面那个结论再来思考，如果要满足这样的一个高性能集群，我们最直观的想法应该是， 每个节点都能接收到请求，并且每个节点的数据都必须要保持一致。要实现各个节点的数据 一致性，就势必要一个 leader 节点负责协调和数据同步操作。这个我想大家都知道，如果在 这样一个集群中没有 leader 节点，每个节点都可以接收所有请求，那么这个集群的数据同步 的复杂度是非常大。
所以，当客户端请求过来时，需要满足，事务型数据和非事务型数据的分开处理方式，就是 leader 节点可以处理事务和非事务型数据。而 follower 节点只能处理非事务型数据。原因是， 对于数据变更的操作，应该由一个节点来维护，使得集群数据处理的简化。同时数据需要能 够通过 leader 进行分发使得数据在集群中各个节点的一致性。

leader 节点如何和其他节点保证数据一致性，并且要求是强一致的。在分布式系统中，每一 个机器节点虽然都能够明确知道自己进行的事务操作过程是成功和失败，但是却无法直接获 取其他分布式节点的操作结果。所以当一个事务操作涉及到跨节点的时候，就需要用到分布 式事务，分布式事务的数据一致性协议有 2PC 协议和 3PC 协议。

#### 关于2PC提交

(Two Phase Commitment Protocol)当一个事务操作需要跨越多个分布式节点的时候，为了保持事务处理的 ACID 特性，就需要引入一个“协调者”(TM)来统一调度所有分布式节点 的执行逻辑，这些被调度的分布式节点被称为 AP。TM 负责调度 AP 的行为，并最终决定这 些 AP 是否要把事务真正进行提交；因为整个事务是分为两个阶段提交，所以叫 2pc。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g67air6gyzj30n60s4q3y.jpg)

##### 阶段一：提交事务请求（投票）

1. 事务询问 协调者向所有的参与者发送事务内容，询问是否可以执行事务提交操作，并开始等待各参与者的 响应
2. 执行事务
各个参与者节点执行事务操作，并将 Undo 和 Redo 信息记录到事务日志中，尽量把提交过程中 所有消耗时间的操作和准备都提前完成确保后面 100%成功提交事务
3. 各个参与者向协调者反馈事务询问的响应 如果各个参与者成功执行了事务操作，那么就反馈给参与者 yes 的响应，表示事务可以执行; 如果参与者没有成功执行事务，就反馈给协调者 no 的响应，表示事务不可以执行，上面这个阶 段有点类似协调者组织各个参与者对一次事务操作的投票表态过程，因此 2pc 协议的第一个阶 段称为“投票阶段”，即各参与者投票表名是否需要继续执行接下去的事务提交操作。

##### 阶段二：执行事务提交

在这个阶段，协调者会根据各参与者的反馈情况来决定最终是否可以进行事务提交操作，正常情况 下包含两种可能:执行事务、中断事务。

##### observer角色

Observer 是 zookeeper3.3 开始引入的一个全新的服务器角色，从字面来理解，该角色充当 了观察者的角色。
观察 zookeeper 集群中的最新状态变化并将这些状态变化同步到 observer 服务器上。 Observer 的工作原理与 follower 角色基本一致，而它和 follower 角色唯一的不同在于 observer 不参与任何形式的投票，包括事物请求 Proposal 的投票和 leader 选举的投票。简 单来说，observer 服务器只提供非事物请求服务，通常在于不影响集群事物处理能力的前提 下提升集群非事物处理的能力（读多写死的场景下）。

#### leader选举

当 leader 挂了，需要从其他 follower 节点中选择一个新的节点进行处理，这个时候就需要涉 及到 leader 选举
从这个过程中，我们推导处了 zookeeper 的一些设计思想。

##### 集群组成

通常 zookeeper 是由 2n+1 台 server 组成，每个 server 都知道彼此的存在。每个 server 都 维护的内存状态镜像以及持久化存储的事务日志和快照。对于 2n+1 台 server，只要有 n+1 台(大多数)server 可用，整个系统保持可用。我们已经了解到，一个 zookeeper 集群如果 要对外提供可用的服务，那么集群中必须要有过半的机器正常工作并且彼此之间能够正常通 信，基于这个特性，如果向搭建一个能够允许 F 台机器 down 掉的集群，那么就要部署 2*F+1 台服务器构成的 zookeeper 集群。因此 3 台机器构成的 zookeeper 集群，能够在挂掉一台 机器后依然正常工作。一个 5 台机器集群的服务，能够对 2 台机器怪调的情况下进行容灾。 如果一台由 6 台服务构成的集群，同样只能挂掉 2 台机器。因此，5 台和 6 台在容灾能力上 并没有明显优势，反而增加了网络通信负担。系统启动时，集群中的 server 会选举出一台 server 为 Leader，其它的就作为 follower(这里先不考虑 observer 角色)。 之所以要满足这样一个等式，是因为一个节点要成为集群中的 leader，需要有超过及群众过 半数的节点支持，这个涉及到 leader 选举算法。同时也涉及到事务请求的提交投票。

### zookeeper的安装部署

安装
zookeeper 有两种运行模式:集群模式和单击模式。
下载 zookeeper 安装包:http://apache.fayea.com/zookeeper/ 下载完成，通过 tar -zxvf 解压
常用命令
1. 启动ZK服务:
 bin/zkServer.sh start 
2. 查看 ZK 服务状态:
   bin/zkServer.sh status 
3. 停止 ZK 服务:
   bin/zkServer.sh stop 
4. 重启 ZK 服务:
   bin/zkServer.sh restart 
5. 连接服务器
   zkCli.sh -timeout 0 -r -server ip:port

单机环境安装

一般情况下，在开发测试环境，没有这么多资源的情况下，而且也不需要特别好的稳定性的 前提下，我们可以使用单机部署;初次使用 zookeeper，需要将 conf 目录下的 zoo_sample.cfg 文件 copy 一份重命名为 zoo.cfg 修改 dataDir 目录，dataDir 表示日志文件存放的路径(关于 zoo.cfg 的其他配置信息后面会 讲)。

集群环境安装
在 zookeeper 集群中，各个节点总共有三种角色，分别是:leader，follower，observer 集群模式我们采用模拟 3 台机器来搭建 zookeeper 集群。分别复制安装包到三台机器上并解 压，同时 copy 一份 zoo.cfg。

1. 修改配置文件
  修改端口
  server.1=IP1:2888:3888 【2888:访问 zookeeper 的端口;3888:重新选举 leader 的端口】 

  server.2=IP2.2888:3888
  server.3=IP3.2888:2888
  server.A=B:C:D:其 中
  A 是一个数字，表示这个是第几号服务器;
  B 是这个服务器的 ip 地址;
  C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口;
  D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，
  选出一个新的 Leader， 而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。 在集群模式下，集群中每台机器都需要感知到整个集群是由哪几台机器组成的，在配置文件中，按照格式 server.id=host:port:port，每一行代表一个机器配置，id: 指的是 server ID,用来标识该机器在集群中的机器序号。

2. 新建 datadir 目录，设置 myid
  在每台 zookeeper 机器上，我们都需要在数据目录(dataDir)下创建一个 myid 文件，该文件 只有一行内容，对应每台机器的 Server ID 数字;比如 server.1 的 myid 文件内容就是 1。【必 须确保每个服务器的 myid 文件中的数字不同，并且和自己所在机器的 zoo.cfg 中 server.id 的 id 值一致，id 的范围是 1~255】

3. 启动 zookeeper

### zookeeper名词复盘

#### [集群角色](https://www.jianshu.com/p/cc5c49157404)

![](http://ww4.sinaimg.cn/large/006y8mN6ly1g67bhhj42aj30y00eqq3n.jpg)

#### 数据模型

zookeeper 的视图结构和**标准的文件系统非常类似**，每一个 节点称之为 ZNode，是 zookeeper 的最小单元。每个 znode 上都可以保存数据以及挂载子节点。构成一个层次化的树形 结构。

持久节点(PERSISTENT)：创建后会一直存在 zookeeper 服务器上，直到主动删除。

持久有序节点(PERSISTENT_SEQUENTIAL) ：每个节点都会为它的一级子节点维护一个顺序。

临时节点(EPHEMERAL)：临时节点的生命周期和客户端的会话绑定在一起，当客户端会话 失效该节点自动清理。

临时有序节点(EPHEMERAL_SEQUENTIAL)： 在临时节点的基础上多了一个顺序性。

CONTAINER（新增）：当子节点都被删除后，Container 也随即删除。

PERSISTENT_WITH_TTL ：超过 TTL 未被修改，且没有子节点。

PERSISTENT_SEQUENTIAL_WITH_TTL ：客户端断开连接后不会 自动删除 Znode，如果该 Znode 没有子 Znode 且在给定 TTL 时 间内无修改，该 Znode 将会被删除;TTL 单位是毫秒，必须大于 0 且小于或等于 EphemeralType.MAX_TTL。

#### stat状态信息

每个节点除了存储数据内容以外，还存储了数据节点本身的 一些状态信息，通过 get 命令可以获得状态信息的详细内容。

![image-20190821152041481](http://ww1.sinaimg.cn/large/006y8mN6ly1g67brmj9irj310s0gemz0.jpg)

#### 版本-保证分布式数据原子性

zookeeper 为数据节点引入了版本的概念，每个数据节点都有三 类版本信息，对数据节点任何更新操作都会引起版本号的变化。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g67bvu11ihj310i04ymxj.jpg)

版本有点和我们经常使用的乐观锁类似。这里有两个概念说 一下，一个是乐观锁，一个是悲观锁 悲观锁:是数据库中一种非常典型且非常严格的并发控制策 略。假如一个事务 A 正在对数据进行处理，那么在整个处理 过程中，都会将数据处于锁定状态，在这期间其他事务无法 对数据进行更新操作。 乐观锁:乐观锁和悲观锁正好想法，它假定多个事务在处理 过程中不会彼此影响，因此在事务处理过程中不需要进行加 锁处理，如果多个事务对同一数据做更改，那么在更新请求 提交之前，每个事务都会首先检查当前事务读取数据后，是 否有其他事务对数据进行了修改。如果有修改，则回滚事务 ，再回到 zookeeper，version 属性就是用来实现乐观锁机制的 “写入校验”。

```java
// 如果stat是老的版本的话，setData会失败
client.setData().withVersion(stat.getVersion()).forPath(path, "init9999".getBytes());
```

#### watcher

zookeeper 提供了分布式数据的发布/订阅功能，zookeeper 允许客户端向服务端注册一个 watcher 监听，当服务端的一 些指定事件触发了 watcher，那么服务端就会向客户端发送一个事件通知。
值得注意的是，Watcher 通知是一次性的，即一旦触发一次 通知后，该 Watcher 就失效了，因此客户端需要反复注册 Watcher，即程序中在 process 里面又注册了 Watcher，否则， 将无法获取 c3 节点的创建而导致子节点变化的事件。

### zookeeper基于java访问

针对 zookeeper，比较常用的 Java 客户端有 zkclient、curator。 由于 Curator 对于 zookeeper 的抽象层次比较高，简化了 zookeeper 客户端的开发量。使得 curator 逐步被广泛应用。
1. 封装 zookeeper client 与 zookeeper server 之间的连接处理
2. 提供了一套 fluent 风格的操作 api
3. 提供 zookeeper 各种应用场景(共享锁、leader 选举)的抽象
封装

```java
CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString(connectString)
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(3000)
        .retryPolicy(new ExponentialBackoffRetry(1000, 3))// 重试策略：重试指定的次数，且每一次重试之间停顿的时间组件增加
        .namespace("/curator")// 命名空间，即客户端对zk上数据节点的任务操作都是相对于/curator目录进行的，这有利于实现不同的zk的业务之间的隔离
        .build();
```

节点的增删改查：参考自己写的demo

#### 节点权限设置

Zookeeper 作为一个分布式协调框架，内部存储了一些分布 式系统运行时的状态的数据，比如 master 选举、比如分布式 锁。对这些数据的操作会直接影响到分布式系统的运行状态。 因此，为了保证 zookeeper 中的数据的安全性，避免误操作 带来的影响。Zookeeper 提供了一套 ACL 权限控制机制来保 证数据的安全。

#### 节点事件监听

Watcher 监听机制是 Zookeeper 中非常重要的特性，我们基 于 zookeeper 上创建的节点，可以对这些节点绑定监听事件，比如可以监听节点数据变更、节点删除、子节点状态变更等 事件，通过这个事件机制，可以基于 zookeeper 实现分布式 锁、集群管理等功能。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g67cxksnfaj31ao0lm76b.jpg)

watcher 机制有一个特性:当数据发生改变的时候，那么 zookeeper 会产生一个 watch 事件并发送到客户端，但是客 户端只会收到一次这样的通知，如果以后这个数据再发生变 化，那么之前设置 watch 的客户端不会再次收到消息。因为 他是一次性的;如果要实现永久监听，可以通过循环注册来 实现。
curator 对节点事件监听提供了很完善的 api，接下来简单演示一下 curator 事件监听的基本使用

~~~javascript
<dependency> 
  <groupId>org.apache.curator</groupId> 

<artifactId>curator-recipes</artifactId>

 <version>4.0.0</version>

</dependency>
~~~

Curator 提供了三种 Watcher 来监听节点的变化。自己写的demo中，都有体现。
1、PathChildCache:监视一个路径下孩子结点的创建、删 除、更新。
2、NodeCache:监视当前结点的创建、更新、删除，并将 结点的数据缓存在本地。
3、TreeCache:PathChildCache 和 NodeCache 的“合体”， 监视路径下的创建、更新、删除事件，并缓存路径下所 有孩子结点的数据。

### 基于curator实现分布式锁

#### 分布式锁的基本场景

如果在多线程并行情况下去访问某一个共享资源，比如说 共享变量，那么势必会造成线程安全问题。那么我们可以 用很多种方法来解决，比如 synchronized、 比如 Lock 之 类的锁操作来解决线程安全问题，那么在分布式架构下， 涉及到多个进程访问某一个共享资源的情况，比如说在电 商平台中商品库存问题，在库存只有 10 个的情况下进来 100 个用户，如何能够避免超卖呢?所以这个时候我们需 要一些互斥手段来防止彼此之间的干扰。 然后在分布式情况下，synchronized 或者 Lock 之类的锁 只能控制单一进程的资源访问，在多进程架构下，这些 api 就没办法解决我们的问题了。怎么办呢?

#### 用zookeeper来实现分布式锁

结合我们前面对 zookeeper 特性的分析和理解，我们可以 利用 zookeeper 节点的特性来实现独占锁，就是同级节点 的唯一性，多个进程往 zookeeper 的指定节点下创建一个 相同名称的节点，只有一个能成功，另外一个是创建失败; 创建失败的节点全部通过 zookeeper 的 watcher 机制来监听 zookeeper 这个子节点的变化，一旦监听到子节点的删 除事件，则再次触发所有进程去写锁；

这种实现方式很简单，但是会产生“惊群效应”，简单来说就 是如果存在许多的客户端在等待获取锁，当成功获取到锁 的进程释放该节点后，所有处于等待状态的客户端都会被 唤醒，这个时候 zookeeper 在短时间内发送大量子节点变 更事件给所有待获取锁的客户端，然后实际情况是只会有 一个客户端获得锁。如果在集群规模比较大的情况下，会 对 zookeeper 服务器的性能产生比较的影响。

#### 利用有序节点来实现分布式锁

我们可以通过有序节点来实现分布式锁，每个客户端都往指定的节点下注册一个临时有序节点，越早创建的节点， 节点的顺序编号就越小，那么我们可以判断子节点中最小 的节点设置为获得锁。如果自己的节点不是所有子节点中 最小的，意味着还没有获得锁。这个的实现和前面单节点 实现的差异性在于，每个节点只需要监听比自己小的节点， 当比自己小的节点删除以后，客户端会收到 watcher 事件， 此时再次判断自己的节点是不是所有子节点中最小的，如 果是则获得锁，否则就不断重复这个过程，这样就不会导 致羊群效应，因为每个客户端只需要监控一个节点。

#### curator分布式锁的基本使用

curator 对于锁这块做了一些封装，curator 提供了 InterProcessMutex 这样一个 api。除了分布式锁之外， 还提供了 leader 选举、分布式队列等常用的功能。 

InterProcessMutex:分布式可重入排它锁 

InterProcessSemaphoreMutex:分布式排它锁 

InterProcessReadWriteLock:分布式读写锁

```java
public static void main(String[] args) {
    client.start();
    final InterProcessMutex lock = new InterProcessMutex(client, path);
    final CountDownLatch countDownLatch = new CountDownLatch(100);
    for (int i = 0; i < 100; i++) {
        new Thread(() -> {
            try {
                countDownLatch.await();
                lock.acquire();
                SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss|SSS");
                String format = simpleDateFormat.format(new Date());
                System.out.println(format);
            } catch (InterruptedException e) {
            } catch (Exception e) {
            } finally {
                try {
                    lock.release();
                } catch (Exception e) {
                }
            }
        }).start();
    }
    try {
        Thread.sleep(3000l);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    countDownLatch.countDown();
}
```

#### curator分布式锁的源码分析

```java
// 最常用
public InterProcessMutex(CuratorFramework client, String path)
{
    this(client, path, new StandardLockInternalsDriver());
}
public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver)
{
    this(client, path, LOCK_NAME, 1, driver);// maxLeases为1，表示可以获得分布式锁的线程数量为1，即为互斥锁
}
InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver)
{
    basePath = PathUtils.validatePath(path);
    // InterProcessMutex将操作委托给它执行
    internals = new LockInternals(client, driver, path, lockName, maxLeases);
}
```

**InterProcessMutex.acquire**

```java
@Override
public void acquire() throws Exception
{
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}
   @Override//限时等待
    public boolean acquire(long time, TimeUnit unit) throws Exception
    {
        return internalLock(time, unit);
    }
```

```java
private boolean internalLock(long time, TimeUnit unit) throws Exception
{
    /*
       Note on concurrency: a given lockData instance
       can be only acted on by a single thread so locking isn't necessary
    */

    Thread currentThread = Thread.currentThread();
    // private final ConcurrentMap<Thread, LockData> threadData = Maps.newConcurrentMap();线程和锁信息的映射关系
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null ) 
    {
        // re-entering 重入锁，count+1
        lockData.lockCount.incrementAndGet();
        return true;
    }
    // 尝试获得锁，下面会讲到
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {   // 成功获得锁
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);// put到ConcurrentMap
        return true;
    }

    return false;
}
    private static class LockData
    {
        final Thread owningThread;
        final String lockPath;
        final AtomicInteger lockCount = new AtomicInteger(1);// 记录重入次数

        private LockData(Thread owningThread, String lockPath)
        {
            this.owningThread = owningThread;
            this.lockPath = lockPath;
        }
    }
```

**LockInternals.attemptLock**

```java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
{
    final long      startMillis = System.currentTimeMillis();
    final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int             retryCount = 0;

    String          ourPath = null;
    boolean         hasTheLock = false;
    boolean         isDone = false;
    while ( !isDone )
    {
        isDone = true;

        try
        {
            // 从 InterProcessMutex 的构造函数可知实际 driver 为 StandardLockInternalsDriver 的实例
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            // 循环等待，下面会讲到
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            // gets thrown by StandardLockInternalsDriver when it can't find the lock node
            // this can happen when the session expires, etc. So, if the retry allows, just try it all again
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
            {
                isDone = false;
            }
            else
            {
                throw e;
            }
        }
    }

    if ( hasTheLock )// 成功获得分布式锁
    {
        return ourPath;
    }

    return null;// 获得分布式锁失败
}
```

**createsTheLock**

```java
@Override
public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception
{
    String ourPath;
    if ( lockNodeBytes != null )
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, lockNodeBytes);
    }
    else // 创建临时顺序节点
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
    }
    return ourPath;
}
```

**internalLockLoop**

```java
// 循环等待来激活分布式锁，实现锁的公平性
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;// 是否需要删除子节点
    try
    {
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            List<String>        children = getSortedChildren();// 获得排序后的子节点列表
            // 获得自己创建的临时顺序子节点的名称
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
            // 判断sequenceNodeName在children排名，一来可以决定是否获得锁，二来决定watch上一个临时顺序节点。下文会提到
            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if ( predicateResults.getsTheLock() )
            {// 获得锁
                haveTheLock = true;
            }
            else
            {// 没获得锁，watch上一个临时顺序节点
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this)
                {
                    try 
                    {// exists()会导致资源泄漏，因此exists()可以监听不存在的znode，因此采用getData()
                     // 上一个临时顺序节点如果被删除，会唤醒当前线程继续竞争锁，正常情况下直接获得锁，因为锁是公平的
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }

                            wait(millisToWait);// 限时等待
                        }
                        else
                        {
                            wait();// 无限等待
                        }
                    }
                    catch ( KeeperException.NoNodeException e ) 
                    {// client.getData()有可能抛出NoNodeException，（previousSequencePath被释放）
                     // 继续while循环
                        // it has been deleted (i.e. lock released). Try to acquire again
                    }
                }
            }
        }
    }
    catch ( Exception e )
    {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;// 发生异常，删除之前自己创建的临时顺序节点
        throw e;
    }
    finally
    {
        if ( doDelete )
        {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```

**getsTheLock**

```java
@Override
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
{
    // 自己创建的临时顺序节点，排在排序后的子节点列表中的索引
    int             ourIndex = children.indexOf(sequenceNodeName);
    validateOurIndex(sequenceNodeName, ourIndex);// 找不到，会抛出异常
    // 由InterProcessMutex 的构造函数可知， maxLeases 为 1，即只有 ourIndex 为 0 时，线程才能持有锁，或者说该线程创建的临时顺序节点激活了锁
    // Zookeeper 的临时顺序节点特性能保证跨多 个 JVM 的线程并发创建节点时的顺序性，越早创建临时 顺序节点成功的线程会更早地激活锁或获得锁
    boolean         getsTheLock = ourIndex < maxLeases;
    // 如果获得锁了，无需监听任何节点，否则需要监听上一个顺序节点
    // 这样子的设计，是为了防止惊群效应，如果监听获得锁的节点，那么一旦释放锁，其他所有节点都会受到通知，并且一起抢占锁，对于性能大为不利
    String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

    return new PredicateResults(pathToWatch, getsTheLock);
}
    static void validateOurIndex(String sequenceNodeName, int ourIndex) throws KeeperException
    {
        if ( ourIndex < 0 )
        {// 容错处理，由于会话过期或者链接丢失等原因，该线程创建的临时顺序节点被zk服务端删除，往外抛出异常
         // 前面的代码会进行catch，如果在重试范围之内，则进行重新尝试获得锁，这会重新生成临时顺序节点
            throw new KeeperException.NoNodeException("Sequential path not found: " + sequenceNodeName);
        }
    }
```

**释放锁的逻辑**

```java
@Override
public void release() throws Exception
{
    /*
        Note on concurrency: a given lockData instance
        can be only acted on by a single thread so locking isn't necessary
     */

    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    if ( lockData == null )
    {
        throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
    }

    int newLockCount = lockData.lockCount.decrementAndGet();// count-1
    if ( newLockCount > 0 )
    {// 锁是可以重入的
        return;
    }
    if ( newLockCount < 0 )
    {
        throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
    }
    try
    {// 释放锁资源
        internals.releaseLock(lockData.lockPath);
    }
    finally
    {// 映射表中移除当前线程的锁信息
        threadData.remove(currentThread);
    }
}
```

LockInternals.releaseLock

```java
void releaseLock(String lockPath) throws Exception
{
    revocable.set(null);
    deleteOurPath(lockPath);// 删除临时顺序节点，只会触发后一顺序节点去获取锁，理论上不存在竞争，只排队，非抢占，公平锁，先到先得
}
private void deleteOurPath(String ourPath) throws Exception
    {
        try
        {// 后台不断尝试删除
            client.delete().guaranteed().forPath(ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {// 会话如果过期，节点已经不再，delete会抛出此异常，不做任何处理
            // ignore - already deleted (possibly expired session, etc.)
        }
    }
```

### 基于curator实现leader选举

在分布式计算中，leader election 是很重要的一个功能， 这个选举过程是这样子的：指派一个进程作为组织者，将任务分发给各节点。在任务开始前，哪个节点都不知道谁是 leader 或者 coordinator。当选举算法开始执行后，每个节点最终会得到一个唯一的节点作为任务 leader。除此之外，选举还经常会发生在 leader 意外宕机的情况下，新的 leader 要被选举出来。Curator 有两种选举 recipe(Leader Latch 和 Leader Election)。

**Leader Latch**

参与选举的所有节点，会创建一个顺序节点，其中最小的节点会设置为 master 节点, 没抢到 Leader 的节点都监听 前一个节点的删除事件，在前一个节点删除后进行重新抢主，当 master 节点手动调用 close()方法或者 master 节点挂了之后，后续的子节点会抢占 master。其中 spark 使用的就是这种方法。

**LeaderSelector**

LeaderSelector 和 Leader Latch 最的差别在于，leader 可以释放领导权以后，还可以继续参与竞争。

```java
public static void main(String[] args) throws Exception {
    client.start();
    LeaderSelector selector = new LeaderSelector(client, path, new LeaderSelectorListenerAdapter() {
        public void takeLeadership(CuratorFramework client) throws Exception {
            System.out.println("成为master:" + System.currentTimeMillis()/1000);
            Thread.sleep(3000l);
            System.out.println("释放master");
        }
    });
    selector.autoRequeue();
    selector.start();
    Thread.sleep(Integer.MAX_VALUE);
}
```

### Zookeeper数据的同步流程

在 zookeeper 中，客户端会随机连接到 zookeeper 集群中 的一个节点，如果是读请求，就直接从当前节点中读取数 据，如果是写请求，那么请求会被转发给 leader 提交事务，然后 leader 会广播事务，只要有超过半数节点写入成功， 那么写请求就会被提交(类 2PC 事务)。

那么问题来了
1. 集群中的 leader 节点如何选举出来?
2. leader 节点崩溃以后，整个集群无法处理写请求，如何快速从其他节点里面选举出新的 leader 呢?
3. leader 节点和各个 follower 节点的数据一致性如何保证？

#### ZAB协议

ZAB(Zookeeper Atomic Broadcast) 协议是为分布式协 调服务 ZooKeeper 专门设计的一种支持崩溃恢复的原子 广播协议。在 ZooKeeper 中，主要依赖 ZAB 协议来实现 分布式数据一致性，基于该协议，ZooKeeper 实现了一种 主备模式的系统架构来保持集群中各个副本之间的数据一 致性。

##### ZAB协议介绍

ZAB 协议包含两种基本模式，分别是

1. 崩溃恢复
2. 原子广播

当整个集群在启动时，或者当 leader 节点出现网络中断、 崩溃等情况时，ZAB 协议就会进入恢复模式并选举产生新 的 Leader，当 leader 服务器选举出来后，并且集群中有过 半的机器和该 leader 节点完成数据同步后(同步指的是数 据同步，用来保证集群中过半的机器能够和 leader 服务器 的数据状态保持一致)，ZAB 协议就会退出恢复模式。 当集群中已经有过半的 Follower 节点完成了和 Leader 状 态同步以后，那么整个集群就进入了消息广播模式。这个 时候，在 Leader 节点正常工作时，启动一台新的服务器加 入到集群，那这个服务器会直接进入数据恢复模式，和leader 节点进行数据同步。同步完成后即可正常对外提供 非事务请求的处理。
需要注意的是:leader 节点可以处理事务请求和非事务请 求，follower 节点只能处理非事务请求，如果 follower 节 点接收到非事务请求，会把这个请求转发给 Leader 服务器

##### 消息广播的实现原理

消息广播的过程实际上是一个 简化版本的二阶段提交过程。

1. leader 接收到消息请求后，将消息赋予一个全局唯一的64 位自增 id，叫:zxid，通过 zxid 的大小比较既可以实现因果有序这个特征。
2. leader 为每个 follower 准备了一个 FIFO 队列(通过 TCP协议来实现，以实现了全局有序这一个特点)将带有 zxid的消息作为一个提案(proposal)分发给所有的 follower。
3. 当 follower 接收到 proposal，先把 proposal 写到磁盘，写入成功以后再向 leader 回复一个 ack。
4. 当 leader 接收到合法数量(超过半数节点)的 ACK 后，leader 就会向这些 follower 发送 commit 命令，同时会在本地执行该消息。
5. 当 follower 收到消息的 commit 命令以后，会提交该消息。

> ps: 和完整的 2pc 事务不一样的地方在于，zab 协议不能 终止事务，follower 节点要么 ACK 给 leader，要么抛弃 leader，只需要保证过半数的节点响应这个消息并提交了 即可，虽然在某一个时刻 follower 节点和 leader 节点的 状态会不一致，但是也是这个特性提升了集群的整体性 能。 当然这种数据不一致的问题，zab 协议提供了一种 恢复模式来进行数据恢复，后续讲解。

这里需要注意的是:
leader 的投票过程，不需要 Observer 的 ack，也就是 Observer 不需要参与投票过程，但是 Observer 必须要同步 Leader 的数据从而在处理请求的时候保证数据的一致性。

#### 崩溃恢复的实现原理

前面我们已经清楚了 ZAB 协议中的消息广播过程，ZAB 协议的这个基于原子广播协议的消息广播过程，在正常情况 下是没有任何问题的，但是一旦 Leader 节点崩溃，或者由 于网络问题导致 Leader 服务器失去了过半的 Follower 节 点的联系(leader 失去与过半 follower 节点联系，可能是 leader 节点和 follower 节点之间产生了网络分区，那么此 时的 leader 不再是合法的 leader 了)，那么就会进入到崩 溃恢复模式。崩溃恢复状态下 zab 协议需要做两件事

1. 选举出新的 leader
2. 数据同步

前面在讲解消息广播时，知道 ZAB 协议的消息广播机制是 简化版本的 2PC 协议，这种协议只需要集群中过半的节点响应提交即可。但是它无法处理 Leader 服务器崩溃带来的 数据不一致问题。因此在 ZAB 协议中添加了一个“崩溃恢复模式”来解决这个问题。那么 ZAB 协议中的崩溃恢复需要保证，如果一个事务 Proposal 在一台机器上被处理成功，那么这个事务应该在 所有机器上都被处理成功，哪怕是出现故障。为了达到这 个目的，我们先来设想一下，在 zookeeper 中会有哪些场 景导致数据不一致性，以及针对这个场景，zab 协议中的 崩溃恢复应该怎么处理。

##### 已经被处理的消息不能丢

当 leader 收到合法数量 follower 的 ACKs 后，就向各 个 follower 广播 COMMIT 命令，同时也会在本地执行 COMMIT 并向连接的客户端返回「成功」。但是如果在各 个 follower 在收到 COMMIT 命令前 leader 就挂了，导 致剩下的服务器并没有执行都这条消息。

##### 被丢弃的消息不能再次出现

当 leader 接收到消 息请求生 成 proposal 后就挂 了，其他 follower 并没有收 到此 proposal ，因此经 过恢复模式重新选 了 leader 后，这条 消息是被 跳过的。 此时，之 前挂了的 leader 重 新启动并 注册成了 follower， 他保留了 被跳过消 息的proposal 状态，与 整个系统 的状态是 不一致 的，需要将其删除。

leader 选举算法：能够确保已经被 leader 提交的事务 Proposal 能够提交、同时丢弃已经被跳过的事务 Proposal。 针对这个要求。

1. 如果 leader 选举算法能够保证新选举出来的 Leader 服 务器拥有集群中所有机器最高编号(ZXID 最大)的事务 Proposal，那么就可以保证这个新选举出来的 Leader 一 定具有已经提交的提案。因为所有提案被 COMMIT 之 前必须有超过半数的 followerACK，即必须有超过半数节点的服务器的事务日志上有该提案的 proposal，因此， 只要有合法数量的节点正常工作，就必然有一个节点保 存了所有被 COMMIT 消息的 proposal 状态。
2. 另外一个，zxid 是 64 位，高 32 位是 epoch 编号，每经 过一次 Leader 选举产生一个新的 leader，新的 leader 会将 epoch 号+1，低 32 位是消息计数器，每接收到一 条消息这个值+1，新 leader 选举后这个值重置为 0.这样 设计的好处在于老的 leader 挂了以后重启，它不会被选 举为 leader，因此此时它的 zxid 肯定小于当前新的 leader。当老的 leader 作为 follower 接入新的 leader 后，新的 leader 会让它将所有的拥有旧的 epoch 号的 未被 COMMIT 的 proposal 清除。

#### 关于ZXID

前面一直提到 zxid，也就是事务 id，那么这个 id 具体起什 么作用，以及这个 id 是如何生成的，简单给大家解释下 为了保证事务的顺序一致性，zookeeper 采用了递增的事 务 id 号(zxid)来标识事务。所有的提议(proposal)都 在被提出的时候加上了 zxid。实现中 zxid 是一个 64 位的 数字，它高 32 位是 epoch(ZAB 协议通过 epoch 编号来 区分 Leader 周期变化的策略)用来标识 leader 关系是否 改变，每次一个 leader 被选出来，它都会有一个新的epoch=(原来的 epoch+1)，标识当前属于那个 leader 的 统治时期。低 32 位用于递增计数。 epoch:可以理解为当前集群所处的年代或者周期，每个 leader 就像皇帝，都有自己的年号，所以每次改朝换代，leader 变更之后，都会在前一个年代的基础上加 1。这样就算旧的 leader 崩溃恢复之后，也没有人听他的了，因为 follower 只听从当前年代的 leader 的命令。

