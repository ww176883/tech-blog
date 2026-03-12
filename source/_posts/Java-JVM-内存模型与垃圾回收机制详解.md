---
title: Java JVM 内存模型与垃圾回收机制详解
date: 2026-03-12 10:30:03
tags:
  - Java
  - JVM
  - 性能优化
  - 垃圾回收
categories: 技术文章
---

# Java JVM 内存模型与垃圾回收机制详解

## 前言

Java 虚拟机（JVM）是 Java 程序运行的基础，理解 JVM 的内存模型和垃圾回收机制对于编写高性能、低内存泄漏的 Java 应用至关重要。本文将深入探讨 JVM 的内存结构、垃圾回收算法以及常见的 GC 调优策略。

<!-- more -->

## 一、JVM 内存模型

### 1.1 内存区域划分

JVM 将运行时数据区划分为以下几个部分：

```
┌─────────────────────────────────────────┐
│              Heap (堆)                   │
│  ┌─────────────┐  ┌─────────────────┐   │
│  │ Eden Space  │  │ Survivor Space  │   │
│  │             │  │  (S0 + S1)      │   │
│  │             │  └─────────────────┘   │
│  │             │  ┌─────────────────┐   │
│  │             │  │   Old Gen       │   │
│  └─────────────┘  └─────────────────┘   │
│  ┌─────────────────────────────────┐    │
│  │      Metaspace (元空间)          │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 1.2 各区域详解

#### 堆（Heap）

- **Eden 区**：新对象优先分配的区域
- **Survivor 区**：存放经过 Minor GC 后存活的对象
- **Old Gen（老年代）**：存放长期存活的对象
- **Metaspace**：存储类的元数据、方法信息等

#### 线程私有区域

| 区域 | 作用 | 是否 OOM |
|------|------|----------|
| 程序计数器 | 记录当前线程执行的字节码行号 | 否 |
| Java 虚拟机栈 | 存储栈帧、局部变量、操作数栈 | 是 |
| 本地方法栈 | 为 Native 方法服务 | 是 |

#### 线程共享区域

| 区域 | 作用 | 是否 OOM |
|------|------|----------|
| 堆 | 存放对象实例 | 是 |
| 元空间 | 类元数据、静态变量 | 是 |

## 二、垃圾回收机制

### 2.1 判断对象可回收的方法

#### 引用计数法（已废弃）

```java
// 循环引用问题
public class ReferenceCountingGC {
    public Object instance = null;
    
    public static void main(String[] args) {
        ReferenceCountingGC obj1 = new ReferenceCountingGC();
        ReferenceCountingGC obj2 = new ReferenceCountingGC();
        obj1.instance = obj2;
        obj2.instance = obj1;
        obj1 = null;
        obj2 = null;
        // 两个对象互相引用，但实际已不可达
    }
}
```

#### 可达性分析算法（GC Roots）

JVM 使用可达性分析算法，从 GC Roots 向下搜索：

```
GC Roots 包括：
├── 虚拟机栈中引用的对象
├── 方法区中类静态属性引用的对象
├── 方法区中常量引用的对象
└── 本地方法栈中 JNI 引用的对象
```

### 2.2 垃圾回收算法

#### 标记 - 清除算法

```
步骤：
1. 标记出所有需要回收的对象
2. 统一清除被标记的对象

缺点：
- 效率不高
- 产生内存碎片
```

#### 复制算法

```
将内存分为两块相等区域：
- 每次只使用一块
- GC 时将存活对象复制到另一块
- 清理已使用区域

优点：无碎片
缺点：内存利用率低（50%）
```

#### 标记 - 整理算法

```
步骤：
1. 标记存活对象
2. 将存活对象向一端移动
3. 清理边界外的内存

适用于：老年代
```

#### 分代收集算法

```
新生代：
- 对象存活率低
- 使用复制算法
- Minor GC 频繁

老年代：
- 对象存活率高
- 使用标记 - 整理算法
- Major GC / Full GC
```

### 2.3 垃圾收集器对比

| 收集器 | 代别 | 算法 | 特点 |
|--------|------|------|------|
| Serial | 新生代 | 复制 | 单线程，停顿时间长 |
| ParNew | 新生代 | 复制 | Serial 的多线程版本 |
| Parallel Scavenge | 新生代 | 复制 | 关注吞吐量 |
| CMS | 老年代 | 标记 - 清除 | 低延迟，有碎片 |
| G1 | 全堆 | 分区回收 | 可预测停顿时间 |
| ZGC | 全堆 | 染色指针 | 超低停顿（<10ms） |

## 三、GC 日志分析

### 3.1 开启 GC 日志

```bash
# JDK 8
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/to/gc.log

# JDK 9+
-Xlog:gc*:file=/path/to/gc.log:time,uptime,level,tags
```

### 3.2 日志解读

```
[GC (Allocation Failure) 
 [PSYoungGen: 1048576K->131072K(1228800K)]  
 1048576K->393216K(4063232K), 0.0123456 secs]
 
解读：
- GC 类型：Minor GC（Allocation Failure）
- 新生代：回收前 1048576K → 回收后 131072K
- 堆总量：回收前 1048576K → 回收后 393216K
- 停顿时间：0.012 秒
```

## 四、常见 GC 问题与调优

### 4.1 内存泄漏排查

```bash
# 生成堆转储
jmap -dump:format=b,file=heap.hprof <pid>

# 使用 MAT 分析
# 查看 Dominator Tree 找出大对象
# 分析 GC Roots 引用链
```

### 4.2 频繁 Full GC 调优

```bash
# 检查老年代使用率
jstat -gcutil <pid> 1000 10

# 调整堆大小
-Xms4g -Xmx4g

# 调整新生代比例
-XX:NewRatio=2

# 设置 Metaspace 上限
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### 4.3 G1 收集器调优

```bash
# 启用 G1
-XX:+UseG1GC

# 设置最大停顿时间
-XX:MaxGCPauseMillis=200

# 设置 Region 大小
-XX:G1HeapRegionSize=16m

# 触发 Mixed GC 的阈值
-XX:InitiatingHeapOccupancyPercent=45
```

## 五、最佳实践

### 5.1 编码层面

```java
// ✅ 推荐：及时关闭资源
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line = reader.readLine();
}

// ❌ 避免：静态集合类导致的内存泄漏
public class Cache {
    private static Map<String, Object> cache = new HashMap<>();
    // 应使用 WeakHashMap 或设置过期策略
}

// ✅ 推荐：使用弱引用
private static Map<String, WeakReference<CacheEntry>> cache = new ConcurrentHashMap<>();
```

### 5.2 JVM 参数建议

```bash
# 服务端应用推荐配置
-server
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof
```

## 六、总结

| 要点 | 说明 |
|------|------|
| 内存模型 | 理解堆、栈、元空间的作用 |
| GC 算法 | 掌握分代收集和各类收集器特点 |
| 调优策略 | 根据应用场景选择合适的 GC |
| 监控工具 | 熟练使用 jstat、jmap、MAT 等工具 |

---

**参考资料：**
- 《深入理解 Java 虚拟机》（周志明）
- [Oracle JVM Documentation](https://docs.oracle.com/javase/specs/jvms/se11/html/)
- [G1 GC Tuning Guide](https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-garbage-collector-tuning.html)

---

*如果你觉得这篇文章有帮助，欢迎点赞和分享！*
