#### 1. Logical View (C4 Component Diagram)

This diagram adds the `Event Bus` as a central messaging component and the new `Notification Service` as a consumer.

```mermaid
C4Component
    title Component Diagram for InkWell System (Event Bus - Refined Layout)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "Container: Go", "Manages user data.")
            Component(postService, "Post Service", "Container: Ruby on Rails", "Manages posts.")
            Component(workerService, "Worker Service", "Container: Python", "Executes background jobs.")
            Component(notificationService, "Notification Service", "Container: Go", "Consumes events.")
        }
        
        System_Boundary(c3, "Infrastructure Layer") {
             Component(eventBus, "Event Bus", "Amazon SNS/SQS", "Decouples services via events.")
             ComponentQueue(cacheQueue, "Cache & Job Queue", "Amazon ElastiCache for Redis", "Provides cache and job storage.")
             ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Source of truth for core data.")
        }
    }

    %% Relationships from User and Gateway
    Rel(writer, gateway, "Uses", "HTTPS/JSON")
    Rel_Down(gateway, postService, "Routes to")
    Rel_Down(gateway, userService, "Routes to")

    %% Inter-service communication
    Rel(postService, userService, "Gets user data from", "HTTPS/JSON")

    %% Relationships to Infrastructure
    Rel_Down(postService, eventBus, "Publishes 'PostPublished' event")
    Rel_Down(notificationService, eventBus, "Consumes event from")
    
    Rel_Down(postService, cacheQueue, "Uses")
    Rel_Down(workerService, cacheQueue, "Uses")
    
    Rel_Down(userService, db, "Reads/Writes")
    Rel_Down(postService, db, "Reads/Writes")
    Rel_Down(workerService, db, "Reads/Writes")
```

#### 2. Physical View (AWS Deployment Diagram)

The physical diagram adds the new `Amazon SNS` and `SQS` services and the new `notification_container`.

```mermaid
graph LR
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        Gateway["Amazon API Gateway"]
        
        subgraph "VPC"
            subgraph "EC2 Instance (Private Subnet)"
                subgraph "Docker"
                    post_container["Post Service"]
                    user_container["User Service"]
                    worker_container["Worker Service"]
                    notification_container["Notification Service"]
                end
            end

            subgraph "Messaging Services"
                sns["SNS Topic<br>(PostEvents)"]
                sqs["SQS Queue<br>(Notifications)"]
            end

            subgraph "Stateful Services"
                redis["Redis Cluster"]
                db_primary["RDS Primary"]
                db_replica["RDS Replica"]
            end
        end
    end

    %% Define User-Facing Flows
    User -- "HTTPS" --> Gateway
    Gateway -- "/posts/*" --> post_container
    Gateway -- "/users/*" --> user_container
    
    %% Define Event-Driven Flow
    post_container -- "Publish Event" --> sns
    sns -- "Push to Queue" --> sqs
    notification_container -- "Polls Queue" --> sqs
    
    %% Define Other Internal Flows
    post_container -- "Cache Check" --> redis
    post_container -- "DB Query" --> db_replica
    post_container -- "API Call" --> user_container
    post_container -- "Enqueue Job" --> redis
    worker_container -- "Dequeue Job" --> redis
    worker_container -- "SQL Update" --> db_primary
    user_container -- "SQL Writes" --> db_primary
    db_primary -- "Replication" --> db_replica
```

#### 3. Component-to-Resource Mapping Table

We add the new components for the event bus and notification service.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | `Amazon API Gateway` | (Unchanged) System entry point. |
| **User Service** | `User Service Container` | (Unchanged) Manages user data. |
| **Post Service** | `Post Service Container` | (Updated Rationale) Now fully decoupled from downstream concerns like notifications by publishing events to SNS. |
| **Event Bus** | `Amazon SNS Topic` & `Amazon SQS Queue` | **Scalability & Resilience:** Using a managed pub/sub system (SNS) with durable queues (SQS) is a robust, highly scalable, and serverless pattern for event-driven architectures. It guarantees event delivery and allows consumers to process events at their own pace. |
| **Notification Service** | `Notification Service Container` | **Decoupled Consumer:** A new, single-purpose service that reacts to business events. Deploying it on the shared EC2 instance is a cost-effective initial step. |
| **Cache & Job Queue**| `Amazon ElastiCache for Redis` | (Unchanged) High-performance cache and job queue. |
| **Worker Service** | `Worker Service Container` | (Unchanged) Asynchronous job processor. |
| **PostgreSQL Database**| `Amazon RDS for PostgreSQL` | (Unchanged) The system's source of truth. |
