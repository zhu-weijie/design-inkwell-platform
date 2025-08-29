#### 1. Logical View (C4 Component Diagram)

This diagram introduces a new "Object Storage" component. Crucially, it shows the `Writer` interacting directly with it for the upload, which is a key part of the new design.

```mermaid
C4Component
    title Component Diagram for InkWell System (Object Storage)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "Container: Go", "Manages user data.")
            Component(postService, "Post Service", "Container: Ruby on Rails", "Manages posts and orchestrates media uploads.")
            Component(workerService, "Worker Service", "Container: Python", "Executes background jobs.")
            Component(notificationService, "Notification Service", "Container: Go", "Consumes events.")
        }
        
        System_Boundary(c3, "Infrastructure Layer") {
             ComponentDb(storage, "Object Storage", "Amazon S3", "Stores all user-uploaded media files.")
             Component(eventBus, "Event Bus", "Amazon SNS/SQS", "Decouples services.")
             ComponentQueue(cacheQueue, "Cache & Job Queue", "Amazon ElastiCache for Redis", "Provides cache and job storage.")
             ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Source of truth for structured data.")
        }
    }

    %% Relationships
    Rel(writer, gateway, "1. Requests pre-signed URL", "HTTPS/JSON")
    Rel(writer, storage, "2. Uploads file directly", "HTTPS")

    Rel(gateway, postService, "Routes URL request to")
    Rel(postService, db, "Saves file URL reference", "SQL")
    
    %% Existing relationships
    Rel(gateway, userService, "Routes to")
    Rel(postService, userService, "Gets user data")
    Rel(postService, eventBus, "Publishes events")
    Rel(notificationService, eventBus, "Consumes events")
```

#### 2. Physical View (AWS Deployment Diagram)

The physical diagram adds the new `Amazon S3 Bucket` and clearly visualizes the new two-part upload flow.

```mermaid
graph LR
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        Gateway["Amazon API Gateway"]
        S3["Amazon S3 Bucket"]

        subgraph "VPC"
            subgraph "EC2 Instance"
                subgraph "Docker"
                    post_container["Post Service"]
                    user_container["User Service"]
                    worker_container["Worker Service"]
                    notification_container["Notification Service"]
                end
            end

            subgraph "Messaging Services"
                sns["SNS Topic"]
                sqs["SQS Queue"]
            end

            subgraph "Stateful Services"
                redis["Redis Cluster"]
                db_primary["RDS Primary"]
                db_replica["RDS Replica"]
            end
        end
    end

    %% Define Upload Flow
    User -- "Req Signed URL" --> Gateway
    Gateway -- " " --> post_container
    post_container -- "Return Signed URL" --> User
    User -- "Upload Direct to S3" --> S3

    %% Define Existing Flows
    Gateway -- "/users/*" --> user_container
    post_container -- "Publish Event" --> sns
    sns -- "Push" --> sqs
    notification_container -- "Poll" --> sqs
    post_container -- "API Call" --> user_container
    worker_container -- "Dequeue Job" --> redis
    post_container -- "SQL" --> db_primary
    db_primary -- "Replication" --> db_replica
```

#### 3. Component-to-Resource Mapping Table

We add the new object storage component.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Object Storage** | `Amazon S3 Bucket` | **Scalability, Durability, and Performance:** S3 is the industry standard for object storage, offering virtually limitless scalability and extreme durability. The pre-signed URL pattern offloads the bandwidth and processing of file uploads from our application servers, which is critical for performance and cost-effectiveness. |
| **Post Service** | `Post Service Container` | (Updated Rationale) Its role in uploads is now orchestration: it validates the request, generates a secure, short-lived upload URL via the AWS SDK, and saves the final file location to the database. It no longer processes the file contents directly. |
| ...(Other components unchanged)... | ... | ... |
