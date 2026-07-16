# Seata 分布式事务与 RocketMQ 消息

> 来源：[Spring Cloud Alibaba 全栈技术专题（三）— Seata + RocketMQ 篇](https://zhuanlan.zhihu.com/p/2055576856636793373)
> 一句话总结：Seata 解决跨服务同步事务一致性（AT/TCC/SAGA/XA），RocketMQ 事务消息解决异步最终一致性，两者覆盖微服务数据一致性的全部场景。

## 一、一致性问题的本质

```
用户下单
  ├─ 订单服务（DB1）：创建订单记录
  ├─ 库存服务（DB2）：扣减库存
  └─ 支付服务（DB3）：扣款
```

三个服务操作三个独立数据库，任何环节失败前面已成功操作必须回滚。传统数据库事务无法跨库——这就是分布式事务的"老大难"。

**两条解决路径：**
- **同步强一致** → Seata（AT / TCC / SAGA / XA）
- **异步最终一致** → RocketMQ 事务消息

---

## 二、Seata — 分布式事务

### 2.1 核心角色

| 角色 | 全称 | 职责 |
| --- | --- | --- |
| TC | Transaction Coordinator（事务协调器） | Server 端，维护全局事务状态 |
| TM | Transaction Manager（事务管理器） | 标注 `@GlobalTransactional` 的服务，负责开启/提交/回滚全局事务 |
| RM | Resource Manager（资源管理器） | 各参与者的数据库，管理本地分支事务 |

### 2.2 四种模式对比

| 模式 | 侵入性 | 性能 | 一致性 | 适用场景 |
| --- | --- | --- | --- | --- |
| AT | 低（注解即可） | 高 | 最终一致 | **主流选择**，关系型 DB |
| TCC | 高（需实现 3 个接口） | 高 | 最终一致 | 自定义资源（发红包、扣积分）、热点数据 |
| SAGA | 中（补偿逻辑） | 高 | 最终一致 | 长流程事务（审批流），补偿容易设计 |
| XA | 低 | 低（全程持锁） | 强一致 | 金融场景严格强一致 |

**选型口诀：**
- 能忍最终一致 + 关系型 DB → **AT 模式**
- 自定义资源或热点数据 → **TCC 模式**
- 长事务链路 → **SAGA 模式**
- 严格强一致 → **XA 模式**

### 2.3 AT 模式原理（最常用）

核心机制：**undo_log 表 + 全局锁**

**一阶段（提交前）：**
1. 解析业务 SQL，生成 `before image`（执行前数据快照）
2. 执行业务 SQL
3. 生成 `after image`（执行后数据快照）
4. 将 before/after image 序列化写入 `undo_log` 表
5. 向 TC 注册分支事务

**二阶段（全局提交/回滚）：**
- **全局提交**：TC 通知各 RM 提交，**异步删除** `undo_log` 记录
- **全局回滚**：TC 通知各 RM 回滚，RM 根据 `undo_log` 中 before image 生成**反向 SQL** 补偿

**undo_log 建表（每个参与者数据库都要建）：**

```sql
CREATE TABLE undo_log (
    id            BIGINT(20)   NOT NULL AUTO_INCREMENT,
    branch_id     BIGINT(20)   NOT NULL COMMENT '分支事务ID',
    xid           VARCHAR(100) NOT NULL COMMENT '全局事务ID',
    context       VARCHAR(128) NOT NULL COMMENT '序列化类型等上下文',
    rollback_info LONGBLOB     NOT NULL COMMENT 'before/after image',
    log_status    INT(11)      NOT NULL COMMENT '0=活跃 1=全局已完成',
    log_created   DATETIME(6)  NOT NULL,
    log_modified  DATETIME(6)  NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY ux_undo_log (xid, branch_id)
) ENGINE=InnoDB;
```

### 2.4 AT 模式快速接入

**依赖：**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

**配置（application.yml）：**

```yaml
seata:
  enabled: true
  application-id: order-service
  tx-service-group: order-tx-group
  service:
    vgroup-mapping:
      order-tx-group: default
  registry:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
```

**业务代码（只需一个注解）：**

```java
@GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
public void createOrder(CreateOrderDTO dto) {
    orderRepository.save(dto.toOrder());
    inventoryClient.deductStock(dto.getStockItems());  // Seata 通过 XID 自动传播
    paymentClient.pay(dto.getPayment());
    // 任何一步抛异常，Seata 自动二阶段回滚所有已提交分支
}
```

> Seata 会自动在 Feign 调用时通过 Request Header 传递全局事务 XID，对业务代码完全透明。

### 2.5 TCC 模式详解

AT 局限：不支持非关系型资源（Redis、MongoDB）、高并发热点数据全局锁可能成为瓶颈。TCC 为这些场景设计。

**TCC = Try + Confirm + Cancel：**

```java
@LocalTCC
public interface InventoryTccAction {
    @TwoPhaseBusinessAction(
        name = "inventoryDeduct",
        commitMethod = "commit",
        rollbackMethod = "rollback"
    )
    boolean try(@BusinessActionContextParameter(paramName = "skuId") Long skuId,
                @BusinessActionContextParameter(paramName = "count") int count,
                BusinessActionContext context);
    boolean commit(BusinessActionContext context);
    boolean rollback(BusinessActionContext context);
}
```

**三阶段职责：**

| 阶段 | 职责 | 类比 |
| --- | --- | --- |
| Try | 资源预留（冻结库存，不真正扣减） | 预授权 |
| Confirm | 将冻结转为真实操作 | 确认扣款 |
| Cancel | 释放冻结的资源 | 撤销预授权 |

**TCC 三大难题：**

| 难题 | 场景 | 解决方案 |
| --- | --- | --- |
| 空回滚 | Try 未执行但 Cancel 先到 | Cancel 中判空检查 |
| 悬挂 | Cancel 比 Try 先执行，Try 后到又执行一次 | Try 前检查是否已 Cancel |
| 幂等 | 任何阶段都可能重试 | 业务表加唯一约束或状态字段 |

### 2.6 生产建议

- **Seata Server 必须集群部署**：单点挂了整个分布式事务体系全瘫痪
- **undo_log 定期清理**：已完成事务的 undo_log 需要定时任务删除，否则表无限膨胀
- **AT 模式慎用于热点数据**：高并发修改同一行导致全局锁争抢，切 TCC

---

## 三、RocketMQ — 分布式消息

### 3.1 MQ 选型对比

| 特性 | RocketMQ | Kafka | RabbitMQ |
| --- | --- | --- | --- |
| 吞吐量 | 极高（10W+/s） | 极高（100W+/s） | 中等（万级/s） |
| 消息延迟 | 毫秒级 | 毫秒级 | 微秒级 |
| 事务消息 | 支持 | 不支持 | 不支持 |
| 顺序消息 | 队列级严格 | 分区有序 | 不支持 |
| 消息回溯 | 支持 | 支持 | 不支持 |
| 适用场景 | 电商、金融 | 日志、大数据 | 企业集成 |

> 如果同时用 Seata + RocketMQ，**事务消息**是两者结合的关键桥梁——解决"发消息和本地事务的一致性"难题。

### 3.2 Spring Cloud Stream 核心抽象

一层 Binder 抽象，**切换消息中间件只需换依赖，业务代码不变**：

```
应用代码
  └─> Binding（绑定器抽象层：input/output 通道）
        └─> Binder（具体实现：RocketMQ / Kafka / RabbitMQ）
              └─> 消息中间件
```

**快速接入配置：**

```yaml
spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
      bindings:
        order-out-0:              # 生产者输出通道
          destination: ORDER_TOPIC
          content-type: application/json
        order-in-0:               # 消费者输入通道
          destination: ORDER_TOPIC
          group: inventory-consumer-group
          consumer:
            orderly: false        # false=并发消费，true=顺序消费
```

**函数式消费（Spring Cloud Stream 3.x）：**

```java
@Bean
public Consumer<OrderCreatedEvent> orderIn() {
    return event -> inventoryService.deductStock(event.getItems());
}
```

**手动发送：**

```java
streamBridge.send("order-out-0", new OrderCreatedEvent(order.getId(), order.getItems()));
```

### 3.3 事务消息机制

RocketMQ 事务消息解决"本地事务 + 发消息"的一致性，比 Seata 侵入性更低：

```
生产者                          RocketMQ Broker               消费者
  │── 1. 发送半消息(PREPARE) ─────>│
  │<── 2. 返回半消息结果 ─────────│   （消息对消费者不可见）
  │   3. 执行本地事务              │
  │      成功 → COMMIT            │
  │      失败 → ROLLBACK          │
  │── 4. 提交/回滚 ──────────────>│
  │                               │── 5. COMMIT后推送 ──────>│
  │   6. [步骤4超时，Broker回查]   │
  │<── 7. 回查事务状态 ───────────│
  │── 8. 返回 COMMIT/ROLLBACK ───>│
```

**代码实现：**

```java
// 发送事务消息
rocketMQTemplate.sendMessageInTransaction(
    "order-out-0",
    MessageBuilder.withPayload(new OrderCreatedEvent(dto))
        .setHeader("orderId", dto.getOrderId()).build(),
    dto
);

// 事务监听器
@RocketMQTransactionListener
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            orderRepository.save(dto.toOrder());
            return COMMIT;    // 本地事务成功 → 消息对消费者可见
        } catch (Exception e) {
            return ROLLBACK;  // 本地事务失败 → 消息被丢弃
        }
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // Broker 超时未收到 COMMIT/ROLLBACK 时回调
        Order order = orderRepository.findById(orderId).orElse(null);
        return order != null ? COMMIT : ROLLBACK;
    }
}
```

### 3.4 消费幂等保障

分布式消息至少投递一次是常态，消费者必须幂等：

```java
@Transactional
public void onMessage(OrderCreatedEvent event) {
    String messageKey = event.getOrderId() + ":" + event.getEventType();
    // 方案1：Redis setNX（快，但 Redis 挂了不可靠）
    Boolean isFirst = redisTemplate.opsForValue()
        .setIfAbsent("msg_dedup:" + messageKey, "1", 24, TimeUnit.HOURS);
    if (Boolean.FALSE.equals(isFirst)) return;

    // 方案2：数据库唯一索引兜底（慢，但可靠）
    try {
        processedMessageRepository.save(new ProcessedMessage(messageKey));
    } catch (DataIntegrityViolationException e) {
        return;  // 重复消息，跳过
    }
    doProcess(event);
}
```

> **折中方案**：Redis 拦截 99.9% 重复 + DB 唯一索引兜底 Redis 故障时的漏网之鱼。

---

## 四、两大组件总结

| 组件 | 解决什么问题 | 关键要点 |
| --- | --- | --- |
| Seata | 跨服务同步事务一致性 | AT 模式最常用（undo_log + 全局锁）；TCC 给热点场景；四种模式按需选 |
| RocketMQ | 异步解耦 + 最终一致性 | 事务消息是"本地事务 + 发消息"一致性的利器；消费务必幂等 |

---

## 复习清单

1. **分布式事务的"老大难"是什么？** 多服务操作多独立数据库，传统事务无法跨库，任一环节失败需回滚。
2. **Seata 三大角色？** TC（事务协调器，Server）、TM（事务管理器，@GlobalTransactional）、RM（资源管理器，各参与者 DB）。
3. **Seata 四种模式选型口诀？** 关系型 DB → AT；自定义资源/热点 → TCC；长流程 → SAGA；强一致 → XA。
4. **AT 模式核心机制？** undo_log 表（存 before/after image）+ 全局锁。一阶段提交并记录快照，二阶段提交删 undo_log / 回滚用反向 SQL 补偿。
5. **AT 模式业务代码怎么用？** 只需 `@GlobalTransactional(rollbackFor = Exception.class)`，Seata 自动通过 Feign Header 传播 XID。
6. **TCC 三阶段？** Try（资源预留/冻结）→ Confirm（确认执行）→ Cancel（释放/回滚）。
7. **TCC 三大难题？** 空回滚（Try 未执行 Cancel 先到）、悬挂（Cancel 比 Try 先执行）、幂等（重复重试）。
8. **RocketMQ 对比 Kafka 的优势？** 支持事务消息、队列级严格顺序消息、消息回溯，更适合电商/金融场景。
9. **Spring Cloud Stream 的作用？** Binder 抽象层，切换消息中间件只需换依赖，业务代码不变。
10. **RocketMQ 事务消息流程？** 发半消息 → 执行本地事务 → COMMIT/ROLLBACK → Broker 超时回查 → 最终推送。
11. **消费幂等怎么做？** Redis setNX 拦截 99.9% + DB 唯一索引兜底。
