# Matching Service UML Class Diagram

```mermaid
classDiagram
    direction TB

    %% Service Layer
    class Service {
        +handler Handler
        +Start()
        +Stop()
    }

    class Handler {
        +engine Engine
        +AddWorkflowTask()
        +AddActivityTask()
        +PollWorkflowTaskQueue()
        +PollActivityTaskQueue()
        +QueryWorkflow()
    }

    %% Engine Layer
    class Engine {
        <<interface>>
        +AddWorkflowTask()
        +AddActivityTask()
        +PollWorkflowTaskQueue()
        +PollActivityTaskQueue()
        +GetTaskQueueUserData()
    }

    class matchingEngineImpl {
        -partitions map~taskQueuePartitionManager~
        -taskManager persistence.TaskManager
        -historyClient HistoryServiceClient
        -matchingRawClient MatchingServiceClient
        -config Config
        +getTaskQueuePartitionManager()
        +unloadTaskQueuePartition()
    }

    %% Partition Manager Layer
    class taskQueuePartitionManager {
        <<interface>>
        +AddTask()
        +PollTask()
        +GetUserDataManager()
        +LegacyDescribeTaskQueue()
    }

    class taskQueuePartitionManagerImpl {
        -engine Engine
        -partition tqid.Partition
        -config Config
        -defaultQueue physicalTaskQueueManager
        -versionedQueues map~physicalTaskQueueManager~
        -userDataManager userDataManager
        +getPhysicalQueue()
    }

    %% Physical Queue Manager Layer
    class physicalTaskQueueManager {
        <<interface>>
        +Start()
        +Stop()
        +AddTask()
        +PollTask()
        +SpoolTask()
        +HasBackloggedTasks()
    }

    class physicalTaskQueueManagerImpl {
        -queue PhysicalTaskQueueKey
        -config Config
        -backlogMgr backlogManager
        -matcher TaskMatcher
        -priMatcher priTaskMatcher
        -liveness liveness
    }

    %% Task Matcher Layer
    class TaskMatcher {
        -config Config
        -fwdr Forwarder
        -taskC chan~internalTask~
        -queryTaskC chan~internalTask~
        +Offer()
        +OfferQuery()
        +MustOffer()
        +Poll()
    }

    class priTaskMatcher {
        -config Config
        -fwdr priForwarder
        -data matcherData
        -readyCond sync.Cond
        +AddTask()
        +AddPoller()
        +WaitForTask()
    }

    class matcherData {
        -taskQueues taskPQ
        -pollerQueues pollerPQ
        +addTask()
        +addPoller()
        +waitingPollerCount()
        +backloggedTaskCount()
    }

    %% Backlog Manager Layer
    class backlogManager {
        <<interface>>
        +Start()
        +Stop()
        +SpoolTask()
        +BacklogStatus()
        +TasksRemaining()
    }

    class backlogManagerImpl {
        -taskQueueDB taskQueueDB
        -taskWriter taskWriter
        -taskReader taskReader
        -ackManager ackManager
        -outstandingTasksLimit int
    }

    %% Persistence Layer
    class taskQueueDB {
        +CreateTasks()
        +CompleteTasksLessThan()
        +GetTasks()
        +UpdateState()
    }

    class taskWriter {
        -taskQueueDB taskQueueDB
        -appendCh chan~writeTaskRequest~
        +appendTask()
    }

    class taskReader {
        -taskQueueDB taskQueueDB
        -dispatchCh chan~internalTask~
        +getTaskBatch()
        +dispatchTasks()
    }

    class ackManager {
        -db taskQueueDB
        -backlogAge int64
        +addTask()
        +completeTask()
        +getAckLevel()
    }

    %% Forwarder Layer
    class Forwarder {
        -client MatchingServiceClient
        -taskQueueID tqid.TaskQueue
        +ForwardTask()
        +ForwardPoll()
        +ForwardQueryTask()
    }

    class priForwarder {
        -client MatchingServiceClient
        -partition tqid.Partition
        +StartForwarding()
        +ForwardTask()
        +ForwardPoll()
    }

    %% User Data Manager
    class userDataManager {
        <<interface>>
        +GetUserData()
        +UpdateUserData()
    }

    %% Rate Limit Manager
    class rateLimitManager {
        -partitionRateLimiter quotas.RateLimiter
        -dispatchRateLimiter quotas.RateLimiter
        +Wait()
    }

    %% Data Types
    class internalTask {
        +event persistencespb.TaskQueueInfo
        +responseC chan~syncMatchResponse~
        +isForwarded bool
        +isQuery()
        +isStarted()
        +finish()
    }

    class PhysicalTaskQueueKey {
        +Partition tqid.Partition
        +VersionSet string
        +TaskType TaskType
    }

    %% Relationships
    Service --> Handler : contains
    Handler --> Engine : uses
    Engine <|.. matchingEngineImpl : implements
    matchingEngineImpl --> taskQueuePartitionManager : manages *

    taskQueuePartitionManager <|.. taskQueuePartitionManagerImpl : implements
    taskQueuePartitionManagerImpl --> physicalTaskQueueManager : defaultQueue
    taskQueuePartitionManagerImpl --> physicalTaskQueueManager : versionedQueues *
    taskQueuePartitionManagerImpl --> userDataManager : uses

    physicalTaskQueueManager <|.. physicalTaskQueueManagerImpl : implements
    physicalTaskQueueManagerImpl --> backlogManager : uses
    physicalTaskQueueManagerImpl --> TaskMatcher : legacy
    physicalTaskQueueManagerImpl --> priTaskMatcher : new
    physicalTaskQueueManagerImpl --> rateLimitManager : uses

    priTaskMatcher --> matcherData : uses
    priTaskMatcher --> priForwarder : uses
    TaskMatcher --> Forwarder : uses

    backlogManager <|.. backlogManagerImpl : implements
    backlogManagerImpl --> taskQueueDB : uses
    backlogManagerImpl --> taskWriter : uses
    backlogManagerImpl --> taskReader : uses
    backlogManagerImpl --> ackManager : uses

    taskWriter --> taskQueueDB : writes to
    taskReader --> taskQueueDB : reads from
    ackManager --> taskQueueDB : updates
```

## 架构层次说明

### 1. 服务层 (Service Layer)
- **Service**: 服务入口,管理生命周期
- **Handler**: gRPC处理器,接收外部请求

### 2. 引擎层 (Engine Layer)
- **matchingEngineImpl**: 核心引擎,管理所有任务队列分区

### 3. 分区管理层 (Partition Manager Layer)
- **taskQueuePartitionManagerImpl**: 逻辑队列分区,管理多个物理队列

### 4. 物理队列层 (Physical Queue Layer)
- **physicalTaskQueueManagerImpl**: 物理队列,包含匹配器和积压管理

### 5. 任务匹配层 (Task Matcher Layer)
- **TaskMatcher**: 传统匹配器 (channel-based)
- **priTaskMatcher**: 新优先级匹配器 (priority queue-based)

### 6. 持久化层 (Persistence Layer)
- **backlogManager**: 积压管理,处理DB操作
- **taskWriter/taskReader/ackManager**: 读写和确认管理

## 核心流程

```
AddTask: Handler → Engine → PartitionManager → PhysicalQueueManager → Matcher/Backlog
PollTask: Handler → Engine → PartitionManager → PhysicalQueueManager → Matcher
```

---

## AddTask 流程图

```mermaid
flowchart TD
    Start([History Service 调用]) --> Handler[Handler.AddWorkflowTask/AddActivityTask]
    Handler --> Engine[matchingEngineImpl.AddWorkflowTask]
    Engine --> GetPartition[getTaskQueuePartitionManager]
    GetPartition --> |不存在| CreatePartition[创建新的 partitionManager]
    GetPartition --> |存在| UsePartition[使用现有 partitionManager]
    CreatePartition --> PartitionAdd
    UsePartition --> PartitionAdd[partitionManager.AddTask]

    PartitionAdd --> GetPhysicalQueue[getPhysicalQueue<br/>根据版本选择队列]
    GetPhysicalQueue --> PhysicalAdd[physicalTaskQueueManager.AddTask]

    PhysicalAdd --> CheckForwarding{是否需要转发?<br/>非root分区}
    CheckForwarding --> |是| Forward[Forwarder.ForwardTask<br/>转发到父分区]
    Forward --> ForwardResult{转发成功?}
    ForwardResult --> |是| ReturnSuccess([返回成功])
    ForwardResult --> |否| TrySyncMatch

    CheckForwarding --> |否| TrySyncMatch[尝试同步匹配]

    TrySyncMatch --> MatcherOffer[TaskMatcher.Offer / priTaskMatcher.AddTask]
    MatcherOffer --> HasPoller{有等待的Poller?}

    HasPoller --> |是| SyncMatch[同步匹配成功<br/>直接交付任务]
    SyncMatch --> ReturnSuccess

    HasPoller --> |否| SpoolTask[SpoolTask 持久化到积压]
    SpoolTask --> BacklogMgr[backlogManager.SpoolTask]
    BacklogMgr --> TaskWriter[taskWriter.appendTask]
    TaskWriter --> DB[(taskQueueDB.CreateTasks)]
    DB --> AckMgr[ackManager.addTask]
    AckMgr --> ReturnSuccess

    style Start fill:#e1f5fe
    style ReturnSuccess fill:#c8e6c9
    style DB fill:#fff3e0
    style SyncMatch fill:#c8e6c9
```

### AddTask 流程说明

1. **入口**: History Service 通过 gRPC 调用 Matching Service
2. **路由**: Engine 根据 TaskQueue 名称和分区找到或创建 PartitionManager
3. **版本选择**: PartitionManager 根据 Worker 版本选择正确的物理队列
4. **转发**: 如果当前是子分区,尝试转发到父分区以提高匹配效率
5. **同步匹配**: 尝试直接匹配等待的 Poller (零延迟)
6. **持久化**: 如果没有等待的 Poller,将任务写入数据库积压

---

## PollTask 流程图

```mermaid
flowchart TD
    Start([Worker 调用]) --> Handler[Handler.PollWorkflowTaskQueue/PollActivityTaskQueue]
    Handler --> Engine[matchingEngineImpl.PollWorkflowTaskQueue]
    Engine --> GetPartition[getTaskQueuePartitionManager]
    GetPartition --> PartitionPoll[partitionManager.PollTask]

    PartitionPoll --> GetPhysicalQueue[getPhysicalQueue<br/>根据 Worker 版本选择]
    GetPhysicalQueue --> PhysicalPoll[physicalTaskQueueManager.PollTask]

    PhysicalPoll --> RateLimit[rateLimitManager.Wait<br/>等待限流令牌]
    RateLimit --> MatcherPoll[TaskMatcher.Poll / priTaskMatcher.AddPoller]

    MatcherPoll --> CheckTask{检查任务来源}

    CheckTask --> |有同步任务| SyncTask[从 Matcher 获取同步任务]
    SyncTask --> RecordStart[recordWorkflowTaskStarted<br/>调用 History Service]

    CheckTask --> |有积压任务| BacklogTask[从 Backlog 获取任务]
    BacklogTask --> TaskReader[taskReader.dispatchTasks]
    TaskReader --> DB[(taskQueueDB.GetTasks)]
    DB --> RecordStart

    CheckTask --> |无任务| CheckForward{是否可以转发?}
    CheckForward --> |是| ForwardPoll[Forwarder.ForwardPoll<br/>转发到父分区]
    ForwardPoll --> ForwardResult{转发成功?}
    ForwardResult --> |是| RecordStart
    ForwardResult --> |否| Wait[等待任务到达<br/>阻塞直到超时]

    CheckForward --> |否| Wait
    Wait --> Timeout{超时?}
    Timeout --> |是| ReturnEmpty([返回空响应])
    Timeout --> |否| CheckTask

    RecordStart --> HistoryCall[historyClient.RecordWorkflowTaskStarted]
    HistoryCall --> AckTask[ackManager.completeTask]
    AckTask --> ReturnTask([返回任务给 Worker])

    style Start fill:#e1f5fe
    style ReturnTask fill:#c8e6c9
    style ReturnEmpty fill:#ffcdd2
    style DB fill:#fff3e0
```

### PollTask 流程说明

1. **入口**: Worker 通过 gRPC 长轮询 Matching Service
2. **限流**: 通过 rateLimitManager 控制分发速率
3. **任务来源优先级**:
   - **同步任务**: 直接从 Matcher 获取刚添加的任务 (最低延迟)
   - **积压任务**: 从数据库读取之前持久化的任务
   - **转发任务**: 转发 Poll 到父分区获取任务
4. **记录启动**: 调用 History Service 记录任务已被 Worker 领取
5. **确认完成**: 更新 ackManager 以便清理已完成的任务
6. **超时处理**: 如果在超时时间内没有任务,返回空响应

---

## 同步匹配机制

```mermaid
flowchart LR
    subgraph AddTask流程
        Task[新任务] --> Offer[Matcher.Offer]
    end

    subgraph Matcher
        Offer --> Check{有等待Poller?}
        Check --> |是| Match[直接匹配]
        Check --> |否| Queue[入队等待]
    end

    subgraph PollTask流程
        Poller[Poller] --> Poll[Matcher.Poll]
        Poll --> Check2{有等待任务?}
        Check2 --> |是| Match
        Check2 --> |否| Wait[入队等待]
    end

    Match --> Success([匹配成功<br/>零延迟交付])

    style Match fill:#c8e6c9
    style Success fill:#c8e6c9
```

同步匹配是 Matching Service 的核心优化:
- 当任务和 Poller 同时存在时,直接匹配,跳过数据库
- 大幅降低任务调度延迟
