线上问题大体可以分为以下几类：

1. **内存相关**
    - **OOM（OutOfMemoryError）**：堆内存溢出、Metaspace溢出、直接内存溢出。
    - **内存泄漏**：对象未被释放，堆持续上涨。
    - **频繁 Full GC / GC 时间长**。
2. **CPU 相关**
    - CPU 使用率过高（单线程死循环 / 大量计算）。
    - CPU 使用率过低（频繁阻塞、IO卡住）。
3. **线程相关**
    - **死锁**：线程互相等待资源。
    - **线程池耗尽**：无空闲线程处理请求。
    - **大量 WAITING / BLOCKED 状态**：IO阻塞、锁竞争严重。
4. **GC 相关**
    - Full GC 频繁，导致系统卡顿。
    - GC 停顿时间过长。
    - GC 后内存回收效果不好。
5. **数据库/缓存相关**
    - 数据库连接池耗尽。
    - SQL 执行慢 / 索引失效。
    - 缓存穿透、击穿、雪崩。
6. **网络相关**
    - 请求超时。
    - 网络抖动、丢包。
    - 服务调用链过长导致延迟。
7. **应用异常**
    - 服务启动报错（配置、依赖冲突）。
    - 运行中频繁报错（NullPointerException、ClassCastException）。
    - 部署版本不一致、依赖 jar 冲突（常见于 Spring Boot/FatJar）。
8. **日志相关**
    - 日志量过大，导致 IO 瓶颈。
    - 日志缺失，排查困难。

---

## 二、常用工具

常用工具可分为 **系统级**、**JVM级** 和 **应用级**。

### 1. 系统级

- **top / htop**：查看CPU、内存、进程占用。
- **vmstat / iostat / sar**：监控CPU、内存、IO、网络。
- **netstat / ss**：查看端口占用、连接状态。
- **dstat / iftop**：网络流量监控。

### 2. JVM级

- **jps**：列出Java进程。
- **jstack**：导出线程堆栈，用于分析死锁、卡顿。
- **jmap**：导出内存快照（heap dump），查看对象分布。
- **jstat**：监控GC情况。
- **jcmd**：多功能诊断工具（GC、线程、堆转储等）。
- **jconsole / VisualVM**：可视化监控JVM状态。
- **MAT (Eclipse Memory Analyzer Tool)**：分析堆dump，定位内存泄漏。

### 3. 应用级

- **Arthas**（阿里出品）：强大的Java线上诊断工具，可以实时查看线程栈、监控方法调用、修改参数。
- **SkyWalking / Pinpoint / Zipkin**：分布式调用链追踪。
- **Prometheus + Grafana**：监控指标收集与展示。
- **日志分析工具**：ELK（Elasticsearch + Logstash + Kibana）、Loki。

---

## 三、常见思路

针对不同问题，排查思路如下：

### 1. 内存问题

- **怀疑内存泄漏/OOM**
    1. `jmap -heap <pid>` 查看堆使用情况。
    2. `jmap -dump:live,format=b,file=heap.hprof <pid>` 导出堆文件，用 MAT 分析。
    3. 关注老年代占用率、某类对象数量是否异常。
- **频繁GC**
    - `jstat -gcutil <pid> 1000` 持续观察。
    - 分析GC日志（推荐使用 [GCViewer](https://github.com/chewiebug/GCViewer)）。

### 2. CPU 问题

- **CPU高**
    1. `top -Hp <pid>` 找到消耗CPU高的线程。
    2. `printf "%x\\n" <tid>` 转换线程ID为十六进制。
    3. `jstack <pid> | grep <nid>` 定位线程栈，找到具体方法。
- **CPU低/卡住**
    - 用 `jstack` 看大量线程是否在 WAITING/BLOCKED，定位是否锁竞争或IO阻塞。

### 3. 线程问题

- **死锁**
    - `jstack` 输出日志，查找 `Found one Java-level deadlock`。
- **线程池耗尽**
    - 查看线程数，是否远大于预期。
    - 检查线程池配置（corePoolSize、queueSize）。

### 4. 数据库/缓存

- **数据库连接池耗尽**
    - 查看连接池指标（HikariCP、Druid有监控页面）。
    - 检查SQL执行时间、慢查询。
- **缓存问题**
    - 查看QPS与命中率。
    - 检查热点key、雪崩保护（预热+随机过期时间）。

### 5. 网络问题

- **请求超时**
    - 检查调用链，是否某服务响应慢。
    - 抓包 (`tcpdump`) 或查看连接数 (`netstat -an | grep ESTABLISHED | wc -l`)。

### 6. 线上调试思路

- **Arthas 常用命令**
    - `thread`：查看线程栈和CPU占用。
    - `jad`：反编译类，确认线上版本。
    - `watch`：监控方法的入参和返回值。
    - `trace`：方法调用链耗时分析。

---

## 四、总结

线上问题排查通常流程是：

1. **快速定位问题范围**（系统 → JVM → 应用）。
    
2. **结合监控 & 日志**（CPU、内存、线程、请求延迟）。
    
3. **用合适的工具抓取数据**（jstack/jmap/Arthas）。
    
4. **分析数据找根因**（死循环、内存泄漏、DB瓶颈）。
    
5. **验证并优化**（代码优化、调参、扩容）。
    
6. **YGC 频繁**
    
    - 增大 **新生代大小**（-Xmn），减少 YGC 次数。
    - 优化代码中短命对象的创建（例如避免频繁创建大对象）。
7. **老年代上涨快 / FGC 频繁**
    
    - 检查是否有 **大对象直接进入老年代**（可调 `XX:PretenureSizeThreshold`）。
    - 排查是否 **内存泄漏** → `jmap -dump` + MAT 分析。
    - 调整老年代大小，或优化缓存/集合对象。
8. **FGC 频繁且效果差**（回收后老年代占用仍高）
    
    - 高概率是内存泄漏。
    - 导出 heap dump，重点看大集合（Map/List）里是否有异常对象未释放。
9. **Metaspace 占用高**
    
    - 检查是否有频繁动态生成的类（CGLIB、反射、Lambda 反序列化）。
    - 增大 `XX:MaxMetaspaceSize`，或优化代码。
10. **GC 耗时长**
    
    - 考虑 **换 GC 收集器**（CMS、G1、ZGC）。
    - 调整 GC 参数，比如并行度、Region 大小（G1）。
    - 如果是 Full GC 卡顿 → 先确认是不是 Stop-The-World 引起的。
