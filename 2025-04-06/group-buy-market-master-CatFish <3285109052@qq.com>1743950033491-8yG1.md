文件名称：group-buy-market-app/src/test/java/cn/cat/test/ApiTest.java
评审结果：

根据提供的diff变更记录，我针对被删除的RabbitMQ测试代码部分进行评审，以下是主要发现和建议：

1. **测试有效性风险**：
- 删除的`test_rabbitmq()`是消息队列的集成测试用例，移除后可能失去对消息发布功能的基础验证
- 建议确认是否有其他测试类覆盖了消息发布逻辑，或通过Mock测试替代集成测试
- 若保留集成测试，推荐添加断言机制验证消息是否到达队列（如使用TestContainers）

2. **资源释放问题**：
- 原测试使用`countDownLatch.await()`永久阻塞线程，可能导致测试线程无法回收
- 推荐添加带超时的等待方式：
```java
countDownLatch.await(30, TimeUnit.SECONDS);
```

3. **代码残留隐患**：
- 删除`EventPublisher`和`routingKey`字段是正确的，但需确认：
  - 是否在`pom.xml`移除了RabbitMQ相关依赖
  - 是否在Spring配置中清理了相关队列定义
  - 是否影响其他需要消息发布的测试用例

4. **测试数据硬编码问题**：
- 原测试使用的订单号"ORD-20231234"等为固定值，建议改为动态生成：
```java
String orderId = "ORD-" + LocalDateTime.now().format(DateTimeFormatter.BASIC_ISO_DATE);
```

5. **日志监控建议**：
- 添加消息发送后的日志记录，方便问题排查：
```java
log.info("Published message to {}: {}", routingKey, message);
```

6. **测试隔离性改进**：
- 如果需要保留此类集成测试，推荐：
  - 使用`@DirtiesContext`注解避免上下文污染
  - 在`@After`中清理测试产生的队列消息

建议根据实际业务需求，选择保留核心集成测试或完全转向单元测试，保持测试金字塔的合理结构。
文件名称：group-buy-market-app/src/test/java/cn/cat/test/infrastructure/RabbitMQTest.java
评审结果：

针对提供的RabbitMQ测试代码，以下是专业级的代码评审意见：

1. **CountDownLatch误用缺陷**
```java
CountDownLatch countDownLatch = new CountDownLatch(1);
countDownLatch.await(); // 永久阻塞
```
- **严重问题**：创建了初始值为1的倒计数器，但未在任何位置调用countDown()，这将导致测试线程永久阻塞
- **影响**：测试用例无法正常结束，在自动化测试场景中会导致构建超时失败
- **改进建议**：
```java
// 增加回调机制或在消费者侧触发countDown
publisher.publish(routingKey, message, () -> countDownLatch.countDown());
```

2. **测试有效性缺失**
- **关键缺陷**：未添加任何断言（assert）或验证逻辑，无法验证消息是否成功投递/消费
- **优化方案**：
```java
// 使用Mockito验证调用次数
verify(consumer, times(5)).handle(any());
// 或使用内存队列收集处理结果
assertThat(processedMessages).hasSize(5);
```

3. **资源泄漏风险**
```java
@Resource
private EventPublisher publisher; // 未关闭AMQP连接
```
- **隐患**：未在测试完成后释放RabbitMQ连接资源
- **改进建议**：
```java
@After
public void tearDown() {
    publisher.cleanup(); // 添加连接关闭逻辑
}
```

4. **测试设计改进点**
```java
// 硬编码测试数据
publisher.publish(routingKey, "订单结算：ORD-20231234");
```
- **优化建议**：
```java
// 参数化测试
@ParameterizedTest
@ValueSource(strings = {"ORD-20231234", "...", "..."})
void test_message(String orderId) {
    publisher.publish(routingKey, "订单结算：" + orderId);
    // 断言逻辑
}
```

5. **日志监控缺失**
```java
@Slf4j
public class RabbitMQTest {
    // 未记录关键节点日志
}
```
- **建议补充**：
```java
publisher.publish(routingKey, message, 
    success -> log.info("Sent {}", message),
    error -> log.error("Failed", error));
```

6. **异常处理优化**
```java
public void test_rabbitmq() throws InterruptedException {
```
- **防御性编程**：
```java
try {
    countDownLatch.await(30, TimeUnit.SECONDS); // 添加超时控制
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    fail("Test interrupted");
}
```

建议的完整改进版本：
```java
@Slf4j
@SpringBootTest
public class RabbitMQTest {
    @Resource private EventPublisher publisher;
    @Value("${...}") private String routingKey;
    @MockBean private MessageConsumer consumer;

    @After
    public void tearDown() {
        publisher.cleanup();
    }

    @ParameterizedTest
    @ValueSource(strings = {"ORD-20231234", "ORD-20231235", "ORD-20231236", "ORD-20231237", "ORD-20231238"})
    void test_message_flow(String orderId) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);
        
        publisher.publish(routingKey, "订单结算：" + orderId, 
            () -> {
                log.info("Published {}", orderId);
                latch.countDown();
            },
            ex -> fail("Publish failed: " + ex.getMessage()));

        assertTrue(latch.await(10, SECONDS));
        verify(consumer, timeout(5000)).handle(argThat(msg -> 
            msg.contains(orderId)));
    }
}
```
