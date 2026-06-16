# Kafka 学习路径

> 本文档是 Kafka 知识体系的导航索引，按由浅入深、承前启后的顺序组织，共 4 个阶段 7 个步骤。  
> 每篇文档建议精读并配合动手实践，预计总学习时间 3~5 天。

---

## 学习路径总览

```
第一阶段（入门认知）       第二阶段（核心原理）        第三阶段（消费者深入）      第四阶段（高级特性）

Step 1 ──→ Step 2 ──→ Step 3 ──→ Step 5 ──→ Step 6
                     │                        │
                     └──→ Step 4 ─────────────┘──→ Step 7
```

- **实线箭头** = 强依赖，必须按顺序学习
- **虚线箭头** = Step 3 / Step 4 都依赖 Step 2，二者可并行学习
- Step 7 依赖前三个阶段全部完成

---

## 第一阶段：入门认知（Why → What）

> 目标：理解"为什么需要消息队列"以及"Kafka 架构长什么样"。

### Step 1 — 为什么要使用消息队列

📄 **[为什么要使用消息队列](./为什么要使用消息队列.md)**

**核心知识点：**
- 系统解耦：生产者与消费者松耦合
- 削峰填谷：缓冲瞬时流量，平滑后端负载
- 异步处理：提升响应速度，避免阻塞
- 容错与持久化：消息不丢失，故障后可恢复
- 广播与多订阅：一对多消费模式
- Kafka 在这些场景下的独特优势

**学习建议：** 先建立「消息队列解决什么问题」的整体认知，带着问题进入后续学习。

---

### Step 2 — 架构原理及存储机制

📄 **[架构原理及存储机制](./架构原理及存储机制.md)**

**核心知识点：**
- Broker / Producer / Consumer 三大角色
- Topic 与 Partition 的关系（逻辑分类 vs 物理并行单位）
- Leader-Follower 副本模型
- Zookeeper / Raft 的集群协调作用
- 日志文件顺序写入 & Segment 分段存储
- 消息保留策略（基于时间 / 基于大小）
- 日志压缩（Log Compaction）
- 零拷贝（Zero Copy）技术

**学习建议：** 这是整个知识体系的基石，重点理解 Partition 和副本模型，后续所有章节都会用到。

---

## 第二阶段：核心原理（How）

> 目标：搞懂 Kafka 为什么快、为什么可靠。

### Step 3 — Kafka 高效率原因

📄 **[Kafka高效率原因](./Kafka高效率原因.md)**

**前置依赖：** Step 2（需要先理解顺序写、分段存储、分布式架构）

**核心知识点：**
- 顺序写入 vs 随机写入的性能差异
- 日志分段（Segment）管理
- 批量发送与批量拉取降低网络开销
- 内存映射文件（Memory-Mapped Files）减少磁盘 I/O
- 消费者 Pull 模型 vs 服务端 Push 模型的优劣
- Page Cache 的利用

**学习建议：** 关注「设计选择」而非实现细节，理解 Kafka 在每个环节做了哪些取舍来换取高吞吐。

---

### Step 4 — 如何保证高可用

📄 **[如何保证高可用](./如何保证高可用.md)**

**前置依赖：** Step 2（需要先理解 Partition / Replica / Leader-Follower）

**核心知识点：**
- 多副本数据复制机制
- ISR（In-Sync Replicas）的概念与作用
- Leader 选举流程（Controller 的角色）
- ACK 机制三档对比：`acks=0` / `acks=1` / `acks=all`
- 分布式架构下的横向扩展与负载均衡
- WAL（Write Ahead Log）持久化
- 故障检测 → Leader 选举 → 分区重分配 → 数据恢复全流程
- `replication.factor` 的配置建议

**学习建议：** Step 3（高效率）和 Step 4（高可用）可以并行学，它们分别从性能和可靠性两个维度解释 Kafka 的核心能力。

---

## 第三阶段：消费者深入（Deep Dive）

> 目标：彻底掌握消费者端的工作机制，这是日常开发中最常接触的部分。

### Step 5 — 消费者策略、Rebalance 机制、Offset 存储机制

📄 **[消费者策略、Rebalance机制、Offset存储机制](./消费者策略、Rebalance机制、Offset存储机制.md)**

**前置依赖：** Step 2（需要理解 Consumer Group / Partition 概念）

**核心知识点：**
- 单独消费者 vs 消费者组模式
- 三种分区分配策略：Range / RoundRobin / Sticky（各自的分配逻辑与适用场景）
- Rebalance 的触发条件（消费者增减、分区数变化、消费者超时）
- Rebalance 流程三阶段与「消费中断」问题
- Offset 存储机制：`__consumer_offsets` 内部 Topic
- 自动提交 vs 手动提交（同步/异步）
- `auto.offset.reset`：`latest` vs `earliest`

**学习建议：** Rebalance 是面试高频考点，重点理解触发条件、Sticky 策略的优势以及如何避免频繁 Rebalance。

---

### Step 6 — 消费调优

📄 **[消费调优](./消费调优.md)**

**前置依赖：** Step 5（需要先理解消费者工作机制和 Rebalance）

**核心知识点：**
- 并行消费：消费者实例数与分区数的匹配原则
- `max.poll.records` 批量拉取调优
- `fetch.min.bytes` / `fetch.max.wait.ms` 拉取效率调优
- `session.timeout.ms` / `max.poll.interval.ms` 超时配置
- 消息处理逻辑优化（poll 后尽快处理 + 及时提交 Offset）
- 监控工具：Confluent Control Center / Prometheus + Grafana

**学习建议：** 这是最贴近生产实践的一步，建议结合实际项目中的消费者配置进行对比和调整。

---

## 第四阶段：高级特性（Advanced）

> 目标：掌握 Kafka 的事务机制，满足对数据一致性有严格要求的场景。

### Step 7 — Kafka 事务

📄 **[Kafka事务](./Kafka事务.md)**

**前置依赖：** 前三个阶段全部完成（需要对架构、存储、生产者、消费者都有理解）

**核心知识点：**
- 事务的原子性 & 一致性保证
- 跨分区 / 跨主题的原子写入
- Exactly Once Semantics（EOS）: 恰好一次语义
- 事务三大组件：Transactional Producer / Transaction Coordinator / Transactional Consumer
- 事务工作流程：开启 → 发送 → 提交/回滚
- 「消费-处理-生产」模式的原子化
- 性能开销与版本要求（Kafka 0.11+）
- Go 语言实战示例（confluent-kafka-go）

**学习建议：** 事务是进阶内容，适合在对基础有充分掌握后深入学习。配合 Go 代码示例动手跑一遍会更直观。

---

## 推荐学习节奏

| 天数 | 内容 | 预计耗时 |
|------|------|----------|
| Day 1 | Step 1 + Step 2（入门概念 + 架构） | 2~3 小时 |
| Day 2 | Step 3 + Step 4（高效率 + 高可用） | 2~3 小时 |
| Day 3 | Step 5（消费者机制，本章信息密度大） | 1.5~2 小时 |
| Day 4 | Step 6 + Step 7（消费调优 + 事务） | 2 小时 |
| Day 5 | 回顾、横向串联、实践 | 灵活安排 |

---

## 知识盲区（后续可补充）

当前文档体系已覆盖 Kafka 的核心概念和主要机制，以下方向可作为进阶补充：

| 主题 | 说明 |
|------|------|
| **生产者调优** | 幂等性（idempotence）、压缩算法选择、`linger.ms` / `batch.size` 调优 |
| **Kafka Streams** | 流处理 DSL、状态存储、窗口操作 |
| **Kafka Connect** | 数据管道、Source/Sink Connector |
| **副本同步机制** | HW（High Watermark）/ LEO（Log End Offset）深入 |
| **幂等性与去重** | Producer ID + Sequence Number 机制深入 |
| **运维实践** | 集群规划、分区重分配、集群监控、升级策略 |
| **KRaft 模式** | 去 Zookeeper 化的新架构（Kafka 2.8+） |

---

## 学习建议总结

1. **先整体后局部**：先看完 Step 1~2 建立全局观，再深入各子主题
2. **动手优先**：每步学完后在本地搭建 Kafka 环境（Docker 一键部署），实际发送/消费消息
3. **带着问题学**：每个阶段结束后问自己——「如果某个 Broker 挂了会怎样？」「消费者组扩缩容时发生了什么？」
4. **面试重点**：架构原理、ISR/ACK 机制、Rebalance 流程是高频考点，建议反复理解
