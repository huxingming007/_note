~~~mysql
-- user_account添加主键
alter table user_account add account_id BIGINT(10) unsigned not Null auto_increment primary key FIRST;
-- 注意分租户
CREATE TABLE `user_menu` (
  `id` bigint(10) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `name` varchar(128) NOT NULL COMMENT '名称，唯一',
  `url` varchar(128) DEFAULT '' COMMENT '链接',
  `sort` char(2) NOT NULL COMMENT '排序',
  `ranks` char(2) NOT NULL DEFAULT '1' COMMENT '级别：1：主菜单，一级,父级；2：二级；3：三级，最多三级',
  `parent_id` bigint(10) DEFAULT 0 COMMENT '父菜单id：默认为0，即没有父级菜单，button必须有父菜单id',
  `menu_type` varchar(10) NOT NULL COMMENT '主菜单：menu，子菜单：page，按钮：button',
  `menu_platform` varchar(10) DEFAULT 'oss' COMMENT '适用的平台',
  `remarks` varchar(255) DEFAULT NULL COMMENT '备注信息',
  `created_by` char(32) NOT NULL COMMENT '创建者',
  `created_time` datetime NOT NULL COMMENT '创建时间',
  `updated_by` char(32) NOT NULL COMMENT '更新者',
  `updated_time` datetime NOT NULL COMMENT '更新时间',
  `del_flag` char(1) DEFAULT '0' COMMENT '是否删除'
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='菜单资源表'

CREATE TABLE `user_role` (
  `id` bigint(10) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `name` varchar(128) NOT NULL COMMENT '名称，唯一',
  `remarks` varchar(255) DEFAULT NULL COMMENT '备注信息',
  `created_by` char(32) NOT NULL COMMENT '创建者',
  `created_time` datetime NOT NULL COMMENT '创建时间',
  `updated_by` char(32) NOT NULL COMMENT '更新者',
  `updated_time` datetime NOT NULL COMMENT '更新时间',
  `del_flag` char(1) DEFAULT '0' COMMENT '是否删除'
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='角色表'

CREATE TABLE `user_account_role` (
  `id` bigint(10) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `account_id` bigint(10) NOT NULL COMMENT '账号id，唯一',
  `role_id` bigint(10) NOT NULL COMMENT '角色id，唯一',
  `remarks` varchar(255) DEFAULT NULL COMMENT '备注信息',
  `created_by` char(32) NOT NULL COMMENT '创建者',
  `created_time` datetime NOT NULL COMMENT '创建时间',
  `updated_by` char(32) NOT NULL COMMENT '更新者',
  `updated_time` datetime NOT NULL COMMENT '更新时间',
  `del_flag` char(1) DEFAULT '0' COMMENT '是否删除'
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='账号和角色关联表'

CREATE TABLE `user_role_menu` (
  `id` bigint(10) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `role_id` bigint(10) NOT NULL COMMENT '角色id，唯一',
  `menu_id` bigint(10) NOT NULL COMMENT '菜单id，唯一',
  `remarks` varchar(255) DEFAULT NULL COMMENT '备注信息',
  `created_by` char(32) NOT NULL COMMENT '创建者',
  `created_time` datetime NOT NULL COMMENT '创建时间',
  `updated_by` char(32) NOT NULL COMMENT '更新者',
  `updated_time` datetime NOT NULL COMMENT '更新时间',
  `del_flag` char(1) DEFAULT '0' COMMENT '是否删除'
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='角色和菜单关联表'

~~~

哪里进行切面，在切面出做点啥。

proxyfactorybean。getobject。创建一个代理对象、构建通知器链。这是XML的方式。

扩展beanpostprocessor。重点在看postprocessorafterinitialization方法。在此里面创建了代理对象。这是注解的方式。





~~~java
//        List<UserMenuEntity> userMenuEntitys1 = userMenuMapper.selectByParentKey(id);
//        if (null != userMenuEntitys1 && !userMenuEntitys1.isEmpty()) {
//            userMenuEntitys1.forEach(o1 -> {
//                deletes.add(o1.getId());
//                if ("menu".equals(o1.getMenuType())) {
//                    List<UserMenuEntity> userMenuEntitys2 = userMenuMapper.selectByParentKey(o1.getId());
//                    if (null != userMenuEntitys2 && !userMenuEntitys2.isEmpty()) {
//                        userMenuEntitys2.forEach(o2 -> {
//                            deletes.add(o2.getId());
//                            if ("menu".equals(o2.getMenuType())) {
//                                List<UserMenuEntity> userMenuEntitys3 = userMenuMapper.selectByParentKey(o2.getId());
//                                if (null != userMenuEntitys3 && !userMenuEntitys3.isEmpty()) {
//                                    // 最多3层
//                                    userMenuEntitys3.forEach(o3 -> deletes.add(o3.getId()));
//                                }
//                            }
//                        });
//                    }
//                }
//            });
//        }
~~~

每个server提出一个提案，最终只能有一个提案被大家认同，也就是说一致性通过。

leader：事务请求的唯一处理者；

Follower：1、处理非事务请求，把事务请求转发给leader；2、提案的投票；3、leader选举投票

zab协议：1、崩溃恢复：当leader服务器崩溃、网络分区网络不可用导致leader服务无法正常对外服务，此时zab协议进入了恢复模式并且选举出新的leader出来，当选举出新的leader出来后，集群中有过半的服务器已经和leader完成数据同步，此时zab协议退出恢复模式。2、消息广播：当一台服务器加入到集群中后，leader在负责消息广播，新加入的服务器自觉进入数据恢复模式。





1、分配内存空间

2、初始化对象

3、instance指向内存空间

4、初次访问对象

2和3的指令重排序。其他线程会访问到没有初始化的实例。





~~~java
public void batchInsert(List<LoanGroupOrderLog> list);

  <!-- 批量插入对象信息(对象中属性不为null入库)-->
  <insert id="batchInsert" parameterType="java.util.List" useGeneratedKeys="true" keyProperty="id">
     insert into loan_group_order_log (file_name, batch_no,
      group_status, content, error_msg,
      remark, tenant_id, del_flag,
      created_by, created_time, updated_by,
      updated_time)
    values
    <foreach collection="list" item="item" index="index" separator=",">
      (#{item.fileName,jdbcType=VARCHAR}, #{item.batchNo,jdbcType=VARCHAR},
      #{item.groupStatus,jdbcType=VARCHAR}, #{item.content,jdbcType=VARCHAR}, #{item.errorMsg,jdbcType=VARCHAR},
      #{item.remark,jdbcType=VARCHAR}, #{item.tenantId,jdbcType=VARCHAR}, #{item.delFlag,jdbcType=CHAR},
      #{item.createdBy,jdbcType=VARCHAR}, #{item.createdTime,jdbcType=TIMESTAMP}, #{item.updatedBy,jdbcType=VARCHAR},
      #{item.updatedTime,jdbcType=TIMESTAMP})
    </foreach>
  </insert>
~~~





dubbo 发布流程

1、创建一个Invoker

2、把Invoker保存到一个map中

3、URL注册到注册中心

dubbo消费流程

1、创建一个动态代理

2、获得目标服务的URL地址

3、实现远程网络通信

4、实现负载均衡

5、实现集群容错



dubbo SPI是对java spi 的扩展升级，是通过键值对的方式进行配置，**我们可以按需加载指定的实现类。**

Dubbo ioc 是通过setter方法注入依赖。首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检查方法名是否具有setter方法特征，若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中。



受到JVM本身设置固定大小上限，无法进行调整，而元空间使用的是直接内存，受本机可用内存的限制，并且永远不会OOM。永久代的GC带来了不必要的复杂度。



Rabbitmq 由于基于erlang开发，并发性能很强，延时很低。吞吐量比kafka和rocketmq低。消息丢失可能性非常低。

rabbitmq可靠性：

1、消息到交换机

2、交换机到队列

3、消息进行持久化

4、rabbitmq做集群

5、消费者异常，断开连接后，rabbitmq把消息重新发送给其他消费者

线程的上下文切换：当前任务在执行完CPU时间片切换到另一个任务之前会先保存自己的状态，以便下次再切换回这个任务时候，可以再加载这个任务的状态。任务从保存到再加载的过程就是一次上下文切换。上下文切换对系统来说意味着消耗大量的CPU时间。

线程遇到monitorenter字节码指令时，会尝试去获得monitor对象的持有权，monitor对象在锁对象的对象头里面，monitor对象内部有一个计数器，当计数器为0时，可以获得持有权即获得锁，获取后计数器加1，当执行monitorexit，将锁计数器设为0，表明锁被释放。

偏向锁：轻量级锁在无竞争的

轻量级锁：CAS代替互斥量的开销。

他们的本意都是为了没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

Happenbefore 前一个操作的结果对于后续操作是可见的！

1、单线程情况下，一个线程的每个操作happens-before于该线程中的任意后续操作；

2、volatile规则，对volatile的写操作一定是happens-before对volatile的读操作；

3、传递性。

4、start规则。

5、join规则。

6、监视器规则。

线程池的监控：扩展方法，beforeExecute 、afterExecute、shutDown





~~~java

            if (null != button && !button.isEmpty()) {
                Iterator<UserMenuDTO> it = button.iterator();
                while (it.hasNext()) {
                    if (!menuIdsExist.contains(it.next())) {
                        it.remove();
                    }
                }
            }
~~~

~~~java
lock：线程被中断时，必须先获得锁，然后响应中断；
~~~



```java
lockInterruptibly：直接抛出异常，不用先获得锁；
```

1、减少锁持有的时间；

2、减小锁粒度；hashmap

3、读写分离锁；arrayListblockqueue，takelock，putlock





Hashmap.key value

userId

渠道code

list<map<string,string>> list = new arraylist<>();

map<string,string> map = new hashmap;

Map.put("userId","1dfasfdsaf");

Map.put("channelCode","QD000001");

List.add(map);



userId，name

回忆JDBC步骤：加载驱动、获得连接、生成preparedstatement、组装参数、执行ps.execute、返回参数进行组装

实例化bean、设置对象属性、检查aware相关接口、beanpostprocessor、afterpropertiesset、init-method、beanpostprocessor、回调、使用中、disposablebean、destory

请求、dispatchservlet、handermapping拿到handlerlist、请求handler返回modelview、视图解析器、视图渲染返回给前端。

观察者模式：事件applicationevent、事件监听applicationlistener、application.publish（event）。

适配器模式：spring在AOP，advice使用过程中，advisoradapter，是是配成methodintercept

工厂、单例、模板方法模式，代理模式、观察者模式、适配器模式

初始化IOC、注入依赖。



redo log

是否在等待锁：show processlist查看当前的状态

主键索引存放的是整行字段的数据

非主键索引存放的是主键字段的值+列值，走两次索引。

采样的方式，进行优化。

覆盖索引：不需要“回表”进行查询。

union会进行去重

大批量数据写导致的问题：1、主从延迟；2、大量binlog日志导致主从延迟；3、同一个事务中进行，对大批量数据进行锁定，导致大量阻塞，占用可用连接，导致应用的延迟。故一定要进行分批次写。

对于大表修改表结构会导致大量锁表。

设计模式：1、简单工厂，提供一个静态方法调用；2、工厂方法模式，调用具体工厂角色实现这些不同的工厂方法，比如我想生成苹果手机，就调用苹果工厂生产苹果手机的的方法；3、抽象工厂模式，比如我要生成苹果手机，调用手机工厂生产苹果手机的方法。4、工厂方法模式和抽象工厂模式的区别：前一个工厂生成单一产品，后一个工厂生成一整产生，这些产品必须互相是有依赖关系的。

使用原型模式可以简化创建对象过程。

策略模式：

jre是运行环境包括了JVM，java类库，java命令，jdk包括jre加javac和工具。

多态：引用变量指向哪个类的实例对象，程序运行期间才能知道。继承、接口。

Jdk1.8在接口中默认实现，和静态方法。

重写equals方法必须重写hashcode，person对象两个属性age和name，hashset....



while循环里面监听客户端的连接，如果有一个连接进来，建立一个socket套接字进行读写操作，这个是同步阻塞的，当没有完成读写操作，其他客户端是连接不过来的，于是就有了BIO通信模型的多线程版本，一个线程对应一个socket连接，这样子会出现问题：1、线程创建和销毁需要性能开销；2、连接大时，系统资源容易被耗尽；于是乎就有了线程池机制来进行改善，线程池可以控制队列的长度和线程的个数，防止系统资源被耗尽。

IO是阻塞的，NIO是非阻塞的，从通道中读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以做别的事情。当通道准备好读写时，得到通知，接着就由这个线程自行进行IO操作，IO操作本身是同步的。

读取数据，从channel到buffer。

写入数据，从buffer到channel。

selector选择器，使得单个线程可以处理多个通道，因此需要较少的线程处理通道，避免了线程之间的切换。

AIO是异步IO，真正的异步非阻塞IO，基于事件和回调机制，也就是当应用操作之后会直接返回，不会阻塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

阻塞IO、非阻塞IO、IO复用模型、信号驱动IO模型、异步IO模型

鱼（文件）从鱼塘（硬盘）转移（拷贝）到鱼篓（用户空间）里面。

应用程序调用内核获取数据。

阻塞IO：慢慢的等待鱼上钩，啥事也不干，一心一意，专心致志；

非阻塞IO：应用进程轮训访问内核，确认数据包有没有准备好；

信号驱动IO模型：先向内核注册一个信号处理函数，当内核数据准备就绪会发送一个信号给进程；

IO复用模型：多用几条鱼竿，select函数，几个鱼竿数据还没准没好时，select函数会进行阻塞，多个channel会在selector进行注册，当某一个channel数据准备好时，select会进行返回，然后进程通过recvfrom来进行数据拷贝。

以上几种都是同步IO，因为数据拷贝过程是同步进行的。

异步IO模型：数据拷贝好时，进行通知到应用进程。

reactor和handler。reactor负责接收新连接，并分派请求到handler处理器中，

单线程reactor模型，reactor线程和handler是同一个线程，handler的阻塞会导致其他handler的阻塞，导致acceptor也被阻塞，基本上很少使用这种模型。

多线程reactor模型，而对于reactor而言可以仍旧是单个线程，handler处理器的执行放入线程池，多线程进行业务处理。可以使用有限的线程个数处理更多的请求，节约了线程的资源。

持续改进：reactor线程改成多线程

时间复杂度：o1-ologn-on（线性增长）-on2次方

二叉树数据结构，二叉查找法，但是会因为多次插入连续的新节点导致不平衡。平衡二叉树红黑树应运而生。红黑树会有一些规则，使他变成一颗平衡二叉树。根节点是黑色、叶子节点都是黑色的空节点，每个红节点的两个子节点都是黑色的。当删除或者插入节点的时候，红黑树的规则有可能被打破，变色和旋转。

Redis事务，几条命令一起执行，multi-exec。他是悲观锁。乐观锁的方式是使用watch，CAS的方式，当监视的key在整个事务期间被人修改了，整个事务就会被取消掉。

每个节点都提出一个提案，只有一个提案被通过，得到大家的认许。达到一致的结果。

Google的chubby最开始解决GFS的master选举的问题。所有的server向chubby创建同一个文件，只有一个server会成功，它就是成为master。其实这就是粗粒度的分布式锁实现，通过创建文件的方式进行加锁。zookeeper就是chubby的开源实现，并不是作为注册中心而设计，他是分布式锁的一种设计。注册中心只是他能够实现的一种功能而已。

分布式事务需要用到的协议是2PC和3PC

zookeeper集群搭建，配置zoo.cfg配置文件，server.A=B:C:D，其中A是一个数字，标识该机器在集群中的机器序号，B是该机器IP地址，C2888该机器与leader服务器交换信息的接口，D3888重新选举leader服务器的端口。设置myid文件，该文件只要一行内容，对应每台机器的server ID数字。

数据模型新增容器节点，当子节点都被删除后，容器也随即删除。TTL节点。

zxid事务ID。

引入版本号，保证分布式数据原子性。version、cversion、aversion。有点类似于乐观锁的概念。乐观锁：假设没有冲突，操作数据之前没有进行加锁操作，对数据进行提交之前，检查这期间有没有其他事务对其进行操作，如果有则回滚，没有则提交。

节点事件监听：监听节点数据变更、节点删除、子节点状态变更等等事件。产生一个watch事件并发送到客户端，但是客户端只收到一次这样的通知。

分布式锁，多个进程往一个ZK上的指定节点创建一个子节点，创建成功获得锁，失败则注册一个子节点变更的watch，当释放锁即删除子节点会通知到其他进程，优点是比较简单，缺点，惊群效应，如果有很多客户端来等到获得锁，ZK会发送非常多的子节点变更事件通知给所有待获取锁的客户端，然后实际情况下只会有一个客户端获得锁。如果在集群规模比较大的情况下，会ZK的服务器性能产生比较大的影响。

有序节点获得分布式锁，指定节点下面创建临时有序节点，越早创建，序号越小，比较序号大小，最小的那个节点获得锁。没有获得锁的客户端只监听比自己小的那个节点，比方说1获得锁，2监听1，3监听2，避免了惊群效应。

消息广播：1、leader接收到消息后，生成一个zxid；2、把带有zxid的消息作为一个提案分发给其他follower；3、follower接收到proposal，先把proposal写到磁盘，写入成功后再向leader回复一个ack。4、当leader获得超过半数的ack，表明这个提案通过，于是向follower发送commit命令。5、follower收到消息后进行commit。

zxid全局唯一递增。

ZAB协议两种基本的模式：崩溃恢复（leader选举）和消息广播

dubbo是一个生态，如果自己实现一个网络通信，需要考虑到：1、底层网络协议的设计；2、序列化和反序列化；3、服务链路的跟踪和监控；4、服务注册和发现；5、负载均衡；6、容错机制，防止服务雪崩。

支持多注册中心，多协议的支持，默认是dubbo，还可以是webservice、rest。

负载均衡：默认是权重随机算法、最小活跃数负载均衡、一致性hash、权重轮训算法。

集群容错：自动重试（默认，适用于读操作）、快速失败（非幂等性）、失败安全（catch住异常，返回一个空结果）、失败后定时重发、并行调用、广播调用。

新功能：配置中心，apollo nacos zk。

新功能：元数据中心，配置信息：接口名称、重试次数、负载均衡、容错策略、版本号等，所有信息都是基于URL的形式配置在ZK中。缺点：1、存储空间；2、消耗网络带宽；3、网络传输慢。改造：把服务治理相关的数据发布到ZK上，其他的配置数据统一发布到元数据中心。ZK获得Redis做元数据中心；

dubbo发布流程：生成一个Invoker（代理）、保存到一个map中、URL地址注册到注册中心。

dubbo消费流程：生成代理、获得目标服务的URL地址、实现远程网络通信、实现负载均衡、实现集群容错；

SPI的机制就是把接口实现类配置在配置文件中，项目在运行过程中根据读取配置文件，动态切换实现类。dubbo SPI是java SPI的增强版本，配置键值对的方式，按需加载，另外实现了IOC的特性。

SPI自适应扩展点，希望拓展方法被调用时，根据**运行时参数进行加载，**

~~~java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class)
  .getAdaptiveExtension();
    
@Adaptive 
<T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
        
Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
~~~

~~~java
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol { public void destroy() {
throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol. destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public int getDefaultPort() {
throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol. getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc. RpcException {
if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument g etUrl() == null");
org.apache.dubbo.common.URL url = arg0.getUrl();
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Pro
tocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionL
oader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName); return extension.export(arg0);
}
public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws o rg.apache.dubbo.rpc.RpcException {
if (arg1 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg1;
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Pro
tocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionL
oader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName); return extension.refer(arg0, arg1);
} }
~~~

new初始化 runnable运行状态 阻塞状态block synchronized 

wait join locksupport.park.time_wait、terminat

上下文切换：任务从保存再到加载的过程就是一次上下文切换。因为切换需要时间，所以上下文切换系统意味着消耗着大量的CPU时间，事实上，可能是操作系统中时间消耗最大的操作。线程1在执行A任务，线程2在执行B任务，CPU核心只能被一个线程使用，线程1使用完时间片后，需要让给线程2执行，这个时候线程1得保存任务，下次切换回来的时候，就知道上一次任务执行到了哪儿。

CPU给线程时间片。

**synchronized**重量级锁，获得锁对象的监视器，每个对象都可以作为锁，对象：对象头和实例数据。监视器锁是依赖于底层的操作，挂起和唤醒线程，都需要操作系统的帮忙完成，用户态转到内核态。偏向锁、轻量级锁、自旋锁、适应性自旋、锁粗化、锁消除等减少锁操作的开销。

Javac变成.class javap字节码信息。。。。synchronized代码块 monitorenter    monitorexit  当执行enter的时候，获得监视器锁，这把锁存放在锁对象的对象头里面，因为是可以重入的，所以monitor里边维护了一个计数器。enter就是加一，exit就是减一。flags：acc_synchronized。标识。

reentrantlock 和 synchronized的区别：1、都是可重入锁；2、一个基于JVM一个是基于java API；3、reentrantlock加入了一些高级功能，等待可中断、公平锁、锁可以绑定多个条件实现可以选择性通知；4、性能已经不再是选择的标准了。

偏向锁：没有竞争的前提下，每次申请锁都是同一个线程，把整个同步都消除掉。一把锁总是由同一个线程多次获得。

轻量级锁：毕竟偏向锁太理想化了，不可能只有一个线程，没有互斥量的开销，加锁和解锁都用到了CAS

自旋锁和自适应自旋：竞争激烈的话，轻量级锁还会有互斥量的开销，升级成重量级锁做的一次努力，挂起线程和恢复线程性能开销会比较大。

锁消除：数据在明显不会出现竞争的场景下，如果使用了锁，JVM会优化，执行锁消除动作，不会去同步操作请求锁的操作，比方说在方法里面定义了一个stringbuffer，stringbuffer是线程安全的，每个方法里面都有同步，虚拟机栈是线程私有的。。。。。

锁粗化：连续的获得锁，释放锁，对性能有影响，干脆就把同步范围块扩大

对象：对象头、实例数据、填充数据；

对象头：Mark Word 、对象指针（确定这个对象是哪个类的实例）

Mark Word：锁相关的标识、GC分代年龄、hashcode、

JMM：解决可见性和有序性的问题。主内存、工作内存。工作内存保存的是主内存中的变量副本。插入内存屏障禁止指令重排序。

happensBefore：1、一个线程中的每个操作，happensbefore后续操作；

2、传递性，a hb b ,b hb c 那么 a肯定hb c

3、volatile，对volatile的写肯定happensbefore对volatile的写；

4、start规则。主线程定义一个X变量，在主线程开始一个线程，这个线程对于X变量可见的

5、join规则。主线程开启一个线程对一个变量X进行修改，t.join。之后的代码对X是可见的；

6、监视器规则，解锁肯定happens-before于对这个锁的加锁动作。

可见性问题：单线程情况在是不会发生这样子的问题。多线程情况下，比方说读和写分别在两个不同的线程中，读线程不能及时读到写线程写入的最新的值。

写屏障：插入一个写屏障，强制变量写入到主内存中；

读屏障：插入一个读屏障，强制去主内存中读取最新的数据。

可见性问题：是由于缓存和指令重排序导致的。JMM可以禁用缓存和重排序。

JMM：java是跨平台，一次编译多平台运行。定一个一个java内存模型屏蔽掉各个操作系统和硬件的访问差异。C++是使用硬件和操作系统的内存模型，不同的平台会有不同的内存模型的差异。

volatile是最轻量级的同步机制。

AQS核心思想：共享资源是空闲的，当前线程来请求，把当前线程置为有效线程。资源被其他线程占用，请求线程只能进入队列FIFO中，等待资源释放后，唤醒线程。共享资源是通过一个int state来表示，volatile修饰，保证可见性，get方法，set方法，cas的方法。

并且实现了独占锁（reentrantlock）、共享锁(countdownlatch等)。

设计模式使用了模板方法模式，模板方法需要子类同步器自己去实现，比方说tryacquire、tryrelease等等呢个

AQS同步队列，获得锁的节点是header节点，当释放锁，唤醒的是header节点的next节点。

Condition.await：增加一个node到队列中，释放锁，唤醒header节点的next节点。

condition.signal：拿到firstwaiter，从condition队列移到AQS队列，跟正常获得锁释放说同样的逻辑。

Countdownlatch：countdown到0之后会唤醒AQS中的线程（所有线程），线程await会进入到AQS等待队列中。

chm的优秀设计：1、addcount，将当前map的元素数量加1。并发低的情况下，采用CAS操作，当一旦CAS失败，也就是说并发上去了，他会初始化一个数组，把每个线程映射到数组里面，进行增加操作。总和就是每个数组元素value之和。采用分片的方式，进行增加操作。2、多线程扩容，为每个线程分配其扩容的桶空间区域。高低位的设计，不需要在每次扩容的时候重新计算hash，极大的提升了效率。

map在JDK1.7和1.8的区别：1.7采用了分段锁的概念，缩小锁粒度来提升并发性能，把一个map分成16个段，通过key的hash值映射到某个段，对段进行加锁，理论上可以支持16个线程并发写入。1.8取消了分段锁的设计，直接对桶元素node进行加锁，另外cas的方式，更加减少了锁的粒度；引入红黑树、CAS操作来确保node的一些操作原子性，这种方式代替了锁。并发扩容，线程帮助扩容，设计了MOVED的状态。