# System Architecture Diagrams

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        MobileApp[Qantas Pay Mobile App]
    end
    
    subgraph "API Gateway Layer"
        APIGateway[Amazon API Gateway]
        Authorizer[Lambda Authorizer]
    end
    
    subgraph "Core Services"
        EnrollmentService[Member Enrollment Service<br/>Lambda + Aurora]
        LoadTrackingService[Load Tracking Service<br/>Lambda + DynamoDB]
        ReconciliationEngine[Reconciliation Engine<br/>Step Functions + Lambda]
        WithdrawalTracker[Withdrawal Tracker<br/>Lambda + DynamoDB Streams]
        SchedulerService[30-Day Window Scheduler<br/>EventBridge Scheduler]
        RewardService[Reward Issuance Service<br/>Lambda + SQS FIFO]
        NotificationService[Notification Service<br/>Lambda + SNS + Pinpoint]
    end
    
    subgraph "Data Storage"
        Aurora[(Amazon Aurora<br/>PostgreSQL)]
        DynamoDB[(Amazon DynamoDB)]
        S3[(Amazon S3)]
    end
    
    subgraph "Integration Layer"
        EventBridge[Amazon EventBridge]
        SQS[Amazon SQS FIFO]
        DataWarehouse[Data Warehouse<br/>Overnight Feed]
    end
    
    subgraph "External Systems"
        PaymentGateway[Payment Gateway<br/>Instant Load Webhooks]
        LoyaltyAPI[Qantas Loyalty<br/>Points API]
    end
    
    subgraph "Observability"
        CloudWatch[CloudWatch Logs & Metrics]
        XRay[AWS X-Ray]
    end
    
    MobileApp -->|HTTPS/REST| APIGateway
    APIGateway --> Authorizer
    APIGateway --> EnrollmentService
    APIGateway --> LoadTrackingService
    
    EnrollmentService --> Aurora
    EnrollmentService --> EventBridge
    
    LoadTrackingService --> DynamoDB
    LoadTrackingService --> EventBridge
    
    PaymentGateway -->|Webhook| LoadTrackingService
    DataWarehouse -->|Daily File| S3
    
    S3 -->|S3 Event| ReconciliationEngine
    ReconciliationEngine --> DynamoDB
    ReconciliationEngine --> Aurora
    ReconciliationEngine --> EventBridge
    
    DynamoDB -->|Streams| WithdrawalTracker
    WithdrawalTracker --> EventBridge
    
    EnrollmentService --> SchedulerService
    SchedulerService -->|30 Days + Grace| RewardService
    
    EventBridge --> RewardService
    RewardService --> SQS
    SQS --> RewardService
    RewardService --> LoyaltyAPI
    RewardService --> NotificationService
    
    NotificationService --> MobileApp
    
    EnrollmentService --> CloudWatch
    LoadTrackingService --> CloudWatch
    ReconciliationEngine --> CloudWatch
    RewardService --> CloudWatch
    
    EnrollmentService --> XRay
    LoadTrackingService --> XRay
    RewardService --> XRay
    
    style MobileApp fill:#e1f5ff
    style Aurora fill:#ff9999
    style DynamoDB fill:#ff9999
    style EventBridge fill:#ffcc99
    style RewardService fill:#99ff99
    style SchedulerService fill:#cc99ff
```

## 2. Member Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> ENROLLED: Member joins program
    
    ENROLLED --> ACTIVE: First load received
    ENROLLED --> DISQUALIFIED: Window expires with zero loads
    
    ACTIVE --> GOAL_ACHIEVED: Cumulative total >= $1000
    ACTIVE --> DISQUALIFIED: Withdrawal detected
    ACTIVE --> DISQUALIFIED: Window expires without goal
    
    GOAL_ACHIEVED --> CONFIRMING: 30-day window expires
    GOAL_ACHIEVED --> DISQUALIFIED: Withdrawal detected
    
    CONFIRMING --> PENDING_REWARD: Grace period complete, eligible
    CONFIRMING --> DISQUALIFIED: Late withdrawal or insufficient funds
    
    PENDING_REWARD --> COMPLETED: Points issued successfully
    PENDING_REWARD --> DISQUALIFIED: Points API failure after retries
    
    COMPLETED --> [*]: Final state
    DISQUALIFIED --> [*]: Final state
    
    note right of ENROLLED
        30-day timer started
        Awaiting first transaction
    end note
    
    note right of ACTIVE
        Real-time balance tracking
        Instant + Settled loads
    end note
    
    note right of GOAL_ACHIEVED
        Goal met, awaiting window expiry
        Withdrawals still disqualify
    end note
    
    note right of CONFIRMING
        48-hour grace period
        Final reconciliation in progress
    end note
    
    note right of PENDING_REWARD
        Queued for points issuance
        Idempotent processing
    end note
```

## 3. Data Reconciliation Flow

```mermaid
flowchart TB
    Start([Transaction Initiated]) --> CheckType{Transaction Type?}
    
    CheckType -->|Instant Load| InstantPath[Instant Load Path]
    CheckType -->|Delayed Load| DelayedPath[Delayed Load Path]
    
    InstantPath --> PaymentGateway[Payment Gateway Processes]
    PaymentGateway --> Webhook[Webhook to Load Tracking Service]
    Webhook --> ValidateInstant[Validate Transaction]
    ValidateInstant --> UpdateDynamo[Update DynamoDB<br/>instantLoadTotal += amount]
    UpdateDynamo --> PublishEvent[Publish Load Event to EventBridge]
    PublishEvent --> UpdateUI[Update Mobile App UI<br/>Sub-100ms latency]
    
    DelayedPath --> BankProcess[Bank Processing<br/>1-3 business days]
    BankProcess --> DataWarehouseETL[Data Warehouse ETL]
    DataWarehouseETL --> S3Upload[Upload to S3<br/>Daily at 2 AM]
    
    S3Upload --> TriggerStepFunctions[Trigger Reconciliation Workflow]
    TriggerStepFunctions --> ParseFile[Parse Transaction File]
    ParseFile --> BatchProcess[Process in Batches of 100]
    
    BatchProcess --> CheckExisting{Transaction<br/>Already in DynamoDB?}
    CheckExisting -->|Yes| MarkSettled[Mark as SETTLED<br/>No amount change]
    CheckExisting -->|No| AddSettled[Update settledLoadTotal<br/>+= amount]
    
    MarkSettled --> UpdateCumulative[Recalculate Cumulative Total]
    AddSettled --> UpdateCumulative
    
    UpdateCumulative --> CheckGoal{Cumulative >= $1000?}
    CheckGoal -->|Yes| TriggerConfetti[Emit Goal Achievement Event]
    CheckGoal -->|No| Continue[Continue Processing]
    
    TriggerConfetti --> NotifyMember[Send Confetti Notification]
    Continue --> NextBatch{More Transactions?}
    NextBatch -->|Yes| BatchProcess
    NextBatch -->|No| GenerateReport[Generate Reconciliation Report]
    
    NotifyMember --> End([End])
    GenerateReport --> End
    UpdateUI --> End
```

## 4. Reward Issuance Sequence

```mermaid
sequenceDiagram
    participant Scheduler as EventBridge Scheduler
    participant Lambda as Eligibility Check Lambda
    participant DynamoDB as DynamoDB
    participant Aurora as Aurora PostgreSQL
    participant SQS as SQS FIFO Queue
    participant RewardLambda as Reward Lambda
    participant LoyaltyAPI as Loyalty Points API
    participant SNS as SNS Notification
    
    Note over Scheduler: 30 days + 48 hours<br/>after enrollment
    
    Scheduler->>Lambda: Trigger eligibility check<br/>{memberId, windowId}
    
    Lambda->>DynamoDB: Get member load tracking
    DynamoDB-->>Lambda: {instantTotal, settledTotal, state}
    
    Lambda->>Aurora: Get enrollment record
    Aurora-->>Lambda: {enrollmentDate, cardType}
    
    Lambda->>DynamoDB: Check withdrawal history
    DynamoDB-->>Lambda: {hasWithdrawal: false}
    
    Note over Lambda: Calculate final total<br/>= instant + settled<br/>= $1,050
    
    alt Total >= $1000 AND No Withdrawal
        Lambda->>Aurora: Update state to PENDING_REWARD
        Aurora-->>Lambda: Success
        
        Lambda->>SQS: Enqueue reward request<br/>MessageDeduplicationId: windowId
        SQS-->>Lambda: MessageId
        
        Lambda->>Scheduler: Delete one-time schedule
        
        Note over SQS,RewardLambda: FIFO queue ensures<br/>exactly-once delivery
        
        SQS->>RewardLambda: Poll message
        
        RewardLambda->>DynamoDB: Conditional write reward record<br/>ConditionExpression: PK not exists
        
        alt Reward already issued
            DynamoDB-->>RewardLambda: ConditionalCheckFailed
            Note over RewardLambda: Idempotent exit<br/>Already processed
        else New reward
            DynamoDB-->>RewardLambda: Success - record created
            
            RewardLambda->>LoyaltyAPI: Issue 2000 points<br/>IdempotencyKey: windowId
            
            alt API Success
                LoyaltyAPI-->>RewardLambda: {transactionId, success}
                RewardLambda->>DynamoDB: Update reward status: ISSUED
                RewardLambda->>Aurora: Update member state: COMPLETED
                RewardLambda->>SNS: Publish success event
                SNS-->>RewardLambda: MessageId
                
            else API Failure
                LoyaltyAPI-->>RewardLambda: Error
                Note over RewardLambda: Retry with<br/>exponential backoff<br/>Max 5 attempts
                
                alt Max retries exceeded
                    RewardLambda->>DynamoDB: Update status: FAILED
                    RewardLambda->>SQS: Move to DLQ
                    Note over RewardLambda: Manual intervention required
                end
            end
        end
        
    else Disqualified
        Lambda->>Aurora: Update state to DISQUALIFIED<br/>{reason, finalTotal}
        Lambda->>SNS: Publish disqualification event
        Lambda->>Scheduler: Delete schedule
    end
```

## 5. Real-Time Load Processing

```mermaid
flowchart LR
    subgraph "Payment Gateway"
        Card[Debit Card /<br/>Apple Pay /<br/>Google Pay]
    end
    
    subgraph "Load Tracking Service"
        Webhook[Webhook Handler]
        Validator[Transaction Validator]
        Processor[Load Processor]
    end
    
    subgraph "DynamoDB"
        LoadTable[(Member Load Tracking)]
        TxnTable[(Transaction History)]
    end
    
    subgraph "Event Processing"
        EventBridge[EventBridge]
        Withdrawal[Withdrawal Tracker]
        Notification[Notification Service]
    end
    
    subgraph "Mobile App"
        UI[Real-time Balance UI]
    end
    
    Card -->|Instant Transaction| Webhook
    Webhook -->|Validate Signature| Validator
    Validator -->|Parse Event| Processor
    
    Processor -->|Optimistic Lock Update| LoadTable
    Processor -->|Insert Transaction| TxnTable
    
    LoadTable -.->|Read Committed| Processor
    Processor -->|Publish Load Event| EventBridge
    
    EventBridge -->|Route by Type| Withdrawal
    EventBridge -->|Route by Type| Notification
    
    Notification -->|Push/SMS| UI
    LoadTable -.->|Query Balance| UI
    
    TxnTable -->|DynamoDB Streams| Withdrawal
    Withdrawal -->|Check Withdrawal| EventBridge
```

## 6. Withdrawal Detection Flow

```mermaid
flowchart TB
    Start([Transaction Recorded]) --> StreamEvent[DynamoDB Stream Event]
    
    StreamEvent --> CheckType{Is Withdrawal<br/>Transaction?}
    CheckType -->|No| End([End - No Action])
    
    CheckType -->|Yes| GetMember[Retrieve Member Enrollment Data]
    GetMember --> CheckWindow{Within 30-Day<br/>Window?}
    
    CheckWindow -->|No| LogOnly[Log for Audit<br/>No Disqualification]
    LogOnly --> End
    
    CheckWindow -->|Yes| UpdateState[Update Member State<br/>to DISQUALIFIED]
    
    UpdateState --> RecordAudit[Record in Withdrawal Audit Table]
    RecordAudit --> CancelSchedule[Cancel 30-Day Scheduler]
    CancelSchedule --> PublishEvent[Publish Disqualification Event]
    
    PublishEvent --> SendNotification[Send Disqualification Notice]
    SendNotification --> End
    
    style UpdateState fill:#ff9999
    style CancelSchedule fill:#ffcc99
    style SendNotification fill:#99ccff
```

## 7. Reconciliation Engine Workflow

```mermaid
flowchart TB
    subgraph "Overnight Data Feed"
        DW[Data Warehouse] -->|Daily at 2 AM| S3File[S3 Transaction File]
    end
    
    subgraph "Step Functions Workflow"
        S3File -->|S3 Event Trigger| StartWorkflow[Start Reconciliation]
        
        StartWorkflow --> DownloadFile[Download File from S3]
        DownloadFile --> ValidateSchema[Validate File Schema]
        
        ValidateSchema --> ParseTransactions[Parse Transactions]
        ParseTransactions --> SplitBatches[Split into Batches<br/>100 records each]
        
        SplitBatches --> ProcessBatch[Process Batch]
        
        ProcessBatch --> ForEachTxn[For Each Transaction]
        
        ForEachTxn --> CheckDuplicate{Already in<br/>DynamoDB?}
        
        CheckDuplicate -->|Yes - Instant Load| MarkSettled[Mark Transaction as SETTLED<br/>settlementDate = today]
        CheckDuplicate -->|No - Delayed Load| AddToSettled[Add to settledLoadTotal<br/>Create transaction record]
        
        MarkSettled --> UpdateCumulative[Recalculate<br/>cumulativeTotal]
        AddToSettled --> UpdateCumulative
        
        UpdateCumulative --> CheckMilestone{Crossed $1000<br/>Threshold?}
        
        CheckMilestone -->|Yes| EmitGoalEvent[Emit Goal Achievement Event]
        CheckMilestone -->|No| NextTxn[Next Transaction]
        
        EmitGoalEvent --> NextTxn
        NextTxn --> MoreInBatch{More in Batch?}
        
        MoreInBatch -->|Yes| ForEachTxn
        MoreInBatch -->|No| BatchComplete[Batch Complete]
        
        BatchComplete --> MoreBatches{More Batches?}
        MoreBatches -->|Yes| ProcessBatch
        MoreBatches -->|No| GenerateReport[Generate Reconciliation Report]
        
        GenerateReport --> UploadReport[Upload Report to S3]
        UploadReport --> PublishComplete[Publish Completion Event]
        PublishComplete --> End([Workflow Complete])
    end
    
    subgraph "Error Handling"
        ValidateSchema -.->|Schema Error| ErrorHandler[Error Handler]
        ProcessBatch -.->|Processing Error| ErrorHandler
        ErrorHandler --> Retry[Retry with Backoff<br/>Max 3 attempts]
        Retry -.->|All Failed| DLQ[Send to Dead Letter Queue]
        DLQ --> Alert[Alert Operations Team]
    end
```

## 8. Multi-Region Disaster Recovery

```mermaid
graph TB
    subgraph "Primary Region - Sydney"
        PrimaryAPI[API Gateway]
        PrimaryLambda[Lambda Functions]
        PrimaryAurora[(Aurora Primary)]
        PrimaryDynamoDB[(DynamoDB)]
        PrimaryS3[(S3 Bucket)]
    end
    
    subgraph "DR Region - Singapore"
        DRRoute53[Route 53 Health Check]
        DRAPI[API Gateway Standby]
        DRLambda[Lambda Functions Standby]
        DRAurora[(Aurora Read Replica)]
        DRDynamoDB[(DynamoDB Global Table)]
        DRS3[(S3 Cross-Region Replication)]
    end
    
    subgraph "Shared Services"
        Route53[Route 53 DNS]
        CloudFront[CloudFront CDN]
    end
    
    Route53 --> CloudFront
    CloudFront -->|Primary| PrimaryAPI
    CloudFront -.->|Failover| DRAPI
    
    PrimaryAPI --> PrimaryLambda
    PrimaryLambda --> PrimaryAurora
    PrimaryLambda --> PrimaryDynamoDB
    PrimaryLambda --> PrimaryS3
    
    PrimaryAurora -->|Async Replication| DRAurora
    PrimaryDynamoDB <-->|Global Tables| DRDynamoDB
    PrimaryS3 -->|CRR| DRS3
    
    DRRoute53 -.->|Failure Detected| DRAPI
    DRAPI -.-> DRLambda
    DRLambda -.-> DRAurora
    DRLambda -.-> DRDynamoDB
    
    style PrimaryAPI fill:#99ff99
    style PrimaryAurora fill:#99ff99
    style PrimaryDynamoDB fill:#99ff99
    style DRAPI fill:#ffcc99
    style DRAurora fill:#ffcc99
    style DRDynamoDB fill:#ffcc99
```

---

## Diagram Legend

- **Blue boxes**: Client-facing components
- **Green boxes**: Active/Primary components
- **Orange boxes**: Standby/Secondary components
- **Red cylinders**: Data storage
- **Purple boxes**: Event processing
- **Solid arrows**: Active data flow
- **Dashed arrows**: Backup/Failover paths