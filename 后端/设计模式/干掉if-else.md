# 干掉 if...else：六大方案对比

> 一句话总结：**消灭 if...else 的本质是「用配置/多态替代分支」**——按「code 是否带业务含义」「场景是否固定流程」选择注解、bean 名、模板方法、策略工厂、责任链、枚举等方案。

---

## 1. 反例：又臭又长的 if...else

```java
public void toPay(String code) {
    if ("alia".equals(code))      { aliaPay.pay(); }
    else if ("weixin".equals(code))   { weixinPay.pay(); }
    else if ("jingdong".equals(code)) { jingDongPay.pay(); }
    else { System.out.println("找不到支付方式"); }
}
```

**问题**：每加一个支付方式都要改 `toPay` 的判断逻辑，违反**开闭原则**（对扩展开放、对修改关闭）和**单一职责原则**。

> **小结**：
> - 分支臃肿 → 可读性差、易遗漏
> - 每改一次都要动同一处方法 → 风险集中
> - 解决思路：**建立 code ↔ 实现的映射关系**，把判断交给容器/枚举/链

---

## 2. 方案对比表

| 方案 | 适用场景 | 是否要业务 code | 容器介入 | 复杂度 | 核心思想 |
|---|---|---|---|---|---|
| **注解 + 监听** | code 可任意（纯数字也行） | 否 | ApplicationListener | 中 | 启动时扫描 `@PayCode` 注解建立 Map |
| **动态拼接 bean 名** | code 有业务含义且稳定 | 是（= bean 前缀） | ApplicationContextAware | 低 | `alia` → bean `aliaPay` |
| **模板方法判断** | 同一接口多个实现，按类型挑 | 否 | InitializingBean | 中 | 实现类自带 `support(code)` 自荐 |
| **策略 + 工厂** | code 有业务含义 | 是 | @PostConstruct | 中 | 工厂类持全局 Map 注册实例 |
| **责任链** | 多步骤校验、可动态增删节点 | 否 | Spring 注入 List | 中高 | 链式传递，不匹配则 next |
| **枚举** | 数字 ↔ 字符串等简单映射 | 否 | 无 | 最低 | `enum` 替代表达式 |
| **三目/Stream** | 简单二选一或集合过滤 | 否 | 无 | 最低 | 语言级语法糖 |

> **小结**：
> - **没有银弹**：选型取决于「**code 是否有业务含义**」「**调用流程是否固定**」
> - 推荐优先级：**枚举 > 注解/拼接 > 策略工厂 > 责任链 > 模板方法**
> - Spring 容器中**优先利用 `ApplicationContextAware` / `ApplicationListener` 自动注册**

---

## 3. 方案一：注解 + ApplicationListener

**核心**：自定义注解 `@PayCode` + 监听 `ContextRefreshedEvent` → 启动时把带注解的 Bean 收入 `Map<value, IPay>`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface PayCode {
    String value();   // code
    String name();    // 业务名
}

@PayCode(value = "alia", name = "支付宝")
@Service
public class AliaPay implements IPay { /* ... */ }

@Service
public class PayService2 implements ApplicationListener<ContextRefreshedEvent> {
    private static Map<String, IPay> payMap = null;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent e) {
        Map<String, Object> beans = e.getApplicationContext()
                                     .getBeansWithAnnotation(PayCode.class);
        beans.forEach((k, v) -> payMap.put(
            v.getClass().getAnnotation(PayCode.class).value(),
            (IPay) v));
    }

    public void pay(String code) { payMap.get(code).pay(); }
}
```

> **小结**：
> - 优点：加新支付**零改动**，只需加一个带注解的类
> - 缺点：启动期才建立 Map，启动失败排查成本高

---

## 4. 方案二：动态拼接 bean 名

**核心**：约定 bean 命名规则 = `code + "Pay"`，通过 `getBean(code + SUFFIX)` 取实例。

```java
@Service
public class PayService3 implements ApplicationContextAware {
    private ApplicationContext ctx;
    private static final String SUFFIX = "Pay";

    public void toPay(String payCode) {
        ((IPay) ctx.getBean(payCode + SUFFIX)).pay();
    }
}
```

> **小结**：
> - 优点：实现最简单，零注解
> - 缺点：**bean 命名受约束**，改名要同步改 code

---

## 5. 方案三：模板方法判断（`support()` 自荐）

**核心**：在接口加 `boolean support(String code)`，容器初始化时把所有实现类塞进 `List`，调用时**遍历匹配**。

```java
public interface IPay {
    boolean support(String code);
    void pay();
}

@Service
public class PayService4 implements ApplicationContextAware, InitializingBean {
    private List<IPay> payList;

    @Override
    public void afterPropertiesSet() {
        payList = new ArrayList<>(
            applicationContext.getBeansOfType(IPay.class).values());
    }

    public void toPay(String code) {
        for (IPay p : payList) {
            if (p.support(code)) { p.pay(); return; }
        }
    }
}
```

> **小结**：
> - 借鉴 Spring `DefaultAdvisorAdapterRegistry.wrap()` 的思想
> - 缺点：`for` 循环仍存在，匹配是 O(N)，节点多时性能差

---

## 6. 方案四：策略 + 工厂

**核心**：策略类用 `@PostConstruct` 自注册到工厂的静态 Map。

```java
public class PayStrategyFactory {
    private static Map<String, IPay> PAY_REGISTERS = new HashMap<>();
    public static void register(String code, IPay pay) { PAY_REGISTERS.put(code, pay); }
    public static IPay get(String code) { return PAY_REGISTERS.get(code); }
}

@Service
public class AliaPay implements IPay {
    @PostConstruct
    public void init() { PayStrategyFactory.register("aliaPay", this); }
    public void pay() { /* ... */ }
}

@Service
public class PayService3 {
    public void toPay(String code) { PayStrategyFactory.get(code).pay(); }
}
```

> **小结**：
> - 策略模式 + 工厂模式合体，调用方完全无 if
> - 缺点：工厂持**静态 Map**，与 Spring 容器生命周期耦合

---

## 7. 方案五：责任链

**核心**：每个 handler 持 `next` 引用，不匹配就 `getNext().pay(code)`，由 `InitializingBean` 在启动时把 Spring 注入的 handler List 串成链。

```java
public abstract class PayHandler {
    @Getter @Setter protected PayHandler next;
    public abstract void pay(String code);
}

@Service
public class AliaPayHandler extends PayHandler {
    public void pay(String code) {
        if ("alia".equals(code)) { /* 支付宝支付 */ }
        else { getNext().pay(code); }   // 不归我管 → 下一个
    }
}
```

> **小结**：
> - 适用：多步骤、节点顺序重要、可**动态增删节点**
> - 缺点：链长时调试栈深、易在链中断点

---

## 8. 其他场景的简化技巧

### 8.1 数字 ↔ 字符串：枚举

```java
public enum MessageEnum {
    SUCCESS(1, "成功"), FAIL(-1, "失败"),
    TIME_OUT(-2, "网络超时"), PARAM_ERROR(-3, "参数错误");
    public static MessageEnum of(int code) {
        return Arrays.stream(values()).filter(x -> x.code == code)
                     .findFirst().orElse(null);
    }
}
```

### 8.2 简单二选一：三目运算符

```java
return code == 1 ? "成功" : "失败";
```

### 8.3 集合过滤：Stream 替 for

```java
return Arrays.stream(values()).filter(x -> x.code == code).findFirst().orElse(null);
```

### 8.4 参数校验：Spring `Assert`

```java
Assert.notNull(code, "code不能为空");
Assert.notNull(name, "name不能为空");
```

> **小结**：
> - 能用**语法糖**就别上设计模式
> - **枚举**几乎可以消灭所有「int → 字符串」类判断
> - 参数校验用 `Assert` 一行搞定

---

## 9. 复习清单

1. **消灭 if...else 的本质思路？** 建立 code 与实现的映射关系，让容器/枚举/链来匹配。
2. **`ApplicationListener` 与 `ApplicationContextAware` 的区别？** 前者监听事件被动触发，后者主动注入 `ApplicationContext`。
3. **`@PayCode` 注解方案的核心步骤？** 注解 + 监听 `ContextRefreshedEvent` + `getBeansWithAnnotation` 建 Map。
4. **拼接 bean 名方案的硬约束？** bean 名必须 = `code + 固定后缀`。
5. **模板方法判断方案的缺点？** 仍存在 `for` 循环，O(N) 匹配。
6. **策略 + 工厂中工厂类为何用 static Map？** 全局共享、无需注入即可访问。
7. **责任链模式的关键字段？** 每个 handler 持 `next` 引用 + 容器启动期 `setNext` 串链。
8. **阿里开发手册对异常的硬性规定？** 禁止用异常做流程/条件控制，异常处理效率低。
9. **枚举比 if...else 的优势？** 类型安全、可遍历、可附加方法。
10. **三目运算符的适用边界？** 仅限二选一简单分支，多分支可读性反而下降。
11. **Spring `Assert` 替代了什么？** 一堆 `if (x == null) throw` 的样板代码。
12. **选型决策树？** 有业务 code → 注解/拼接；固定流程 → 模板；多节点校验 → 责任链；简单映射 → 枚举。
