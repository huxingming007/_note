### 为什么要用dubbo

单体架构演变成分布式架构，多了一个服务之间的远程通信，不管是SOA还是微服务，他们本质上都是对于业务服务的提炼和复用。远程通信这个领域其实有很多技术：java的RMI、webservice、hessian、dubbo、thrift等RPC框架。服务与服务之间的调用无非是跨进程通信而已，我们可以使用socket来实现通信，我们也可以用NIO来实现高性能通信。那我们为什么使用RPC框架呢（DUBBO）？我认为dubbo是一个生态。如果我们自己开发一个网络通信，需要考虑到：

1. 底层网络通信协议的处理
2. 序列化和反序列化的处理工作
3. 服务链路变长后，如何实现对服务链路的跟踪和监控
4. 服务的大规模集群下，需要依赖第三方注册中心来实现服务的发现和服务的感知问题
5. 服务通信之间的异常，需要一种保护机制防止一个节点故障引发大规模的系统故障，所以需要容错机制
6. 服务的大规模集群下，需要负载均衡实现请求分发

dubbo是一个服务治理框架，针对大规模服务化以后，服务之间的路由、负载均衡、容错机制、服务降级这些问题的解决方案。

### 注册中心

dubbo提供了许多注册中心的支持：consul、sofa、Redis、multicast、zookeeper等等。使用最多的还是zookeeper。

使用ZK作为注册中心，需要依赖jar包，curator。

#### 多注册中心的支持

~~~java
<dubbo:registry address="zookeeper://192.168.13.102:2181" id="registryCenter1"/>
<dubbo:registry address="zookeeper://192.168.13.102:2181" id="registryCenter2"/>
~~~

### dubbo的多协议支持

~~~java
<dubbo:protocol name="dubbo" port="20880" /> // 默认情况下是使用dubbo协议
~~~

支持webservice协议，需要添加相应的jar包

~~~java
<!-- 用 dubbo 协议在 20880 端口暴露服务 -->
<dubbo:protocol name="dubbo" port="20880" />
<dubbo:protocol name="webservice" port="8080" server="jetty"/>
<!-- 声明需要暴露的服务接口 -->
<dubbo:service interface="com.gupaoedu.practice.LoginService" registry="registryCenter1"
    ref="loginService" protocol="dubbo,webservice" />
  
~~~

另外还可以支持REST协议，当然还要添加相应的jar包。

~~~java
<dubbo:protocol name="rest" port="8888" server="jetty"/>
~~~

### dubbo监控平台安装

Dubbo 的监控平台也做了更新，不过目前的功能还没有完善， 在这个网站上下载 Dubbo-Admin 的包: https://github.com/apache/dubbo-admin
1. 修改dubbo-admin-server/src/main/resources/application.properties 中的配置信息
2. mvn clean package 进行构建
3. mvn –projects dubbo-admin-server spring-boot:run
4. 访问 localhost:8080

### dubbo的终端操作方法

之前有个同学问我，如果不看 zookeeper、也不看监控平台，如何知道这个 服务是否正常呢?
Dubbo 里面提供了一种基于终端操作的方法来实现服务治理
使用 telnet localhost 20880 连接到服务对应的端口
常见命令
ls
ls: 显示服务列表
ls -l: 显示服务详细信息列表
ls XxxService: 显示服务的方法列表
ls -l XxxService: 显示服务的方法详细信息列表 ps
ps: 显示服务端口列表
ps -l: 显示服务地址列表
ps 20880: 显示端口上的连接信息
ps -l 20880: 显示端口上的连接详细信息
cd

cd XxxService: 改变缺省服务，当设置了缺省服务，凡是需要输入服务名作 为参数的命令，都可以省略服务参数
cd /: 取消缺省服务
pwd
pwd: 显示当前缺省服务 count
count XxxService: 统计 1 次服务任意方法的调用情况
count XxxService 10: 统计 10 次服务任意方法的调用情况
count XxxService xxxMethod: 统计 1 次服务方法的调用情况 count XxxService xxxMethod 10: 统计 10 次服务方法的调用情况

### 负载均衡

当服务端存在多个节点的集群时，zookeeper上会维护不同集群节点，对于客户端而言，他需要一种负载均衡机制来实现目标服务的请求负载。

~~~java
// roundrobin、random、leastactive、consistenthash
<dubbo:service interface="..." loadbalance="roundrobin" /> // 服务端配置
<dubbo:reference interface="..." loadbalance="roundrobin" />// 客户端配置
// 注解的配置
@Service(loanbalance="roundrobin")
@Reference(loanbalance="roundrobin")
~~~

#### RandomLoanBalance

权重随机算法，根据权重值进行随机负载。dubbo缺省的负载方式。

算法：假如servers=[A,B,C]，对应的权重[5,3,2]，权重总和为10。[0,5)属于服务器A，[5,8)属于服务器B，[8,10)属于服务器C，接下来通过随机数生成一个范围在[0,10)之间的随机数，然后计算这个随机数会落在哪个区间上。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。

~~~java
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    private final Random random = new Random();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();
        int totalWeight = 0;
        boolean sameWeight = true;
        // 下面这个循环有两个作用，第一是计算总权重 totalWeight，
        // 第二是检测每个服务提供者的权重是否相同
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // 累加权重
            totalWeight += weight;
            // 检测当前服务提供者的权重与上一个服务提供者的权重是否相同，
            // 不相同的话，则将 sameWeight 置为 false。
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        
        // 下面的 if 分支主要用于获取随机数，并计算随机数落在哪个区间上
        if (totalWeight > 0 && !sameWeight) {
            // 随机获取一个 [0, totalWeight) 区间内的数字
            int offset = random.nextInt(totalWeight);
            // 循环让 offset 数减去服务提供者权重值，当 offset 小于0时，返回相应的 Invoker。
            // 举例说明一下，我们有 servers = [A, B, C]，weights = [5, 3, 2]，offset = 7。
            // 第一次循环，offset - 5 = 2 > 0，即 offset > 5，
            // 表明其不会落在服务器 A 对应的区间上。
            // 第二次循环，offset - 3 = -1 < 0，即 5 < offset < 8，
            // 表明其会落在服务器 B 对应的区间上
            for (int i = 0; i < length; i++) {
                // 让随机值 offset 减去权重值
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    // 返回相应的 Invoker
                    return invokers.get(i);
                }
            }
        }
        
        // 如果所有服务提供者权重值相同，此时直接随机返回一个即可
        return invokers.get(random.nextInt(length));
    }
}
~~~

#### LeastActiveLoadBalance

最小活跃数负载均衡。活跃调用数越小，表明该服务提供者效率越高，单位时间内可以处理更多的请求。

算法：每个服务都会对应一个活跃数active，初始值为0，每收到一个请求，active加1，完成请求active减1。当服务运行一段时间后，性能好的服务处理请求的速度更快，因此活跃数下降的也越快，此时这样子的服务理应优先获取到新的请求，这就是最小活跃数负载均衡算法的基本思想。另外还引入了权重值，当两个服务的active相等，权重值越高，获得新请求的概率就越大。

源码：略。比较简单。

1. 遍历invokers列表，寻找活跃数最小的Invoker
2. 如果有多个Invoker具有相同的最小活跃数，此时记录这些Invoker在invokers集合中的下标，并累加他们的权重，比较他们的权重值是否相等
3. 如果只有一个Invoker具有最小的活跃数，此时直接返回该Invoker即可
4. 如果有多个Invoker具有最小活跃数，且他们的权重不相等，此时处理方式和RandomLoadBalance一致
5. 如果有多个Invoker具有最小活跃数，但他们的权重相等，此时随机返回一个即可

#### ConsistentHashLoadBalance

[参考](http://dubbo.apache.org/zh-cn/docs/source_code_guide/loadbalance.html)

#### RoundRobinLoadBalance

加权轮询负载均衡。简单的轮询实现的缺点是等量的请求分配给性能较差的服务器，显然是不合理的。因此这个时候我们需要对轮询过程进行加权。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。

### 集群容错

[dubbo官方文档](http://dubbo.apache.org/zh-cn/docs/source_code_guide/cluster.html)

~~~java
@Service(loadbalance = "random", cluster = "failsafe") // 消费端也是可以配置的
~~~

#### Failover Cluster

缺省。失败自动切换，当出现失败，重试其他服务器。通常用于读操作，但重试会带来更长的延迟。可通过retries="2"来设置重试次数（不含第一次）。

~~~java
        // 省略一部分代码：比方说获得重试次数 
        // 循环调用，失败重试
        for (int i = 0; i < len; i++) {
            if (i > 0) {
                checkWhetherDestroyed();
                // 在进行重试前重新列举 Invoker，这样做的好处是，如果某个服务挂了，
                // 通过调用 list 可得到最新可用的 Invoker 列表
                copyinvokers = list(invocation);
                // 对 copyinvokers 进行判空检查
                checkInvokers(copyinvokers, invocation);
            }
            // 通过负载均衡选择 Invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            // 添加到 invoker 到 invoked 列表中，下次会从invoked剔除，因为要重试其他服务
            invoked.add(invoker);
            // 设置 invoked 到 RPC 上下文中
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 调用目标 Invoker 的 invoke 方法
                Result result = invoker.invoke(invocation);
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
       // 后面的代码省略，如有需要，请看dubbo的官方文档！
~~~



#### Failfast Cluster

快速失败，只发起一次调用，失败立即报错。

通常用于非幂等性的写操作，比如新增记录。

~~~java
@Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0,
                    "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName()
                            + " select from all providers " + invokers + " for service " + getInterface().getName()
                            + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost()
                            + " use dubbo version " + Version.getVersion()
                            + ", but no luck to perform the invocation. Last error is: " + e.getMessage(),
                    e.getCause() != null ? e.getCause() : e);
        }
    }
~~~

#### Failsafe Cluster

失败安全，出现异常时，直接忽略，仅会打印异常，而不会抛出异常。

通常用于写入审计日志等操作。

~~~java
@Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }
~~~

#### Failback Cluster

失败自动恢复，后台记录失败请求，**定时重发**。返回一个空结果给服务提供者。

通常用于消息通知操作。

~~~java
    @Override
    protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        Invoker<T> invoker = null;
        try {
            checkInvokers(invokers, invocation);
            invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                    + e.getMessage() + ", ", e);
            // 重点：定时重发的逻辑
            addFailed(loadbalance, invocation, invokers, invoker);
            // 返回一个空
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }

private void addFailed(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker) {
        if (failTimer == null) {
            synchronized (this) {
                if (failTimer == null) {
                    failTimer = new HashedWheelTimer(// 构建一个时间轮
                            new NamedThreadFactory("failback-cluster-timer", true),
                            1,
                            TimeUnit.SECONDS, 32, failbackTasks);
                }
            }
        }
        
        RetryTimerTask retryTimerTask = new RetryTimerTask(loadbalance, invocation, invokers, lastInvoker, retries, RETRY_FAILED_PERIOD);
        try {// RETRY_FAILED_PERIOD默认5秒，即5秒后执行retryTimerTask中的run方法
            failTimer.newTimeout(retryTimerTask, RETRY_FAILED_PERIOD, TimeUnit.SECONDS);
        } catch (Throwable e) {
            logger.error("Failback background works error,invocation->" + invocation + ", exception: " + e.getMessage());
        }
    }

        @Override
        public void run(Timeout timeout) {
            try {
                Invoker<T> retryInvoker = select(loadbalance, invocation, invokers, Collections.singletonList(lastInvoker));
                lastInvoker = retryInvoker;
                retryInvoker.invoke(invocation);
            } catch (Throwable e) {
                logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
                if ((++retryTimes) >= retries) { // 超过重试次数，就不再定时重发
                    logger.error("Failed retry times exceed threshold (" + retries + "), We have to abandon, invocation->" + invocation);
                } else {
                    rePut(timeout);// 继续定时重发（5秒后继续执行）
                }
            }
        }

~~~



#### Forking Cluster

并行调用多个服务器，只要一个成功即返回。

通常用于实时性要求高的读操作，但需要浪费更多服务资源。

可通过forks="2"来设置最大并行数。

~~~java
// 使用的时候要注意：系统资源有可能被耗尽
private final ExecutorService executor = Executors.newCachedThreadPool(
            new NamedInternalThreadFactory("forking-cluster-timer", true));
for (final Invoker<T> invoker : selected) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            Result result = invoker.invoke(invocation);
                            ref.offer(result);
                        } catch (Throwable e) {
                            int value = count.incrementAndGet();
                            if (value >= selected.size()) {
                                ref.offer(e);
                            }
                        }
                    }
                });
            }
~~~



#### Broadcast Cluster

广播调用所有提供者，逐个调用，任意一台报错则报错。

通常用于通知所有提供者更新缓存或日志等本地资源信息。

~~~java
    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        RpcContext.getContext().setInvokers((List) invokers);
        RpcException exception = null;
        Result result = null;
        for (Invoker<T> invoker : invokers) {// 逐个调用
            try {
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e);
                logger.warn(e.getMessage(), e);
            }
        }
        if (exception != null) { // 任意一台报错则报错
            throw exception;
        }
        return result;
    }
~~~

#### 最佳实践

查询语句建议使用默认的 "failover"，而增删改建议使用"failfast"或者"failover retries=0"，防止出现数据重复添加等等其它问题。

### 服务降级

概念：举个例子

1. 对一些非核心服务进行人工降级，在大促之前通过降级开关关闭那些推荐内容、评价等对主流程没有影响的功能；
2. 故障降级，比如调用远程服务挂了，网络故障、或者PRC服务返回异常。那么可以直接降级，降级的方案比如设置默认值、采用兜底数据（系统推荐的广告挂了，可以提前准备静态页面做返回）等等；
3. 限流降级，比如秒杀这种流量比较集中的情况下，可以采用限流来限制流量，当达到阈值时，后续请求被降级，比如进入排队页面，比如跳转到错误页等等；

```java
public class SayHelloServiceMock implements ISayHelloService {
    @Override
    public String sayHello() {
        return "服务端异常，被降级，返回兜底数据。。。。";
    }
}
```

```java
@Reference(loadbalance = "roundrobin", timeout = 10000000, cluster = "failfast", 
           check = false, // 关闭启动时检查，sayHelloService服务没有启动，该服务也能启动，比如，测试时有些服务不关心，或者出现了循环依赖。默认为true
           mock = "com.xavier.dubbo.dubboclient.SayHelloServiceMock" // 降级配置
           )
private ISayHelloService sayHelloService;
```

### 多版本支持

当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间 不引用。
可以按照以下的步骤进行版本迁移:
1. 在低压力时间段，先升级一半提供者为新版本
2. 再将所有消费者升级为新版本
3. 然后将剩下的一半提供者升级为新版本

```java
@Reference(loadbalance = "roundrobin", timeout = 10000000, cluster = "failfast",
        check = false,
        mock = "com.xavier.dubbo.dubboclient.SayHelloServiceMock",
        version = "1.0.1")
```

```java
@Service(loadbalance = "random", timeout = 50000, cluster = "failover", version = "1.0.1")
```

### dubbo新功能

#### 配置中心

简单来说就是把dubbo.properties中的属性进行集中存储，存储在其他的服务器上。目前 Dubbo 能支持的配置中心有:apollo、nacos、zookeeper。

打开dubbo admin，以下使用了zookeeper作为配置中心。

![](http://ww1.sinaimg.cn/large/006y8mN6ly1g6m18ei0myj315c0u0ac8.jpg)

> 虽然使用了配置中心 但是项目的配置必须需要的（保证可靠性） 以配置中心为主（配置中心的优先级高）

![数据存储到了zk](http://ww2.sinaimg.cn/large/006y8mN6ly1g6m1dgj47tj30vi0jmq4q.jpg)

#### 元数据中心

目前支持Redis和zookeeper。官方推荐Redis。在dubbo2.7之前，所有的配置信息，比如服务接口名称、重试次数、负载均衡策略、容错策略、版本号等等，所有的参数都是基于URL形式配置在zookeeper上的。这种方式会造成一些问题：

1. URL内容过多，导致数据存储空间增大；
2. URL需要涉及到网络传输，数据量过大会造成网络传输慢；
3. 网络传输慢，会造成服务地址感知的延迟变大，影响服务的正常响应；

服务提供者这边的配置参数有 30 多个，有一半是不需要作为注册中心进行存储和粗暗 地的。而消费者这边可配置的参数有 25 个以上，只有个别是需要传递到注册中心的。 所以，在 Dubbo2.7 中对元数据进行了改造，简单来说，就是把属于服务治理的数据 发布到注册中心，其他的配置数据统一发布到元数据中心。这样一来大大降低了注册 中心的负载。

```java
#元数据中心配置
dubbo.metadata-report.address=zookeeper://10.200.133.10:2181
dubbo.registry.simplified=true
#注册到注册中心的URL是否采用精简模式的 (与低版本兼容)
```

配置了元数据之后，注册中心的存储变化如下：

~~~java
dubbo%3A%2F%2F10.200.20.231%3A20882%2Fcom.xavier.dubbo.api.ISayHelloService%3F
  application%3Dspringboot-dubbo-provider%26cluster%3Dfailover%26deprecated%3Dfalse%26
  dubbo%3D2.0.2%26release%3D2.7.2%26timeout%3D50000%26timestamp%3D1567470284078%26version%3D
  1.0.1
~~~

没有配置元数据

~~~java
dubbo%3A%2F%2F10.200.20.231%3A20880%2Fcom.xavier.dubbo.api.ISayHelloService%3F
  anyhost%3Dtrue%26application%3Dspringboot-dubbo-provider%26bean.name%3DServiceBean%3A
  com.xavier.dubbo.api.ISayHelloService%3A1.0.1%26cluster%3Dfailover%26deprecated%3Dfalse%26
  dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3D
  com.xavier.dubbo.api.ISayHelloService%26methods%3DsayHello%26pid%3D47107%26
  register%3Dtrue%26release%3D2.7.2%26revision%3D1.0.1%26side%3Dprovider%26
  timeout%3D50000%26timestamp%3D1567472779619%26version%3D1.0.1
~~~

配置了元数据之后，其他参数数据存储的位置（使用zk作为元数据中心）

~~~java
[zk: localhost:2181(CONNECTED) 30] get /dubbo/metadata/com.xavier.dubbo.api.ISayHelloService/provider/springboot-dubbo-provider
{"parameters":{"cluster":"failover","side":"provider","release":"2.7.2","methods":"sayHello",
"deprecated":"false","dubbo":"2.0.2","interface":"com.xavier.dubbo.api.ISayHelloService",
"generic":"false","timeout":"50000","qos-enable":"false",
"application":"springboot-dubbo-provider","dynamic":"true","register":"true",
"bean.name":"ServiceBean:com.xavier.dubbo.api.ISayHelloService","anyhost":"true"},
"canonicalName":"com.xavier.dubbo.api.ISayHelloService",
"codeSource":"file:/Users/huxingming/Documents/_my_code/_practice_demo/dubbo-springboot/dubbo-api/target/classes/","methods":[{"name":"sayHello","parameterTypes":[],"returnType":"java.lang.String"}],"types":[{"type":"java.lang.String","properties":{"value":{"type":"char[]"},"hash":{"type":"int"}}},{"type":"int"},{"type":"char"}]}
~~~

### dubbo发布简单流程

开始于spring容器初始化后，发布刷新事件，入口方法是ServiceBean的onApplicationEvent方法。

1. 创造一个Invoker；
2. 把Invoker保存到exportMap中；
3. 把dubbo协议的URL地址注册到注册中心；

#### invoker

Invoker是dubbo领域模型中非常重要的一个概念，和extensionloader的重要性是一样的，如果Invoker没有搞懂，那么不算是看懂了dubbo的源码。我们继续回到serviceConfig中的export的代码，这段代码是还没有分析过的。以这个作为入口来分析我们前面export出去的Invoker到底是啥东西。

~~~java
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader
                   (ProxyFactory.class).getAdaptiveExtension();

// 生成Invoker，其实是一个代理
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, 
                                             registryURL.addParameterAndEncoded(EXPORT_KEY, 
                                             url.toFullString()));
// 进行包装
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
Exporter<?> exporter = protocol.export(wrapperInvoker);
~~~

proxyFactory是一个spi扩展点，并且默认的扩展实现是javassit，这个接口中有三个方法，并且都是加了@Adaptive的自适应扩展点。所以调用getInvoker方法，应该返回一个ProxyFactory$Adaptive。

```java
@SPI("javassist")
public interface ProxyFactory {
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```

ProxyFactory$Adaptive

- 通过ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);获得了一个指定名称的扩展点；
- 在dubbo-rpc-api/resources/META-INF/com.alibaba.dubbo.rpc.ProxyFactory中，定义了 javassis=JavassisProxyFactory
- 调用JavassistProxyFactory的getInvoker方法

~~~java
// 其他代码省略
public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0,
java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws
org.apache.dubbo.rpc.RpcException {
        if (arg2 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if(extName == null) throw new IllegalStateException("Failed to get
extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString()
+ ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension =
(org.apache.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(org.apache
.dubbo.rpc.ProxyFactory.class).getExtension(extName);
return extension.getInvoker(arg0, arg1, arg2);
~~~

#### JavassistProxyFactory.getInvoker

javassist是一个动态类库，用来实现动态代理的。

proxy：接口的实现：com.xxx.SayHelloServiceImpl

Type：接口全称：com.xxx.ISayHelloService

Url：协议地址：registry://.....

~~~java
    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
~~~

#### javassist生成的动态代理代码

通过断点的方式进入wrapper258行，在wrapper.getWrapper中的makeWrapper，会创建一个动态代理，核心的方法InvokerMethod代码如下

~~~java
 public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws
java.lang.reflect.InvocationTargetException {
        com.gupaoedu.dubbo.practice.ISayHelloService w;
        try {
            w = ((com.gupaoedu.dubbo.practice.ISayHelloService) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        try {
            if ("sayHello".equals($2) && $3.length == 1) {
                return ($w) w.sayHello((java.lang.String) $4[0]);
            }
        } catch (Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
}
        throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not
found method \"" + $2 + "\" in class
com.gupaoedu.dubbo.practice.ISayHelloService.");
~~~

构建好了代理类之后，返回一个AbstractproxyInvoker,并且它实现了doInvoke方法，这个地方似乎看 到了dubbo消费者调用过来的时候触发的影子，因为wrapper.invokeMethod本质上就是触发上面动态 代理类的方法doInvoke。

~~~java
return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
~~~

#### 总结

Invoker本质上应该是一个代理，经过层层包装最终进行了发布。当消费者发起请求的时候，会获得这个Invoker进行调用。最终发布出去的Invoker，也不是单纯的一个代理，也是经过多层包装。

InvokerDelegate(DelegateProviderMetaDataInvoker(AbstractProxyInvoker()))

### dubbo消费简单流程

自己思考下：

1. 生成远程服务的代理；
2. 获得目标服务的URL地址；
3. 实现远程网络通信；
4. 实现负载均衡；
5. 实现集群容错；

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g6vcu22m7dj310u0ksgmj.jpg)

消费端的代码解析是从下面这段代码开始的

~~~java
<dubbo:reference id="xxxService" interface="xxx.xxx.Service"/>
~~~

注解的方式的初始化入口是

~~~java
 ReferenceAnnotationBeanPostProcessor->ReferenceBeanInvocationHandler.init- >
   ReferenceConfig.get() 获得一个远程代理类
~~~

####  ReferenceConfig.get()

~~~java
    public synchronized T get() {
        checkAndUpdateSubConfigs();

        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        if (ref == null) {
            init();// 重点
        }
        return ref;
    }

    private void init(){
      // ....
      ref = createProxy(map);
      // ....
    }
~~~

#### createProxy

代码比较长，但是逻辑相对比较清晰
1. 判断是否为本地调用，如果是则使用injvm协议进行调用
2. 判断是否为点对点调用，如果是则把url保存到urls集合中，如果url为1，进入步骤4，如果urls>1
，则执行5
3. 如果是配置了注册中心，遍历注册中心，把url添加到urls集合，url为1，进入步骤4，如果urls>1
，则执行5
4. 直连构建invoker
5. 构建invokers集合，通过cluster合并多个invoker
6. 最后调用 ProxyFactory 生成代理类

```java
// 其他代码省略，重点看这一段，REF_PROTOCOL是自适应扩展点，此时的场景是被包装过的RegistryProtocol
invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
```

#### RegistryProtocol.refer

~~~java
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        // 根据配置的协议，生成注册中心的URL：zookeeper://10.200.133......
        url = URLBuilder.from(url)
                .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
                .removeParameter(REGISTRY_KEY)
                .build();
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }
        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
~~~

#### doRefer

- 构建一个RegistryDirectory
- 构建一个consumer://协议的地址注册到注册中心
- 订阅zookeeper中的节点变化
- 调用cluster.join方法，生成一个Invoker

```java
// 重点看这段代码
Invoker invoker = cluster.join(directory);
```

```java
// cluster是通过set方法注入进去的
public void setCluster(Cluster cluster) {
    this.cluster = cluster;
}
```

```java
@SPI(FailoverCluster.NAME)
public interface Cluster {
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

}
```

~~~java
// 通过 extName 获得真正的实现类，默认是failover，比如getExtension("failfast")，理论上应该返回FailFastCluster
// 但实际上，这里做了包装，MockClusterWrapper(FailFastCluster)
public class Cluster$Adaptive implements org.apache.dubbo.rpc.cluster.Cluster {
    public org.apache.dubbo.rpc.Invoker
join(org.apache.dubbo.rpc.cluster.Directory arg0) throws
org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new
IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument ==
null");
        if (arg0.getUrl() == null) throw new
IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument
getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("cluster", "failover");
        if(extName == null) throw new IllegalStateException("Failed to get
extension (org.apache.dubbo.rpc.cluster.Cluster) name from url (" +
url.toString() + ") use keys([cluster])");
        org.apache.dubbo.rpc.cluster.Cluster extension =
(org.apache.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(org.apa
che.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
        return extension.join(arg0);
    }
}
~~~

~~~java
mock=org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterWrapper // 会对每一个cluster进行一次包装
failover=org.apache.dubbo.rpc.cluster.support.FailoverCluster
failfast=org.apache.dubbo.rpc.cluster.support.FailfastCluster
failsafe=org.apache.dubbo.rpc.cluster.support.FailsafeCluster
failback=org.apache.dubbo.rpc.cluster.support.FailbackCluster
forking=org.apache.dubbo.rpc.cluster.support.ForkingCluster
available=org.apache.dubbo.rpc.cluster.support.AvailableCluster
mergeable=org.apache.dubbo.rpc.cluster.support.MergeableCluster
broadcast=org.apache.dubbo.rpc.cluster.support.BroadcastCluster
registryaware=org.apache.dubbo.rpc.cluster.support.RegistryAwareCluster
~~~

通过debug的方式得到Invoker，验证了上面的理论。MockClusterWrapper(FailFastCluster(directory))

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g6vdp0vwjmj3212082gn3.jpg)

#### proxyFactory.getProxy

拿到Invoker之后，会调用获得一个动态代理类

```java
return (T) PROXY_FACTORY.getProxy(invoker);
```

这里的proxyFactory是一个自适应扩展点，所以会进入下面的方法。

#### JavassistProxyFactory.getProxy

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

Proxy.getProxy这个方法中会生成一个动态代理类，通过debug的形式看到动态代理类的原貌。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g6ve0nzspnj32280eutb0.jpg)

~~~java
// @Reference注入的一个对象实例本质上就是一个动态代理类
public java.lang.String sayHello(){
  Object[] args = new Object[0]; 
  // handler即InvokerInvocationHandler
  Object ret = handler.invoke(this, methods[0], args); 
  return (java.lang.String)ret;
}
~~~

InvokerInvocationHandler的Invoker方法，最终调用生成Invoker对象的Invoker方法，后续进行分析。

~~~java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
~~~

讲到这里，Invoker生成了，Invoker的动态代理类也生成了！接下去非常简洁的讲下，网络连接的建立。

回到RegistryProtocol.refer -> doRefer -> directory.subscribe

~~~java
    public void subscribe(URL url) {
        setConsumerUrl(url);// 设置consumer的URL
        // 把当前RegistryDirectory作为listener，去监听zk上节点的变化
        CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
        serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
        registry.subscribe(url, this);// 订阅 -> 这里的registry是zookeeperRegistry
    }
~~~

这里的registry 是ZookeeperRegistry ，会去监听并获取路径下面的节点。监听的路径是:

/dubbo/org.apache.dubbo.demo.DemoService/providers、/dubbo/org.apache.dubbo.demo.DemoService/configurators、/dubbo/org.apache.dubbo.demo.DemoService/routers节点下面的子节点变动。

ZookeeperRegistry没有实现subscribe方法，他的父类FailbackRegistry有。

```java
public void subscribe(URL url, NotifyListener listener) {
    super.subscribe(url, listener);
    removeFailedSubscribed(url, listener);// 移除失效的listener
    try {
        // 进行订阅
        doSubscribe(url, listener);
    } catch (Exception e) {
        // catch里边的逻辑就是check为true要抛出异常
        Throwable t = e;

        List<URL> urls = getCacheUrls(url);
        if (CollectionUtils.isNotEmpty(urls)) {
            notify(url, listener, urls);
            logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
        } else {
            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true);
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
        }

        // Record a failed registration request to a failed list, retry regularly
        addFailedSubscribed(url, listener);
    }
}
```

ZookeeperRegistry.doSubscribe这个方法是订阅，逻辑实现比较多。。。。。

