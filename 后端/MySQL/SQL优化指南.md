# SQL 常见优化指南

> 一句话总结：SQL 优化的核心思路是**减少数据访问量**（少扫行、少回表）和**利用索引加速查询**（建对索引、用对索引），辅以表结构设计和参数调优。

## 一、索引优化

### 1.1 索引设计原则

| 原则 | 说明 |
|------|------|
| 区分度高的列优先 | 区分度越高，索引过滤效果越好（如手机号 > 性别） |
| 联合索引把区分度高的列放左边 | 最左前缀匹配，高区分度列靠前能更快缩小范围 |
| 覆盖索引减少回表 | 将查询的列包含在联合索引中，索引叶子节点直接返回数据 |
| 控制索引数量 | 索引越多，写操作越慢（INSERT/UPDATE/DELETE 都要维护索引） |
| 短字段优先 | 索引列越短，一个页能存更多索引项，B+树更矮，I/O 更少 |

### 1.2 索引失效的常见场景

| 场景 | 示例 | 原因 |
|------|------|------|
| 对索引列做函数运算 | `WHERE YEAR(create_time) = 2024` | 索引列被函数包裹，无法走索引 |
| 隐式类型转换 | `WHERE phone = 13800001111`（phone 是字符串） | 字符串与数字比较，MySQL 隐式转换导致不走索引 |
| 左模糊查询 | `WHERE name LIKE '%小林'` | 以通配符开头，无法利用索引的有序性 |
| 使用 OR 连接非索引列 | `WHERE name = 'a' OR age = 20`（age 无索引） | OR 导致优化器放弃索引 |
| 不符合最左前缀 | 联合索引(a,b,c)，查询只用 `WHERE c = 1` | 联合索引从左到右匹配，跳过左边列则失效 |
| 使用 `!=` 或 `NOT IN` | `WHERE status != 0` | 优化器倾向于全表扫描 |
| `IS NOT NULL` | `WHERE name IS NOT NULL` | 大部分值为 NOT NULL 时，优化器倾向全表扫描 |

**正确写法对比：**

```sql
-- 错误：函数导致索引失效
SELECT * FROM user WHERE YEAR(create_time) = 2024;
-- 正确：范围查询走索引
SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- 错误：左模糊
SELECT * FROM user WHERE name LIKE '%小林';
-- 正确：右模糊可走索引
SELECT * FROM user WHERE name LIKE '小林%';
```

## 二、查询语句优化

### 2.1 避免 SELECT *

```sql
-- 差：取所有列，无法利用覆盖索引
SELECT * FROM order WHERE user_id = 1;
-- 好：只取需要的列，可利用覆盖索引
SELECT id, amount, status FROM order WHERE user_id = 1;
```

### 2.2 避免子查询，改用 JOIN

```sql
-- 差：子查询可能产生临时表
SELECT * FROM user WHERE id IN (SELECT user_id FROM order WHERE amount > 100);
-- 好：改用 JOIN
SELECT DISTINCT u.* FROM user u INNER JOIN order o ON u.id = o.user_id WHERE o.amount > 100;
```

### 2.3 避免在大表上做 JOIN

| 问题 | 优化方案 |
|------|---------|
| JOIN 的两张表都很大 | 分库分表，确保 JOIN 在分片内完成 |
| 驱动表选择不当 | 小表驱动大表（优化器通常会自动选择） |
| JOIN 列无索引 | 确保 JOIN 的关联列有索引 |

### 2.4 用 EXISTS 替代 IN（子查询结果集大时）

```sql
-- 当子查询结果集较大时，IN 性能差
SELECT * FROM user WHERE id IN (SELECT user_id FROM order WHERE status = 1);
-- EXISTS 更高效：找到匹配即返回，不需要遍历全部
SELECT * FROM user u WHERE EXISTS (SELECT 1 FROM order o WHERE o.user_id = u.id AND o.status = 1);
```

### 2.5 LIMIT 深度分页优化

| 方案 | SQL | 适用场景 |
|------|-----|---------|
| 子查询延迟关联 | `WHERE id >= (SELECT id FROM t ORDER BY id LIMIT 1000000, 1)` | 主键索引 |
| 延迟关联 | `t1 JOIN (SELECT id FROM t ORDER BY name LIMIT 1000000, 10) t2 ON t1.id = t2.id` | 非主键索引 |
| 游标分页 | `WHERE id > {last_id} ORDER BY id LIMIT 10` | 数据导出、瀑布流 |

> 详见 [MySQL 分页优化](MySQL分页优化.md)

### 2.6 COUNT 优化

| 写法 | 性能 | 说明 |
|------|------|------|
| `COUNT(*)` | 好 | InnoDB 会选最小的二级索引遍历，不取行数据 |
| `COUNT(1)` | 好 | 与 `COUNT(*)` 等价，InnoDB 优化后无区别 |
| `COUNT(列名)` | 差 | 需要取列值判断是否为 NULL，更慢 |
| `COUNT(DISTINCT 列名)` | 更差 | 需要额外去重操作 |

## 三、EXPLAIN 执行计划分析

### 3.1 关键字段解读

| 字段 | 关注点 |
|------|--------|
| **type** | 访问类型，从好到差：NULL > system > const > eq_ref > ref > range > index > ALL |
| **key** | 实际使用的索引，NULL 表示没用索引 |
| **key_len** | 使用的索引长度，越长说明用的索引列越多 |
| **rows** | 预估扫描行数，越少越好 |
| **Extra** | 额外信息（见下表） |

### 3.2 Extra 常见值

| Extra 值 | 含义 | 好坏 |
|----------|------|------|
| Using index | 覆盖索引，不需要回表 | 好 |
| Using where | 在存储引擎取数后 Server 层再过滤 | 一般 |
| Using index condition | 索引下推（ICP），在存储引擎层过滤 | 好 |
| Using temporary | 使用了临时表 | 差 |
| Using filesort | 需要额外排序（内存或磁盘） | 差 |
| Select tables optimized away | 某些聚合操作直接从索引获取，无需访问表 | 好 |

## 四、表结构优化

### 4.1 字段类型选择

| 原则 | 说明 |
|------|------|
| 能用数字就不用字符串 | 数字比较比字符串快，占用空间小 |
| 能用小类型就不用大类型 | `TINYINT`（1 字节）> `INT`（4 字节）> `BIGINT`（8 字节） |
| 固定长度用 CHAR，可变长度用 VARCHAR | CHAR 填充空格，VARCHAR 按实际长度存储 |
| 时间类型用 DATETIME 或 TIMESTAMP | 避免用字符串存时间，否则无法利用时间函数优化 |
| 避免 NULL | NULL 值需要额外空间标记，且影响索引效率，设 NOT NULL + 默认值 |

### 4.2 范式与反范式

| 维度 | 范式（减少冗余） | 反范式（适当冗余） |
|------|---------------|-----------------|
| 优点 | 减少数据冗余，更新一致性好 | 减少JOIN，查询性能好 |
| 缺点 | 查询需要多表 JOIN | 数据冗余，更新需维护多处 |
| 适用 | 写多读少，数据一致性要求高 | 读多写少，查询性能优先 |

> 实际工程中通常是**折中**：核心数据保持范式，频繁查询的统计/冗余字段采用反范式。

### 4.3 大表拆分

| 策略 | 说明 |
|------|------|
| 垂直拆分 | 大表拆为多张小表，按功能/访问频率分离（如基础信息表 + 扩展信息表） |
| 水平拆分 | 按规则分到多张结构相同的表（如按 user_id 取模分 16 张表） |

## 五、参数与架构优化

### 5.1 InnoDB 关键参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `innodb_buffer_pool_size` | Buffer Pool 大小，缓存数据页和索引页 | 物理内存的 50%~70% |
| `innodb_log_buffer_size` | redo log buffer 大小 | 16MB~64MB |
| `innodb_flush_log_at_trx_commit` | redo log 刷盘策略 | 1（安全）/ 2（折中） |
| `innodb_io_capacity` | InnoDB I/O 容量 | SSD: 2000；HDD: 200 |

### 5.2 架构层面优化

| 方案 | 说明 |
|------|------|
| 读写分离 | 主库写，从库读，降低主库压力 |
| 缓存（Redis） | 热点数据放缓存，减少数据库查询 |
| 分库分表 | 单表数据量 > 千万时考虑水平拆分 |
| 引入 ES | 复杂搜索/全文检索场景，用 Elasticsearch 替代 MySQL |

## 六、优化思路总结

| 层次 | 手段 | 效果 |
|------|------|------|
| **SQL 层** | 避免 SELECT *、用 JOIN 替子查询、正确使用 LIMIT | 减少数据传输和计算量 |
| **索引层** | 合理建索引、避免索引失效、利用覆盖索引 | 减少扫描行数和回表次数 |
| **表结构层** | 合理选类型、避免 NULL、适当反范式 | 从存储层面减少 I/O |
| **参数层** | 调整 Buffer Pool、redo log 刷盘策略 | 提升整体吞吐量 |
| **架构层** | 读写分离、缓存、分库分表、引入 ES | 从根本上解决数据量和并发问题 |

> **优化原则**：先 EXPLAIN 看执行计划定位瓶颈 → SQL/索引层面优化 → 表结构优化 → 参数调优 → 架构优化。不要上来就加索引或改架构。

## 七、复习清单

1. **索引设计的核心原则？** 区分度高的列优先、联合索引高区分度列放左边、覆盖索引减少回表、控制索引数量、短字段优先。
2. **哪些操作会导致索引失效？** 函数运算、隐式类型转换、左模糊、OR 连接非索引列、不符合最左前缀、`!=`、`NOT IN`。
3. **如何避免隐式类型转换？** 保证查询值与字段类型一致，字符串字段用引号包裹。
4. **为什么避免 SELECT *？** 无法利用覆盖索引，增加网络传输和内存开销。
5. **EXPLAIN 中 type 字段从好到差？** NULL > system > const > eq_ref > ref > range > index > ALL。
6. **EXPLAIN 中 Extra 哪些值表示差？** Using temporary（临时表）和 Using filesort（额外排序）。
7. **COUNT(*) 和 COUNT(列名) 的区别？** `COUNT(*)` 不取行值直接计数，`COUNT(列名)` 需要取值判断 NULL，更慢。
8. **覆盖索引是什么？** 查询的列全部包含在索引中，无需回表查数据行，Extra 显示 Using index。
9. **范式和反范式怎么选？** 核心数据保持范式保证一致性，频繁查询的冗余字段用反范式提升查询性能。
10. **Buffer Pool 建议设多大？** 物理内存的 50%~70%，用于缓存数据页和索引页。
11. **SQL 优化的整体思路？** EXPLAIN 定位瓶颈 → SQL/索引优化 → 表结构优化 → 参数调优 → 架构优化。
12. **索引下推（ICP）是什么？** 在存储引擎层利用索引过滤条件，减少回表次数，Extra 显示 Using index condition。