 1️⃣  消息丢失和 Consumer 重连问题 (Critical)

  问题 #2230 (2026-01-12): Consumer 重连导致消息丢失
  - 根本原因: Kafka broker 重启时，controller 会创建新的 consumer 实例和新的
  eventChan，但 dispatcher 仍在监听旧的 eventChan，导致所有新消息丢失
  - 解决方案:
    - 添加 onStopped 回调通知 controller consumer 停止
    - 添加 consumerRunning 标志跟踪 consumer 状态
    - 重连时复用同一个 consumer 实例，保持 eventChan 不变
    - Dispatcher 持续接收消息而不中断

  问题 #2195 (2025-12-23): Consumer 无法自动重连
  - 根本原因: Consumer 连接失败后会停止，但因为 credentials
  未变化，ReconcileConsumer 不会被触发，导致 consumer 引用保持非 nil 但已停止
  - 解决方案:
    - Consumer 停止时将引用设为 nil（mutex 保护）
    - 当 consumer 为 nil 时触发 ReconcileConsumer
    - 改进日志以便调试

  问题 #2100 (2025-11-17): Consumer 重启导致消息丢失
  问题 #2100 (2025-11-17): Consumer 重启导致消息丢失
  - 根本原因: Consumer 重启时未从已提交的 offset 恢复，导致消息被跳过
  - 解决方案:
    - 从数据库中读取上次提交的 offset
    - 使用 topic@partition 格式作为 key
    - Consumer 启动时从正确的 offset 位置恢复

  2️⃣  Offset 管理和数据库问题 (Critical)

  问题 #2137 (2025-12-08): Kafka offset 重复键错误
  - 症状: PostgreSQL 错误 ON CONFLICT DO UPDATE command cannot affect row a second time
  - 根本原因: status.transport 表仅使用 topic name 作为主键，当同一 topic 有多个 partition
  时，批量提交会产生重复键
  - 解决方案:
    - 改用 topic@partition 格式作为主键
    - 在 conflation_committer.go 中使用新格式
    - 在 generic_consumer.go 中解析格式
    - 添加自动迁移逻辑从旧格式迁移到新格式:
        - 从现有 payload JSON 提取 partition 信息
      - 创建新格式记录
      - 在事务中删除旧记录

  3️⃣  Migration 事件处理问题 (High Priority)

  问题 #2360 (2026-03-31): 重复 migration 事件导致错误的成功报告
  - 根本原因: Agent 重启时多个 Registering 事件同时到达，重复事件被跳过但 defer 块仍然报告成功，导致 manager
   过早进入 Cleaning 阶段
  - 解决方案: 添加 isDuplicateEvent 标志防止 defer 块为跳过的重复事件报告状态

  问题 #2348 (2026-03-30): Migration stage 顺序混乱和竞态条件
  - 根本原因: Kafka 事件重新投递导致重复执行同一 stage
  - 解决方案:
    - 添加 mutex 保护的 completedStages map 跟踪 stage 状态（"in-progress" 或 "completed"）
    - 执行前标记为 "in-progress" 防止并发执行
    - 失败时清除 in-progress 状态允许重试
    - Rollbacking 阶段除外（可针对不同 sub-stage 重复）

  问题 #2343 (2026-03-27): Migration 事件过期时间管理
  - 解决方案:
    - 在 manager status handler 添加过期检查跳过过期事件
    - 通过 context 转发 expirytime
    - 移除硬编码的 10 分钟超时，使用动态计算

  问题 #2128 (2025-11-27): 跨 hub 的 migration 状态干扰
  - 根本原因: 所有 managed hub 共享单一的 migration state key (latestMigrationTime)，多个 hub 迁移时产生干扰
  - 解决方案:
    - 使用 topic-specific state key: {topic}--{event-type}
    - 从 event 的 Kafka extension 提取 topic
    - 每个 managed hub 的 migration 状态独立跟踪

  问题 #2047 (2025-11-28): Migration deploying stage 消息大小限制
  - 根本原因: 大量集群迁移时单个 MigrationResourceBundle 超过 Kafka 消息大小限制
  - 解决方案:
    - 实现批量发送机制（800KB 批次大小限制）
    - 使用 CloudEvents extension 跟踪总集群数
    - Agent 处理多批次并跟踪已处理的集群数
    - 所有集群处理完成后确认完成

  4️⃣  资源优化 (Medium Priority)

  问题 #2249 (2026-01-22): Kafka 存储无限增长
  - 症状: 负载测试期间 Kafka PVC 3 天内从 53GB 增长到 94GB
  - 根本原因: 没有配置 retention policy
  - 解决方案:
    - 配置双重 retention 策略:
        - cleanup.policy: "compact,delete"
      - retention.ms: "86400000" (24 小时)
      - retention.bytes: "1073741824" (1GB per partition)
    - 当任一限制达到时删除消息
    - 提取为常量避免魔法数字

  5️⃣  依赖升级

  - 2025-12-31: Kafka 升级到 4.1.0, MAQ 升级到 3.1 (#2211)
  - 2026-03-23: confluent-kafka-go/v2 升级到 v2.13.3 (#2337)
  - 2026-03-19: cloudevents kafka_confluent/v2 升级 (#2304)
  - 多次: librdkafka digest 更新 (#2303, #2215, #2095, #2072)

  6️⃣  其他重要修复

  问题 #2060 (2025-10): Transport config reconciliation 触发机制改进
  问题 #2197 (2025-12): 添加 bundle 大小限制到 event emitter
  问题 #2115 (2025-11): Agent dispatcher 并发 map 访问 panic
  问题 #2222 (2026-01): 使用异步 dispatcher 防止长时间同步阻塞

  🎯 关键收获

  1. 可靠性是首要问题: 大部分修复围绕消息丢失、重连、重复处理
  2. 分布式系统挑战: Migration 相关问题主要是分布式状态管理和事件顺序
  3. 性能优化: Retention policy 和批量发送改进了扩展性
  4. 数据一致性: Offset 管理和数据库主键设计的重要性
  