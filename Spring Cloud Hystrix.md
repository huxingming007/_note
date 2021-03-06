### 背景

一个单元出现故障，就很容易因依赖关系而引发故障的蔓延，最终导致整个系统瘫痪，Hystrix的出现就是为了避免故障在分布式系统中的蔓延。比方说，订单系统创建完订单后，会调用库存系统进行减库存的操作，此时库存系统因自身问题造成响应非常缓慢，会直接导致创建订单服务的线程被挂起，以等待库存系统的响应，用户因为请求库存失败而导致创建订单失败，若在高并发的情况下，挂起的线程得不到释放，占用CPU，占用内存，导致订单服务异常，故障就开始蔓延，其他依赖订单服务，都会慢慢变得异常。

### 解决方案

类似于电路中的保险丝，发生电路过载进行熔断。当被调用方服务出现故障，向调用方返回一个错误，而不是长时间的等待。

### 正文

Hystrix具有非常强大的功能，服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并及其监控。

#### 依赖隔离

和docker一样通过”**舱壁模式**“来实现线程池的隔离，给每一个依赖服务分配一个线程池，当这个依赖服务出现延迟过高的情况，也只是对此依赖服务的调用产生影响，而不会拖慢其他的依赖服务。优势如下：

1. 即使给依赖服务的线程池填满，也不会影响应用自身的其他部分
2. 我们可以对线程池进行监控，通过失败次数、延时、超时、拒绝等指标，来实时动态刷新属性
3. 同步依赖服务构建异步访问

线程池隔离会有性能开销，官方的数据，使用线程池隔离会有9ms的延迟，对于大多数的需求来说是微乎其微的，更何况为系统稳定性带来了巨大的提升。当然，我们可以使用信号量隔离，开销比线程池隔离小，但是它不能设置超时和异步访问，所以当在依赖服务是足够可靠的情况下才使用信号量。

#### 使用详解

1. 可以实现同步执行和异步执行（返回Future）。
2. 一般情况下对执行写操作的命令和批处理或者离线计算的命令，不去实现降级逻辑。
3. 默认情况下对异常HystrixBadRequestException以外的异常进行降级逻辑，当然可以进行配置。
4. HTTP相比于其他高性能的通信协议在速度上并不占用优势，高并发条件下可能会成为瓶颈，于是出现了请求缓存和请求合并两大功能。
5. Hystrix仪表盘，主要用来实时监控各项指标的信息。