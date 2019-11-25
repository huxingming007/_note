### dubbo启动预热

大概思想是，当一个服务刚刚启动，权重分配的低一点，避免刚启动收到大流量的请求负载，导致服务发生一些问题，使用这个预热来进行缓冲，JVM也是带有预热的过程。这样看来，当项目刚刚启动，权重都会是1，并且随着时间的慢慢的增加到真实权重，当过了系统的预热时间（十分钟），则进入正常的调用逻辑。

~~~java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), WEIGHT_KEY, DEFAULT_WEIGHT);
        if (weight > 0) {
            long timestamp = invoker.getUrl().getParameter(REMOTE_TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
                // 服务启动的时间
                int uptime = (int) (System.currentTimeMillis() - timestamp);
                // 预热时间，默认为10分钟
                int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
                if (uptime > 0 && uptime < warmup) {
                    weight = calculateWarmupWeight(uptime, warmup, weight);
                }
            }
        }
        return weight >= 0 ? weight : 0;
    }

    static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
        return ww < 1 ? 1 : (ww > weight ? weight : ww);
    }
~~~

[参考](https://blog.csdn.net/qq_31748587/article/details/84958330)

### 负载均衡代码设计

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```

```java
/*
 定义一个抽象类，把各个负载均衡实现类的公共方法进行抽离。
 */
package org.apache.dubbo.rpc.cluster.loadbalance;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.utils.CollectionUtils;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.cluster.LoadBalance;

import java.util.List;

import static org.apache.dubbo.rpc.cluster.Constants.DEFAULT_WARMUP;
import static org.apache.dubbo.rpc.cluster.Constants.DEFAULT_WEIGHT;
import static org.apache.dubbo.rpc.cluster.Constants.REMOTE_TIMESTAMP_KEY;
import static org.apache.dubbo.rpc.cluster.Constants.WARMUP_KEY;
import static org.apache.dubbo.rpc.cluster.Constants.WEIGHT_KEY;
public abstract class AbstractLoadBalance implements LoadBalance {
    static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
        return ww < 1 ? 1 : (ww > weight ? weight : ww);
    }

    // 外部只需要调用此方法即可  
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        return doSelect(invokers, url, invocation);
    }
    // 各个负载均衡实现类，自己实现！
    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
    protected int getWeight(Invoker<?> invoker, Invocation invocation) {
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), WEIGHT_KEY, DEFAULT_WEIGHT);
        if (weight > 0) {
            long timestamp = invoker.getUrl().getParameter(REMOTE_TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
                int uptime = (int) (System.currentTimeMillis() - timestamp);
                int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
                if (uptime > 0 && uptime < warmup) {
                    weight = calculateWarmupWeight(uptime, warmup, weight);
                }
            }
        }
        return weight >= 0 ? weight : 0;
    }

}
```

~~~java
// 实现类。其他就不举例了
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();
        // Every invoker has the same weight?
        boolean sameWeight = true;
        // the weight of every invokers
        int[] weights = new int[length];
        // the first invoker's weight
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        // The sum of weights
        int totalWeight = firstWeight;
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // save for later use
            weights[i] = weight;
            // Sum
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
}
~~~

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g6lbt16lf0j30vk0r4wfi.jpg)

### 双重检查锁和putIfAbsent

dubbo里面使用了非常多的缓存设计，必然会有很多共享变量，dubbo是设计的呢？！

- 双重检查锁

```java
// ExtensionLoader实例存在hashmap缓存中，是共享的，使用双重检查锁
// dubbo使用目的不是延迟初始化，而是用户缓存的前提下，只要缓存中有的时候我们尽量去缓存中取，只在缓存中没有的时候，我们再去创建
public T getExtension(String name) {
    // 代码省略。。。。
    Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

- putIfAbsent

~~~java
// 静态常量作为缓存，ExtensionLoader实例必然会被其他线程共享，发生线程安全问题
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>();        

    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 代码省略。。。。。。
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            // 使用了CHM的putIfAbsent，就不用双重检查锁了，也算是一种优化
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
~~~

双重检查锁是线程不安全的，指令重排序导致的。[详细参考地址和解决方案](https://blog.csdn.net/wpb92/article/details/76714329)

dubbo也已经考虑到了

```java
public class Holder<T> {
    // 重点是这里，用volatile进行修饰，禁止指令重排序
    private volatile T value;
    public void set(T value) {
        this.value = value;
    }
    public T get() {
        return value;
    }
}
// 继续看上面那个例子
public T getExtension(String name) {
    // 代码省略。。。。
    Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) { // instance就是 private volatile T value;被volatile修饰，禁止指令重排序
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

### Wrapper

org.apache.dubbo.rpc.Protocol文件

~~~java
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
MyProtocol=org.apache.dubbo.rpc.protocol.MyProtocol
~~~

![](http://ww4.sinaimg.cn/large/006y8mN6ly1g6pk7qjxusj31u80fy0uh.jpg)

~~~java
// 第一步：加载wrapper，放到ConcurrentHashSet容器中
// 加载org.apache.dubbo.rpc.Protocol文件：clazz为文件中的value，比如说org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        // ......
        } else if (isWrapperClass(clazz)) {// 判断是否是wrapper
            // 重点
            cacheWrapperClass(clazz);
        } else {
           // ......
    }
    private boolean isWrapperClass(Class<?> clazz) {
        try {
            clazz.getConstructor(type);// type是org.apache.dubbo.rpc.Protocol
            return true;// 判断是否有构造函数（参数是type类型的）
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
    private void cacheWrapperClass(Class<?> clazz) {
        if (cachedWrapperClasses == null) {
            cachedWrapperClasses = new ConcurrentHashSet<>();
        }
        cachedWrapperClasses.add(clazz);// 添加到hashset容器中
    }
~~~

~~~java
// 第二步：构建完instance，对instance进行包装    
private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            // ---------包装开始--------
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                // 循环创建Wrapper实例
                for (Class<?> wrapperClass : wrapperClasses) {
                    // 把instance作为参数传给Wrapper的构造方法，并通过反射创建Wrapper实例
                    // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            // ---------包装结束--------
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
~~~

### Filter

默认情况下，会先执行原生的filter，再依次执行自定义filter，继而回溯到原点。

dubbo原生的filter定义在/META-INF/dubbo/internal/org.apache.dubbo.rpc.Filter文件中

~~~java
cache=org.apache.dubbo.cache.filter.CacheFilter
validation=org.apache.dubbo.validation.filter.ValidationFilter
echo=org.apache.dubbo.rpc.filter.EchoFilter
generic=org.apache.dubbo.rpc.filter.GenericFilter
genericimpl=org.apache.dubbo.rpc.filter.GenericImplFilter
token=org.apache.dubbo.rpc.filter.TokenFilter
accesslog=org.apache.dubbo.rpc.filter.AccessLogFilter
activelimit=org.apache.dubbo.rpc.filter.ActiveLimitFilter
classloader=org.apache.dubbo.rpc.filter.ClassLoaderFilter
context=org.apache.dubbo.rpc.filter.ContextFilter
consumercontext=org.apache.dubbo.rpc.filter.ConsumerContextFilter
exception=org.apache.dubbo.rpc.filter.ExceptionFilter
executelimit=org.apache.dubbo.rpc.filter.ExecuteLimitFilter
deprecated=org.apache.dubbo.rpc.filter.DeprecatedFilter
compatible=org.apache.dubbo.rpc.filter.CompatibleFilter
timeout=org.apache.dubbo.rpc.filter.TimeoutFilter
trace=org.apache.dubbo.rpc.protocol.dubbo.filter.TraceFilter
future=org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter
monitor=org.apache.dubbo.monitor.support.MonitorFilter
metrics=org.apache.dubbo.monitor.dubbo.MetricsFilter

~~~

~~~java
// dubbo原生filter举例
@Activate(group = {CONSUMER, PROVIDER}, value = VALIDATION_KEY, order = 10000)
public class ValidationFilter implements Filter {
~~~

1. @Activate是否是dubbo filter必须的，其上的group和order分别扮演什么角色？

   对于原生的filter，@Activate注解是必须的，其group用于provider/consumer的站队，而order值是filter顺序的依据，但是对于自定义filter而言，注解@Activate没被用到，其分组和顺序，完全由用户手工配置指定。**如果自定义filter添加了@Activate注解, 并指定了group了, 则这些自定义filter将升级为原生filter组。**

2. filter的顺序是否可以自己调整，如何实现？

   可以调整，通过“-”符号可以去除某些filter，而default代表默认激活的原生filter子链，通过重排default和自定义filter的顺序，达到实现顺序控制的目的。案例实战：

   ```java
   `<dubbo:reference filter=``"filter1,filter2"``/>`
   ```

   ~~~java
   // 执行顺序为filter1->filter2->原生filter
   <dubbo:reference filter="filter1,filter2,default"/>
   ~~~

   ~~~java
   // 执行顺序为filter1->原生filter->filter2，同时去掉原生的TokenFilter(token=org.apache.dubbo.rpc.filter.TokenFilter)
   <dubbo:service filter="filter1,default,filter2,-token"/>
   ~~~

源码解读：

入口是ProtocolFilterWrapper，上文提到过这个类。

~~~java
// 构建filter链
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        // 获得所有激活的filter（已经排好序了）
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {
                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }
                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }
                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }
                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        Result asyncResult;
                        try {
                            // filter指向的是当前节点，而传入的Invoker参数是其下一个节点
                            asyncResult = filter.invoke(next, invocation);
                        } catch (Exception e) {
                            // onError callback
                            if (filter instanceof ListenableFilter) {
                                Filter.Listener listener = ((ListenableFilter) filter).listener();
                                if (listener != null) {
                                    listener.onError(e, invoker, invocation);
                                }
                            }
                            throw e;
                        }
                        return asyncResult;
                    }
                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }
                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return new CallbackRegistrationInvoker<>(last, filters);
    }
~~~

~~~java
public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> exts = new ArrayList<>();
        List<String> names = values == null ? new ArrayList<>(0) : Arrays.asList(values);
        // names中不包含"-default"时，才进入if，才会加载原生的filter
        if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
            getExtensionClasses();
            for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Object activate = entry.getValue();

                String[] activateGroup, activateValue;

                if (activate instanceof Activate) {
                    activateGroup = ((Activate) activate).group();
                    activateValue = ((Activate) activate).value();
                } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                    activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                    activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
                } else {
                    continue;
                }
                if (isMatchGroup(group, activateGroup)) {
                    T ext = getExtension(name);
                    if (!names.contains(name)
                            // 剔除"-"开头的filter
                            && !names.contains(REMOVE_VALUE_PREFIX + name)
                            && isActive(activateValue, url)) {
                        exts.add(ext);
                    }
                }
            }
            exts.sort(ActivateComparator.COMPARATOR);
        }
        // 走到这一步dubbo原生的filter已经被添加完毕了，下面是用户自己处理扩展的filter
        List<T> usrs = new ArrayList<>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            // "-"开头的filter不会进入if，即不会添加
            if (!name.startsWith(REMOVE_VALUE_PREFIX)
                    && !names.contains(REMOVE_VALUE_PREFIX + name)) {
                if (DEFAULT_KEY.equals(name)) {
                    if (!usrs.isEmpty()) {
                        // 调整位置用，即usrs排在default（原生filter）之前，添加到exts
                        exts.addAll(0, usrs);
                        usrs.clear();
                    }
                } else {
                    T ext = getExtension(name);
                    // 添加
                    usrs.add(ext);
                }
            }
        }
        if (!usrs.isEmpty()) {
            // 把添加到exts
            exts.addAll(usrs);
        }
        return exts;
    }
~~~

```java
// 责任链的模式
@Activate(group = CommonConstants.CONSUMER)
public class MyFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        System.out.println("enter myfilter ..........");
        return invoker.invoke(invocation);
    }
}
```

**总结：**

这个设计精妙。通过简单的配置'-'可以手动剔除dubbo原生的filter，通过default代表dubbo原生的filter子链，通过配置指定从而实现filter链的顺序控制！值得拜读和学习。

### 健壮性

1. 监控中心宕掉不影响使用，只是丢失部分采样数据
2. 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
3. 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
4. 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
5. 服务提供者无状态，任意一台宕掉后，不影响使用
6. 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

