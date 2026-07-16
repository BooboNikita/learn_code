# SCA 综合实战与生产踩坑

> 来源：[Spring Cloud Alibaba 全栈技术专题（四）— 综合实战与生产踩坑](https://zhuanlan.zhihu.com/p/2055576959875392742)
> 一句话总结：把 Nacos/Sentinel/Gateway/Seata/RocketMQ 拼成完整电商 Demo，配上可观测性体系（Metrics + Tracing + Logging），交出 5 个生产真实踩坑及解法。

## 一、综合架构：订单-库存-支付三服务

### 1.1 系统全貌

```
客户端请求
  └─> Gateway（8080）
        │── /api/order/** → lb://order-service（8081）
        │                        ├─ Feign → inventory-service（8082）
        │                        ├─ Seata 全局事务 ─────────┘
        │                        └─ RocketMQ → payment-service（8083）
        └── /api/user/**  → lb://user-service（8084）

基础设施：
  Nacos（8848）            — 注册中心 + 配置中心
  Sentinel Dashboard（8079）— 限流熔断管理
  Seata Server（7091）     — 分布式事务协调
  RocketMQ（9876）        — 消息队列
  SkyWalking OAP（11800）  — 链路追踪
  Prometheus + Grafana     — 指标监控
```

### 1.2 核心调用链

**同步链（Seata 全局事务）：**

```
Gateway 鉴权
  └─> OrderService.createOrder(@GlobalTransactional)
        ├─ 1. 写订单表（分支事务1）
        ├─ 2. Feign → InventoryService.deductStock()（分支事务2，XID 自动传播）
        └─ 3. Feign → PaymentService.pay()（分支事务3）
  任一失败 → Seata 二阶段回滚全部 / 全部成功 → TC 通知全局提交
```

**异步链（RocketMQ 消息驱动）：**

```
OrderService.createOrder 成功后
  └─> RocketMQ → ORDER_TOPIC
        ├─> NotificationService（发短信）
        └─> PointsService（赠积分）
```

### 1.3 订单服务核心代码（TM 入口）

```java
@GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
@SentinelResource(value = "createOrder",
    blockHandler = "createOrderBlock",
    fallback = "createOrderFallback")
public Order createOrder(CreateOrderDTO dto) {
    Order order = orderRepository.save(dto.toOrder());          // 1. 写订单
    inventoryClient.deductStock(dto.getItems());                 // 2. 扣库存（XID 自动传播）
    paymentClient.pay(dto.getPayment());                        // 3. 支付
    streamBridge.send("order-out-0",                             // 4. 异步通知
        new OrderCreatedEvent(order.getId(), dto.getItems()));
    return order;
}
```

---

## 二、可观测性三根柱子

```
Metrics（指标）  → "系统现在状态"：CPU、QPS、错误率
Tracing（链路）  → "请求走过哪里"：端到端调用链
Logging（日志）  → "发生了什么"：带上下文的事件记录
```

### 2.1 Metrics：Micrometer + Prometheus + Grafana

**自定义业务指标：**

```java
@Service
public class OrderMetricService {
    private final Counter orderCreatedCounter;
    private final Timer orderProcessTimer;

    public OrderMetricService(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("order.created.total")
            .tag("env", "prod").register(registry);
        this.orderProcessTimer = Timer.builder("order.process.duration")
            .publishPercentiles(0.5, 0.95, 0.99).register(registry);
    }

    public Order createOrder(CreateOrderDTO dto) {
        return orderProcessTimer.record(() -> {
            Order order = doCreate(dto);
            orderCreatedCounter.increment();
            return order;
        });
    }
}
```

### 2.2 Tracing：SkyWalking（零侵入 Java Agent）

```bash
java -javaagent:/opt/skywalking/skywalking-agent.jar \
     -Dskywalking.agent.service_name=order-service \
     -Dskywalking.collector.backend_service=skywalking-oap:11800 \
     -jar order-service.jar
```

**自动采集范围**：HTTP 请求、JDBC、Redis、RocketMQ/Kafka、Feign、@Async 线程

**手动创建细粒度 Span：**

```java
public void deductStock(Long skuId, int count) {
    AbstractSpan span = ContextManager.createLocalSpan("inventory.deduct");
    span.tag("skuId", String.valueOf(skuId));
    try {
        doDeductStock(skuId, count);
    } catch (Exception e) {
        span.errorOccurred();
        throw e;
    } finally {
        ContextManager.stopSpan();
    }
}
```

### 2.3 Logging：ELK 日志聚合

```
各服务日志（logback 打印 traceId）
  └─> Filebeat 采集
        └─> Logstash 结构化处理
              └─> Elasticsearch 索引
                    └─> Kibana 查询
```

**logback 关键配置（关联 Tracing 和 Logging）：**

```xml
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern>
```

> 在 Kibana 搜索一个 traceId，即可拿到该请求在所有服务上的全部日志。

---

## 三、灰度发布全流程

```
Step 1: 部署 v2 实例，Nacos 元数据标记 version=v2
Step 2: 测试请求带 X-Version:v2 Header → Gateway 灰度路由到 v2
Step 3: Sentinel 对 v2 设更严格流控（QPS=10），防新版本崩了扩散
Step 4: 验证通过，Nacos 控制台逐步调高 v2 权重、调低 v1 权重
Step 5: v1 权重降为 0，观察 10 分钟无异常 → 下线 v1
```

---

## 四、五大生产踩坑

### 坑 1：Nacos 缓存导致调用已下线节点

| 维度 | 内容 |
| --- | --- |
| 现象 | Nacos 控制台下线实例后，调用方还在请求它，报 Connection Refused |
| 根因 | Nacos 客户端本地文件缓存 + LoadBalancer 内存缓存，两重缓存延迟叠加 |
| 解决 | LoadBalancer 缓存 TTL 降至 10s；下线前先调注销 API，等 30s 再关闭应用 |

```yaml
spring:
  cloud:
    nacos:
      discovery:
        cache-dir: /opt/app/nacos-cache
    loadbalancer:
      cache:
        ttl: 10s
        capacity: 100
```

### 坑 2：Sentinel 规则重启丢失

| 维度 | 内容 |
| --- | --- |
| 现象 | Dashboard 配的规则重启后全没了 |
| 根因 | 默认规则存 JVM 内存，Dashboard 只推到内存不持久化 |
| 解决 | 规则持久化到 Nacos，修改 JSON 后所有实例实时生效 |

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: 127.0.0.1:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
```

### 坑 3：Seata AT 模式死锁

| 维度 | 内容 |
| --- | --- |
| 现象 | 高并发订单接口大量超时，频繁 LockConflictException |
| 根因 | AT 模式全局行锁，多事务并发修改同一行（热门 SKU 库存），锁争抢堆到雪崩 |
| 解决 | 热点库存改 Redis 扣减 + 异步同步 DB；热点场景切 TCC 模式 |

```yaml
seata:
  client:
    rm:
      lock:
        retry-interval: 10
        retry-times: 30
        retry-policy-branch-rollback-on-conflict: true
```

### 坑 4：Feign 超时 < Sentinel 慢调用阈值

| 维度 | 内容 |
| --- | --- |
| 现象 | 配了 Sentinel 慢调用阈值 1000ms，但线上无慢调用统计 |
| 根因 | Feign read-timeout=800ms 先超时抛 RetryableException，Sentinel 统计到异常而非慢调用 |
| 解决 | **铁律：Feign read-timeout > Sentinel 慢调用 RT 阈值** |

| Sentinel 慢调用阈值 | Feign read-timeout | 判定 |
| --- | --- | --- |
| 1000ms | 3000ms | 正确 |
| 1000ms | 800ms | 错误 |
| 2000ms | 2000ms | 危险（行为不稳定） |

### 坑 5：RocketMQ 消息重复消费

| 维度 | 内容 |
| --- | --- |
| 现象 | 消费者 ACK 未及时返回，Broker 重投，同一条订单被重复处理 |
| 根因 | 分布式消息"至少投递一次"语义，网络不可靠重复投递是常态 |
| 解决 | Redis setNX（拦截 99.9%）+ DB 唯一索引（兜底）双保险 |

```java
// 第一道防线：Redis setNX
Boolean isFirst = redisTemplate.opsForValue()
    .setIfAbsent("msg_dedup:" + messageKey, "1", 24, TimeUnit.HOURS);
if (Boolean.FALSE.equals(isFirst)) return;

// 第二道防线：DB 唯一索引
try {
    messageDedupMapper.insert(new MessageDedup(messageKey));
} catch (DuplicateKeyException e) {
    return;  // 重复消息，跳过
}
```

---

## 五、SCA 全系列组件速查

| 组件 | 角色比喻 | 解决的问题 |
| --- | --- | --- |
| Nacos Discovery | 电话本 | 谁在线？IP 是什么？ |
| Nacos Config | 总控制台 | 配置改了不用重启 |
| OpenFeign + LoadBalancer | 快递员 | 说找谁就找谁，自动选路 |
| Sentinel | 保险丝 | 流量太大？别进，等一等 |
| Spring Cloud Gateway | 门禁 | 进来前先验身份，然后分流 |
| RocketMQ | 邮件系统 | 异步通知，分批处理 |
| Seata | 合同律师 | 多个操作必须一起成功，否则全取消 |
| SkyWalking + Prometheus | X 光机 | 透明化，哪出了问题一眼看到 |

---

## 复习清单

1. **可观测性三根柱子？** Metrics（指标/CPU/QPS）、Tracing（链路/调用路径）、Logging（日志/事件记录）。
2. **SkyWalking 核心优势？** Java Agent 插桩，零代码改动，自动采集 HTTP/JDBC/Redis/MQ/Feign。
3. **如何关联 Tracing 和 Logging？** logback 打印 `%X{traceId}`，Kibana 按 traceId 搜索全链路日志。
4. **灰度发布五步流程？** 部署 v2 → Header 路由到 v2 → Sentinel 严格限流 → 逐步切流权重 → v1 下线。
5. **Nacos 缓存导致调用下线节点怎么解决？** 降低 LoadBalancer 缓存 TTL 到 10s；下线前先注销再关闭。
6. **Sentinel 规则重启丢失怎么解决？** 持久化到 Nacos，配置 datasource 指向 Nacos 的 JSON Data ID。
7. **Seata AT 模式死锁根因？** 全局行锁在高并发修改同一行时争抢导致雪崩；解决：热点改 Redis + 异步同步 DB 或切 TCC。
8. **Feign 超时与 Sentinel 慢调用阈值的铁律？** `Feign read-timeout > Sentinel 慢调用 RT 阈值`，否则 Sentinel 永远收不到慢调用数据。
9. **RocketMQ 消费幂等双保险？** Redis setNX 拦截 99.9% + DB 唯一索引兜底。
10. **自定义业务 Metrics 用什么？** Micrometer 的 Counter（计数）和 Timer（耗时 P50/P95/P99），配合 Prometheus + Grafana 可视化。
11. **生产环境持久化三件套？** Nacos 用 MySQL 存储注册/配置数据、Sentinel 规则推 Nacos、Seata Server 落库。
