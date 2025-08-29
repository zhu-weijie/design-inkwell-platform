#### 1. Logical View (C4 Component Diagram)

This diagram adds the new `Job Queue` and `Worker Service` and shows how they interact with the existing `Post Service`.

```mermaid
C4Component
    title Component Diagram for InkWell System (Async Processing)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic to the appropriate internal service.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "Container: Go", "Handles user registration, authentication, and profile management.")
            Component(postService, "Post Service", "Container: Ruby on Rails", "Handles initial post creation and management.")
            Component(workerService, "Worker Service", "Container: Python", "Executes long-running, asynchronous jobs.")
        }

        ComponentQueue(queue, "Job Queue", "Amazon ElastiCache for Redis", "Stores background jobs to be processed asynchronously.")
        ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Managed database cluster with separate schemas.")
    }

    Rel(writer, gateway, "Uses", "HTTPS/JSON")

    Rel_Down(gateway, userService, "Routes to", "HTTPS/JSON")
    Rel_Down(gateway, postService, "Routes to", "HTTPS/JSON")

    Rel(postService, userService, "Gets user data from", "HTTPS/JSON")

    Rel(postService, queue, "Enqueues 'process-post' job", "Redis Command")
    Rel(workerService, queue, "Dequeues job", "Redis Command")
    
    Rel(workerService, db, "Updates post status", "SQL")
    Rel(userService, db, "Reads/Writes [User Schema]", "SQL")
    Rel(postService, db, "Reads/Writes [Post Schema]", "SQL")
```

#### 2. Physical View (AWS Deployment Diagram)

We will add the new `worker-service` container to our existing EC2 instance and introduce a new managed `Amazon ElastiCache` service for the Redis queue.

```mermaid
graph TD
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "EC2 Instance (Private Subnet)"
                subgraph "Docker"
                    direction LR
                    post_container["Post Service Container<br>(Rails App)"]
                    user_container["User Service Container<br>(Go App)"]
                    worker_container["Worker Service Container<br>(Python App)"]
                end
            end

            subgraph "Amazon RDS"
                direction LR
                db_primary["RDS Primary Instance<br>(PostgreSQL)"]
                db_replica["RDS Read Replica<br>(PostgreSQL)"]
                db_primary -- "Replication" --> db_replica
            end
            
            subgraph "Amazon ElastiCache"
                redis["Redis Cluster"]
            end

            Gateway["Amazon API Gateway"]
        end
    end

    User -- "HTTPS" --> Gateway
    Gateway -- "/posts/*" --> post_container
    Gateway -- "/users/*" --> user_container
    
    post_container -- "API Call" --> user_container
    post_container -- "ENQUEUE" --> redis
    worker_container -- "DEQUEUE" --> redis
    
    worker_container -- "SQL Update" --> db_primary
    user_container -- "SQL" --> db_primary
    post_container -- "SQL" --> db_primary
```

#### 3. Component-to-Resource Mapping Table

We add the new components for the queue and the worker.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | `Amazon API Gateway` | (Unchanged) Routes traffic to synchronous, user-facing services. |
| **User Service** | `User Service Container` | (Unchanged) Manages user data. |
| **Post Service** | `Post Service Container` | (Updated Rationale) Now offloads slow operations by enqueuing jobs into Redis, resulting in faster API responses. |
| **Job Queue** | `Amazon ElastiCache for Redis` | **Reliability and Simplicity:** Using a managed service for our queue (a stateful component) offloads the operational burden of managing a reliable Redis cluster. It's a proven, high-performance choice for background job queues. |
| **Worker Service** | `Worker Service Container` running on the shared EC2 instance. | **Stateless Scalability:** This is a stateless processing component. For now, running it on the shared EC2 instance is the most cost-effective solution. In the future, this is a prime candidate to be moved to a serverless or auto-scaling container platform. |
| **PostgreSQL Database** | `Amazon RDS for PostgreSQL` | (Unchanged) The source of truth for all core data. |
