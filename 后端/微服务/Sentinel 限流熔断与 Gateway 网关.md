# Sentinel 限流熔断与 Gateway 网关

> 来源：[Spring Cloud Alibaba 全栈技术专题（二）— Sentinel + Gateway 篇](https://zhuanlan.zhihu.com/p/2055576758343284022)
> 一句话总结：Sentinel 守住系统不被流量打垮（限流/降级/熔断），Gateway 统一入口做鉴权限流灰度，两者组合让系统从"能跑"升级到"扛得住"。

## 一、限流·降级·熔断 — Sentinel

### 1.1 三个概念精确区分

| 概念 | 触发条件 | 目的 | 恢复方式 |
| --- | --- | --- | --- |
| 限流 | 请求量超过阈值 | 保护下游资源 | 超出请求直接拒绝 |
| 降级 | 响应慢 / 错误率高 | 快速失败，避免资源耗尽 | 熔断窗口结束后探测恢复 |
| 熔断 | 广义的降级，强调断路器状态机 | 同上 | CLOSED → OPEN → HALF_OPEN |

> **类比**：限流 = 景区满员不卖票；降级 = 设施排队过久暂时关停；熔断 = 设施挂"暂停开放"牌子，过一段试着开。

### 1.2 Sentinel 处理链（Slot Chain）

基于**责任链模式**，每个 Slot 负责一类规则检查，任一触发即抛 `BlockException` 拦截请求：

```
请求进入
  └─> NodeSelectorSlot     （构建调用树）
        └─> ClusterBuilderSlot （统计集群数据）
              └─> StatisticSlot    （实时统计 QPS、RT、异常数）
                    └─> AuthoritySlot   （来源授权规则，黑白名单）
                          └─> SystemSlot      （系统自适应保护）
                                └─> FlowSlot        （流控规则）
                                      └─> DegradeSlot     （熔断降级规则）
                                            └─> 业务逻辑执行
```

> **核心优势**：早拦截，零消耗——请求在到达业务代码前就被拦截。

### 1.3 核心埋点：@SentinelResource

```java
@SentinelResource(
    value = "createOrder",              // 资源名（Dashboard 中可见）
    blockHandler = "createOrderBlock", // 限流/熔断处理（处理 BlockException）
    fallback = "createOrderFallback"   // 业务异常降级（处理 Throwable）
)
public Order createOrder(CreateOrderDTO dto) { ... }

// 限流处理：方法签名和原方法一致，最后加 BlockException 参数
public Order createOrderBlock(CreateOrderDTO dto, BlockException ex) {
    throw new TooManyRequestsException("系统繁忙，请稍后重试");
}

// 业务异常降级：最后加 Throwable 参数
public Order createOrderFallback(CreateOrderDTO dto, Throwable t) {
    throw new ServiceDegradedException("服务降级，订单创建失败");
}
```

**blockHandler vs fallback 区别：**

| 维度 | blockHandler | fallback |
| --- | --- | --- |
| 处理异常类型 | BlockException（限流/降级规则触发） | 业务异常（数据库挂了、下游 500） |
| 含义 | "系统主动保护了你" | "系统出了问题" |
| 优先级 | 先判断，是则走 blockHandler | 不是 BlockException 才走 fallback |

### 1.4 流控规则详解

**QPS 限流配置示例：**

```json
{
  "resource": "createOrder",
  "grade": 1,          // 1=QPS，0=并发线程数
  "count": 100,        // 阈值：100 QPS
  "strategy": 0,       // 0=直接，1=关联，2=链路
  "controlBehavior": 0 // 0=快速失败，1=Warm Up，2=排队等待
}
```

**三种流控效果对比：**

| 效果 | 原理 | 适用场景 |
| --- | --- | --- |
| 快速失败（0） | 超过阈值直接拒绝 | 最简单高效，通用场景 |
| Warm Up（1） | 从 1/3 阈值线性增长到满阈值（配合 warmUpPeriodSec） | **冷启动**，避免刚启动时被流量压垮 |
| 排队等待（2） | 超出阈值请求排队（配合 maxQueueingTimeMs） | **突发流量**削峰填谷 |

### 1.5 熔断降级规则

**三种熔断策略选择：**

| 策略 | grade | 适用场景 | 配置建议 |
| --- | --- | --- | --- |
| 慢调用比例 | 0 | 下游 RT 升高（还能响应但很慢） | count=1000ms, slowRatioThreshold=0.5 |
| 异常比例 | 1 | 下游 500 错误频繁 | count=0.5（异常比例 50%） |
| 异常数 | 2 | 低流量但完全不可用 | count=5（1 分钟内 5 个异常即熔断） |

**熔断器状态机：**

```
CLOSED ──(请求数>=minRequest 且 慢调用比例>=threshold)──> OPEN
  ^                                                      │
  │  探测请求成功                                          │ timeWindow 结束
  │                                                      ↓
  └─────────────────── HALF_OPEN <────────────────────────┘
                            │
                            │ 探测请求失败
                            ↓
                          OPEN（重新计时）
```

### 1.6 规则持久化（生产必做）

默认规则存 JVM 内存，**重启全丢**。生产必须持久化到 Nacos：

```java
@Configuration
public class SentinelNacosConfig {

    @PostConstruct
    public void init() {
        // 从 Nacos 加载流控规则
        ReadableDataSource<String, List<FlowRule>> flowRuleDS =
            new NacosDataSource<>(nacosAddr, "SENTINEL_GROUP",
                appName + "-flow-rules",
                source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
        FlowRuleManager.register2Property(flowRuleDS.getProperty());

        // 从 Nacos 加载熔断规则
        ReadableDataSource<String, List<DegradeRule>> degradeRuleDS =
            new NacosDataSource<>(nacosAddr, "SENTINEL_GROUP",
                appName + "-degrade-rules",
                source -> JSON.parseObject(source, new TypeReference<List<DegradeRule>>() {}));
        DegradeRuleManager.register2Property(degradeRuleDS.getProperty());
    }
}
```

> 持久化后，在 Nacos 控制台修改 JSON 规则，所有服务实例**实时生效**。

---

## 二、API 网关 — Spring Cloud Gateway

### 2.1 核心三要素

```
Route（路由）
  ├─ id：唯一标识
  ├─ uri：目标服务（lb://service-name 或 http://...）
  ├─ Predicate（断言）：请求匹配条件
  └─ Filter（过滤器）：请求/响应处理逻辑（鉴权、加 Header、限流等）
```

> Gateway 基于 **Spring WebFlux + Netty** 响应式模型，对比 Zuul 的 Servlet 阻塞模型，高并发性能优势显著。

### 2.2 路由配置示例

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://order-service          # lb:// 从注册中心负载均衡发现
          predicates:
            - Path=/api/order/**           # 路径匹配
            - Method=GET,POST              # HTTP 方法匹配
            - Header=X-Request-Source, app # Header 匹配
          filters:
            - StripPrefix=1               # 去掉 /api 前缀
            - AddRequestHeader=X-Gateway-Version, 1.0
            - name: RequestRateLimiter    # 网关级限流（需 Redis）
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@ipKeyResolver}"
```

### 2.3 自定义全局过滤器（JWT 鉴权）

生产最常用 Filter——**网关层统一验签，下游只关心业务**：

```java
@Component
@Order(-1)  // 值越小优先级越高，鉴权必须最先执行
public class AuthGlobalFilter implements GlobalFilter {

    private static final List<String> WHITE_LIST = Arrays.asList(
        "/api/user/login", "/api/user/register"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        // 白名单放行
        if (WHITE_LIST.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }
        // 提取并验证 Token
        String token = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);
        // 验证通过后，将用户信息透传给下游（避免重复解析）
        Claims claims = jwtUtils.parseToken(token.substring(7));
        ServerWebExchange mutated = exchange.mutate()
            .request(exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Role", claims.get("role", String.class))
                .build()).build();
        return chain.filter(mutated);
    }
}
```

### 2.4 灰度路由实现

基于请求 Header 版本标识将流量路由到不同版本实例——**零停机发布**核心能力：

```java
@Component
public class GrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        String version = headers.getFirst("X-Version");
        // 按 version 元数据匹配实例，无匹配则回退到全量实例池
        List<ServiceInstance> matched = instances.stream()
            .filter(i -> version.equals(i.getMetadata().get("version")))
            .collect(Collectors.toList());
        List<ServiceInstance> pool = matched.isEmpty() ? instances : matched;
        return new DefaultResponse(pool.get(random.nextInt(pool.size())));
    }
}
```

**灰度发布全流程：**
1. 新版本部署时在 Nacos 元数据标记 `version: v2`
2. 测试请求带 `X-Version: v2` Header → 路由到 v2 实例
3. 对 v2 实例设置更严格的 Sentinel 流控规则，防止新版本 bug 扩散
4. 验证通过后，Nacos 控制台逐步调高 v2 权重、调低 v1 权重，完成切流

### 2.5 生产建议

- **Gateway 只做"透传 + 切面"**，不要塞重业务逻辑
- **避免在 Filter 中做耗时 IO**，Gateway 基于 Netty 事件循环，同步阻塞严重拖累吞吐
- **结合 Sentinel 做网关限流**，流量入口就拦住，比每个服务各自限流更高效
- **开启 CORS 配置**，前后端分离场景必备

---

## 三、两大组件总结

| 组件 | 解决什么问题 | 关键要点 |
| --- | --- | --- |
| Sentinel | 防止系统雪崩 | 流控/熔断/降级三种策略各有场景；生产务必规则持久化到 Nacos |
| Gateway | 统一入口管理 | JWT 鉴权网关层统一做；灰度路由靠 Nacos 元数据 + 自定义 LoadBalancer |

---

## 复习清单

1. **限流、降级、熔断三者区别？** 限流=超量拒绝；降级=慢/错时快速失败；熔断=断路器状态机（CLOSED→OPEN→HALF_OPEN）。
2. **Sentinel 处理链设计模式？** 责任链模式，每个 Slot 检查一类规则，任一触发即抛 BlockException，早拦截零消耗。
3. **blockHandler 和 fallback 区别？** blockHandler 处理 BlockException（系统主动保护），fallback 处理业务异常（系统出了问题），blockHandler 优先级更高。
4. **三种流控效果分别适用什么场景？** 快速失败=通用；Warm Up=冷启动；排队等待=突发流量削峰。
5. **Warm Up 原理？** 从 1/3 阈值线性增长到满阈值，配合 warmUpPeriodSec 参数，避免冷启动被流量压垮。
6. **三种熔断策略的选择依据？** 慢调用比例（RT 高）、异常比例（500 频繁）、异常数（低流量但完全不可用）。
7. **熔断器状态机三个状态？** CLOSED（正常）→ OPEN（熔断中）→ HALF_OPEN（探测恢复）。
8. **Sentinel 规则为什么必须持久化？** 默认存 JVM 内存，重启全丢。生产持久化到 Nacos，修改后所有实例实时生效。
9. **Gateway 核心三要素？** Route（路由）+ Predicate（断言匹配）+ Filter（过滤器处理）。
10. **Gateway 为什么比 Zuul 性能好？** 基于 Spring WebFlux + Netty 响应式模型，非 Zuul 的 Servlet 阻塞模型。
11. **灰度路由怎么实现？** Nacos 元数据标记版本 + 自定义 LoadBalancer 按 Header 版本匹配实例 + 逐步切流。
