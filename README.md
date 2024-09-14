# Distributed Task Scheduler
 
### Project Overview

The **Distributed Task Scheduler** is a fault-tolerant and scalable distributed system that dynamically assigns tasks to worker nodes. It uses **Zookeeper** for worker coordination, leader election, and task assignment. The system ensures task execution in a distributed environment, with built-in recovery for worker failures.



---

### Table of Contents
- [Introduction](#introduction)
- [Goals and Objectives](#goals-and-objectives)
- [Features](#features)
- [Component Tree](#component-tree)
- [Architecture](#architecture)
- [High-Level Design (HLD)](#high-level-design-hld)
- [Low-Level Design (LLD)](#low-level-design-lld)
- [Core Components](#core-components)
- [Zookeeper Integration](#zookeeper-integration)
- [Failure Handling](#failure-handling)
- [Deployment](#deployment)
- [Technologies Used](#technologies-used)
- [Future Enhancements](#future-enhancements)

---

## Introduction

The **Distributed Task Scheduler** is designed to efficiently distribute tasks across multiple worker instances. It ensures that tasks are reliably assigned, executed, and monitored in a fault-tolerant manner. Zookeeper acts as a coordination service to manage leader election, task assignment, and failure detection.

---

## Goals and Objectives

### Goals:
- **Scalability**: Handle an increasing number of tasks and workers by scaling horizontally.
- **Fault-Tolerance**: Ensure that no task is left unexecuted in the event of a worker failure.
- **At Least Once Guarantee**: Each task should be executed at least once, with recovery mechanisms in place for failed executions.
- **Efficient Coordination**: Use Zookeeper to efficiently manage task distribution and worker state.
- **Flexible Worker Selection**: Provide multiple strategies for worker selection, such as round-robin and random assignment.

### Objectives:
- Enable dynamic task submission and monitoring through a REST API.
- Ensure reliable worker registration and task assignment through Zookeeper.
- Implement automatic recovery in case of worker or leader failure.
- Provide extensible worker assignment strategies for load distribution.
- Ensure high performance by avoiding bottlenecks in task assignment and execution.

---

## Features

- **Leader-based Coordination**: The system uses a leader-follower model to assign tasks. Only one leader is responsible for assigning tasks to workers.
- **Fault Recovery**: Detects worker failures and reassigns incomplete tasks to available workers.
- **At Least Once Execution**: Ensures that every task is executed at least once, even in case of failure.
- **Zookeeper-backed Coordination**: The system leverages Zookeeper for leader election, task assignment, and worker failure detection.
- **Customizable Worker Assignment**: Allows task assignment strategies like random selection or round-robin.
- **REST API for Task Submission**: Clients can submit tasks and monitor their status via a RESTful API.

---

## Component Tree

Here is the detailed component tree of the project based on the system structure:

```plaintext
├── src/main/java
│   ├── com.umar.taskscheduler
│   │   ├── App.java
│   │   ├── AppConfiguration.java
│   │   └── JobDetail.java
│   ├── com.umar.taskscheduler.callbacks
│   │   ├── AssignmentListener.java
│   │   ├── JobAssigner.java
│   │   ├── JobsListener.java
│   │   └── WorkersListener.java
│   ├── com.umar.taskscheduler.core
│   │   └── ZKDao.java
│   ├── com.umar.taskscheduler.DistributedTaskScheduler
│   │   └── App.java
│   ├── com.umar.taskscheduler.module
│   │   └── GuiceModule.java
│   ├── com.umar.taskscheduler.resources
│   │   ├── Client.java
│   │   ├── Job.java
│   │   └── Worker.java
│   ├── com.umar.taskscheduler.service
│   │   ├── ClientService.java
│   │   └── WorkerService.java
│   ├── com.umar.taskscheduler.strategy
│   │   ├── RandomWorker.java
│   │   ├── RoundRobinWorker.java
│   │   └── WorkerPickerStrategy.java
│   └── com.umar.taskscheduler.util
│       └── ZKUtils.java
├── src/main/resources
│   └── log4j.properties
├── src/test/java
│   ├── com.umar.taskscheduler.DistributedTaskScheduler
│   │   └── AppTest.java
│   ├── com.umar.taskscheduler.strategy
│   │   └── RoundRobinWorkerTest.java
└── JRE System Library [JavaSE-17]
```

### Key Components:

- **com.umar.taskscheduler**: 
  - **App.java**: The entry point of the application.
  - **AppConfiguration.java**: Handles application configurations.
  
- **callbacks**: 
  - **AssignmentListener.java**: Handles task assignment callbacks.
  - **JobAssigner.java**: Manages the assignment of jobs to workers.
  - **JobsListener.java**: Listens for new job creation and assigns them.
  - **WorkersListener.java**: Monitors worker availability and handles worker failure events.
  
- **core**: 
  - **ZKDao.java**: Zookeeper Data Access Object for handling direct interactions with Zookeeper.
  
- **resources**: 
  - **Client.java**: Handles client interactions and task submission.
  - **Job.java**: REST resource for job management.
  - **Worker.java**: REST resource for worker management.
  
- **service**: 
  - **ClientService.java**: Provides business logic for handling client requests and submitting tasks.
  - **WorkerService.java**: Manages worker operations including task execution and failure recovery.
  
- **strategy**: 
  - **RandomWorker.java**: Implements random worker selection strategy.
  - **RoundRobinWorker.java**: Implements round-robin worker selection strategy.
  - **WorkerPickerStrategy.java**: Interface for worker selection strategies.
  
- **util**: 
  - **ZKUtils.java**: Utility class for managing Zookeeper paths and constants.

---

## Architecture

The architecture of the **Distributed Task Scheduler** is built to ensure efficient task scheduling and execution in a distributed system. Below is the architecture diagram that illustrates the interaction between components:

### Architecture Components:

1. **Client**: 
   - Submits tasks via a REST API and listens for task completion notifications.
   - Provides the input parameters required for task execution.
  
2. **Zookeeper**: 
   - Manages worker nodes, leader election, and task coordination.
   - Tasks are stored under `/jobs`, and task assignments are stored under `/assignments`.
  
3. **Leader**: 
   - Elected from the pool of worker nodes.
   - Responsible for assigning tasks to other workers and managing the task queue.
  
4. **Worker**: 
   - Executes assigned tasks and updates task status in Zookeeper.
   - Monitors its assignment path in Zookeeper for incoming tasks.
  
5. **Task Assignment & Execution**: 
   - The leader assigns tasks to workers based on a worker selection strategy.
   - Workers execute tasks asynchronously and update their status in Zookeeper.
  
### Workflow:

1. **Task Submission**: Clients submit tasks to Zookeeper under the `/jobs` path.
2. **Leader Election**: One of the workers is elected as the leader, which is responsible for job assignment.
3. **Task Assignment**: The leader assigns tasks to workers by creating entries under `/assignments/{worker-id}/{job-id}`.
4. **Task Execution**: Workers execute tasks and update the task status in Zookeeper under `/status/{job-id}`.
5. **Failure Recovery**: If a worker fails, the leader reassigns its tasks to other workers.

### Communication Flow:
- **Client → Zookeeper**: Task submission.
- **Leader → Workers**: Task assignment.
- **Workers → Zookeeper**: Task status updates.
- **Zookeeper → Client**: Task completion notification.

---


## High-Level Design (HLD)

The High-Level Design provides an abstract view of the system components, their interactions, and their responsibilities.

### 1. **Client**:
   - Submits tasks via REST APIs.
   - Listens for task completion notifications.
   - Provides input parameters for the tasks to be executed by the workers.

### 2. **Zookeeper Coordination**:
   - **Leader Election**: Zookeeper selects one worker to act as the leader that assigns jobs.
   - **Task Registry**: Clients register tasks in Zookeeper under `/jobs/{job-id}`. The task includes details like the job's data and its parameters.
   - **Task Assignment**: The leader assigns tasks to workers and creates nodes under `/assignments/{worker-id}/{job-id}`.
   - **Task Status Update**: After task execution, workers update the job's status under `/status/{job-id}`.
   - **Worker Monitoring**: Zookeeper monitors worker nodes, detects failures, and notifies the leader when a worker goes down.

### 3. **Leader Election**:
   - Leader election happens through **Zookeeper’s ephemeral nodes**, where only one worker node becomes the leader.
   - The leader is responsible for assigning tasks to the workers by reading from `/jobs/` and writing to `/assignments/`.

### 4. **Worker Behavior**:
   - Workers monitor their own assignment paths (`/assignments/{worker-id}`).
   - When a task is assigned, the worker fetches it, executes it, and updates the task’s status.

### Sequence of Events:
1. Client submits a task.
2. Zookeeper stores the task.
3. Leader assigns tasks to workers.
4. Workers execute tasks and update their status.
5. Clients are notified of task completion.

---

## Low-Level Design (LLD)

The Low-Level Design explains the detailed structure of each component and the logic flow within the system.

### 1. **ClientService**:
   - The `ClientService` registers tasks by serializing them into a Zookeeper node (`/jobs/{job-id}`).
   - Tasks are serialized using Java’s `ObjectOutputStream` and stored as a byte array in Zookeeper.

### 2. **JobsListener**:
   - The `JobsListener` monitors the `/jobs` path for new tasks.
   - When a task is found, it passes the task to the `JobAssigner` for assignment to workers.

### 3. **JobAssigner**:
   - The `JobAssigner` selects a worker from the available workers using the `WorkerPickerStrategy`.
   - Strategies include random worker selection (`RandomWorker`) or round-robin selection (`RoundRobinWorker`).
   - After assigning a job, it removes the task from `/jobs/{job-id}` and creates an assignment node under `/assignments/{worker-id}/{job-id}`.

### 4. **WorkerService**:
   - Each worker monitors the `/assignments/{worker-id}` path.
   - Upon detecting a task, the worker deserializes the task data and executes it in a thread pool.
   - The worker then updates the task status in Zookeeper under `/status/{job-id}`.

### 5. **AssignmentListener**:
   - Monitors the `/assignments` path for new tasks assigned to workers.
   - The worker fetches the task details from Zookeeper, executes it, and updates the status upon completion.

### 6. **WorkerPickerStrategy**:
   - Implements the logic for selecting workers. Two strategies are supported:
     1. **RandomWorker**: Randomly selects a worker.
     2. **RoundRobinWorker**: Selects workers in a round-robin fashion, ensuring even distribution.
        

---


## Zookeeper Integration

Zookeeper acts as the backbone of the **Distributed Task Scheduler**, handling coordination, synchronization, and leader election among worker nodes.


### Key Zookeeper Operations:
- **Leader Election**: Uses ephemeral nodes to elect a single leader that is responsible for assigning tasks.
- **Task Submission**: Clients submit tasks to the `/jobs` path.
- **Task Assignment**: The leader assigns tasks by creating nodes under `/assignments/{worker-id}`.
- **Task Completion**: Workers update the task status under `/status/{job-id}`.
- **Watcher Mechanism**: Zookeeper’s watchers are used to notify workers of new tasks and notify the leader of worker failures.


### CuratorFramework:
- The project uses **CuratorFramework** to interact with Zookeeper, which simplifies many Zookeeper operations like leader election, node creation, and watcher management.

---

## Failure Handling

1. **Worker Failure**:
   - When a worker goes down, Zookeeper detects the failure via its ephemeral nodes.
   - The leader reassigns the failed worker’s tasks to another available worker.

2. **Leader Failure**:
   - When the leader fails, Zookeeper triggers a new leader election, ensuring that task assignment continues.
   - The new leader takes over and resumes monitoring the `/jobs` and `/workers` paths.

3. **Task Reassignment**:
   - In the event of worker failure, the tasks assigned to that worker are reassigned to other workers by the leader, ensuring that no task remains unexecuted.

---

## Deployment

### Prerequisites:
- **Java 17**
- **Maven**
- **Zookeeper** running on port `2181`

### Steps to Deploy:
1. Clone the repository:
   ```bash
   git clone https://github.com/umar/TaskScheduler.git
   ```
2. Start Zookeeper:
   ```bash
   zookeeper-server-start.sh config/zookeeper.properties
   ```
3. Build the project:
   ```bash
   mvn clean install
   ```
4. Run the application:
   ```bash
   java -jar target/TaskScheduler-1.0.jar server config.yml
   ```

---

## Maven Dependency

Use the following maven dependency:

```xml
<dependency>
    <groupId>com.snehasishroy</groupId>
    <artifactId>TaskScheduler</artifactId>
    <version>1.0</version>
</dependency>
```

## Starting Zookeeper Server

This service requires **Zookeeper Server version 3.5.4 or higher** because it uses TTL Nodes. Additionally, the `extendedTypesEnabled` flag must be set when starting the Zookeeper Server.

### Configuring Zookeeper

Edit the `zkEnv.sh` file to set the necessary environment variables:

```bash
vim {ZK}/bin/zkEnv.sh
```

Add the following line:

```bash
export SERVER_JVMFLAGS="-Xmx${ZK_SERVER_HEAP}m $SERVER_JVMFLAGS -Dzookeeper.extendedTypesEnabled=true"
```

### Sample zoo.cfg

Here is a sample `zoo.cfg` configuration file:

```bash
tickTime = 200
dataDir = /data/zookeeper
clientPort = 2181
initLimit = 5
syncLimit = 2
```

### Starting the Zookeeper Server

Run the following command to start the Zookeeper server in the foreground:

```bash
sudo ./zkServer.sh start-foreground
```


## Technologies Used

- **Java 17**
- **Dropwizard** for building RESTful APIs.
- **Zookeeper** for distributed coordination.
- **CuratorFramework** for Zookeeper interaction.
- **JUnit** for testing.
- **Lombok** to simplify Java code.

---

## Future Enhancements

- **Task Prioritization**: Implement the ability to prioritize tasks based on client-defined rules.
- **Docker Task Execution**: Enable the system to execute tasks inside Docker containers, allowing for more complex and isolated workloads.
- **Metrics and Monitoring**: Add Prometheus and Grafana for real-time monitoring of the system’s performance and task statuses.
- **Better Fault Recovery**: Improve fault detection mechanisms and speed up worker recovery after failure.
- **Job Dependencies**: Allow clients to submit jobs that depend on the completion of other jobs.

### License

```
This project is licensed under the @2024 Umar Mohammad
```

