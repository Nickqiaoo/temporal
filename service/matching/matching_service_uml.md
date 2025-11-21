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

## AddTask 流程图 (priTaskMatcher)

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
    GetPhysicalQueue --> TrySyncMatch[physicalTaskQueueManager.TrySyncMatch]

    TrySyncMatch --> PriOffer[priTaskMatcher.Offer]
    PriOffer --> MatchImmediate[matcherData.MatchTaskImmediately]

    MatchImmediate --> HasPoller{有等待的Poller<br/>或TaskForwarder?}
    HasPoller --> |是 - Poller| SyncMatch[同步匹配成功<br/>直接交付给Poller]
    HasPoller --> |是 - TaskForwarder| ForwardTask[后台goroutine<br/>priForwarder.ForwardTask]
    ForwardTask --> ParentPartition[转发到父分区]
    ParentPartition --> ReturnSuccess([返回成功])
    SyncMatch --> ReturnSuccess

    HasPoller --> |否| CheckBlock{是root且<br/>转发来的backlog?}
    CheckBlock --> |是| EnqueueWait[matcherData.EnqueueTaskAndWait<br/>阻塞等待匹配]
    EnqueueWait --> WaitMatch[等待Poller到达]
    WaitMatch --> ReturnSuccess

    CheckBlock --> |否| SyncFail[同步匹配失败]
    SyncFail --> SpoolTask[physicalTaskQueueManager.SpoolTask]
    SpoolTask --> BacklogMgr[backlogManager.SpoolTask]
    BacklogMgr --> TaskWriter[taskWriter写入DB]
    TaskWriter --> DB[(持久化到数据库)]
    DB --> TaskReader[taskReader读取任务]
    TaskReader --> AddToMatcher[priMatcher.AddTask]
    AddToMatcher --> EnqueueNoWait[matcherData.EnqueueTaskNoWait<br/>加入任务队列等待匹配]
    EnqueueNoWait --> ReturnSuccess

    style Start fill:#e1f5fe
    style ReturnSuccess fill:#c8e6c9
    style DB fill:#fff3e0
    style SyncMatch fill:#c8e6c9
    style ForwardTask fill:#e8f5e9
```

### AddTask 流程说明 (pri逻辑)

1. **入口**: History Service 通过 gRPC 调用 Matching Service
2. **路由**: Engine 根据 TaskQueue 名称和分区找到或创建 PartitionManager
3. **版本选择**: PartitionManager 根据 Worker 版本选择正确的物理队列
4. **同步匹配尝试**:
   - `priTaskMatcher.Offer` → `matcherData.MatchTaskImmediately`
   - 如果有等待的Poller,直接匹配交付
   - 如果匹配到TaskForwarder (子分区后台goroutine),任务被转发到父分区
5. **阻塞等待**: Root分区收到转发来的backlog任务会阻塞等待Poller
6. **持久化**: 同步匹配失败后,写入backlog,taskReader读取后调用`AddTask`加入匹配队列

---

## PollTask 流程图 (priTaskMatcher)

```mermaid
flowchart TD
    Start([Worker 调用]) --> Handler[Handler.PollWorkflowTaskQueue/PollActivityTaskQueue]
    Handler --> Engine[matchingEngineImpl.PollWorkflowTaskQueue]
    Engine --> GetPartition[getTaskQueuePartitionManager]
    GetPartition --> PartitionPoll[partitionManager.PollTask]

    PartitionPoll --> GetPhysicalQueue[getPhysicalQueue<br/>根据 Worker 版本选择]
    GetPhysicalQueue --> PhysicalPoll[physicalTaskQueueManager.PollTask]

    PhysicalPoll --> MatcherPoll[priTaskMatcher.Poll]
    MatcherPoll --> CreatePoller[创建 waitingPoller<br/>设置 forwardCtx 和 pollMetadata]
    CreatePoller --> EnqueuePoller[matcherData.EnqueuePollerAndWait]

    EnqueuePoller --> CheckMatch{matcherData 匹配}
    CheckMatch --> |有等待的任务| GetTask[从任务队列获取任务]
    CheckMatch --> |无任务| WaitInQueue[Poller加入等待队列]

    WaitInQueue --> WaitForTask[等待任务到达或超时]
    WaitForTask --> |超时| Timeout([返回 errNoTasks])
    WaitForTask --> |任务到达| GetTask

    GetTask --> CheckTaskType{任务类型}
    CheckTaskType --> |同步任务| SyncTask[Offer来的同步任务]
    CheckTaskType --> |Backlog任务| BacklogTask[taskReader读取的任务]
    CheckTaskType --> |转发来的任务| ForwardedTask[从父分区转发来]

    SyncTask --> CheckExpired
    BacklogTask --> CheckExpired
    ForwardedTask --> CheckExpired{任务是否过期?}

    CheckExpired --> |是| ExpireTask[标记过期,继续等待]
    ExpireTask --> EnqueuePoller
    CheckExpired --> |否| RecordStart[recordWorkflowTaskStarted<br/>调用 History Service]

    RecordStart --> ReturnTask([返回任务给 Worker])

    style Start fill:#e1f5fe
    style ReturnTask fill:#c8e6c9
    style Timeout fill:#ffcdd2
    style GetTask fill:#e8f5e9
```

### 子分区后台 Goroutine

```mermaid
flowchart LR
    subgraph 子分区后台处理
        direction TB
        FT[forwardTasks goroutine] --> EnqueueAsPoller[以TaskForwarder身份<br/>EnqueuePollerAndWait]
        EnqueueAsPoller --> GetBacklogTask[获取backlog任务]
        GetBacklogTask --> Forward[priForwarder.ForwardTask<br/>转发到父分区]

        FP[forwardPolls goroutine] --> EnqueueForwarderTask[以ForwarderTask身份<br/>EnqueueTaskAndWait]
        EnqueueForwarderTask --> GetPoller[获取等待的Poller]
        GetPoller --> ForwardPoll[priForwarder.ForwardPoll<br/>转发到父分区]
        ForwardPoll --> |成功| FinishMatch[FinishMatchAfterPollForward]
        ForwardPoll --> |失败| ReenqueuePoller[ReenqueuePollerIfNotMatched<br/>重新入队]
    end

    style Forward fill:#e8f5e9
    style ForwardPoll fill:#e8f5e9
```

### PollTask 流程说明 (pri逻辑)

1. **入口**: Worker 通过 gRPC 长轮询 Matching Service
2. **创建Poller**: 创建 `waitingPoller` 包含转发上下文和元数据
3. **入队等待**: `matcherData.EnqueuePollerAndWait` 将Poller加入等待队列
4. **匹配机制**: matcherData 统一管理任务和Poller的匹配:
   - 如果有等待的任务,立即匹配返回
   - 否则Poller加入队列等待任务到达
5. **任务来源**:
   - **同步任务**: 通过 `Offer` 添加的任务
   - **Backlog任务**: taskReader 读取后通过 `AddTask` 添加
   - **转发任务**: 从子分区转发来的任务
6. **子分区转发**: 后台goroutine负责:
   - `forwardTasks`: 将本地backlog任务转发到父分区
   - `forwardPolls`: 将本地Poller转发到父分区获取任务
7. **过期检查**: 返回前检查任务是否过期,过期则丢弃继续等待

---

## 同步匹配机制 (matcherData)

```mermaid
flowchart LR
    subgraph AddTask流程
        Task[新任务] --> Offer[priTaskMatcher.Offer]
        Offer --> MatchImm[MatchTaskImmediately]
    end

    subgraph matcherData
        MatchImm --> CheckPoller{pollerQueues<br/>有等待Poller?}
        CheckPoller --> |是| Match[直接匹配]
        CheckPoller --> |否| EnqueueTask[EnqueueTaskAndWait<br/>加入taskQueues]

        EnqueuePoll --> CheckTask{taskQueues<br/>有等待任务?}
        CheckTask --> |是| Match
        CheckTask --> |否| WaitPoller[加入pollerQueues<br/>等待任务]
    end

    subgraph PollTask流程
        Poller[Poller] --> Poll[priTaskMatcher.Poll]
        Poll --> EnqueuePoll[EnqueuePollerAndWait]
    end

    Match --> Success([匹配成功<br/>零延迟交付])

    style Match fill:#c8e6c9
    style Success fill:#c8e6c9
```

### matcherData 核心数据结构

- **taskQueues (taskPQ)**: 等待匹配的任务优先级队列
- **pollerQueues (pollerPQ)**: 等待任务的Poller优先级队列
- **关键方法**:
  - `MatchTaskImmediately`: 快速路径,尝试立即匹配
  - `EnqueueTaskAndWait`: 任务入队并阻塞等待Poller
  - `EnqueuePollerAndWait`: Poller入队并阻塞等待任务
  - `EnqueueTaskNoWait`: Backlog任务入队不阻塞

同步匹配是 Matching Service 的核心优化:
- 当任务和 Poller 同时存在时,直接匹配,跳过数据库
- 大幅降低任务调度延迟
