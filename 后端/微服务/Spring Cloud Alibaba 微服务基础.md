# Spring Cloud Alibaba 微服务基础

> 来源：[Spring Cloud Alibaba 全栈技术专题（一）— Nacos 篇](https://zhuanlan.zhihu.com/p/2055576247065956791)
> 一句话总结：Nacos 解决服务注册发现与动态配置，OpenFeign + LoadBalancer 解决声明式调用与负载均衡，三者构成微服务基础设施底座。

## 一、SCA 选型背景

### 1.1 微服务框架演进

```
单体应用
  └─> Spring Cloud Netflix（2016～2018）
        Eureka + Ribbon + Hystrix + Zuul + Feign
        └─> Netflix 组件进入维护模式（2018）
              └─> Spring Cloud Alibaba（2018 至今）
                    Nacos + Sentinel + RocketMQ + Seata + Gateway
```

Netflix 2018 年宣布核心组件（Hystrix、Ribbon、Eureka）进入维护模式，推动阿里巴巴开源 SCA，2019 年正式成为 Spring Cloud 官方孵化项目。

### 1.2 Netflix vs SCA 核心组件对照

| 功能 | Netflix 方案 | SCA 方案 | 核心优势 |
| --- | --- | --- | --- |
| 服务注册发现 | Eureka | Nacos Discovery | 支持 AP/CP 切换，内置配置中心 |
| 配置中心 | Spring Cloud Config | Nacos Config | 动态推送，无需 Bus |
| 负载均衡 | Ribbon | Spring Cloud LoadBalancer | 官方维护，响应式支持 |
| API 网关 | Zuul | Spring Cloud Gateway | 响应式，性能更高 |

### 1.3 版本兼容矩阵

| Spring Cloud Alibaba | Spring Cloud | Spring Boot |
| --- | --- | --- |
| 2022.0.0.0 | 2022.0.0 (Kilburn) | 3.0.x |
| 2021.0.5.0 | 2021.0.5 (Jubilee) | 2.6.x |
| 2.2.10.RELEASE | Hoxton.SR12 | 2.3.x |

> **建议**：新项目选 `2022.0.0.0` + Spring Boot 3.x；存量升级重点验证版本兼容性。

---

## 二、Nacos Discovery — 服务注册发现

### 2.1 Nacos 三大优势

1. **一个组件两种能力**：既是注册中心，又是配置中心
2. **支持 AP/CP 切换**：可根据场景选择一致性模型
3. **主动健康检查**：对持久实例支持 TCP/HTTP/MySQL 等多种探测

### 2.2 数据模型（四级层次）

```
Namespace（命名空间）       — 隔离环境（dev/test/prod）
  └─ Group（分组）          — 隔离业务域，默认 DEFAULT_GROUP
       └─ Service（服务）   — 具体微服务
            ├─ 临时实例（Ephemeral）  — 客户端心跳，默认
            └─ 持久实例（Persistent） — 服务端健康检查
```

### 2.3 AP vs CP 模式对比

| 维度 | AP 模式（临时实例，默认） | CP 模式（持久实例） |
| --- | --- | --- |
| 协议 | Distro（阿里自研） | Raft |
| 适用场景 | 大多数微服务注册 | 配置信息、不频繁变动的元数据 |
| 节点故障处理 | 自动摘除不健康实例 | 保留实例，标记为不健康 |
| 数据一致性 | 最终一致 | 强一致 |

> **选择建议**：99% 场景用 AP（临时实例）。仅 DNS 元数据、核心路由表等极少变更但一致性要求极高的场景才用 CP。临时实例默认 **15 秒** 无心跳即摘除。

### 2.4 快速接入

**依赖（pom.xml）：**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>2022.0.0.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

**配置（application.yml）：**

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev-namespace-id   # 命名空间 ID，非名称
        group: ORDER_GROUP
        weight: 1.0                   # 权重 0~1，影响负载均衡概率
        metadata:
          version: v1
          region: cn-east
```

**启动类（SCA 2021+ 可省略 `@EnableDiscoveryClient`）：**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderApplication { ... }
```

### 2.5 生产建议

- **集群部署**：至少 3 节点，奇数个，避免脑裂
- **数据持久化**：生产必须配置 MySQL 存储，否则重启丢数据
- **心跳超时**：`heart-beat-timeout` 默认 15s，网络抖动时适当调大
- **注销保护**：开启 `protectThreshold`，防止大量实例下线触发雪崩
- **优雅下线**：先调注销 API，再关闭应用

---

## 三、Nacos Config — 分布式配置中心

### 3.1 核心价值

传统痛点：改配置需重新打包、发布、重启。Nacos Config 通过**长轮询 + 主动推送**实现配置实时生效，消灭"重启发版"。

### 3.2 长轮询机制

```
客户端                              Nacos Server
  │── 发起请求（携带 MD5 checksum）──────>│
  │                                     │ 等待配置变化（最多 30s）
  │<── 返回变更的 Data ID 列表 ──────────│
  │── 根据 Data ID 拉取最新配置 ─────────>│
  │<── 返回配置内容 ─────────────────── │
  │ 本地缓存更新，触发 @RefreshScope ────│
```

> **兜底降级**：服务端挂了，客户端可从本地缓存（`${user.home}/nacos/config`）读取上次拉到的配置，保障服务可用。

### 3.3 快速接入

**依赖：**

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!-- Spring Boot 2.4+ 需额外引入 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

**配置（bootstrap.yml，优先级高于 application.yml）：**

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev-namespace-id
        group: ORDER_GROUP
        file-extension: yaml
        # Data ID 规则：${spring.application.name}.${file-extension}
        # 即：order-service.yaml
        extension-configs:
          - data-id: common-datasource.yaml
            group: COMMON_GROUP
            refresh: true
          - data-id: common-redis.yaml
            group: COMMON_GROUP
            refresh: true
```

**动态刷新 Bean：**

```java
@RestController
@RefreshScope  // 配置变更时销毁并重新创建该 Bean
public class OrderController {
    @Value("${order.timeout:5000}")
    private int timeout;

    @GetMapping("/timeout")
    public String getTimeout() {
        return "当前超时配置: " + timeout + "ms";
    }
}
```

### 3.4 @RefreshScope 原理

Spring 为每个被注解的 Bean 创建**作用域代理（Scope Proxy）**，实际 Bean 实例存在缓存中。配置变更回调触发 `RefreshScope.refreshAll()` → 缓存清空 → 下次访问重新创建 Bean → 注入最新配置。

> **注意**：`@RefreshScope` 每次调用走代理，有微性能开销，不要滥用，只对真正需要动态刷新的 Bean 使用。

### 3.5 多环境隔离方案

| 隔离方式 | 隔离级别 | 适用场景 |
| --- | --- | --- |
| Namespace | 物理隔离 | **推荐**用于环境隔离（dev/test/prod） |
| Group | 逻辑分组 | 同环境下不同业务域隔离 |

```
Namespace: dev-xxxxxxxx   → 开发环境所有服务配置
Namespace: test-xxxxxxxx  → 测试环境所有服务配置
Namespace: prod-xxxxxxxx  → 生产环境所有服务配置
```

### 3.6 生产建议

- **配置分层**：公共配置（数据库、Redis）放 `common-*.yaml`，业务配置放各服务私有 Data ID
- **灰度发布**：Nacos 支持按 IP 灰度推送配置，大改动先验证少量实例
- **回滚**：Nacos 控制台保留历史版本，可一键回滚
- **密码安全**：不要明文存密码，结合 Jasypt 加密或接入 KMS

---

## 四、OpenFeign + LoadBalancer — 声明式调用与负载均衡

### 4.1 演进路线

```
RestTemplate（手写 URL）
  └─> Feign（声明式，Netflix）
        └─> OpenFeign（官方维护，Spring 深度集成）
```

OpenFeign 让你**像调用本地方法一样调用远程服务**——定义接口 + 注解，框架自动生成 HTTP 客户端。

### 4.2 快速接入

**依赖：**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**定义 Feign 客户端：**

```java
@FeignClient(
    name = "inventory-service",       // Nacos 注册的服务名
    path = "/api/inventory",
    fallback = InventoryClientFallback.class   // 降级类
)
public interface InventoryClient {

    @GetMapping("/stock/{skuId}")
    StockDTO getStock(@PathVariable("skuId") Long skuId);

    @PostMapping("/deduct")
    Boolean deductStock(@RequestBody DeductRequest request);
}
```

**降级实现：**

```java
@Component
public class InventoryClientFallback implements InventoryClient {
    @Override
    public StockDTO getStock(Long skuId) {
        return StockDTO.builder().skuId(skuId).stock(0).build();
    }

    @Override
    public Boolean deductStock(DeductRequest request) {
        throw new ServiceException("库存服务暂不可用");
    }
}
```

### 4.3 负载均衡策略

| 策略 | 说明 |
| --- | --- |
| 轮询（RoundRobin，默认） | 按顺序依次分配 |
| 随机（Random） | 随机选择实例 |
| Nacos 权重联动 | 根据实例权重分配流量 |

> Nacos 控制台调整某实例权重为 0.5，即分配 50% 流量。需自定义 `NacosWeightedLoadBalancer` Bean 实现权重联动。

### 4.4 关键配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:              # 全局配置
            connect-timeout: 2000
            read-timeout: 5000
            logger-level: BASIC
          inventory-service:    # 针对特定服务
            connect-timeout: 1000
            read-timeout: 3000
      okhttp:
        enabled: true           # 开启 okhttp（生产推荐）
      compression:
        request:
          enabled: true         # Gzip 压缩
        response:
          enabled: true
```

### 4.5 生产建议

- **优先用 OkHttp**：连接池复用 + HTTP/2 支持，性能比默认 URLConnection 差 3~5 倍
- **超时配置**：connect-timeout 2s 足够；read-timeout 按业务设置（订单 500ms，报表 30s）
- **日志级别**：开发用 FULL，生产用 BASIC 或 NONE
- **Feign 拦截器**：统一透传 traceId、userId 等上下文，避免手动传参

---

## 五、三大组件总结

| 组件 | 解决什么问题 | 关键要点 |
| --- | --- | --- |
| Nacos Discovery | 服务的"电话本" | AP 模式 99% 场景够用，生产必须 MySQL 持久化 |
| Nacos Config | 配置动态生效 | `@RefreshScope` 实现零重启切换配置，长轮询 + 本地缓存兜底 |
| OpenFeign + LoadBalancer | 声明式远程调用 + 负载均衡 | 接口即客户端，Nacos 权重直接驱动流量分配 |

---

## 复习清单

1. **为什么从 Netflix 迁移到 SCA？** Netflix 核心组件 2018 年进入维护模式，不再添加新功能。
2. **Nacos 相比 Eureka 的三大优势？** 注册+配置二合一、AP/CP 可切换、主动健康检查。
3. **Nacos 数据模型四级层次？** Namespace → Group → Service → Instance。
4. **AP 和 CP 模式怎么选？** 99% 用 AP（临时实例）；仅元数据等极少变更且一致性要求极高的场景用 CP。
5. **临时实例多久被摘除？** 默认 15 秒无心跳即自动摘除。
6. **Nacos Config 配置实时生效原理？** 长轮询 + 推送，客户端监听变更后触发 `@RefreshScope` 重新创建 Bean。
7. **`@RefreshScope` 有什么副作用？** 每次调用走代理有微性能开销，不要滥用。
8. **多环境隔离推荐用什么？** Namespace（物理隔离），而非 Group（逻辑分组）。
9. **OpenFeign 的 fallback 是什么？** 服务降级实现，远程调用失败时返回兜底数据或抛出业务异常。
10. **生产环境 Feign 用什么 HTTP 客户端？** OkHttp（连接池复用 + HTTP/2），性能比默认 URLConnection 高 3~5 倍。
11. **Nacos 集群部署最少几个节点？** 至少 3 个，奇数个，避免脑裂。
