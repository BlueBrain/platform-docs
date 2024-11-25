
# Accounting service draft

* [Requirements](#requirements)
  * [Functional requirements](#functional-requirements)
  * [Non-functional requirements](#non-functional-requirements)
* [High level architecture / data flow](#high-level-architecture--data-flow)
* [General idea](#general-idea)
* [Compute services](#compute-services)
* [Storage report service](#storage-report-service)
* [Accounting service](#accounting-service)
  * [Ledger Accessor](#ledger-accessor)
  * [Event processor](#event-processor)
  * [Job charger](#job-charger)
  * [Cost component](#cost-component)
  * [AWS cost tracker](#aws-cost-tracker)
  * [Watchdog](#watchdog)
* [Business logic](#business-logic)
  * [Longrun jobs](#longrun-jobs)
  * [Oneshot jobs](#oneshot-jobs)
  * [Storage](#storage)
* [SQS event format](#sqs-event-format)
* [DB schemas](#db-schemas)

## Requirements

### Functional requirements

* Users should be able to create and top-up their (funding) accounts.
* Subscription management for users.
* Funds management across v-lab/projects.
* Resource usage collection, user cost calculation and billing.
* Job termination when funds are exhausted.
* Users should have access to cost breakdown at both virtual lab and project level.
* Users should be able to see account balances, including reserved costs for currently running jobs.
* Admins should be able to control service prices/margins.
* Admins should be able to trace all transactions, see detailed and aggregated costs.

### Non-functional requirements

* High availability with minimal downtime for critical services.
* Double-spending/overspending prevention.
* Frequent user balance updates.

## High level architecture / data flow

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%

graph
  subgraph accountingSvc[Accounting service]
    sqs["AWS SQS (FIFO Queue)"]
    db[(AWS RDS)]
    accountingAPI[Accounting API]
    usageReceiverAPI[Event receiver API]
    paymentReceiverAPI[Payment receiver API]
    ledger[Ledger]
    pricing[Pricing]

    subgraph tasks[Tasks]
      jobCharger[Job Charger]
      eventProcessor[Event processor]
      watchdog[Watchdog]
      awsCostTracker[AWS cost tracker]
    end
  end

  subgraph svc[Services]
    subgraph longrunSvc[Longrun services]
      singleCellSim[Single cell simulations]
      meModel[ME-model simulations]
      smallCircuitSim[Small circuit simulations]
    end

    subgraph oneshotSvc[Oneshot services]
      ml[ML]
    end

    subgraph storageSvc[Storage report service]
      nexus[Nexus storage]
    end
  end

  virtualLabAPI[Virtual Lab API]
  costExplorerAPI[AWS Cost Explorer API]

  svc -- usage events ---> usageReceiverAPI
  usageReceiverAPI -- usage events ---> sqs

  longrunSvc -- pre-run cost reservation --> accountingAPI
  jobCharger -- job termination --> longrunSvc

  oneshotSvc -- pre-run cost reservation --> accountingAPI

  paymentReceiverAPI -- top-up events --> sqs

  sqs -- events --> eventProcessor
  eventProcessor -- usage reports --> db

  pricing --> db

  virtualLabAPI <--> accountingAPI

  ledger ---> db
  jobCharger ---> db
  jobCharger --- ledger
  jobCharger --- pricing

  watchdog --> db
  watchdog --- ledger

  costExplorerAPI -- AWS costs --> awsCostTracker
  awsCostTracker <--> db

  %% styles
  style svc fill:transparent

  style sqs fill:#FF9900
  style costExplorerAPI fill:#FF9900
  style db fill:#FF9900


  style paymentReceiverAPI fill:#87CEEB
  style usageReceiverAPI fill:#87CEEB
  style accountingAPI fill:#87CEEB
  style virtualLabAPI fill:#87CEEB
```

## General idea

The accounting service is meant to collect usage statistics from compute/storage services and to translate it into the user cost, which means:
* Updating virtual-lab/project budgets.
* Providing the user with detailed information about the usage and the cost of the resources spent.

## Compute services

Will handle two types of jobs by their billing model:
* `longrun`, which are billed by running time and used resources (e.g. underlying EC2 instance type, number of nodes). Examples: model building, simulations, analysis.
* `oneshot`, which are billed per execution with a fixed cost. Example: ML API calls.

These services will be responsible for:
* Executing pre-run checks to estimate the cost and reserve user funds for each execution.
* Providing usage statistics in a form of events which mark the start and the end of usage sessions as well as hearbeat signals.
* Listening for job termination requests from the accounting service (longrun jobs only).

## Storage report service

Is responsible to periodically collect the usage statistics for shared S3 buckets per each virtual-lab/project and provide it to the accounting service.

## Accounting service

### Ledger accessor

It is one of the core components of the accounting service, it's role is to track, manage and report funds across various system and user (v-lab/project) accounts. Uses [double entry accounting](https://en.wikipedia.org/wiki/Double-entry_bookkeeping) model.

The minimum list of accounts that are required for the service:
  * User related accounts:
    * Main account per each virtual lab. This is a target for top-ups.
    * Main account per each project.
    * Reservation account per each project.
  * System accounts:
    * Main platform account

Later on more accounts can be added, for example to track real AWS costs or other expenses.

SQL transactions **must** be used where applicable for concurrency control as well as to ensure data consistency and integrity.

### Event processor

It's responsible to collect from the AWS Simple Queue Service (SQS) and initiate processing for:
* Usage events from:
  * The compute service.
  * The storage report service.
* Top-up events from the payment provider.

### Job charger

A separate process that periodically charges users for their used storage and running jobs.

### Cost component

The component handles the following tasks:
1) Calculates user costs based on the usage reports by applying cost coefficient and fixed costs.
2) Handles cost estimation and reservation before any task is actually started (needed for double spent prevention).

### AWS cost tracker

Will provide insights about user billed costs and underlying platform AWS costs by mapping usage reports to data from AWS Cost Explorer API.
Can be used by platform administrators for analysis and decision making on service costs.

### Watchdog

It's responsible for:

- detecting abnormal termination of longrun jobs, i.e. when some time has passed without receiving any running or finish event.
  In this case, the job should be marked as terminated in the db, and the job charger should charge the user for the partial cost.
- detecting reserved job never started, i.e. when some time has passed without receiving any start event.
  In this case, the job should be marked as cancelled, and the job charger should release the reservation.

## Business logic

The process diagrams in this section are color-coded according the the job/event status as follows:

```mermaid
graph
  started[Started]
  running[Running]
  finished[Finished]

  style started fill:#FFFACD
  style running fill:#90EE90
  style finished fill:#ADD8E6
```

### Longrun jobs

#### Sequence diagram

```mermaid
sequenceDiagram
  participant svc as Compute service

  box Accounting
    participant api as API
    participant cost as Cost
    participant ledger as Ledger
    participant queue as Queue
    participant jobCharger as Job Charger
    participant watchdog as Watchdog
  end

  svc ->> api: Request job reservation
  api ->> cost: Get estimated cost
  cost ->> api: Estimated cost

  api ->> ledger: Reserve estimated cost
  ledger ->> api: Reservation result

  break if not enough funds
    api ->> svc: Rejected
  end

  api ->> svc: Allowed

  Note over svc: Job is starting

  break if no events received
    watchdog ->> jobCharger: Cancel job
    jobCharger ->> ledger: Release reservation
    jobCharger ->> svc: Terminate
  end

  rect rgb(233, 247, 248)
    svc --) queue: Started evt

    loop
      par
        svc --) queue: Running evt
        
      and
        jobCharger ->> cost: Compute cost for last interval
        cost ->> jobCharger: Cost
        jobCharger ->> ledger: Charge reservation + project

        opt if no funds left
          jobCharger ->> svc: Terminate
        end
      end
    end

    break if no events received
      watchdog ->> jobCharger: Terminate job
      jobCharger ->> ledger: Return unspent reservation
      jobCharger ->> svc: Terminate
    end

    Note over svc: Job finishes

    svc --) queue: Finished evt
  end

  jobCharger ->> ledger: Return unspent reservation
```

#### Process diagrams

##### Job execution request HTTP API calls

```mermaid
graph LR
  start([Start])
  End([End])

  receive[/HTTP API:<br/>Receive job reservation request/]
  estimateCost[Cost service:<br/>Estimate job cost]
  checkAvailableFunds{Ledger:<br/>enough funds}
  createUsageReport[DB:<br/>Create reserved job]
  reserve[Ledger:<br/>reserve estimated cost]

  reject[Reject job exec]
  allow[Allow job exec]

  start --> receive
  receive --> estimateCost
  estimateCost --> checkAvailableFunds

  checkAvailableFunds --> |No| reject

  checkAvailableFunds --> |Yes| createUsageReport
  createUsageReport --> reserve
  reserve --> allow
  allow --> End

  reject --> End

  %% styles
  style estimateCost fill:#FFFACD
  style checkAvailableFunds fill:#FFFACD
  style createUsageReport fill:#FFFACD
  style reserve fill:#FFFACD
  style allow fill:#FFFACD
  style reject fill:#FFFACD
```

##### SQS events

```mermaid
flowchart
  start([Start event processing loop])
  getEvent[Queue:<br/>Get event]

  checkStatus{Event status}

  initUsageReport[DB:<br/>Update reserved job with<br/>started_at and<br/>last_alive_at]
  updateFinishedAt[DB:<br/>Update finished_at<br/>and last_alive_at]
  updateLastAliveAt[DB:<br/>Update last_alive_at]
  deleteEvent[Delete event<br/>from the queue]

  start --> getEvent
  getEvent --> checkStatus

  checkStatus --> |"Started"| initUsageReport
  initUsageReport --> deleteEvent

  checkStatus --> |"Running"| updateLastAliveAt
  updateLastAliveAt --> deleteEvent

  checkStatus --> |Finished| updateFinishedAt
  updateFinishedAt --> deleteEvent

  deleteEvent --> getEvent

  %% styles

  %% Started
  style initUsageReport fill:#FFFACD

  %% Running
  style updateLastAliveAt fill:#90EE90

  %% Finished
  style updateFinishedAt fill:#ADD8E6
```

##### Periodic charging

```mermaid
graph
  start([Start periodic charge processing])
  getJobToBeCharged[DB:<br/>Get longrun job to be charged]
  sleep[Sleep]

  compute_unfinished_uncharged[Cost service:<br/>compute fixed cost and<br/>cost for first running time]
  compute_unfinished_charged[Cost service:<br/>compute cost for running time<br/>since the last charge]
  compute_finished_uncharged[Cost service:<br/>compute fixed costs and cost<br/>for the full running time]
  compute_finished_charged[Cost service:<br/>compute cost for<br/>the final running time]
  compute_finished_overcharged[Cost service:<br/>compute cost for extra time<br/>previously overcharged]

  balance_unfinished{Available funds on<br/>reservation + proj?}
  charge_unfinished[Ledger:<br/>Charge<br/>reservation + proj]
  drain_unfinished[Ledger:<br/>Drain<br/>reservation + proj]
  terminateJob[Compute svc:<br/>Terminate job]

  balance_finished{Available funds on<br/>reservation + proj?}
  charge_finished[Ledger:<br/>Charge<br/>reservation + proj]
  drain_finished[Ledger:<br/>Drain<br/>reservation + proj]

  releaseUnspentReservation[Ledger:<br/>Release unspent<br/>reservation]

  refund_overcharged["Ledger:<br/>Refund extra time<br/>previously overcharged"]

  start --> getJobToBeCharged
  getJobToBeCharged --> |job not finished<br/>not charged| compute_unfinished_uncharged
  getJobToBeCharged --> |job not finished<br/>partially charged| compute_unfinished_charged
  getJobToBeCharged --> |job finished<br/>not charged| compute_finished_uncharged
  getJobToBeCharged --> |job finished<br/>partially charged| compute_finished_charged
  getJobToBeCharged --> |job finished<br/>overcharged<br/>*edge case*| compute_finished_overcharged

  compute_unfinished_uncharged --> balance_unfinished
  compute_unfinished_charged --> balance_unfinished
  compute_finished_uncharged --> balance_finished
  compute_finished_charged --> balance_finished
  compute_finished_overcharged --> refund_overcharged --> sleep

  balance_finished --> |< txn| drain_finished --> sleep
  balance_finished --> |>= txn| charge_finished --> releaseUnspentReservation --> sleep

  balance_unfinished --> |< txn| drain_unfinished --> terminateJob --> sleep
  balance_unfinished --> |>= txn| charge_unfinished --> sleep

  sleep --> getJobToBeCharged

  %% styles
  style compute_unfinished_uncharged fill:#90EE90
  style compute_unfinished_charged fill:#90EE90
  style balance_unfinished fill:#90EE90
  style drain_unfinished fill:#90EE90
  style charge_unfinished fill:#90EE90
  style terminateJob fill:#90EE90

  style compute_finished_uncharged fill:#ADD8E6
  style compute_finished_charged fill:#ADD8E6
  style balance_finished fill:#ADD8E6
  style drain_finished fill:#ADD8E6
  style charge_finished fill:#ADD8E6
  style releaseUnspentReservation fill:#ADD8E6

  style compute_finished_overcharged fill:#ADD8E6
  style refund_overcharged fill:#ADD8E6
```

### Oneshot jobs

#### Sequence diagram

```mermaid
sequenceDiagram
  participant svc as Compute service

  box Accounting
    participant api as API
    participant cost as Cost
    participant ledger as Ledger
    participant queue as Queue
    participant jobCharger as Job Charger
    participant watchdog as Watchdog
  end

  svc ->> api: Request job reservation
  api ->> cost: Get estimated cost
  cost ->> api: Estimated cost
  api ->> ledger: Reserve estimated cost
  ledger ->> api: Reservation result

  break if not enough funds
    api ->> svc: Rejected
  end

  api ->> svc: Allowed

  rect rgb(233, 247, 248)
    Note over svc: Job starts

    break if no events received
      watchdog ->> jobCharger: Cancel job
      jobCharger ->> ledger: Release reservation
    end

    Note over svc: Job finishes
  end

  svc --) queue: Usage evt
  jobCharger ->> ledger: Charge reservation
```

#### Process diagrams

##### Job execution request HTTP API calls

```mermaid
graph LR
  start([Start])
  End([End])

  receive[/HTTP API:<br/>Receive job reservation request/]

  computeCost[Cost service:<br/>Compute job cost]
  checkAvailableFunds{Ledger:<br/>enough funds}
  createUsageReport[DB:<br/>Create reserved job]
  reserve[Ledger:<br/>reserve estimated cost]

  reject[Reject job exec]
  allow[Allow job exec]

  start --> receive
  receive --> computeCost
  computeCost --> checkAvailableFunds

  checkAvailableFunds --> |No| reject

  checkAvailableFunds --> |Yes| createUsageReport
  createUsageReport --> reserve
  reserve --> allow
  allow --> End

  reject --> End

  style computeCost fill:#FFFACD
  style checkAvailableFunds fill:#FFFACD
  style createUsageReport fill:#FFFACD
  style reserve fill:#FFFACD
  style reject fill:#FFFACD
  style allow fill:#FFFACD
```

##### SQS events

```mermaid
graph LR
  %% event processing

  start([Start event processing loop])

  getEvent[Queue:<br/>Get event]
  closeUsageReport[DB:<br/>Update started_at,<br/>last_alive_at, finished_at]

  deleteEvent[Queue:
    Delete event]

  start --> getEvent
  getEvent --> closeUsageReport
  closeUsageReport --> deleteEvent
  deleteEvent --> getEvent

  %% styles
  %% Running
  style closeUsageReport fill:#ADD8E6
```

##### Periodic charging

The oneshot jobs are charged by the job charger in a separate task so that the event processor doesn't need to handle money transactions.

```mermaid
graph LR
  start([Start periodic charge processing])
  getJobToBeCharged[DB:<br/>Get job to be charged]
  sleep[Sleep]

  computeTxnAmount[Cost service:<br/>compute txn amount]

  charge[Ledger:<br/>Charge<br/>reservation]

  start --> getJobToBeCharged
  getJobToBeCharged --> computeTxnAmount
  sleep --> getJobToBeCharged

  computeTxnAmount --> charge
  charge --> sleep

  %% styles
  style computeTxnAmount fill:#ADD8E6
  style charge fill:#ADD8E6
```


### Storage

#### Sequence diagram

```mermaid
sequenceDiagram
  participant svc as Storage stats service

  box Accounting
    participant queue as Queue
    participant cost as Cost
    participant ledger as Ledger
    participant jobCharger as Job Charger
  end

  rect rgb(233, 247, 248)
    loop
      par
        svc --) queue: Usage evt
      and
        jobCharger ->> cost: Compute cost for last interval
        cost ->> jobCharger: Cost
        jobCharger ->> ledger: Charge main
      end
    end
  end
```

#### Process diagrams

#### SQS Events

```mermaid
graph LR
  %% event processing

  start([Start event processing loop])
  getEvent[Queue:<br/>Get event]
  createUsageReport[DB:<br/>Create new<br/>usage report]
  closePreviousUsageReport[DB:<br/>Close previous<br/>storage usage report]

  deleteEvent[Queue:<br/>Delete event]

  start --> getEvent
  getEvent --> closePreviousUsageReport
  closePreviousUsageReport --> createUsageReport
  createUsageReport --> deleteEvent
  deleteEvent --> getEvent

  %% styles
  %% Running
  style createUsageReport fill:#90EE90
  style closePreviousUsageReport fill:#90EE90
```

##### Periodic charging

```mermaid
graph

  start([Start periodic charge processing])

  getStorageReportsToBeCharged["DB:<br/>Get storage reports<br/>not charged/partially charged"]

  computeTxnAmount[Cost service:<br/>compute runnning txn amount]
  chargeMain[Ledger:<br/>Charge project]

  computeFinalTxnCost[Cost service:<br/>Compute closing txn cost]
  chargeDeltaFromMain[Ledger:<br/>Create closing txn]

  sleep[Sleep]

  start --> getStorageReportsToBeCharged
  getStorageReportsToBeCharged --> |not finished| computeTxnAmount --> chargeMain --> sleep
  getStorageReportsToBeCharged --> |finished| computeFinalTxnCost --> chargeDeltaFromMain --> sleep
  sleep --> getStorageReportsToBeCharged

  %% styles
  %% Running
  style computeTxnAmount fill:#90EE90
  style chargeMain fill:#90EE90

  %% Finished
  style chargeDeltaFromMain fill:#ADD8E6
  style computeFinalTxnCost fill:#ADD8E6
```

## SQS event format

```mermaid

erDiagram
  StorageUsageEvent {
    string type "'storage'"
    uuid vlab_id
    uuid proj_id
    bigint size
    timestamp timestamp
  }

  LongrunJobUsageEvent {
    string type "'longrun'"
    string subtype "'single-cell-sim'"
    enum status "'started' | 'running' | 'finished'"
    uuid vlab_id
    uuid proj_id
    uuid job_id
    int instances
    string instance_type
    timestamp timestamp
  }

  OneshotJobUsageEvent {
    string type "'oneshot'"
    string subtype "'ml-query'"
    uuid vlab_id
    uuid proj_id
    uuid job_id
    int count
    timestamp timestamp
  }

  TopUpEvent["TopUpEvent TBD"] {
    enum type "'top-up'"
    uuid vlab_id
    uuid proj_id
    string amount "To check format from Stripe"
    timestamp timestamp
  }
```

Notes:
- All the payloads must be json dicts with keys and values formatted as strings.
- The timestamps must be represented as unix time in milliseconds.
- The separator and case used for type and subtype should be always the same for consistency:
  - use lowercase names
  - use `-` instead of `_`
  - prefer single to plural names
- The list of valid subtypes needs to be decided yet, and it could evolve in the future.

## DB schemas

```mermaid
erDiagram

  ACCOUNT {
    uuid id PK
    enum account_type "'sys', 'vlab', 'proj', 'rsv'"
    uuid parent_id FK
    string name
    decimal balance
    bool enabled
  }

  JOURNAL {
    bigint id PK
    datetime transaction_datetime
    enum transaction_type "'top-up', 'assign', 'reserve', 'release', 'charge-oneshot', 'charge-longrun', 'charge-storage', 'refund'"
    uuid job_id FK
    int price_id FK
    dict properties "'reason', 'charge_period_start', 'charge_period_end'..."
  }
  
  LEDGER {
    bigint id PK
    uuid account_id FK
    bigint journal_id FK
    Decimal amount
  }
  
  JOB {
    uuid id PK
    uuid vlab_id FK
    uuid proj_id FK
    enum service_type "'storage', 'longrun', 'oneshot'"
    enum service_subtype
    int usage_value
    datetime reserved_at
    datetime started_at
    datetime last_alive_at
    datetime last_charged_at
    datetime finished_at
    datetime cancelled_at
    dict properties "['instances', 'instance_type'...]"
  }

  PRICE {
    int id PK
    enum service_type
    enum service_subtype
    datetime valid_from
    datetime valid_to
    decimal multiplier
    decimal fixed_cost
    uuid vlab_id FK
  }

  JOB ||--|{ JOURNAL : "Triggers"
  JOURNAL ||--|{ LEDGER : "Is detailed by"
  ACCOUNT ||--|{ LEDGER : "Has"
  ACCOUNT ||--o| ACCOUNT : "Has parent in"
  PRICE ||--|{ JOURNAL : "Is used by"
  ACCOUNT ||--|{ PRICE : "Is charged with"
  ACCOUNT ||--|{ JOB : "Started"
  ```
