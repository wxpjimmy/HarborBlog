title: 如何打造优雅的Thrift RPC封装库 - Part 1
date: 2017-10-22 22:00:01
categories: rpc
tags: [rpc, thrift, zookeeper]
----------
本系列文章我们重点对Thrift RPC封装库进行介绍，计划分成几部分来讲：
    * 一个简易版本的通用Thrift RPC库封装，支持服务注册、服务发现。
    * Thrift RPC熔断、降级、限流
    * Thrift RPC异步化（swift）

这篇主要讲第一部分。

## Thrift RPC简介   
Thrift 是Facebook 开源的提供跨语言支持的服务框架，一般在业务开发中会被用于以下两种场景：
* 结构化对象序列化，这个主要得益于Thrift提供的完善的序列化/反序列化机制，可以很好的应用于结构化数据存储（结合Kafka、Scribe、Hive等）
* RPC服务框架，Thrift RPC相比Restful API的好处主要是结构化+bianry data传输效率更高，缺点是表达性不如Restful API直白，一般用于企业内部服务之间的通信。   

具体介绍Thrift的概念和文档已经很多了，这里就不详细展开了，感兴趣的同学可以移步[Thrift 官网](https://thrift.apache.org/)了解详情。
## Thrift RPC调用
首先，我们先看一个正常Thrift RPC调用的例子。我们首先定义一个thrift 服务描述文件Calculator.thrift

```
namespace java tutorial

service Calculator {
    void ping(),
    i32 add(1:i32 num1, 2:i32 num2),
    oneway void zip()
}
```

通过thrift命令

```
thrift --gen <language> <Thrift filename>
```

项目中引用生成的java代码 （推荐使用maven thrift plugin插件），启动server代码如下：

```
Calculator.Processor processor = new Calculator.Processor(new CalculatorServiceImpl());

try {
  TServerTransport serverTransport = new TServerSocket(9090);
  TServer server = new TSimpleServer(processor, serverTransport);

  // Use this for a multithreaded server
  // TServer server = new TThreadPoolServer(new TThreadPoolServer.Args(serverTransport).processor(processor));

  System.out.println("Starting the simple server...");
  server.serve();
} catch (Exception e) {
  e.printStackTrace();
}
```

Server thrift提供了多种类型，这里demo只用了最简单的TSimpleServer。
我们看下启动过程：
* 首先，使用生成代码中的Processor类，构造使用的参数是我们提供的服务实现CalculatorServiceImpl对象。 
* 创建TServerTransport，服务绑定的端口是9090
* 使用Processor和serverTransport创建TSimpleServer，并通过server.serve启动服务。

接下来我们看下client 调用 server端代码：

```
   try {
       TTransport transport;
       transport = new TSocket("localhost", 9090);
       transport.open();
       TProtocol protocol = new TBinaryProtocol(transport);
       Calculator.Client client = new Calculator.Client(protocol);
       client.ping();
       int result = client.add(100, 20);
       System.out.println(result);
   } catch (TTransportException e) {
       e.printStackTrace();
   } catch (TException e) {
       e.printStackTrace();
   }
```

可以看到，客户端指定传输协议，通过socket连接到本机9090端口；接下来指定编码协议是使用二进制编码TBinaryProtocol；最后构造Calculator Client对象，并通过创建的Client对象调用远程服务接口进行操作。


## 为什么需要封装通用的Thrift RPC库
刚才我们描述了一个最简单的thrift服务启动和客户端调用流程。如果我们只有这么一个服务，这种方式可以work得很happy。而实际上往往没有这么简单，每次都需要这么一段代码来创建一个客户端，并且如果下游服务有多台机器，我还要考虑如何管理链接、如何进行负载均衡，诸如此类事情如果让每个服务定义方/使用方自己去做，无疑代价是巨大的，后续的维护升级都不是一件容易的事情，这也是为什么通常我们都需要封装一个通用的Thrift RPC库的最直接的原因之一。

那么我们该如何来定义通用的Thrift RPC库呢？

## 最简单的Thrift RPC封装

我们再来回顾下刚才的代码示例，首先为了通用，我们需要约束序列化的机制（比如都使用Bianry），统一了序列化之后，我们先看下服务端，在服务端我们要做的其实很简单，只需要框架能够根据提供的服务名（这里比如Calculator.Iface)能创建对应的TProcessor就可以。这个可以通过反射来完成，因为Thrift自动生成的stub代码编译后，Processor class文件都是以<Service>$Processor来命名的。示例代码如下：

```
static <T> TServer getServer(Class<T> clazz, T serviceImplInstance, int port) {
   //get Processor class name
   String simpleName = clazz.getName();
   String name = simpleName;
   if (simpleName.endsWith("$Iface")) {
       name = simpleName.substring(0, simpleName.indexOf("$Iface"));
   }
   name = name + "$Processor";
   //verify processpr class exist
   Class<?> processorClass;
   try {
       processorClass = Class.forName(name);
   } catch (ClassNotFoundException e) {
       System.out.println("Class not found: " + name);
       return null;
   }
   //create server
   try {
       TServerTransport serverTransport = new TServerSocket(port);
       Constructor<?> constructor = processorClass.getConstructor(clazz);
       TProcessor tProcessor = (TProcessor) constructor.newInstance(serviceImplInstance);
       TServer server = new TSimpleServer(tProcessor, serverTransport);
       return server;
   } catch (TTransportException e) {
       e.printStackTrace();
   } catch (Exception e) {
       e.printStackTrace();
   }

   return null;
}
```

启动服务代码如下：

```
TServer server = getServer(Calculator.Iface.class, new CalculatorServiceImpl(), 9090);
if (server != null) {
    System.out.println("Starting the simple server...");
    server.serve();
} else {
    System.out.println("Get server failed!");
}
```

客户端测也类似，我们可以看到，客户端需要构建Client对象，而所有服务的Client对象实际上都实现了TServerClient和对应的Iface接口：

```
public static class Client implements TServiceClient, Iface {
```

Client的构造函数也是固定的使用TProtocol，所以基于这些信息，我们可以通过反射创建出TServerClient，然后返回给客户端的时候转为对应的Iface类型，这样客户端就可以通过简单得提供Iface class来获取相应的客户端操作对象，并进行后续的调用处理。代码示例如下：

```
static <T> T getClient(Class<T> clazz, String host, int port) {
   //get client class name
   String simpleName = clazz.getName();
   String name = simpleName;
   if(simpleName.endsWith("$Iface")) {
       name = simpleName.substring(0, simpleName.indexOf("$Iface"));
   }
   name  = name + "$Client";
   //verify client class exist
   Class<?> clientClass;
   try {
       clientClass = Class.forName(name);
   } catch (ClassNotFoundException e) {
       System.out.println("Class not found: " + name);
       return null;
   }
   //create client
   try {
       TTransport transport;
       transport = new TSocket(host, port);
       transport.open();
       TProtocol protocol = new TBinaryProtocol(transport);
       Constructor<?> cons = clientClass.getConstructor(TProtocol.class);
       T res = (T) cons.newInstance(protocol);
       return res;
   } catch (TTransportException e) {
       e.printStackTrace();
   } catch (Exception e) {
       e.printStackTrace();
   }

   return null;
}
```

使用方式如下：

```
Calculator.Iface ifaceClient = getClient(Calculator.Iface.class, "localhost", 9090);
if (ifaceClient != null) {
  ifaceClient.ping();
  int result = ifaceClient.add(100, 20);
  System.out.println("100 + 20 = " + result);
} else {
  System.out.println("create client failed!");
}
```

至此我们已经可以很简单的构造一个通用服务启动代理和客户端代理来进行RPC调用了。

## 升级版Thrift RPC代理 -- 服务注册、服务发现
前面我们已经介绍了最简单的服务和客户端封装，我们可以看到调用服务的地址都是固定的某台机器，实际线上使用过程中，下游服务往往都是会部署多台，为了简化服务提供方和服务使用方的工作，我们需要在Thrift RPC框架中支持服务注册和服务发现，下面我们就分别介绍一下我们该如何在刚才封装的基础上来支持服务注册和服务发现。
### 服务注册
目前通用的服务注册中心有Zookeeper、Etcd、Eureka、Consul等，这里我们选用Zookeeper作为demo（其他流程类似，感兴趣可以自行实现）。
服务注册的目的是在服务器启动的时候能够将自己注册到Zookeeper上，在服务关闭的时候将自己从zk的注册列表中移除。我们约定服务在zk注册的形式如下：

```
/registry/<service full name>/Nodes/<ip>:<port>
```

我们定义服务注册接口ServiceRegistry如下：

```
public interface ServerRegistry {
    void register() throws Exception;
    void unregister() throws Exception;
}
```

基于CuratorFramework实现的ZookeeperServeRegistry如下：

```
public class ZookeeperServerRegistry implements ServerRegistry {

    private CuratorFramework client;
    private String zkPathPattern = "/registry/%s/Nodes/%s:%d";
    private String zkPath;
    public ZookeeperServerRegistry(String zkhost, int zkPort, int serverPort, String serverName) {
        client = CuratorFrameworkFactory.newClient(
                zkhost + ":" + zkPort,
                new RetryNTimes(10, 5000)
        );
        client.start();
        System.out.println("zk client start successfully!");
        String ip = Utils.getIp();

        zkPath = String.format(zkPathPattern, serverName, ip, serverPort);
    }


    public void register() throws Exception {
        Stat stat = null;
        try {
            stat = client.checkExists().creatingParentContainersIfNeeded().forPath(zkPath);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if(stat == null) {
            try {
                client.create().creatingParentsIfNeeded().forPath(zkPath);
            } catch (Exception e) {
                e.printStackTrace();
                throw e;
            }
        }
    }

    public void unregister() throws Exception {
        Stat stat = null;
        try {
            stat = client.checkExists().creatingParentContainersIfNeeded().forPath(zkPath);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if (stat != null) {
            try {
                client.delete().deletingChildrenIfNeeded().forPath(zkPath);
            } catch (Exception e) {
                e.printStackTrace();
                throw e;
            }
        }
    }
}
```

下面我们看下服务启动通用注册流程：

```
static <T> void startServer(Class<T> ifaceClass, T serverImpleInstance, int serverPort) throws Exception {
   final TServer server = getServer(ifaceClass, serverImpleInstance, serverPort);
   if (server != null) {
       System.out.println("Starting the simple server...");
       Thread thread = new Thread(new Runnable() {
           public void run() {
               server.serve();
           }
       });
       thread.start();
       final ZookeeperServerRegistry registry = new ZookeeperServerRegistry("127.0.0.1", 2181, 9090, getServerName(ifaceClass));
       registry.register();
       Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
           public void run() {
               try {
                   registry.unregister();
               } catch (Exception e) {
                   e.printStackTrace();
               }
           }
       }));
   } else {
       System.out.println("Get server failed!");
   }
}
```

至此我们完成了基于Zookeeper的服务注册。
### 服务发现
服务注册完成后，所有服务的使用者就不再需要自己去在代码里维护一个服务节点列表，直接从ZK获取就好，并且可以做到服务扩容/缩容的及时响应。
我们定义服务发现接口如下：

```
public interface ServiceDiscovery {
    public List<ServerInstance> discoverService(Class<?> ifaceClazz);
}

public class ServerInstance {
    private String ip;
    private int port;

    public ServerInstance(String ip, int port) {
        this.ip = ip;
        this.port = port;
    }

    public String getIp() {
        return ip;
    }

    public int getPort() {
        return port;
    }

    @Override
    public String toString() {
        return "ServerInstance{" +
                "ip='" + ip + '\'' +
                ", port=" + port +
                '}';
    }
}
```

同样基于CuratorFramework实现ZookeeperServiceDiscovery：

```
public class ZookeeperServiceDiscovery implements ServiceDiscovery {

    private CuratorFramework client;
    private String zkPathPattern = "/registry/%s/Nodes";
    private String zkPath;
    private List<ServerInstance> serverInstances;

    public ZookeeperServiceDiscovery(String zkhost, int zkPort, Class<?> ifaceClass) {
        client = CuratorFrameworkFactory.newClient(
                zkhost + ":" + zkPort,
                new RetryNTimes(10, 5000)
        );
        client.start();

        String serverName = Utils.getServerName(ifaceClass);
        zkPath = String.format(zkPathPattern, serverName);
    }

    public List<ServerInstance> discoverService(Class<?> ifaceClazz) {
        if (serverInstances != null) {
            return serverInstances;
        }

        try {
            List<String> items = client.getChildren().usingWatcher(new CuratorWatcher() {
                public void process(WatchedEvent watchedEvent) throws Exception {
                    if (watchedEvent.getType() == Watcher.Event.EventType.NodeChildrenChanged) {
                        update();
                    } else if (watchedEvent.getType() == Watcher.Event.EventType.NodeDeleted) {
                        serverInstances = null;
                    }
                }
            }).forPath(zkPath);
            serverInstances = parseServer(items);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return serverInstances;
    }

    private void update() {
        try {
            List<String> items = client.getChildren().forPath(zkPath);
            serverInstances = parseServer(items);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private List<ServerInstance> parseServer(List<String> items) {
        if (items == null) {
            return null;
        }
        List<ServerInstance> servers = new ArrayList<ServerInstance>();
        for(String item: items) {
            String[] parts = StringUtils.split(item, ":");
            servers.add(new ServerInstance(parts[0], Integer.valueOf(parts[1])));
        }
        return servers;
    }
}
```

客户端的封装更改如下：

```
static <T> T getClient(Class<T> clazz) {
   ZookeeperServiceDiscovery serviceDiscovery = new ZookeeperServiceDiscovery("127.0.0.1", 2181, clazz);
   List<ServerInstance> serverInstances = serviceDiscovery.discoverService(clazz);
   if (serverInstances == null || serverInstances.isEmpty()) {
       throw new RuntimeException("No server instance found!");
   }
   //here choose one endpoint, for demo we just take the first one
   ServerInstance instance = serverInstances.get(0);
   return getClient(clazz, instance.getIp(), instance.getPort());
}
```

这样我们就可以通过简单的指定服务定义的Iface类来获取到对应服务的客户端实例进行操作。

## 简单总结
本文主要介绍的是最简易版本的Thrift RPC通用封装，支持基于Zookeeper的服务注册、服务发现，还远远称不上是一个完备的RPC封装库，架子已经搭好，接下来的一篇会介绍如何对RPC封装进行通用的负载均衡、监控、熔断、降级、限流等功能的完善，敬请期待。



