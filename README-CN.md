# <a id="0"></a>Event Sourcing with PostgreSQL

- [Introduction](#1)
- [Example domain](#2)
- [Event sourcing and CQRS basics](#3)
    - [State-oriented persistence](#3-1)
    - [Event sourcing](#3-2)
    - [Snapshotting](#3-3)
    - [Querying the data](#3-4)
    - [CQRS](#3-5)
    - [Event handlers](#3-6)
    - [Domain events vs Integration events](#3-7)
    - [Advantages of CQRS](#3-8)
    - [Advantages of event sourcing](#3-9)
- [Solution architecture](#4)
    - [Component diagram](#4-1)
    - [ER diagram](#4-2)
    - [Optimistic concurrency control](#4-3)
    - [Snapshotting](#4-4)
    - [Loading any revision of the aggregate](#4-5)
    - [Synchronously updating projections](#4-6)
    - [Asynchronously sending integration events to a message broker](#4-7)
      - [Transactional outbox using transaction ID](#4-7-1)
      - [Transactional outbox using table-level lock](#4-7-2)
      - [Database polling](#4-7-3)
      - [Listen/Notify as an alternative to database polling](#4-7-4)
    - [Adding new asynchronous event handlers](#4-8)
    - [Drawbacks](#4-9)
- [Project structure](#5)
    - [Gradle subprojects](#5-1)
    - [Database schema migrations](#5-2)
    - [Class diagrams](#5-3)
        - [Class diagram of the domain model](#5-3-1)
        - [Class diagram of the projections](#5-3-2)
        - [Class diagram of the service layer](#5-3-3)
- [How to adapt it to your domain?](#6)
- [How to run the sample?](#7)

<!-- Table of contents is made with https://github.com/eugene-khyst/md-toc-cli -->

## <a id="1"></a>介绍

我们的应用程序通常操作领域对象的当前状态，但有时候，我们需要知道领域对象变更的完整历史记录。
例如，我们想知道订单是如何进入当前状态的.

审核跟踪（也称为审核日志）是影响系统的操作的历史记录和详细信息的按时间顺序的记录。
审计跟踪可能是监管或业务要求。

我们可以将领域对象状态的所有更改存储到仅增事件流中。因此，事件流将包含完整的更改历史记录。
但是，我们如何确保此历史记录真实无误？
我们可以将事件流用作系统中的主要事实来源。
要获取对象的当前状态，我们按照发生顺序重播所有事件，此模式称为事件溯源。
用于存储事件流的数据库称为事件存储。
事件溯源提供了对系统所做的所有更改的完整和准确的记录。
事件溯源是实现审核跟踪的行业标准。

市面上有一类数据库专门做事件溯源。为这些专用数据库公司工作的开发者布道师过去常说，你不应该使用传统的关系型或文档型数据库来实现事件溯源。
这是真的还是只是一种营销策略？

专用的事件溯源数据库很方便，可以直接使用，但是世界上最先进的开源数据库 PostgreSQL 也适用于事件溯源。
你可以使用无需额外框架或扩展的 PostgreSQL 作为事件存储，而不是设置和维护专用的事件溯源数据库。

本仓库提供了一个使用 `Spring Boot` 构建的事件溯源系统的参考实现，该系统使用 PostgreSQL 作为事件存储。
[Fork](https://github.com/eugene-khyst/postgresql-event-sourcing/fork) 仓库并将其用作项目的模板。
或者克隆仓库并运行端到端测试，来了解整体是如何运行的。

![PostgreSQL Logo](img/potgresql-logo.png)

<!--See also-->
参看

* [Event Sourcing with EventStoreDB](https://github.com/eugene-khyst/eventstoredb-event-sourcing)
* [Event Sourcing with Kafka and ksqlDB](https://github.com/eugene-khyst/ksqldb-event-souring)

## <a id="2"></a>例子中的模型

本示例使用了一个简化的打车系统领域模型。

* 乘客可以下单打车，指定路线和价格。

[//]: # (todo 支付更多的钱来减少等待？还是其它可行的翻译)
* 乘客可以编辑订单价格，在需求量很大的情况下，可以支付更多的钱，而不是等待。
* 司机可以接受订单。
* 司机可以完成之前接受的订单。
* 订单可以在完成之前取消。

![Domain use case diagram](img/domain-1.svg)

![Domain state diagram](img/domain-2.svg)

## <a id="3"></a>事件溯源和 CQRS 基础

### <a id="3-1"></a>面向状态的持久化

[//]: # (todo 能否简化为 DML 语句？)
面向状态的持久化（CRUD）应用程序仅存储实体的最新版本。
数据库记录当前的实体。
当实体更新时，相应的数据库记录也会更新。
使用 SQL `INSERT`，`UPDATE` 和 `DELETE` 语句。

![State-oriented persistence](img/state-oriented-persistence.svg)

### <a id="3-2"></a>事件源

事件溯源应用程序将实体的状态持久化为一系列**不可变**的状态更改事件。

![Event sourcing](img/event-sourcing-1.svg)

每当实体的状态发生更改时，都会将新事件附加到事件列表中。
仅使用 SQL `INSERT` 语句。
事件是不可变的，因此不会使用 SQL `UPDATE` 和 `DELETE` 语句。

![Event sourcing](img/event-sourcing-2.svg)

可以通过重播实体的所有事件来恢复实体的当前状态。

事件溯源与领域驱动设计（DDD）密切相关，并共享一些术语。

一个实体在事件溯源中被称为**聚合**。

相同聚合的事件序列称为**流**。

事件溯源最适合只有少量事件的短期实体（例如，订单）。

通过重播所有事件来恢复短期实体的状态不会对性能产生任何影响。因此，不需要为短期实体恢复状态进行任何优化。

对于具有数千个事件的无限存储实体（例如，用户，银行账户），通过重播所有事件来恢复状态并不是最佳选择，应该考虑快照。

### <a id="3-3"></a>快照

[//]: # (todo 类似跳表？)
快照是一种优化技术，其中还保存了聚合状态的快照，因此应用程序可以从聚合快照而不是从所有事件（可能有数千个）中恢复当前的状态。

在每个 *nth* 事件上，通过存储聚合状态及其版本来创建聚合快照。

为了恢复聚合状态：

1. 首先读取最新的快照，
2. 然后从快照指向的版本开始，从原始流中向前读取事件。

![Snapshotting in event souring](img/event-sourcing-snapshotting.svg)

### <a id="3-4"></a>查询数据

按 ID 查找聚合很容易，但是其他查询很难。
由于聚合以不可变事件的仅增列表的形式存储，因此使用 SQL 查询数据是不可能的。
要按某个字段查找聚合，我们需要首先读取所有事件并重播它们以恢复所有聚合。

为了恢复关系型数据库提供的所有查询功能，我们可以从事件流中派生出一个专用的只读模型。

事件流是写入模型和主要事实来源。

只读模型是写入模型的“非规范化”视图，允许更快，更方便的查询。
只读模型是系统状态的投影。
因此，只读模型也称为**投影**。

投影为单个聚合类型提供数据视图，或执行聚合并组合来自多个聚合类型的数据。

这就是 CQRS 的用武之地。

### <a id="3-5"></a>CQRS

命令查询责任分离（CQRS）是指在命令（写请求）和查询（读请求）之间分离责任。
写请求和读请求由不同的处理器处理。

一个命令生成零个或多个事件，或导致错误。

![CQRS](img/cqrs-1.svg)

CQRS 是一种自给自足的架构模式，不需要事件源。
但是在实践中，事件源通常与 CQRS 结合使用。
事件存储常用作写库，而 SQL 或 NoSQL 数据库则用作读库。

![CQRS with event sourcing](img/cqrs-2.svg)

### <a id="3-6"></a>事件处理器

命令生成事件。
事件处理由**事件处理器**完成。
作为事件处理的一部分，我们可能需要更新投影，向消息代理发送消息或调用 API。

这里有两种类型的事件处理器：**同步**和**异步**。

存储写模型和读模型在一个数据库内，允许对读模型进行事务更新。
每次我们追加一个新事件，投影都会在同一个事务中**同步**更新。
投影与事件流是**一致**的。

当一个事件处理器需要与外部系统或中间件通信（例如，向 Kafka 发送消息）时，它应该在更新写模型的事务之后**异步**运行。
异步执行导致**最终一致性**。

与外部系统交流不会发生在更新写模型的同一事务中。
外部系统调用可能会成功，但事务稍后将被回滚，导致不一致。

总之，分布式系统需要考虑最终一致性。

### <a id="3-7"></a>领域事件 vs 集成事件

事件源中的事件是**领域事件**。
领域事件是有界上下文的一部分，不应该原样用于与其他有界上下文的集成。

**集成事件**用于有界上下文之间的通信。
集成事件表示聚合的当前状态，而不仅仅是作为领域事件的聚合更改。

### <a id="3-8"></a>CQRS 的优势

* 可以独立扩展读写数据库。
* 优化读库的数据模式（例如，读数据库可以是非规范化的）。
* 更简单的查询（例如，可以避免复杂的 `JOIN` 操作）。

### <a id="3-9"></a>事件源的优势

* 系统的真实历史（审核和可追溯性）。
  实现审核跟踪的行业标准。
* 能够将系统置于任何先前的状态（例如，用于调试）。
* 可以根据（稍后）需要从事件创建新的读模型投影。
  它允许响应未来的需求和新的需求。

## <a id="4"></a>解决方案架构

PostgreSQL 可以用作事件存储。
它将原生支持追加事件，并发控制和读取事件。
订阅事件需要额外的实现。

### <a id="4-1"></a>组件图

![PostgreSQL event store component diagram](img/postgresql-event-sourcing.svg)

### <a id="4-2"></a>ER 图

![PostgreSQL event store ER diagram](img/er-diagram.svg)

事件存储在 `ES_EVENT` 表中。

### <a id="4-3"></a>乐观并发控制

最新的聚合版本存储在 `ES_AGGREGATE` 表中。
版本检查用于乐观并发控制。
版本检查使用版本号来检测冲突更新（并防止丢失更新）。

追加事件操作由单个事务中的 2 条 SQL 语句组成：

1. 检查实际版本和期望版本是否匹配，并递增版本
    ```sql
    UPDATE ES_AGGREGATE
       SET VERSION = :newVersion
     WHERE ID = :aggregateId
       AND VERSION = :expectedVersion
    ```
2. 插入新事件
    ```sql
    INSERT INTO ES_EVENT (TRANSACTION_ID, AGGREGATE_ID, VERSION, EVENT_TYPE, JSON_DATA)
    VALUES(pg_current_xact_id(), :aggregateId, :version, :eventType, :jsonObj::json)
    RETURNING ID, TRANSACTION_ID::text, EVENT_TYPE, JSON_DATA
    ```
   `pg_current_xact_id()` 返回当前事务的 ID。稍后将解释这一需求。

![Optimistic Concurrency Control](img/event-sourcing-concurrency.svg)

### <a id="4-4"></a>快照

在每一个 *nth* 事件插入一个聚合状态（快照）到 `ES_AGGREGATE_SNAPSHOT` 表中，指定版本

```sql
INSERT INTO ES_AGGREGATE_SNAPSHOT (AGGREGATE_ID, VERSION, JSON_DATA)
VALUES (:aggregateId, :version, :jsonObj::json)
```

给一个聚合类型配置快照可以在 [`application.yml`](src/main/resources/application.yml) 中禁用和配置

```yaml
event-sourcing:
    snapshotting:
        # com.example.eventsourcing.domain.AggregateType
        ORDER:
            enabled: true
            # Create a snapshot on every nth event
            nth-event: 10
```

### <a id="4-5"></a>加载聚合的任何修订版本

还原聚合的任何修订版本：

1. 首先读取最新快照的数据
    ```sql
    SELECT a.AGGREGATE_TYPE,
           s.JSON_DATA
      FROM ES_AGGREGATE_SNAPSHOT s
      JOIN ES_AGGREGATE a ON a.ID = s.AGGREGATE_ID
     WHERE s.AGGREGATE_ID = :aggregateId
       AND (:version IS NULL OR s.VERSION <= :version)
     ORDER BY s.VERSION DESC
     LIMIT 1
    ```
2. 然后从快照指向的修订版本开始，从事件流中向前读取事件
    ```sql
    SELECT ID,
           TRANSACTION_ID::text,
           EVENT_TYPE,
           JSON_DATA
      FROM ES_EVENT
     WHERE AGGREGATE_ID = :aggregateId
       AND (:fromVersion IS NULL OR VERSION > :fromVersion)
       AND (:toVersion IS NULL OR VERSION <= :toVersion)
     ORDER BY VERSION ASC
    ```

### <a id="4-6"></a>同步更新投影

使用 PostgreSQL 作为事件存储和读数据库允许对读模型进行事务更新。
每次我们追加一个新事件，投影都会在同一个事务中同步更新。
这是一个很大的优势，因为有时候一致性并不容易实现。

[//]: # (todo 这一条并不显然，因为 pg 可以在事务执行代码段，然后利用 raft 等机制保证一致性)
你无法使用单独的数据库作为事件存储时获得一致的投影。

### <a id="4-7"></a>异步将集成事件发送到消息代理

在更新写模型的事务之后，应该异步发送集成事件。

[//]: # (todo 第二段翻译)
PostgreSQL 不允许订阅更改，因此解决方案是事务外箱模式。
使用数据库的服务将事件作为本地事务的一部分插入到外箱表中。
一个独立的进程将插入到数据库的事件发布到消息代理。

![Transactional outbox pattern](img/transactional-outbox-1.svg)

[//]: # (todo 第二段)
我们可能有多个异步事件处理器或所谓的订阅。
订阅概念是：保持追踪不同的事件处理器中的已传递的事件。
最新的事件处理器（订阅）处理的事件存储在单独的表 `ES_EVENT_SUBSCRIPTION` 中。
新的事件通过轮询 `ES_EVENT` 表来处理。

![Transactional outbox pattern with subscriptions](img/transactional-outbox-2.svg)

当应用程序的实例并行运行时，我们需要确保任何处理仅影响事件一次。
我们不希望多个事件处理器实例处理相同的事件。

这通过在 `ES_EVENT_SUBSCRIPTION` 表的行上获取锁来实现。
我们锁定当前处理订阅的行（`SELECT FOR UPDATE`）。

为了不挂起其他后端实例，我们希望跳过已锁定的行（`SELECT FOR UPDATE SKIP LOCKED`）并锁定“下一个”订阅。
它允许多个后端实例选择不重叠的订阅。
这样，我们提高了可用性和可扩展性。

事件订阅处理器每秒轮询 `ES_EVENT_SUBSCRIPTION` 表以获取新事件（间隔可配置）并处理它们：

1. 读取订阅处理的最后一个事务 ID 和事件 ID 并获取锁
    ```sql
    SELECT LAST_TRANSACTION_ID::text,
           LAST_EVENT_ID
      FROM ES_EVENT_SUBSCRIPTION
     WHERE SUBSCRIPTION_NAME = :subscriptionName
       FOR UPDATE SKIP LOCKED
    ```
2. 读取新的事件
    ```sql
    SELECT e.ID,
           e.TRANSACTION_ID::text,
           e.EVENT_TYPE,
           e.JSON_DATA
      FROM ES_EVENT e
      JOIN ES_AGGREGATE a on a.ID = e.AGGREGATE_ID
     WHERE a.AGGREGATE_TYPE = :aggregateType
       AND (e.TRANSACTION_ID, e.ID) > (:lastProcessedTransactionId::xid8, :lastProcessedEventId)
       AND e.TRANSACTION_ID < pg_snapshot_xmin(pg_current_snapshot())
     ORDER BY e.TRANSACTION_ID ASC, e.ID ASC
    ```
   `(a, b) > (c, d)` 是一种行比较，等价于 `a > c OR (a = c AND b > d)`。
3. 订阅处理更新最新的一个事务 ID 和事件 ID
    ```sql
    UPDATE ES_EVENT_SUBSCRIPTION
       SET LAST_TRANSACTION_ID = :lastProcessedTransactionId::xid8,
           LAST_EVENT_ID = :lastProcessedEventId
     WHERE SUBSCRIPTION_NAME = :subscriptionName
    ```

#### <a id="4-7-1"></a>使用 transaction ID 的事务外箱

仅用事件 ID 来跟踪订阅处理的事件是不可靠的，可能会导致事件丢失。

`ES_EVENT` 的 `ID` 列是 `BIGSERIAL` 类型。
它是一种便利的记号，用于创建 ID 列，其默认值从 `SEQUENCE` 生成器分配。

PostgreSQL 序列无法回滚。
`SELECT nextval('ES_EVENT_ID_SEQ')` 递增并返回序列值。
即使事务尚未提交，新的序列值也会对其他事务可见。

如果事务 #2 在事务 #1 之后开始但首先提交，
事件订阅处理器可以读取事务 #2 创建的事件，
更新最后处理的事件 ID，
从而丢失事务 #1 创建的事件。

![PostgreSQL naive transactional outbox](img/postgresql-naive-outbox.svg)

我们使用事务 ID 和事件 ID 来构建可靠的 PostgreSQL 轮询机制，不会丢失事件。

每个事件都附带有当前事务 ID。
`pg_current_xact_id()` 返回类型为 `xid8` 的当前事务 ID。
`xid8` 值严格单调递增，并且在数据库集群的生命周期内不能重复使用。

最新的事务可以“安全”的在当前快照的 `xmin` 之前处理。
`pg_current_snapshot()` 返回当前的快照，其数据结构显示哪些事务 ID 正在进行中。
`pg_snapshot_xmin(pg_snapshot)` 返回快照的 `xmin`。
`xmin` 是仍处于活动状态的最小事务 ID。
所有小于 `xmin` 的事务 ID 要么已提交并可见，要么已回滚（事务均已结束）。

即使事务 #2 在事务 #1 之后开始并首先提交，
直到事务 #1 提交后，事件订阅处理器才会读取事务 #2 创建的事件。

![PostgreSQL reliable transactional outbox using transaction ID](img/postgresql-reliable-outbox.svg)


> **NOTE**
> The transaction ID solution is used by default as it is non-blocking.

#### <a id="4-7-2"></a>Transactional outbox using table-level lock

With the transaction ID solution, event subscription processor doesn't wait for in-progress transactions to complete.
Events created by already committed transactions will not be available for processing
while transactions started earlier are still in-progress.
These events will be processed immediately after these earlier transactions have completed.

An alternative solution is to use PostgreSQL explicit locking to make event subscription processor wait for in-progress transactions.

Before reading new events, the event subscription processor
* gets the most recently issued ID sequence number,
* very briefly locks the table for writes to wait for all pending writes to complete.

The most recently issued `ES_EVENT_ID_SEQ` sequence number is obtained using the `pg_sequence_last_value` function:
`SELECT pg_sequence_last_value('ES_EVENT_ID_SEQ')`.

Events are created with the command `INSERT INTO ES_EVENT...` that acquires the `ROW EXCLUSIVE` lock mode on `ES_EVENT` table.
`ROW EXCLUSIVE (RowExclusiveLock)` lock mode is acquired by any command that modifies data in a table.

The command `LOCK ES_EVENT IN SHARE ROW EXCLUSIVE MODE` acquires the `SHARE ROW EXCLUSIVE` lock mode on `ES_EVENT` table.
`SHARE ROW EXCLUSIVE (ShareRowExclusiveLock)` mode protects a table against concurrent data changes,
and is self-exclusive so that only one session can hold it at a time.

`SHARE ROW EXCLUSIVE` lock must be acquired in a separate transaction (`Propagation.REQUIRES_NEW`).
The transaction must contain only this command and commit quickly to release the lock, so writes can resume.

When the lock is acquired and released, it means
that there are no more uncommitted writes with an ID less than or equal to the ID returned by `pg_sequence_last_value`.

![PostgreSQL reliable transactional outbox using table-level lock](img/postgresql-reliable-outbox-with-lock.svg)

#### <a id="4-7-3"></a>数据库轮询

为了从表 `ES_EVENT` 中获取最新的事务，应用程序必须轮询数据库。
轮询周期越短，持久化新事件和订阅处理之间的延迟就越短。
但是滞后是不可避免的。如果轮询周期为 1 秒，则滞后最多为 1 秒。

轮询算法实现 [ScheduledEventSubscriptionProcessor](postgresql-event-sourcing-core/src/main/java/eventsourcing/postgresql/service/ScheduledEventSubscriptionProcessor.java)
使用 Spring 注解 [@Scheduled](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/Scheduled.html) 固定周期轮询

轮询事件订阅处理可以在 [`application.yml`](event-sourcing-app/src/main/resources/application.yml) 中启用和配置

```yaml
event-sourcing:
    subscriptions: polling # Enable database polling subscription processing
    polling-subscriptions:
        polling-initial-delay: PT1S
        polling-interval: PT1S
```

#### <a id="4-7-3"></a>使用监听/通知替代数据库轮询

为了减少与数据库轮询相关的滞后，轮询周期可以设置为非常低的值，例如 1 秒。
但是这意味着即使没有新事件，每小时将有 3600 个数据库查询，每天将有 86400 个数据库查询。

PostgreSQL 的 `LISTEN` 和 `NOTIFY` 功能可以代替轮询。
该机制允许在数据库连接之间发送异步通知。
通知不是直接从应用程序发送的，而是通过表的 [trigger](event-sourcing-app/src/main/resources/db/migration/V2__notify_trigger.sql) 发送的。

要使用此功能，需要一个未共享的 [PgConnection](https://jdbc.postgresql.org/documentation/publicapi/org/postgresql/jdbc/PgConnection.html) 保持打开状态。
用于接收通知的持久 JDBC `Connection` 必须使用 `DriverManager` API 创建，而不是从池化的 `DataSource` 获取。

PostgreSQL JDBC 驱动程序无法接收异步通知，并且必须轮询后端以检查是否发出了任何通知。
超时可以给出轮询函数 `getNotifications(int timeoutMillis)`，但是其他线程的语句执行将被阻塞。
当 `timeoutMillis` = 0 时，永远阻塞，或者直到至少收到一个通知。
这意味着通知几乎立即传递，没有滞后。
如果将要接收多个通知，则将以批次返回。

这个解决方案显著减少了发出的查询数量，并且解决了轮询解决方案所遇到的滞后问题。

Listen/Notify 机制的实现 [PostgresChannelEventSubscriptionProcessor](postgresql-event-sourcing-core/src/main/java/eventsourcing/postgresql/service/PostgresChannelEventSubscriptionProcessor.java)
受到 Spring Integration 类 [PostgresChannelMessageTableSubscriber](https://github.com/spring-projects/spring-integration/blob/v6.0.0/spring-integration-jdbc/src/main/java/org/springframework/integration/jdbc/channel/PostgresChannelMessageTableSubscriber.java)
启发

Listen/Notify 事件订阅处理可以在 [`application.yml`](event-sourcing-app/src/main/resources/application.yml) 中启用

```yaml
event-sourcing:
    subscriptions: postgres-channel # Enable Listen/Notify event subscription processing
```

> **NOTE**
默认使用监听/通知机制，因为更有效率。

### <a id="4-8"></a>添加新的异步事件处理器

在重启后端后，现有的订阅将只处理最后处理的事件之后的新事件，而不是从第一个事件开始处理。

> **警告**
> 由于潜在的风险，关键内容需要立即引起用户的注意。
> 第一次轮询中的新订阅（事件处理器）将读取并处理所有事件。
> 如果事件太多，处理它们可能需要很长时间。

### <a id="4-9"></a>缺点

使用 PostgreSQL 作为事件存储有许多优势，但是也有缺点。

1. **异步事务处理器会处理相同事务多次。**
    它可能在处理事件之后但在记录其已完成的事实之前崩溃。
    当它重新启动时，它将再次处理相同的事件（例如，发送集成事件）。
    集成事件使用 **至少一次** 传递保证。
    由于双写，很难实现完全一次性传递保证。
    双写描述了这样一种情况：您需要在不使用两阶段提交（2PC）的情况下以原子方式更新数据库并发布消息。
    集成事件的消费者应该是幂等的，并过滤重复和无序事件。
2. 异步事件处理导致写模型和发送集成事件之间的 **最终一致性**。
    使用固定延迟轮询新事件的数据库表会导致完全一致性滞后大于或等于轮询之间的间隔（默认为 1 秒）。
3. **一个长时间运行的事务将有效地“暂停”所有事件处理程序**
    `pg_sanpshot_xmin(pg_snaphost)` 将会返回这个长时间运行事务的 ID，
    并且所有后续事务创建的事件将只有在这个长时间运行事务提交后才会被事件订阅处理器读取。

## <a id="5"></a>项目结构

### <a id="5-1"></a>Gradle 子项目

这个参考实现可以很容易地扩展成符合你的领域模型。

事件源相关的代码和应用程序特定的代码位于单独的 Gradle 子项目中：

* [`postgresql-event-sourcing-core`](postgresql-event-sourcing-core)：事件源和 PostgreSQL 相关的代码，
    一个共享库，`eventsourcing.postgresql` 包，
* [`event-sourcing-app`](event-sourcing-app)：应用程序特定的代码，一个简化的打车示例，`com.example.eventsourcing` 包。

`event-sourcing-app` 依赖 `postgresql-event-sourcing-core`:

```groovy
dependencies {
    implementation project(':postgresql-event-sourcing-core')
}
```

### <a id="5-2"></a>数据库结构迁移

事件源相关的表结构：

* [V1__eventsourcing_tables.sql](event-sourcing-app/src/main/resources/db/migration/V1__eventsourcing_tables.sql)
* [V2__notify_trigger.sql](event-sourcing-app/src/main/resources/db/migration/V2__notify_trigger.sql)

应用程序相关的表结构：

* [V3__projection_tables.sql](event-sourcing-app/src/main/resources/db/migration/V3__projection_tables.sql)

### <a id="5-3"></a>类图

#### <a id="5-3-1"></a>领域模型类图

![Class diagram of the domain model](img/class-domain.svg)

#### <a id="5-3-2"></a>投影类图

![Class diagram of the projections](img/class-projection.svg)

#### <a id="5-3-3"></a>服务层类图

![Class diagram of the service layer](img/class-service.svg)

## <a id="6"></a>如何适配到你的领域中？

为了适配你的领域模型，对 `event-sourcing-app` 子项目进行更改。
不需要对 `postgresql-event-sourcing-core` 子项目进行更改。

## <a id="7"></a>如何运行这个例子

1. 下载 & 安装 [SDKMAN!](https://sdkman.io/install).

2. 安装 JDK 21
    ```bash
    sdk list java
    sdk install java 21-tem
    ```

3. 安装 [Docker](https://docs.docker.com/engine/install/)
   和 [Docker Compose](https://docs.docker.com/compose/install/).

4. 构建 Java 项目和 Docker 镜像
    ```bash
    ./gradlew clean build jibDockerBuild -i
    ```

5. 运行 PostgreSQL，`Kafka` 和 `event-sourcing-app`
    ```bash
    docker compose --env-file gradle.properties up -d --scale event-sourcing-app=2
    # wait a few minutes
    ```

6. 跟踪应用程序的日志
    ```bash
    docker compose logs -f event-sourcing-app
    ```

7. 运行 E2E 测试并查看输出
    ```bash
    E2E_TESTING=true ./gradlew clean test -i
    ```

8. 在 http://localhost:8181 中探索数据库。
    在 [docker-compose.yml](docker-compose.yml) 查看库名，用户名和密码。

你也可以手动调用 REST API。

1. 安装 [curl](https://curl.se/) 和 [jq](https://stedolan.github.io/jq/)
    ```bash
    sudo apt install curl jq
    ```

2. 下一个新订单
    ```bash
    ORDER_ID=$(curl -s -X POST http://localhost:8080/orders -d '{"riderId":"63770803-38f4-4594-aec2-4c74918f7165","price":"123.45","route":[{"address":"Kyiv, 17A Polyarna Street","lat":50.51980052414157,"lon":30.467197278948536},{"address":"Kyiv, 18V Novokostyantynivska Street","lat":50.48509161169076,"lon":30.485170724431292}]}' -H 'Content-Type: application/json' | jq -r .orderId)
    ```

3. 获取已下单的订单
    ```bash
    curl -s -X GET http://localhost:8080/orders/$ORDER_ID | jq
    ```

4. 接收订单
    ```bash
    curl -s -X PUT http://localhost:8080/orders/$ORDER_ID -d '{"status":"ACCEPTED","driverId":"2c068a1a-9263-433f-a70b-067d51b98378"}' -H 'Content-Type: application/json'
    ```

5. 获取已接收的订单
    ```bash
    curl -s -X GET http://localhost:8080/orders/$ORDER_ID | jq
    ```

6. 完成订单
    ```bash
    curl -s -X PUT http://localhost:8080/orders/$ORDER_ID -d '{"status":"COMPLETED"}' -H 'Content-Type: application/json'
    ```

7. 获取已完成的订单
    ```bash
    curl -s -X GET http://localhost:8080/orders/$ORDER_ID | jq
    ```

8. 尝试取消已完成的订单以模拟业务规则违反
    ```bash
    curl -s -X PUT http://localhost:8080/orders/$ORDER_ID -d '{"status":"CANCELLED"}' -H 'Content-Type: application/json' | jq
    ```

9. 打印集成事件
    ```bash
    docker compose exec kafka /bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic order-events --from-beginning --property print.key=true --timeout-ms 10000
    ```
