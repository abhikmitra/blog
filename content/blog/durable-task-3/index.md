---
title: Durable Task Framework Internals - Part 3 (Tracker Queue and Azure Storage)
date: "2020-04-25T18:00:00.284Z"
description: "Deep-dive into inner workings of DTF - Part 3."
---
### Durable Task Framework Series
This post is part 2 of a series of posts on DTF.
1. [Durable Task Framework Internals - Part 1 (Dataflow and Reliability)](https://abhikmitra.github.io/blog/durable-task/)
2. [Durable Task Framework Internals - Part 2 (The curious case of Orchestrations)](https://abhikmitra.github.io/blog/durable-task-2/)
3. [Durable Task Framework Internals - Part 3 (Tracker Queue and Azure Storage)](https://abhikmitra.github.io/blog/durable-task-3/)

---
 
### The Tracker Queue - (WIP)

```mermaid

sequenceDiagram

Client->>Orchestrator: ExecutionStarted

Orchestrator-->>Hub: ExecutionStarted

Hub-->>TestOrchestration: Orchestration Invoked

Hub-->>Worker: Task Scheduled

Hub-->>Tracker: OrchestratorStarted

Hub-->>Tracker: ExecutionStarted

Hub-->>Tracker: TaskScheduled

Hub-->>Tracker: OrchestratorCompleted

Hub-->>Tracker: HistoryState

Worker-->>Hub: Task Scheduled

Hub-->>TaskActivity: Task Invoked

Hub->>Orchestrator: TaskCompleted

Tracker-->>Hub: OrchestratorStarted

Tracker-->>Hub: ExecutionStarted

Tracker-->>Hub: TaskScheduled

Tracker-->>Hub: OrchestratorCompleted

Tracker-->>Hub: HistoryState

Orchestrator-->>Hub: TaskCompleted

Hub-->>TestOrchestration: Orchestration Invoked & Completed

Hub-->>Tracker: OrchestratorStarted

Hub-->>Tracker: TaskCompleted

Hub-->>Tracker: ExecutionCompleted

Hub-->>Tracker: OrchestratorCompleted

Hub-->>Tracker: HistoryState

  

Tracker-->>Hub: OrchestratorStarted

Tracker-->>Hub: TaskCompleted

Tracker-->>Hub: ExecutionCompleted

Tracker-->>Hub: OrchestratorCompleted

Tracker-->>Hub: HistoryState

```



