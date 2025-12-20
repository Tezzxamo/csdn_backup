# 概要

MQ主要作用为：

- 解耦：减少模块间的耦合，即使有新增的模块，也只需要订阅相关的数据，对原系统无任何影响。
- 异步：允许消息不立即处理，在需要的时候慢慢处理。
- 削峰：将高峰期流量放入MQ中，可以按照系统自身的处理能力进行消费，避免因流量过大，导致系统崩溃。

MQ的主要问题：

- 如何保证顺序消费？
    - 顺序消费：顺序消费是按照消息的发送顺序被消费（如 A → B → C 需按此顺序处理），需要生产、存储、消费三个环节协同保证。
    - 生产端：单线程发送消息，或者按统一标识（如订单ID）路由到同一队列/分区，避免发送乱序。
    - 存储端：MQ需要按照接收顺序存储消息，不允许内部重排（如单队列、单分区存储）
    - 消费端：单线程消费统一队列/分区的消息，或者通过分布式锁保证同一标识消息串行处理。
- 如何保证消息不丢失？
- 如何保证不重复消费？
- 如何保证数据一致性？

引入MQ会带来“异步”和“中间件依赖”的复杂化代价，这也是引入MQ所要承担的代价。

# MQ：

## 一、RabbitMQ：

### 1、核心组件：

| 组件         | 核心作用                                              | 生产环境注意事项                             |
|------------|---------------------------------------------------|--------------------------------------|
| Broker     | rabbitmq服务节点/集群的统称（包含exchange、queue、binding等所有资源） | 至少部署三个节点                             |
| Virtual    | 虚拟主机，实现资源隔离（队列、交换机、用户权限）                          | 不同业务线用独立虚拟主机                         |
| Connection | 客户端与Broker的TCP连接                                  | 必须使用连接池，避免频繁创建连接                     |
| Channel    | 轻量级连接（基于Connection复用），所有消息操作（发/收）都通过Channel完成     | 每个线程对应一个Channel，避免多线程共享channel导致并发问题 |
| Exchange   | 交换机，接收生产者发送的消息，根据【绑定规则】路由到队列。                     | 需根据业务选择交换机类型，核心是【路由键+绑定健】匹配          |
| Binding    | 交换机与队列绑定关系（包含绑定键Routing Key）                      | 绑定健设计需符合业务规则（如订单业务用`order.#`匹配所有子路由） |
| Queue      | 消息存储的最终载体，消费者从队列拉取/订阅消息                           | 核心属性：持久化、排他性、自动删除、死信参数               |
| Producer   | 消息生产者，发送消息。                                       | 需开启生产者确认机制，避免消息丢失                    |
| Consumer   | 消息消费者，接收消息。                                       | 需开启手动ACK，消费失败时合理重试/入死信队列             |

`消息流转核心链路`：

核心逻辑：生产者不直接发送消息到队列，而是先发送到交换机，由交换机根据绑定规则将消息路由到对应队列，消费者从队列消费消息。

```text
// 消息流转全链路：
producer → connection → channel → exchange（按路由键） → binding → queue → channel → consumer
// 消息流转简述：
producer → exchange → queue → consumer
```

### 2、交换机类型：

| 交换机类型                 | 路由规则                                           | 经典场景                       | 
|-----------------------|------------------------------------------------|----------------------------|
| 直连交换机,Direct Exchange | 消息的`路由键`与Binding的`绑定键`完全匹配，才会路由到队列。            | 精准路由（如订单超时死信路由）            |
| 扇出交换机,Fanout Exchange | 忽略`路由键`，将消息广播到所有绑定的队列。                         | 发布订阅（如订单创建后通知库存、物流）        |
| 主题交换机,Topic Exchange  | `路由键`支持通配符（`#`匹配任意多级路由，`*`匹配任意一级路由）            | 模糊路由（如`order.#`匹配所有订单相关消息） |               |                     |
| 头交换机,Headers Exchange | 忽略`路由键`,根据消息头（Header）的键值对匹配（如`x-match=all`全匹配） | 复杂属性路由（极少用，Topic可替代）       |

`示例：交换机配置（订单业务）`

```java
import java.io.IOException;// ========== 1. 声明两个不同类型的交换机，主题交换机和直连交换机，两个参数：true=持久化，false=不自动删除 ==========

@Bean
public DirectExchange orderDirectExchange() {
    return new DirectExchange("order.direct.exchange", true, false);
}

@Bean
public TopicExchange orderTopicExchange() {
    return new TopicExchange("order.topic.exchange", true, false);
}

// ========== 2. 声明2个队列 ==========
// 队列1：接收订单创建消息（路由键：order.create）
@Bean
public Queue orderCreateQueue() {
    return QueueBuilder.durable("order.create.queue").build();
}

// 队列2：接收所有订单消息（路由键：order.#）
@Bean
public Queue orderAllQueue() {
    return QueueBuilder.durable("order.all.queue").build();
}

// ========== 3. 绑定到主题交换机（通配符绑定键） ==========
// 绑定：队列1 绑定 order.create
@Bean
public Binding orderCreateBindingTopic(Queue orderCreateQueue, TopicExchange orderTopicExchange) {
    return BindingBuilder.bind(orderCreateQueue).to(orderTopicExchange).with("order.create");
}

// 绑定：队列2 绑定 order.#
@Bean
public Binding orderAllBindingTopic(Queue orderAllQueue, TopicExchange orderTopicExchange) {
    return BindingBuilder.bind(orderAllQueue).to(orderTopicExchange).with("order.#");
}

// 绑定到直连交换机（精准匹配：order.create）
@Bean
public Binding orderCreateBindingDirect(Queue orderCreateQueue, DirectExchange orderDirectExchange) {
    return BindingBuilder.bind(orderCreateQueue).to(orderDirectExchange).with("order.create");
}

// ========== 4. 生产者发送消息 ==========
@Autowired
private RabbitTemplate rabbitTemplate;

public void sendOrderMsg(OrderDTO createMsg, OrderDTO payMsg) {
    // 发送 order.create → 两个队列都接收
    rabbitTemplate.convertAndSend("order.topic.exchange", "order.create", createMsg);
    // 发送到直连交换机
    rabbitTemplate.convertAndSend("order.direct.exchange", "order.create", createMsg);
    // 发送 order.pay.success → 仅orderAllQueue接收
    rabbitTemplate.convertAndSend("order.topic.exchange", "order.pay.success", payMsg);
}

// ========== 5. 消费者接收消息 ==========
@RabbitListener(queues = "order.create.queue")
public void handlerOrderCreateQueueMsg(String msg, Message message, Channel channel) throws IOException {
    try {
        System.out.println("消费消息（来源：多交换机）：" + msg);
        // 识别消息来源交换机
        String exchange = message.getMessageProperties().getReceivedExchange();
        System.out.println("消息来自交换机：" + exchange);
        // 业务处理（需要加幂等校验，避免不同交换机的重复消息）
        processOrderMsg(msg);
        // 手动ACK
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        // 当前消息处理失败，重新放回队列
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
    }
}

private void processOrderMsg(String msg) {
    // 统一处理订单消息（无论来自哪个交换机）
}
```

### 3、队列类型：

| 队列类型   | 描述                                | 适用场景                          |
|--------|-----------------------------------|-------------------------------|
| 普通队列   | 基础队列，无特殊参数，仅实现消息的存储与转发，是所有高级队列的基础 | 常规业务消息传输（如订单创建通知、物流状态推送）      |
| 死信队列   | 专门接收“死信”（无法正常消费的信息）的队列            | 订单支付超时、消费失败的兜底处理、消息过期/队列满的留存  |
| 延时队列   | 实现消息延迟消费的队列（rabbitmq无原生延迟队列）      | 订单15分钟超时、1小时候提醒评价、接口调用失败延迟重试  |
| 优先级队列  | 支持消息按照优先级消费的队列，优先级高的消息优先被处理       | VIP订单优先处理、紧急告警信息、核心业务优先于非核心业务 |
| 惰性队列   | 优先将消息存储到磁盘的队列                     | 日志采集、批量数据同步、非实时消费的海量消息场景      |
| 镜像队列   | 实现队列高可用的队列                        | 核心业务队列、要求高可用的关键消息链路           |
| 排他队列   | 仅对创建它的“连接”可见，连接关闭后队列自动删除          | 客户端与服务端的临时会话、一次性任务的临时队列       |
| 自动删除队列 | 当最后一个消费者断开连接后，队列自动删除              | 服务间临时数据传输、测试环境临时队列            |
| 临时队列   | 非持久化的临时队列，rabbitmq重启后队列和消息全部丢失    | 测试环境、非核心临时消息（如实时监控、实时统计数据）    |

`示例：队列创建`

```java
public void queueCreateDemo() {
    // 1、持久化队列：durable():持久化
    Queue normalQueue = QueueBuilder.durable("normal.queue").build();
    // 2、临时队列：nonDurable():非持久化
    Queue tempQueue = QueueBuilder.nonDurable("temp.queue").build();
    // 3、优先级队列，优先级范围：0-255，性能最好范围：0-10
    Queue priorityQueue = QueueBuilder.durable("priority.queue").withArgument("x-max-priority", 10).build();
    // 4、惰性队列,也可以手动设置.withArgument("x-queue-mode", "lazy")
    Queue lazyQueue = QueueBuilder.durable("lazy.queue").lazy().build();
    // 5、自动删除队列：autoDelete():自动删除
    Queue autoDeleteQueue = QueueBuilder.durable("auto.delete.queue").autoDelete().build();
    // 6、排他队列：exclusive():排他队列
    Queue exclusiveQueue = QueueBuilder.exclusive().name("exclusive.queue").build();
    // 7、镜像队列：无法通过java代码配置，需要通过rabbitmq控制台/命令行设置集群策略
    // 命令行配置（为所有order开头的队列设置镜像策略）:ha-mode=all:镜像到所有节点；ha-sync-mode=automatic:自动同步数据
    // rabbitmq set_policy ha-all "^order\." '{"ha-mode":"all","ha-sync-mode":"automatic"}'
    // 镜像队列仅需持久化，镜像策略由集群配置
    Queue mirrorQueue = QueueBuilder.durable("mirror.queue").build();
    // 8、死信队列
    Queue deadLetterQueue = QueueBuilder.durable("dead.letter.queue").build();
    // 9、延时队列：rabbitmq没有原生的延时队列，但是可以通过普通队列+TTL+死信队列实现延时队列：
    //    消息在普通队列超时后，进入死信队列，消费者则订阅该死信队列，处理死信队列中的消息，实现延时队列。
    Map<String, Object> args = new HashMap<>(3);
    // 绑定死信交换机
    args.put("x-dead-letter-exchange", DLX_EXCHANGE);
    // 死信路由键
    args.put("x-dead-letter-routing-key", DLX_ROUTING_KEY);
    // 队列级别TTL（可选，15分钟）
    args.put("x-message-ttl", 15 * 60 * 1000);
    // 该队列不进行订阅，只用于消息存储，等到消息超时后，死信队列接收该消息，进行业务处理
    Queue delayQueue = QueueBuilder.durable("delay.queue")
            .withArguments(args)
            .build();
}
```

### 4、核心工作模式：

`7种工作模式`

- 简单模式：
    - 原理:1个生产者 → 1个交换机（可省略，用默认交换机） → 1个队列 → 1个消费者
    - 场景：简单的一对一消息传输，如用户注册后发送注册成功消息给用户
    - 核心代码：

```java
// 生产者发送到默认交换机（""），路由键=队列名
rabbitTemplate.convertAndSend("","simple.queue","用户注册消息：userId=10001");

// 消费者
@RabbitListener(queues = "simple.queue")
public void simpleQueueListener(String msg) {
    System.out.println("消费者收到消息：" + msg);
}
```

- 工作队列模式：
    - 原理：1个生产者 → 1个队列 → 多个消费者，消息被`轮训/公平分发`给消费者
    - 场景：任务并行处理（如批量处理订单；日志解析）
    - 核心配置：开启公平分发（避免空闲消费者等待，忙碌消费者挤压）：

```java
// 消费者配置：每次只拉取一条消息，处理完再拉取
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setConcurrentConsumers(1); // 公平分发核心参数
    return factory;
}
```

- 发布订阅模式：
    - 原理：1个生产者 → 扇出交换机 → 多个队列 → 多个消费者（每个消费者都能收到全量消息）
    - 场景：消息广播（如订单创建后，库存、物流、积分系统都需要收到消息）
- 路由模式：
    - 原理：1个生产者 → 直连交换机 → 多个队列（不同绑定键） → 按路由键精准路由
    - 场景：精准消息分发（如订单支付成功路由到支付队列，订单取消路由到取消队列）
- 主题模式：
    - 原理：基于通配符的灵活路由，是生产模式最常用的模式。
    - 场景：多维度消息路由（如`order.pay.success`路由到支付成功队列，`order.refund.*`路由到退款队列）
- 死信模式：
    - 原理：普通队列绑定死信交换机，异常消息（过期/消费失败/队列满）自动转为死信，路由到死信队列。
    - 场景：订单超时未支付、消费失败兜底处理。
- rpc模式：
    - 原理：
    - 场景：

### 5、关键特性

- 持久化机制：rabbitmq的持久化分为3层，需要全部开启才能保证【Broker重启后消息不丢失】
    - 队列持久化：创建队列式指定durable=true；
    - 交换机持久化：创建交换机式指定durable=ture；
    - 消息持久化：spring amqp默认消息持久化;也可以发送消息时设置持久化；
    ```java
    rabbitTemplate.convertAndSend(
        "exchangeName", 
        "routingKey",
        msg,
        message -> {
            message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
            return message;
        },
        new CorrelationData(UUID.randomUUID().toString())
    );
    ```
- 确认机制：保障消息可靠传输
    - 生产者确认：确保消息成功发送到交换机/队列，分两种模式：
        - 单条确认：同步等待确认，性能低，适合少量消息；
        - 批量确认：异步批量确认，性能高，生产环境首选；
        - 配置示例：
      ```yaml
      spring:
        rabbitmq:
          publisher-confirm-type: correlated # 开启异步确认
          publisher-returns: true # 开启消息返回（路由失败时回调）
      ```
      ```java
      // 生产者确认回调
      @Bean
      public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
          RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
          // 设置消息转换器
          rabbitTemplate.setMessageConverter(messageConverter());
          
          // 设置发消息前开启事务，通过事务可以保证消息不丢失，可以与外部事务联动（如数据库事务）
          // rabbitTemplate.setChannelTransacted(true);
          
          // confirm模式：开启返回回调
          rabbitTemplate.setMandatory(true);
          // 消息发送到exchange回调
          rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
              if (!ack) {
                  log.error("消息发送失败：{}，原因：{}", correlationData.getId(), cause);
                  // 失败重试/入库补偿
              } else {
                  log.info("消息发送成功:correlationData({}),cause({})", correlationData, cause);
              }
          });
          // 注册返回回调处理器:消息从exchange发送到queue失败回调
          rabbitTemplate.setReturnsCallback(returned -> {
              Message message = returned.getMessage();
              String exchange = returned.getExchange();
              String routingKey = returned.getRoutingKey();
              int replyCode = returned.getReplyCode();
              String replyText = returned.getReplyText();
              // 记录错误日志
              log.error("消息投递失败: exchange={}, routingKey={}, replyCode={}, replyText={}",
                      exchange, routingKey, replyCode, replyText);
              log.error("失败的消息内容: {}", new String(message.getBody()));
      
              // 根据不同的失败原因采取不同的处理策略
              handleFailedMessage(message, exchange, routingKey, replyCode, replyText);
          });
          return rabbitTemplate;
      }
      ```
    - 消费者ACK：避免消息重复/丢失，生产环境必须开启`手动ACK`，禁用自动ACK（自动ACK会导致消费中宕机时消息丢失）
        - 配置示例：
      ```yaml
      spring:
        rabbitmq:
          listener:
            simple:
              acknowledge-mode: manual # 手动ACK
      ```
      ```java
      // 消费者处理逻辑：
      @RabbitListener(queues = "order.queue")
      @RabbitHandler
      public void handleMessage(String msg, Message message, Channel channel) throws IOException {
          boolean success = false;
          int retryCount = 3;
          while (!success && retryCount-- > 0) {
              try {
                  // 处理消息
                  log.info("收到消息: {}, deliveryTag = {}", msg, message.getMessageProperties().getDeliveryTag());
                  if (message.equals("dead-letter")) {
                      throw new RuntimeException("收到死信");
                  }
                  // 正常处理完毕，手动确认
                  success = true;
                  channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
              } catch (Exception e) {
                  log.error("程序异常：{}", e.getMessage());
              }
          }
          // 达到最大重试次数后仍然消费失败
          if (!success) {
              // 手动删除，移至死信队列
              try {
                  channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
      }
      ```
- 延迟队列：
    - 方式一：普通队列TTL + 死信队列（消费者订阅死信队列），固定延迟。
    - 方式二：rabbitmq_delayed_message_exchange插件：支持动态延迟（不同消息不同超时），灵活度高；
- 优先级队列：核心业务优先处理
    - 队列设置最大优先级（0-255，建议0-10），消息设置优先级，积压时先处理高优先级消息。
    - 代码示例：
    ```java
    // 声明优先级队列
    @Bean
    public Queue priorityQueue() {
        return QueueBuilder.durable("order.priority.queue")
                .withArgument("x-max-priority", 10) // 最大优先级10
                .build();
    }
    // 发送高优先级消息（VIP订单）
    rabbitTemplate.convertAndSend(exchange, routingKey, orderMsg, message -> {
        message.getMessageProperties().setPriority(9); // VIP订单优先级9
        return message;
    });
    ```
- 镜像队列：集群高可用

### 6、常见问题：

- 消息丢失：
    - 原因：生产者丢消息、MQ丢消息（未持久化、队列未镜像）、消费者丢消息；
    - 解决方案：
        - 开启队列/交换机/消息持久化；
        - 开启生产者确认 + 消费者手动ACK；
        - 核心队列配置镜像策略；
- 消息重复消费：
    - 原因：消费者宕机导致ACK未提交、网络重试、消息重发；
    - 解决方案：
        - 消费者实现幂等性（如订单ID作为唯一键，处理前在DB校验状态）；
        - 分布式锁/redis记录消费状态；
        - 死信队列避免无限重试；
- 消息积压：
    - 原因：消费者能力不足、队列消息过多、惰性队列未开启；
    - 解决方案：
        - 扩容消费者（提高 `max-concurrency`)
        - 海量消息场景使用`惰性队列`（优先存磁盘，避免OOM）；
        - 临时增加消费结点，批量消费积压消息；
- 死信队列堆积：
    - 原因：死信消费失败、业务异常未处理；
    - 解决方案：
        - 配置死信队列监控告警（如堆积＞1000条出发告警）；
        - 死信消息入库，人工介入处理；
        - 区分【可重试死信】和【不可重试死信】，避免无效堆积。
- 集群脑裂：
    - 原因：集群节点网络分区，多个节点认为自己是主节点；
    - 解决方案：
        - 配置`cluster_partition_handling=pause_minority`（少数派节点暂停服务），或者使用
          `cluster_partition_handling=autoheal`（自动合并集群，可能丢失部分数据，仅适用于非核心业务）。
        - 监控网络分区并告警、优化网络结构、故障及时恢复网络。
    - 注意事项：
        - 镜像队列是基础，核心业务必须配置镜像队列；
        - 策略选择适配业务：核心业务选择`pause_minority`，非核心业务可选`autoheal`；
        - 不建议关闭分区检测，部分场景下有人关闭分区检测`cluster_partition_handling=ignore`，会导致数据混乱，绝对禁止。

### 7、rabbitmq保证顺序消费：

- 普通情况：
    - 顺序错乱原因：
        - 多消费者并发消费：一个队列被多个消费者同时消费，由于各自处理速度不同，后发送的消息可能先被处理完。
        - 消息确认机制不当：如果使用自动确认，消息可能在处理完成前就被确认为已消费，若此时消费者失败，消息重新入队会被其他消费者处理，导致乱序。
        - 消息重试与死信队列：处理失败的消息若被重试或者进入死信队列，可能会脱离原有的顺序。
    - 解决方案：
        - ①单队列与单消费者：最简单直接的方案，严格保证了FIFO。
            - 实现原理：创建一个专用队列，并确保该队列只有一个消费者，且该消费者在单线程模式下工作。
            - 优点：实现简单、能严格保证全局顺序。
            - 缺点：无法水平扩展消费者，吞吐量低，容易成为性能瓶颈；
            - 适用场景：消息量不大、对顺序性要求极其严格的业务。
        - ②消息分组（分片队列）
            - 实现原理：放弃全局顺序，保证分组顺序。使用一个业务ID（如订单ID、用户ID）作为分片键，通过计算（如哈希取模）将相同分片键的消息式中路由到同一个队列。每个队列仍然配置但消费者，从而保证同一业务ID的消息顺序。
            - 工作流程：
                - 1）生产者根据消息的业务ID计算路由键，发送到对应的队列。
                - 2）相同orderId的消息总是进入同一个Queue。
                - 3）每个队列只有一个消费者，保证队列内消息顺序处理。
            - 代码示例：（生产者路由逻辑）：
          ```java
          // 订单ID取模，将订单ID路由到对应的分区，假设有3个队列
          String routingKey = "order." + Math.abs(message.getOrderId().hashCode() % 3);
          rabbitTemplate.convertAndSend("order.exchange", routingKey, message);
          ```
            - 优点：横向扩展性好，可以通过增加队列数来提高整体吞吐量。
            - 缺点：只能保证局部（同一分组内）顺序；需要提前规划好队列数量，分片不均可能导致数据倾斜。
            - 适用场景：电商订单状态流、用户行为跟踪等，同一实体的状态变更需要保证顺序的场景。
            - 优化：消费者使用多线程，但是同一个业务ID的消息必须单线程串行执行，不能多线程执行同一个业务ID的消息。
- 特殊情况：
  - 情况：当严格顺序的消息链（如 A→B→C→D）中某条消息（如 B）发送失败时。
  - 解决方案：
    - 核心逻辑：断链不消费 + 补链再执行。先兜底不丢消息，再阻塞后续消息消费，最后补全失败消息并按原顺序执行。
    - 处理：消费前校验“当前消息的前序所有序号都已消费成功”，否则拒绝消费（重新入队）。
    - 优化：基于redis的前序校验（替代DB查询），消息消费成功后，将如`order:0123:success_seq 2`写入redis，查询时得到已经消费到哪一个序号。

---

## 2、Kafka：

- 核心逻辑：基于分区有序，一个主题的多个分区独立有序，单个分区内消息绝对有序。
- 实现方式：生产者通过指定分区键（如订单ID）将消息路由到同一分区；消费端一个消费组内，一个分区仅分配给一个消费者，且消费者单线程处理该分区消息。

## 3、RocketMQ：

- 核心逻辑：支持全局有序和分区有序，灵活性更高。
- 实现方式：全局有序需要使用单主题+单队列；分区有序通过MessageQueueSelector指定队列（按业务标识路由），消费端按队列顺序处理，且不并行消费同一队列。


