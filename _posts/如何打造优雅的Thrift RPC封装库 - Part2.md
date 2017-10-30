title: 如何打造优雅的Thrift RPC封装库 - Part2
date: 2017-10-30 23:20:01
categories: rpc
tags: [rpc, thrift, zookeeper]
----------
上一篇我们介绍了如何一步步打造一个包含基础功能，支持服务注册、服务发现的Thrift RPC封装库，本文叙接上文，主要围绕以下几个方面来介绍：

* 负载均衡
* 连接池管理
* 客户端代理封装V2
* 统一监控

## 负载均衡
常用的负载均衡算法参见[常用负载均衡算法介绍](http://www.wxpjimmy.com/2015/10/30/%E5%B8%B8%E7%94%A8%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AE%97%E6%B3%95/)
所谓负载均衡，说白了就是根据下游多台机器的负载情况来动态调整流量分配。一个好的负载均衡算法要考虑到以下几种情况：
* 服务器异构，不同服务器处理能力不同
* 某一台或多台服务异常

代码示例结构如下：
{% asset_img Snip20171030_3.png [负载均衡代码结构] %}

我们定义负载均衡算法接口ILoadBalancer如下：

```
public interface ILoadBalancer {
    /**
     * choose one server instance
     * 
     * @param all  all available server instances
     * @param blacklist  server instance can't be connected during one request
     * @return
     */
    ServerInstance getServerInstance(List<ServerInstance> all, List<ServerInstance> blacklist);
}
```

其他几个类（RoundRobinLoadBalancer/CoHashLoadBalancer/PredictiveLoadBalancer）都是具体实现，分别代表round-robin算法、一致性hash算法以及根据下游负载以及处理能力动态调整算法。

## 连接池管理
目前应用的最普遍的连接池管理工具包是apache common pool，我们就选用最新的v2版本来作为我们的连接池管理工具。
具体maven依赖如下：

```
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-pool2</artifactId>
  <version>2.4.2</version>
</dependency>
```

连接池管理代码结构如下：
{% asset_img Snip20171030_4.png [连接池管理代码结构] %}
其中定义了两个接口IConnectionPool和IConnectionPoolFactory：

```
public interface IConnectionPool<K, V> {
    V borrowObject(K key);
    void returnObject(K key, V value);
    void discardObject(K key, V value);
    void shutdown();
}

public interface IConnectionPoolFactory<K, V> {
    /**
     * get a connection pool based on provided ifaceClazz 
     * 
     * @param ifaceClazz  (server class definition)
     * @return
     */
    IConnectionPool<K,V> getConnectionPool(Class<?> ifaceClazz);
}
```

ConnectionPoolConfig类提供了连接池的参数设置，这些参数都是Commons pool中指定的，我们选取了部分重要的参数开放给用户设定：
{% asset_img Snip20171030_7.png [连接池重要参数] %}


ConnectionKey类是common pool连接池管理中每个连接对应的key，它主要包含三个字段：
{% asset_img Snip20171030_5.png [连接池管理key对象] %}


ThriftConnectionKeyedPooledObjectFactory 继承自common pool的BaseKeyedPooledObjectFactory类，key是ConnectionKey，返回的value是TServiceClient，这个类主要提供具体创建连接的方法实现，具体如下：

```
public class ThriftConnectionKeyedPooledObjectFactory extends BaseKeyedPooledObjectFactory<ConnectionKey, TServiceClient> {

    public TServiceClient create(ConnectionKey key) throws Exception {
        //create client
        try {
            TSocket socket = new TSocket(key.getServerInstance().getIp(), key.getServerInstance().getPort(), key.getTimeout());
            TTransport transport = new TFramedTransport(socket);
            transport.open();
            TProtocol protocol = new TBinaryProtocol(transport);
            Constructor<?> cons = key.getiClientClass().getConstructor(TProtocol.class);
            TServiceClient res = (TServiceClient) cons.newInstance(protocol);
            return res;
        } catch (TTransportException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }

        return null;
    }

    public PooledObject<TServiceClient> wrap(TServiceClient value) {
        if (value == null)
            return null;
        return new DefaultPooledObject<TServiceClient>(value);
    }
}
```

ThriftConnectionPool是真正的Thrift连接池实现类，它依赖commons pool的对象池GenericKeyedObjectPool来管理连接，提供创建连接、归还连接、删除连接以及关闭连接池的功能。

```
public class ThriftConnectionPool implements IConnectionPool<ConnectionKey, TServiceClient> {
    private static Logger logger = LoggerFactory.getLogger(ThriftConnectionPool.class);

    private GenericKeyedObjectPool<ConnectionKey, TServiceClient> connectionPool;
    private ConnectionPoolConfig config;

    public ThriftConnectionPool(ConnectionPoolConfig config) {
        this.connectionPool = new GenericKeyedObjectPool<ConnectionKey, TServiceClient>(new ThriftConnectionKeyedPooledObjectFactory());

        this.config = config;
        this.connectionPool.setMaxTotal(config.getMaxTotal());
        this.connectionPool.setMaxTotalPerKey(config.getMaxTotalPerKey());
        this.connectionPool.setMaxIdlePerKey(config.getMaxIdlePerKey());
        this.connectionPool.setBlockWhenExhausted(config.isBlockWhenExhausted());
        this.connectionPool.setMaxWaitMillis(config.getMaxWaitTimeInMillis());
        this.connectionPool.setTestOnBorrow(config.isTestOnBorrow());
        this.connectionPool.setTestOnReturn(true);
        this.connectionPool.setTimeBetweenEvictionRunsMillis(config.getTimeBetweenEvictionRunsMillis());
    }

    public TServiceClient borrowObject(ConnectionKey key) {
        TServiceClient client = null;
        try {
            client = this.connectionPool.borrowObject(key);
        } catch (Exception e) {
            logger.error("borrow client with key={} failed, Exception: {}", key, e);
            return null;
        }
        return client;
    }

    public void returnObject(ConnectionKey key, TServiceClient client) {
        this.connectionPool.returnObject(key, client);
    }

    public void discardObject(ConnectionKey key, TServiceClient value) {
        try {
            this.connectionPool.invalidateObject(key, value);
        } catch (Exception e) {
           logger.warn("discard client failed! key={}, Exception={}", key, e);
        }
    }

    public void shutdown() {
        this.connectionPool.close();
    }
}
```

DefaultConnectionPoolFactory和PerServiceConnectionPoolFactory是连接池工厂的具体实现，分别代表的是统一连接池管理和分服务不同连接池管理，后者主要是为了减少不同服务依赖之间的互相影响。

## 客户端代理封装V2
好了，前面分别介绍了负载均衡和连接池管理，那么我们是如何将这些功能串起来的呢？   

我们先来回忆下第一篇中我们介绍的最简单封装，那种情况下我们是直接创建好了服务的Client对象直接返还给使用者，使用者直接使用这个Client来发起调用，逻辑很简单，也很好理解；但是当我们加入连接池的时候情况就不一样了，有了连接池，我们不仅要创建好Client，这个Client还应该是可以重用的，那么这种情况下如果我们只是简单提供给用户返还Client对象的方法就很容易出问题，你永远不要去猜测用户的使用习惯，最好的方法就是我们提供封装库来统一管理连接的创建/借用/归还/关闭。

那么我们该怎么做呢？

很显然，前一篇介绍的方法在这里很难再继续走下去，我们需要换个思路，如果我们来管理连接，那么就意味着用户是不需要关心具体调用使用的是哪个Client，只要我们保证在用户发起调用的时候能正确找到一个可用的连接，发起调用，调用成功后归还连接、调用失败关闭连接就好了。这里Java的代理机制就派上用场了，我们可以创建一个代理类来代理Client类，所有细节都封装在这个代理类中（具体Java代理的机制我们这里就不展开细说了，不熟悉的同学可以去baidu/google一下）。我们返还给客户端的实际上是这个代理类，因为这个代理类实现了跟Client对象完全一样的接口，对使用者来说是感受不到差别的。

我们看下代码结构：
{% asset_img Snip20171030_8.png [客户端代理V2代码结构] %}

ClientProxyConfigFactory和ClientProxyV1是第一篇中我们介绍的封装，后者这里只是改了个名字。
ServiceProxy还是第一篇中介绍的封装，这里没有做任何改动（后续改进会单独讲）。

我们重点看一下ThriftClientConfig、ThriftClientFactory以及ThriftClientProxy三个类：

* ThriftClientConfig是客户端配置类，可以用它指定要使用的LoadBalancer、ConnectionPoolFactory、ServiceDiscovery、timeout等配置；
* ThriftClientProxy是一个实现了InvocationHandler的代理类，他主要负责拦截用户发起的调用，根据要调用的服务找到一个可用的连接，然后通过反射发起调用，同时负责连接生命周期的维护；
* ThriftClientFactory是提供给用户使用的接口。

具体看下ThriftClientProxy类的实现：

```
class ThriftClientProxy implements InvocationHandler {
    private Class<?> ifaceClass;
    private Class<?> iclientClass;
    private ThriftClientConfig config;
    private IConnectionPool<ConnectionKey, TServiceClient> connectionPool;
    private ServiceDiscovery serviceDiscovery;

    public ThriftClientProxy(Class<?> ifaceClass, ThriftClientConfig config) {
        this.ifaceClass = ifaceClass;
        this.config = config;
        this.iclientClass = getIClientClass(ifaceClass);
        this.connectionPool = config.getConnectionPoolFactory().getConnectionPool(ifaceClass);

        serviceDiscovery = config.getServiceDiscovery();
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        IConfig properties = config.getConfigProvider().getConfig(ifaceClass);
        List<ServerInstance> invalidInstances = new ArrayList<ServerInstance>();
        while (true) {
            List<ServerInstance> serverInstances = serviceDiscovery.discoverService(ifaceClass);
            String loadbalancer = properties.getProperty(Constants.RPC_LOADBALANCER, null);
            ILoadBalancer loadBalancer = getLoadBalancer(loadbalancer);
            ServerInstance instance = loadBalancer.getServerInstance(serverInstances, invalidInstances);
            
            if (instance == null) {
               throw new RuntimeException("Can't find a valid server instance!");
            }

            ConnectionKey connectionKey = new ConnectionKey(iclientClass, instance, config.getTimeout());
            TServiceClient client = this.connectionPool.borrowObject(connectionKey);
            if (client == null) {
                if (config.isFailFast()) {
                    throw new RuntimeException("Can't borrow client from server: " + instance);
                } else {
                    invalidInstances.add(instance);
                    continue;
                }
            }
            boolean exceptionOccurred = false;
            try {
                return method.invoke(client, args);
            } catch (Exception e) {
                exceptionOccurred = true;
                this.connectionPool.discardObject(connectionKey, client);
                if (config.isFailFast()) {
                    throw new RuntimeException(e);
                } else {
                    invalidInstances.add(instance);
                    continue;
                }
            } finally {
                if (!exceptionOccurred) {
                    this.connectionPool.returnObject(connectionKey, client);
                }
            }
        }
    }
}
```

所有重要的逻辑都是invoke方法中，它支持两种模式：retry-until-succeed以及fail-fast。前者是当调用失败的时候会将当前的服务节点标记为不可用，然后重新选取一个节点发起调用直到成功为止；后者是一旦有失败立即终止，更适合于latency比较敏感的服务，比如广告。

我们再看下具体逻辑，首先通过serverdiscovery找到可用的所有节点，然后从中根据指定的Loadbalancer算法选取一个节点，接下来从连接池中借用/创建一个该节点对应的连接，并通过反射直接发起调用，调用成功则归还连接并返回结果；如果失败则关闭当前连接，并根据指定的模式来决定是否重试。

我们可以看到，这种模式非常灵活，所有服务发现、节点选取、连接池管理的逻辑都隐藏在代理类中，用户使用的时候只需要简单的调用ThriftClientFactory获取连接代理即可，ThriftClientFactory实现如下：

```
public class ThriftClientFactory {
    public static <T> T getClient(Class<T> ifaceClazz, ThriftClientConfig config) {
        ThriftClientProxy proxy = new ThriftClientProxy(ifaceClazz, config);

        T clientProxy = (T) Proxy.newProxyInstance(ifaceClazz.getClassLoader(),
                new Class<?>[]{ifaceClazz, TServiceClient.class}, proxy);
        return clientProxy;
    }
}
```

下面是客户端使用的一个示例：

```
static void V2Demo() throws TException {
   ThriftClientConfig clientConfig = new ThriftClientConfig(true);
   clientConfig.setConfigProvider(new ZookeeperConfigProvider());

   Calculator.Iface ifaceClient = ThriftClientFactory.getClient(Calculator.Iface.class, clientConfig);
   if (ifaceClient != null) {
       ifaceClient.ping();
       int result = ifaceClient.add(100, 50);
       System.out.println("100 + 50 = " + result);

       ifaceClient.ping();
       ifaceClient.ping();
       ifaceClient.ping();

       result = ifaceClient.add(100, 80);
       System.out.println("100 + 80 = " + result);
   } else {
       System.out.println("create client failed!");
   }
   ifaceClient.zip();
}
```

## 统一监控
其实有了前面的代理，统一监控就非常容易实现了，我们直接在代理类中添加监控就好，可以细分到service和method级别，添加qps、latency、超时等指标的监控，这里就不再具体展示了。

## 小结
本文从如何更好的进行负载均衡和连接池管理出发，引出对前一篇基本封装的改进完善，通过引入java代理机制，使得实现更优美，功能更灵活，也更容易支持客户端封装的统一监控。下一篇我们会介绍在V2基础上如何支持熔断、降级和限流，敬请期待。







