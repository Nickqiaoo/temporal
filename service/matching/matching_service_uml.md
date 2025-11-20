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
