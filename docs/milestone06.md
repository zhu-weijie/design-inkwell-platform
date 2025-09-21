#### 1. Logical View (C4 Component Diagram)

We will update the description of the Redis component and add a new relationship from the `Post Service` to it, representing the cache read operation.

```mermaid
C4Component
    title Component Diagram for InkWell System (Caching Layer)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic to the appropriate internal service.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "Container: Go", "Handles user registration, authentication, and profile management.")
            Component(postService, "Post Service", "Container: Ruby on Rails", "Handles post management and serves post content, with caching.")
            Component(workerService, "Worker Service", "Container: Python", "Executes long-running, asynchronous jobs.")
        }

        ComponentQueue(cacheQueue, "Cache & Job Queue", "Amazon ElastiCache for Redis", "Provides a high-speed cache and stores background jobs.")
        ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Managed database cluster with separate schemas.")
    }

    Rel(writer, gateway, "Uses", "HTTPS/JSON")

    Rel_Down(gateway, userService, "Routes to", "HTTPS/JSON")
    Rel_Down(gateway, postService, "Routes to", "HTTPS/JSON")

    Rel(postService, userService, "Gets user data from", "HTTPS/JSON")

    Rel(postService, cacheQueue, "Enqueues job / Reads from cache", "Redis Command")
    Rel(workerService, cacheQueue, "Dequeues job", "Redis Command")
    
    Rel(workerService, db, "Updates post status", "SQL")
    Rel(userService, db, "Reads/Writes [User Schema]", "SQL")
    Rel(postService, db, "Reads/Writes [Post Schema] on cache miss", "SQL")
```

#### 2. Physical View (AWS Deployment Diagram)

The physical infrastructure doesn't change, but the data flow becomes more sophisticated. We add new arrows to represent the cache check and the subsequent database query on a cache miss.

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
                    post_container["Post Service Container<br>(Rails App)"]
                    user_container["User Service Container<br>(Go App)"]
                    worker_container["Worker Service Container<br>(Python App)"]
                end
            end

            subgraph "Stateful Services"
                subgraph "Amazon ElastiCache"
                    redis["Redis Cluster"]
                end

                subgraph "Amazon RDS"
                    db_primary["RDS Primary Instance"]
                    db_replica["RDS Read Replica"]
                end
            end
        end
    end

    %% Define Flows
    User -- "HTTPS" --> Gateway
    Gateway -- "/posts/*" --> post_container
    Gateway -- "/users/*" --> user_container
    
    post_container -- "Cache Check" --> redis
    post_container -- "DB Query on Miss" --> db_replica
    
    post_container -- "API Call" --> user_container
    post_container -- "Enqueue Job" --> redis
    
    worker_container -- "Dequeue Job" --> redis
    worker_container -- "SQL Update" --> db_primary
    
    user_container -- "SQL Writes" --> db_primary
    user_container -- "SQL Reads" --> db_replica

    db_primary -- "Replication" --> db_replica
```

#### 3. Component-to-Resource Mapping Table

We update the name and rationale for the Redis component and the `Post Service`.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | `Amazon API Gateway` | (Unchanged) Routes traffic to synchronous, user-facing services. |
| **User Service** | `User Service Container` | (Unchanged) Manages user data. |
| **Post Service** | `Post Service Container` | (Updated Rationale) Implements a read-through cache to reduce database load and improve latency for frequent reads. |
| **Cache & Job Queue** | `Amazon ElastiCache for Redis` | (Updated Rationale) This managed service now serves two critical roles: a high-performance, in-memory cache for hot data and a reliable queue for background jobs. Using a single Redis cluster for both is cost-effective at this stage. |
| **Worker Service** | `Worker Service Container` | (Unchanged) Stateless component for asynchronous processing. |
| **PostgreSQL Database**| `Amazon RDS for PostgreSQL` | (Unchanged) The source of truth for all core data. Its load is now protected by the caching layer. |
