## 1. System Overview

```mermaid
flowchart TB
  subgraph C["Client"]
  Client["Client"]
  end
 subgraph A["Woshi service"]
        TaskQueue["TaskQueue"]
        WoshiService["WoshiService"]        
  end
 subgraph M1["Manager (Engine Client)"]
        Manager1["Manager1"]
        EngineInstance1["EngineInstance1"]
        ParallelInstance1["ParallelInstance1"]
        ParallelInstance2["ParallelInstance2"]
        ParallelInstance3["ParallelInstance3"]
        ParallelInstance4["ParallelInstance4"]
  end

    EngineInstance1 --- |spawns| ParallelInstance1 & ParallelInstance2 & ParallelInstance3 & ParallelInstance4
    Client <-- requests schedule operations --> WoshiService    
    Manager1 --- |spawns and monitors| EngineInstance1
    TaskQueue@{ shape: cyl}
    WoshiService --> |writes task to TaskQueue| TaskQueue
    TaskQueue --> |reads task from TaskQueue| WoshiService
    Manager1 --> |polls service for tasks| WoshiService
```

## 2. Database Schema

```mermaid
erDiagram
    MANAGERS {
        uuid manager_id PK
        string manager_name
        string host_address
        int port
        enum status "offline,online"
        enum registration_status "unregistered,pending,approved,rejected"
        datetime created_at
        datetime last_seen
        datetime approved_at
        json current_engine_state
        int max_parallel_per_schedule
        int total_cores
    }

    SCHEDULE_CALCULATIONS {
        uuid schedule_id PK
        uuid assigned_manager_id FK
        enum status "pending,running,failed,stopped"
        datetime started_at
        datetime completed_at
        datetime last_heartbeat
        datetime stop_deadline_at
        json calculation_metadata
        string output_path
        string error_message
    }

    TASKS {
        uuid task_id PK
        uuid schedule_id FK
        uuid assigned_manager_id FK
        enum task_type "start_calculation,stop_calculation"
        enum status "queued,pending,accepted,completed,failed,obsolete"
        string task_data_path
        datetime created_at
        datetime assigned_at
        datetime accepted_at
        datetime completed_at
        datetime acceptance_deadline_at
        datetime execution_deadline_at
        int timeout_retry_count
        string error_message
    }

    TASK_RESULTS {
        uuid result_id PK
        uuid task_id FK
        enum result_type "success,failure"
        json result_data
        string output_file_path
        string error_details
        datetime created_at
        long file_size_bytes
    }

    TASK_HISTORY {
        uuid history_id PK
        uuid task_id FK
        enum previous_status
        enum new_status
        string reason
        json metadata
        datetime timestamp
    }

    SYSTEM_EVENTS {
        uuid event_id PK
        uuid manager_id FK
        uuid task_id FK
        uuid schedule_id FK
        enum event_type "manager_registered,manager_offline,task_failed,engine_crash"
        string description
        json event_data
        enum severity "info,warning,error,critical"
        datetime timestamp
    }

    %% Relationships
    SCHEDULE_CALCULATIONS ||--o{ TASKS : "has"
    MANAGERS ||--o{ SCHEDULE_CALCULATIONS : "executes"
    MANAGERS ||--o{ TASKS : "assigned_to"
    TASKS ||--o| TASK_RESULTS : "produces"
    TASKS ||--o{ TASK_HISTORY : "tracked_by"
    TASKS ||--o{ SYSTEM_EVENTS : "referenced_by"
    MANAGERS ||--o{ SYSTEM_EVENTS : "referenced_by"
    SCHEDULE_CALCULATIONS ||--o{ SYSTEM_EVENTS : "referenced_by"
```



## 3. Manager Selection Process

```mermaid
flowchart TD
    A[New Schedule Task] --> B[Calculate Available Cores per Manager]
    B --> C[Assign to Manager with Most Available Cores]

    subgraph "Manager Pool - Available Cores"
        M1["Manager 1<br/>16 total cores<br/>4 cores used<br/>📊 12 cores available"]
        M2["Manager 2<br/>12 total cores<br/>8 cores used<br/>📊 4 cores available"]
        M3["Manager 3<br/>8 total cores<br/>7 cores used<br/>📊 1 core available"]
    end

    subgraph "Core Calculation"
        CC["Calculate how many cores the schedule would get if assigned to this manager (based on total cores and max parallels per schedule)."]
    end

    B --> CC
    C --> M1
    
    style M1 fill:#90EE90
    style M2 fill:#FFE4B5
    style M3 fill:#FFA07A
```

## 4. Manager-Server Communication

### Manager Communication (Manager POV)

```mermaid
flowchart TD
    Start([Manager Starts]) --> Connect[Connect to Server]
    Connect --> ConnSuccess{Connection<br/>Successful?}
    
    ConnSuccess -->|No| Retry[Wait & Retry]
    Retry --> Connect
    
    ConnSuccess -->|Yes| Poll[Poll Server]
    Poll --> Response{Server Response?}
    
    Response -->|Not Registered<br/>Registration Not Approved| Poll
    Response -->|Connection Lost| Connect
    
    Response -->|Response + Expected State| StateMatch{State<br/>Match?}
    StateMatch -->|No| CorrectState[Synchronize with server state and reject tasks]
    StateMatch -->|Yes| TaskCheck{Task<br/>Assignment?}
    
    TaskCheck -->|No| Poll
    TaskCheck -->|Yes| ProcessTask[Process Task]
    ProcessTask --> ReportResult[Report Result]
    
    CorrectState --> Poll
    ReportResult --> Poll
```

### Manager Communication (Server POV)

```mermaid
flowchart TD
    Start([Manager Poll]) --> Registered{Manager<br/>Registered?}
    
    Registered -->|No| CreateEntry[Create Pending Entry]
    CreateEntry --> NotRegistered[Send: Not Registered]
    
    Registered -->|Yes| Approved{Manager<br/>Approved?}
    Approved -->|No| NotApproved[Send: Not Approved]
    
    Approved -->|Yes| UpdateHeartbeat[Update Last Seen]
    UpdateHeartbeat --> CheckStatus[Update Manager Status to Online]
    CheckStatus --> SendResponse[Send Tasks + Expected State]
    
    NotRegistered --> End([End])
    NotApproved --> End
    SendResponse --> End
```

## 5. State Reconciliation

### Manager-Server State Reconciliation Process

```mermaid
flowchart TD
    A[Manager Polls Server] --> B[Server Sends Expected Schedules + Tasks]
    B --> C{Expected vs Actual<br/>Schedules Match?}
    
    C -->|Yes| A
    C -->|No| E[Reject All Tasks & Reconcile]
    
    E --> F{For each mismatch}
    F -->|Schedule is running on manager but not on server| G[Stop Schedule]
    F -->|Schedule not running but server expects it to be running| H[CRITICAL ERROR<br/>Alert system]
    
    note1[Server believes calculation is running<br/>but manager has no active process.<br/>Data integrity issue - clients may think<br/>work is progressing when it isn't.]
    H -.- note1
    
    G --> I{More mismatches?}
    H --> I
    I -->|Yes| F
    I -->|No| A
    
    
    style H fill:#ff4444
    style note1 fill:#ffffcc
```

## 6. State Diagrams

### Schedule Calculation States

```mermaid
stateDiagram-v2
    [*] --> pending: Start Calculation Task Created
    pending --> running: Start Calculation Task Completed Successfully
    pending --> failed: Start Calculation Task Failed
    
    running --> completed: Calculation Finished Successfully
    running --> stopped: Stop Calculation Task Completed
    running --> failed: Calculation Failed/Error
    
    completed --> [*]
    stopped --> [*]
    failed --> [*]
    
    Note: This represents the SCHEDULE_CALCULATIONS.status field
```

### Start Calculation Task States

```mermaid
stateDiagram-v2
    [*] --> queued: Task Created
    queued --> pending: Assigned to Manager
    pending --> accepted: Manager Accepts
    pending --> failed: Manager Rejects/Error
    pending --> failed: Acceptance Timeout

    accepted --> completed: Engine Started Successfully
    accepted --> failed: Engine Failed to Start
    accepted --> failed: Execution Timeout

    completed --> [*]
    failed --> [*]
```

### Stop Calculation Task States

```mermaid
stateDiagram-v2
    [*] --> queued: Task Created
    queued --> pending: Assigned to Manager
    pending --> accepted: Manager Accepts
    pending --> failed: Manager Rejects/Error
    pending --> failed: Acceptance Timeout

    accepted --> completed: Stop Successful
    accepted --> failed: Stop Failed
    accepted --> failed: Execution Timeout

    completed --> [*]
    failed --> [*]
```

## 7. Core Process Flows

### Start Calculation Sequence

```mermaid
flowchart TD
    A[POST /send-start-calculation] --> B{Server: Schedule running?}
    B -->|Yes| C[Server: Return 200 + SCHEDULE_ALREADY_RUNNING]
    B -->|No| D{Server: Manager available?}
    D -->|No| E[Server: Return 200 + NO_MANAGER_AVAILABLE]
    D -->|Yes| F[Server: Create ScheduleTask queued]
    F --> G[Server: Set task deadline]
    G --> H{Manager: State sync check}
    H -->|Out of sync| HA[Manager: Reject task & reconcile state]
    HA --> HB[Server: Return 200 + STATE_OUT_OF_SYNC]
    H -->|In sync| I{Manager: Accept task?}
    I -->|Timeout| J[Server: Mark task failed]
    J --> K[Server: Return 200 + TASK_TIMEOUT]
    I -->|Yes| L[Server: Update task to accepted]
    L --> M[Manager: Write schedule files]
    M --> N[Manager: Run Engine.exe]
    N --> O{Manager: Engine execution result}
    O -->|Success| P[Server: Create SCHEDULE_CALCULATIONS running]
    P --> Q[Server: Return 200 + SUCCESS]
    O -->|Failure| R[Server: Return 200 + ENGINE_FAILED]
    O -->|Invalid| S[Server: Return 200 + INVALID_REQUIREMENTS]
```

### Stop Calculation Sequence

```mermaid
flowchart TD
    V[POST /send-stop-calculation] --> W{Server: Schedule found & running?}
    W -->|No| X[Server: Return 200 + SCHEDULE_NOT_FOUND]
    W -->|Yes| Y{Server: Manager available?}
    Y -->|No| Z[Server: Return 200 + MANAGER_UNAVAILABLE]
    Y -->|Yes| AA[Server: Create stop task queued]
    AA --> BB[Server: Set task deadline]
    BB --> CC{Manager: State sync check}
    CC -->|Out of sync| CCA[Manager: Reject task & reconcile state]
    CCA --> CCB[Server: Return 200 + STATE_OUT_OF_SYNC] 
    CC -->|In sync| DD{Manager: Accept task?}
    DD -->|Timeout| EE[Server: Task failed]
    DD -->|Yes| FF[Server: Update task to accepted]
    FF --> GG{Manager: Schedule actually running?}
    GG -->|No| HH[Server: Return 200 + SCHEDULE_NOT_RUNNING]
    GG -->|Yes| II[Manager: Stop engine & cleanup]
    II --> JJ[Server: Update SCHEDULE_CALCULATIONS stopped]
    JJ --> KK[Server: Return 200 + SUCCESS]
```

## 8. Timeout and Monitoring Processes

### Task Deadline Process

```mermaid
flowchart TD
    Start([Task Created]) --> SetDeadline[Set Task Acceptance Deadline]
    SetDeadline --> Queue[Add to Task Queue]
    Queue --> Assign[Assign to Manager]
    
    Assign --> Processing[Task Processing]
    Processing --> ManagerAccept{Manager Accepts<br/>Before Deadline?}
    
    ManagerAccept -->|Yes| SetExecDeadline[Set Task Execution Deadline]
    ManagerAccept -->|No| Timeout1[Task Timeout - Acceptance]
    
    SetExecDeadline --> Execute[Execute Task]
    Execute --> Completes{Completes<br/>Before Execution Deadline?}
    Completes -->|Yes| Success[Task Success]
    Completes -->|No| Timeout2[Task Timeout - Execution]
    
    Timeout1 --> HandleTimeout[Handle Timeout]
    Timeout2 --> HandleTimeout
    
    HandleTimeout --> FinalFailure[Final Failure]
    
    Success --> UpdateState[Update System State]
    FinalFailure --> UpdateState
    
    UpdateState --> Notify[Notify Client]
    
```

### Timeout Manager

```mermaid
flowchart TD
    Start([Timeout Manager Starts]) --> Init[Initialize Timeout Parameters]
    Init --> CheckTasks[Check for Timed Out Tasks]
    Init --> CheckSchedules[Check for Timed Out Schedules]
    
    CheckTasks --> TaskTimeout{Task Deadline<br/>Expired?}
    TaskTimeout -->|Yes| HandleTaskTimeout[Handle Task Timeout]
    TaskTimeout -->|No| CheckSchedules
    
    CheckSchedules --> ScheduleTimeout{Schedule Heartbeat<br/>Timeout?}
    ScheduleTimeout -->|Yes| HandleScheduleTimeout[Handle Schedule Timeout]
    ScheduleTimeout -->|No| Wait[Wait for Next Interval]
    
    HandleTaskTimeout --> AtomicUpdate[Attempt Atomic State Update]
    AtomicUpdate --> UpdateSuccessful{Update Successful?}
    
    UpdateSuccessful -->|Yes| LogTaskTimeout[Log Task Timeout Event]
    UpdateSuccessful -->|No| AlreadyHandled[Task already handled by manager]
    
    LogTaskTimeout --> NotifyTaskTimeout[Notify System of Task Failure]
    NotifyTaskTimeout --> CheckSchedules
    
    HandleScheduleTimeout --> UpdateScheduleStatus[Update Schedule Status to Failed]
    UpdateScheduleStatus --> LogScheduleTimeout[Log Schedule Timeout Event]
    LogScheduleTimeout --> CleanupResources[Cleanup Orphaned Resources]
    CleanupResources --> NotifyScheduleTimeout[Notify System of Schedule Failure]
    NotifyScheduleTimeout --> Wait
    
    AlreadyHandled --> CheckSchedules
    
    Wait -->|Interval Elapses| CheckTasks
```

## 9. Engine Execution Details

### Engine Started with script_id

```mermaid
flowchart TD
    A[Engine Started] --> B[Parse Arguments & Determine File Paths]
    B --> C{LLS File Exists?}
    
    C -->|No| D[Write Error Output & Exit]
    C -->|Yes| E[Write PID File]
    
    E --> F[Read LLS and OLD Files]
    F --> G[Generate & Write OUT File]
    G --> H[Enter Monitoring Loop]
    
    H --> I{My LLS File Still Exists?}
    
    I -->|No| J[Terminate Parallels & Clean All Files]
    I -->|Yes| K[Touch OUT File & Check for New Parallel LLS Files]
    
    K --> L[Spawn Parallel Processes if Found]
    L --> H
    
    J --> M[Exit]
    
    style J fill:#ffcccc
```

# Resource Cleanup and Orphan Detection Process

```mermaid
flowchart TD
    Start([Manager Health Check<br/>Every X seconds]) --> Cross[Cross-Reference:<br/>LLS, PID ↔ Process<br/>pid file must have process and process must have pid file]
    Cross --> Problems{Mismatches Found?}
    
    Problems -->|No| Wait[Wait X seconds]
    Problems -->|Yes| Classify(Classify mismatch)
    
    Classify -->|LLS exists but process dead| Restart[Restart Engine Process<br/>& Notify Server of Recovery]
    Classify -->|Stale files only| CleanFiles[Remove Stale Files]
    Classify -->|Orphaned processes| TermProc[Terminate Orphaned Processes]
    
    Restart --> Wait
    CleanFiles --> Wait
    TermProc --> Wait
    
    Wait --> Start
    
    style Problems fill:#ffeeaa
    style Classify fill:#ffcccc
```



