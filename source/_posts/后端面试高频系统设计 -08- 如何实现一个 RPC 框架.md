---
title: 后端面试高频系统设计 -08- 如何实现一个 RPC 框架
date: 2026-03-12 16:50:00
tags:
  - 系统设计
  - RPC
  - 框架
  - 面试
categories: 系统设计
---

# 如何自己实现一个 RPC 框架？⭐

## 📌 题目分析

**RPC（Remote Procedure Call）** 是分布式系统的基石，考察点包括：

- 网络通信能力
- 序列化/反序列化
- 服务注册与发现
- 负载均衡
- 动态代理

**核心组件：**
1. 服务注册中心
2. 服务提供者（Server）
3. 服务消费者（Client）
4. 通信协议

<!-- more -->

---

## 🏗️ 整体架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                         服务注册中心                              │
│                        (Zookeeper/Nacos)                        │
└────────────┬─────────────────────────────────┬──────────────────┘
             │                                 │
             │ 注册服务                         │ 发现服务
             │                                 │
    ┌────────▼────────┐               ┌────────▼────────┐
    │   服务提供者     │               │   服务消费者     │
    │    (Server)     │               │    (Client)     │
    ├─────────────────┤               ├─────────────────┤
    │  服务暴露模块    │               │  动态代理模块    │
    │  网络通信模块    │               │  负载均衡模块    │
    │  线程池模块      │               │  网络通信模块    │
    └────────┬────────┘               └────────┬────────┘
             │                                 │
             └──────────────┬──────────────────┘
                            │
                     ┌──────▼──────┐
                     │   Netty     │
                     │  (TCP 通信)  │
                     └─────────────┘
```

---

## 💡 核心模块实现

### 1. 服务接口定义

```java
// 服务接口
public interface UserService {
    User getUserById(Long id);
    void saveUser(User user);
}

// 服务实现（服务端）
@Service
public class UserServiceImpl implements UserService {
    @Override
    public User getUserById(Long id) {
        return new User(id, "张三", 25);
    }
    
    @Override
    public void saveUser(User user) {
        // 保存逻辑
    }
}
```

### 2. 服务注册中心

```java
// 服务注册接口
public interface ServiceRegistry {
    /**
     * 注册服务
     * @param serviceName 服务名称
     * @param address 服务地址（IP:Port）
     */
    void register(String serviceName, String address);
    
    /**
     * 发现服务
     * @param serviceName 服务名称
     * @return 服务地址列表
     */
    List<String> discover(String serviceName);
}

// Zookeeper 实现
public class ZookeeperRegistry implements ServiceRegistry {
    private final ZooKeeper zk;
    private static final String ROOT_PATH = "/rpc";
    
    public ZookeeperRegistry(String connectString) throws Exception {
        zk = new ZooKeeper(connectString, 5000, null);
    }
    
    @Override
    public void register(String serviceName, String address) {
        try {
            String path = ROOT_PATH + "/" + serviceName;
            // 创建服务节点
            if (zk.exists(path, false) == null) {
                zk.create(path, new byte[0], 
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, 
                    CreateMode.PERSISTENT);
            }
            // 创建临时子节点（服务实例）
            zk.create(path + "/instance-", address.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL);
        } catch (Exception e) {
            throw new RuntimeException("服务注册失败", e);
        }
    }
    
    @Override
    public List<String> discover(String serviceName) {
        try {
            String path = ROOT_PATH + "/" + serviceName;
            List<String> children = zk.getChildren(path, true);
            List<String> addresses = new ArrayList<>();
            for (String child : children) {
                byte[] data = zk.getData(path + "/" + child, false, null);
                addresses.add(new String(data));
            }
            return addresses;
        } catch (Exception e) {
            throw new RuntimeException("服务发现失败", e);
        }
    }
}
```

### 3. 服务暴露（服务端）

```java
// 服务暴露注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface RpcService {
    Class<?> interfaceClass();
    String version() default "1.0.0";
}

// 服务暴露处理器
public class ServiceExporter {
    private final ServiceRegistry registry;
    private final int port;
    private final Map<String, Object> serviceMap = new ConcurrentHashMap<>();
    
    public ServiceExporter(ServiceRegistry registry, int port) {
        this.registry = registry;
        this.port = port;
    }
    
    // 注册服务
    public void export(Object service) {
        Class<?>[] interfaces = service.getClass().getInterfaces();
        for (Class<?> interfaceClass : interfaces) {
            RpcService rpcService = interfaceClass.getAnnotation(RpcService.class);
            if (rpcService != null) {
                String serviceName = interfaceClass.getName();
                serviceMap.put(serviceName, service);
                // 注册到注册中心
                String address = getLocalIp() + ":" + port;
                registry.register(serviceName, address);
            }
        }
    }
    
    // 启动 Netty 服务器
    public void start() throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline()
                            .addLast(new RpcDecoder())      // 解码器
                            .addLast(new RpcEncoder())      // 编码器
                            .addLast(new RpcServerHandler(serviceMap));
                    }
                });
            
            ChannelFuture future = bootstrap.bind(port).sync();
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
    
    private String getLocalIp() {
        // 获取本机 IP
        return InetAddress.getLocalHost().getHostAddress();
    }
}
```

### 4. 服务消费（客户端）

```java
// 动态代理实现
public class RpcProxy {
    private final ServiceRegistry registry;
    
    public RpcProxy(ServiceRegistry registry) {
        this.registry = registry;
    }
    
    // 创建代理
    @SuppressWarnings("unchecked")
    public <T> T createProxy(Class<T> interfaceClass) {
        return (T) Proxy.newProxyInstance(
            interfaceClass.getClassLoader(),
            new Class<?>[]{interfaceClass},
            (proxy, method, args) -> {
                // 构建 RPC 请求
                RpcRequest request = new RpcRequest();
                request.setServiceName(interfaceClass.getName());
                request.setMethodName(method.getName());
                request.setParameterTypes(method.getParameterTypes());
                request.setParameters(args);
                
                // 服务发现
                List<String> addresses = registry.discover(interfaceClass.getName());
                if (addresses.isEmpty()) {
                    throw new RuntimeException("服务不可用");
                }
                
                // 负载均衡（随机）
                String address = addresses.get(new Random().nextInt(addresses.size()));
                String[] hostPort = address.split(":");
                
                // 发送请求
                return sendRequest(hostPort[0], Integer.parseInt(hostPort[1]), request);
            }
        );
    }
    
    private Object sendRequest(String host, int port, RpcRequest request) {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline()
                            .addLast(new RpcEncoder())
                            .addLast(new RpcDecoder())
                            .addLast(new RpcClientHandler(request));
                    }
                });
            
            ChannelFuture future = bootstrap.connect(host, port).sync();
            RpcClientHandler handler = future.channel().pipeline().get(RpcClientHandler.class);
            return handler.getResponse(); // 等待响应
        } catch (Exception e) {
            throw new RuntimeException("RPC 调用失败", e);
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

### 5. 序列化/反序列化

```java
// RPC 请求
@Data
public class RpcRequest implements Serializable {
    private String serviceName;
    private String methodName;
    private Class<?>[] parameterTypes;
    private Object[] parameters;
}

// RPC 响应
@Data
public class RpcResponse implements Serializable {
    private Object data;
    private String error;
    private boolean success;
}

// JSON 序列化器
public class JsonSerializer implements Serializer {
    @Override
    public byte[] serialize(Object obj) {
        return JSON.toJSONString(obj).getBytes(StandardCharsets.UTF_8);
    }
    
    @Override
    public <T> T deserialize(byte[] data, Class<T> clazz) {
        return JSON.parseObject(new String(data, StandardCharsets.UTF_8), clazz);
    }
}
```

### 6. 负载均衡策略

```java
// 负载均衡接口
public interface LoadBalancer {
    String select(List<String> addresses);
}

// 随机负载均衡
public class RandomLoadBalancer implements LoadBalancer {
    private final Random random = new Random();
    
    @Override
    public String select(List<String> addresses) {
        return addresses.get(random.nextInt(addresses.size()));
    }
}

// 轮询负载均衡
public class RoundRobinLoadBalancer implements LoadBalancer {
    private final AtomicInteger index = new AtomicInteger(0);
    
    @Override
    public String select(List<String> addresses) {
        int i = index.getAndIncrement() % addresses.size();
        return addresses.get(i);
    }
}

// 一致性哈希负载均衡
public class ConsistentHashLoadBalancer implements LoadBalancer {
    private final TreeMap<Long, String> virtualNodes = new TreeMap<>();
    private static final int VIRTUAL_NODE_COUNT = 100;
    
    @Override
    public String select(List<String> addresses) {
        // 构建虚拟节点环
        virtualNodes.clear();
        for (String address : addresses) {
            for (int i = 0; i < VIRTUAL_NODE_COUNT; i++) {
                long hash = hash(address + "#" + i);
                virtualNodes.put(hash, address);
            }
        }
        
        // 选择节点
        long hash = hash(UUID.randomUUID().toString());
        SortedMap<Long, String> tailMap = virtualNodes.tailMap(hash);
        return tailMap.isEmpty() ? virtualNodes.firstEntry().getValue() : tailMap.firstEntry().getValue();
    }
    
    private long hash(String key) {
        return MurmurHash.hash64(key);
    }
}
```

---

## 📊 通信协议设计

```
┌─────────────────────────────────────────────────────────────┐
│                      TCP 数据包                              │
├──────────────┬──────────────┬───────────────────────────────┤
│ 魔数 (4 字节)  │ 版本 (1 字节)  │ 数据长度 (4 字节)               │
├──────────────┴──────────────┴───────────────────────────────┤
│                    序列化数据 (变长)                          │
└─────────────────────────────────────────────────────────────┘

// 协议常量
public class RpcProtocol {
    public static final int MAGIC_NUMBER = 0x12345678;
    public static final byte VERSION = 1;
}

// 编码器
public class RpcEncoder extends MessageToByteEncoder<Object> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) {
        out.writeInt(RpcProtocol.MAGIC_NUMBER);
        out.writeByte(RpcProtocol.VERSION);
        
        byte[] data = JsonSerializer.serialize(msg);
        out.writeInt(data.length);
        out.writeBytes(data);
    }
}

// 解码器
public class RpcDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        if (in.readableBytes() < 9) return;
        
        in.markReaderIndex();
        int magic = in.readInt();
        if (magic != RpcProtocol.MAGIC_NUMBER) {
            in.resetReaderIndex();
            throw new RuntimeException("魔数不匹配");
        }
        
        byte version = in.readByte();
        int length = in.readInt();
        
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }
        
        byte[] data = new byte[length];
        in.readBytes(data);
        
        out.add(JsonSerializer.deserialize(data, RpcResponse.class));
    }
}
```

---

## 🔧 完整使用示例

### 服务端启动

```java
public class RpcServerApplication {
    public static void main(String[] args) throws Exception {
        // 1. 创建注册中心
        ServiceRegistry registry = new ZookeeperRegistry("127.0.0.1:2181");
        
        // 2. 创建服务暴露器
        ServiceExporter exporter = new ServiceExporter(registry, 8080);
        
        // 3. 注册服务
        UserService userService = new UserServiceImpl();
        exporter.export(userService);
        
        // 4. 启动服务器
        System.out.println("RPC Server starting on port 8080...");
        exporter.start();
    }
}
```

### 客户端调用

```java
public class RpcClientApplication {
    public static void main(String[] args) {
        // 1. 创建注册中心
        ServiceRegistry registry = new ZookeeperRegistry("127.0.0.1:2181");
        
        // 2. 创建代理
        RpcProxy proxy = new RpcProxy(registry);
        
        // 3. 获取服务代理
        UserService userService = proxy.createProxy(UserService.class);
        
        // 4. 远程调用
        User user = userService.getUserById(1L);
        System.out.println("用户：" + user.getName());
        
        userService.saveUser(new User(2L, "李四", 30));
    }
}
```

---

## ⚠️ 进阶功能

### 1. 超时控制

```java
// 客户端添加超时控制
public class RpcClientHandler extends SimpleChannelInboundHandler<RpcResponse> {
    private final CountDownLatch latch = new CountDownLatch(1);
    private RpcResponse response;
    private final long timeout = 5000; // 5 秒超时
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcResponse msg) {
        this.response = msg;
        latch.countDown();
    }
    
    public Object getResponse() throws Exception {
        if (latch.await(timeout, TimeUnit.MILLISECONDS)) {
            if (response.isSuccess()) {
                return response.getData();
            } else {
                throw new RuntimeException(response.getError());
            }
        } else {
            throw new RuntimeException("RPC 调用超时");
        }
    }
}
```

### 2. 重试机制

```java
// 客户端添加重试
public class RetryProxy {
    private static final int MAX_RETRY = 3;
    
    public Object invokeWithRetry(RpcRequest request, String host, int port) {
        int retryCount = 0;
        while (retryCount < MAX_RETRY) {
            try {
                return sendRequest(host, port, request);
            } catch (Exception e) {
                retryCount++;
                if (retryCount >= MAX_RETRY) {
                    throw e;
                }
                // 指数退避
                try {
                    Thread.sleep(100 * (1 << retryCount));
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
        return null;
    }
}
```

### 3. 熔断降级

```java
// 熔断器实现
public class CircuitBreaker {
    private final int failureThreshold = 5;
    private final int successThreshold = 3;
    private final long timeout = 60000; // 1 分钟
    
    private AtomicInteger failureCount = new AtomicInteger(0);
    private AtomicInteger successCount = new AtomicInteger(0);
    private volatile State state = State.CLOSED;
    private volatile long lastFailureTime = 0;
    
    enum State { CLOSED, OPEN, HALF_OPEN }
    
    public boolean allowRequest() {
        switch (state) {
            case CLOSED:
                return true;
            case OPEN:
                if (System.currentTimeMillis() - lastFailureTime > timeout) {
                    state = State.HALF_OPEN;
                    return true;
                }
                return false;
            case HALF_OPEN:
                return true;
        }
        return false;
    }
    
    public void recordSuccess() {
        successCount.incrementAndGet();
        if (state == State.HALF_OPEN && successCount.get() >= successThreshold) {
            state = State.CLOSED;
            failureCount.set(0);
            successCount.set(0);
        }
    }
    
    public void recordFailure() {
        failureCount.incrementAndGet();
        lastFailureTime = System.currentTimeMillis();
        if (failureCount.get() >= failureThreshold) {
            state = State.OPEN;
        }
    }
}
```

---

## ✅ 面试加分项

1. **提到 Netty**：高性能 NIO 框架
2. **提到序列化对比**：JSON vs Protobuf vs Hessian
3. **提到连接池**：避免频繁创建连接
4. **提到异步调用**：CompletableFuture 实现
5. **提到监控**：调用链路追踪、指标采集

---

## 🎯 下一篇文章

[如何设计一个动态线程池？](/2026/03/12/后端面试高频系统设计 -09-如何设计一个动态线程池/)

> 💡 **提示**：RPC 框架是分布式系统的基础，掌握核心原理比死记代码更重要！
