---
title: 【学习打卡】Day 5 — RabbitMQ 消息队列快速入门
date: 2026-04-10 12:00:00
tags:
  - 消息队列
  - RabbitMQ
  - Java后端
categories:
  - 学习打卡
---

# 【学习打卡】Day 5 — RabbitMQ 消息队列快速入门

> 作者：初魁 | 学习周期：Java后端面试冲刺第5天 | 适合人群：Java后端求职者

---

## 一、为什么需要消息队列？

传统架构中，业务系统之间直接调用，一个服务崩了全部受影响。

引入消息队列后，变成**异步解耦**：

```
用户服务 ──发消息──▶ 消息队列 ──取消息──▶ 订单服务
                                        ──取消息──▶ 库存服务
                                        ──取消息──▶ 支付服务
```

**三大核心价值：**

| 作用 | 说明 | 举例 |
|------|------|------|
| **异步** | 非核心流程异步执行，接口响应更快 | 下单后发短信/邮件异步处理 |
| **解耦** | 生产者和消费者独立演化，互不影响 | 新增业务只需对接队列，不用改原有代码 |
| **削峰** | 流量洪峰时缓存消息，平滑处理 | 秒杀活动时把请求积压，慢慢处理 |

---

## 二、RabbitMQ 核心架构

### 角色与组件

```
Producer（生产者）─── Publish ───▶  Exchange（交换机）
                                            │
                                            ▼
                                       Queue（队列）
                                            │
Consumer（消费者）◀── Subscribe ◀────────────┘
```

| 组件 | 作用 |
|------|------|
| **Producer** | 生产消息的一方 |
| **Consumer** | 消费消息的一方 |
| **Exchange** | 接收消息，根据规则路由到队列 |
| **Queue** | 存储消息的容器 |
| **Routing Key** | 路由键，生产者发送消息时指定 |

---

## 三、Exchange 的四种类型

这是 RabbitMQ 最核心的概念，面试必问。

| 类型 | 路由规则 | 典型场景 |
|------|---------|---------|
| **Direct** | 精确匹配 Routing Key | 订单支付成功 → 专门的队列处理 |
| **Fanout** | 广播给所有队列，无视 Routing Key | 系统公告 → 短信/邮件/推送同时收到 |
| **Topic** | 模糊匹配（`*`一个词，`#`零个或多个）| `order.*.pay` 匹配 order.created.pay |
| **Headers** | 根据消息头属性匹配，现实中用得少 | — |

---

## 四、RabbitMQ 工作模式

### 简单队列（Hello World）

一个 Producer 发，一个 Consumer 收，最基础的模式。

### 工作队列（Work Queue）

多个 Consumer 竞争消费，一条消息只能被一个 Consumer 处理。适合**并发消费**，比如处理订单、发送短信等耗时任务。

### 发布/订阅（Fanout）

Fanout 模式，一个消息被所有消费者同时收到。适合**广播通知**场景。

### 路由模式（Direct）

根据 Routing Key 精确路由。

### 主题模式（Topic）

最灵活，支持模糊匹配通配符 `*` 和 `#`。

---

## 五、RabbitMQ 如何保证消息不丢失？

**要从三个层面解决：**

### 1. 生产端：Publisher Confirm 机制

消息发送到 Broker 后，必须等 Broker 返回 ACK 才算成功。失败则重试或记录日志。

```java
channel.confirmSelect(); // 开启确认模式
channel.addConfirmListener((deliveryTag, multiple) -> {
    // ACK 回调
}, (deliveryTag, multiple) -> {
    // NACK 回调，重发
});
```

### 2. 存储端：持久化

队列和消息都需要设置持久化：

```java
// 队列持久化
channel.queueDeclare("my-queue", true, false, false, null);

// 消息持久化
channel.basicPublish("", "my-queue",
    MessageProperties.PERSISTENT_TEXT_PLAIN,  // 持久化消息
    body.getBytes());
```

### 3. 消费端：手动 ACK

关闭自动确认，改为手动 ACK，确认处理成功后才删除消息：

```java
channel.basicConsume("queue", false, callback);
channel.basicAck(deliveryTag, false);  // 确认成功
channel.basicNack(deliveryTag, false, true);  // 失败，重新入队
```

---

## 六、延迟队列的两种实现方式

黑马课讲了两种延迟队列实现：

### 方式一：死信交换机 + TTL

消息超时（TTL）未消费会变成死信（Dead Letter），进入死信队列：

```
普通队列 → 消息TTL超时 → 死信交换机 → 死信队列
```

### 方式二：延迟队列插件 DelayExchange

声明交换机时加 `delayed=true`，发消息时加 `x-delay` 头指定延迟时间：

```java
// 声明延迟交换机
channel.exchangeDeclare("delayed.exchange", "x-delayed-message", true);

// 发送延迟消息，延迟3秒
channel.basicPublish("delayed.exchange", "order.delay",
    new AMQP.BasicProperties.Builder()
        .header("x-delay", 3000)
        .build(),
    body.getBytes());
```

---

## 七、面试高频提问

**Q：RabbitMQ 和 Kafka 的区别？**
> Kafka 面向高吞吐（日志、大数据），RabbitMQ 面向可靠消息（业务事务）。Kafka 用追加写磁盘，RabbitMQ 用内存+磁盘混合。Kafka 适合 CDC/日志收集，RabbitMQ 适合业务消息。

**Q：RabbitMQ 消息堆积怎么处理？**
> 消息堆积说明消费速度跟不上生产速度。解决方案：① 增加消费者实例并发消费；② 使用惰性队列（接收到消息直接存磁盘）；③ 临时扩容消费者。

**Q：消息队列的弊端？**
> ① 系统复杂度增加（幂等性、死信处理）；② 消息延迟；③ 重复消费风险（需业务方保证幂等）；④ 消费者挂掉时消息积压。

---

**往期回顾：**
- [Day 4：JVM内存模型与MySQL索引](https://chukui2005.github.io/blog/)

---

*学习过程中有理解错误的地方欢迎指正，共同进步 💪*
