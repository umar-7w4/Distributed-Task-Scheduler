# Distributed Task Scheduler

### Overview

The **Distributed Task Scheduler** is a highly scalable and fault-tolerant system that efficiently assigns and executes tasks across multiple worker instances. It leverages **Zookeeper** as a centralized coordination system to ensure task assignment, execution, and monitoring of worker health in a distributed setup.

---

### Table of Contents
- [Introduction](#introduction)
- [Features](#features)
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

The **Distributed Task Scheduler** aims to provide a robust task execution framework that dynamically allocates tasks to worker nodes in a distributed system. The system guarantees task execution, manages worker failures, and ensures that tasks are completed even if workers fail during execution.

Key goals of the system:
1. **Scalability**: Add or remove workers dynamically.
2. **Fault-Tolerance**: Reassign tasks in case of worker failures.
3. **Leader Election**: Use Zookeeper for leader election to manage job assignments in a coordinated fashion.

---

## Features

- **Leader-based Task Assignment**: Only one worker (the leader) assigns tasks to others, avoiding conflicts and race conditions.
- **At Least Once Guarantee**: Every task is guaranteed to be executed at least once, even in the case of worker failures.
- **Customizable Worker Selection**: Tasks can be assigned using various strategies such as round-robin or random.
- **Failure Detection and Recovery**: Automatically detects failed workers and reassigns tasks to active workers.
- **Scalable and Distributed**: Horizontal scaling by adding more worker instances.

---

## Architecture

The architecture of the **Distributed Task Scheduler** involves several key components that communicate via Zookeeper to ensure task scheduling and fault tolerance.

### Components Overview:

1. **Client**: 
   - Submits tasks to the system via REST APIs.
   - Receives task completion notifications.

2. **Zookeeper**:
   - Manages leader election among worker instances.
   - Acts as a central registry for task and worker data.
   - Notifies workers and clients of task completions and failures.

3. **Worker**:
   - Executes the tasks assigned by the leader.
   - Monitors Zookeeper for task assignments.
   - Updates the status of tasks in Zookeeper upon completion.

4. **Leader**:
   - Elected among worker nodes using Zookeeper’s leader election mechanism.
   - The leader monitors the job path and assigns tasks to available workers.

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

## Core Components

1. **Client**: Submits tasks and listens for task completion via REST APIs.
2. **Leader**: The elected leader node is responsible for assigning tasks to workers.
3. **Worker**: Executes tasks and updates Zookeeper with the task status.
4. **JobAssigner**: Assigns tasks to workers based on predefined strategies.
5. **WorkerPickerStrategy**: Defines the logic for selecting the worker that should execute the next task.
6. **Zookeeper**: Acts as the central coordination system for managing job submissions, task assignments, and status updates.

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

### Prerequisites
- **Java 17**: Required to run the project.
- **Zookeeper**: Zookeeper instance running locally or remotely (default port `218

1`).
- **Maven**: For building and managing dependencies.

### Steps to Deploy:
1. Clone the repository:
   ```bash
   git clone https://github.com/umar/TaskScheduler.git
   ```
2. Start Zookeeper on your system (default port `2181`).
3. Build the project using Maven:
   ```bash
   mvn clean install
   ```
4. Run the application:
   ```bash
   java -jar target/TaskScheduler-1.0.jar server config.yml
   ```

---

## Technologies Used

- **Java 17**: The primary programming language for the project.
- **Dropwizard**: A lightweight Java framework used for building RESTful web services.
- **Zookeeper**: Manages leader election, task assignment, and worker coordination.
- **CuratorFramework**: A high-level library for working with Zookeeper.
- **Lombok**: Reduces boilerplate code in Java (e.g., getter/setter generation).
- **JUnit 5**: Used for testing the application.

---

## Future Enhancements

- **Task Prioritization**: Implement the ability to prioritize tasks based on client-defined rules.
- **Docker Task Execution**: Enable the system to execute tasks inside Docker containers, allowing for more complex and isolated workloads.
- **Metrics and Monitoring**: Add Prometheus and Grafana for real-time monitoring of the system’s performance and task statuses.
- **Better Fault Recovery**: Improve fault detection mechanisms and speed up worker recovery after failure.
- **Job Dependencies**: Allow clients to submit jobs that depend on the completion of other jobs.
