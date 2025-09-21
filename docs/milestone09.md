#### 1. Logical View (C4 Component Diagram)

This diagram introduces the observability stack as a separate logical block that supports all our application services.

```mermaid
C4Component
    title Component Diagram for InkWell System (with Observability)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "...")
            Component(postService, "Post Service", "...")
            Component(workerService, "Worker Service", "...")
            Component(notificationService, "Notification Service", "...")
        }
        
        System_Boundary(c3, "Infrastructure Layer") {
             ComponentDb(storage, "Object Storage", "...")
             Component(eventBus, "Event Bus", "...")
             ComponentQueue(cacheQueue, "Cache & Job Queue", "...")
             ComponentDb(db, "PostgreSQL Database", "...")
        }

        System_Boundary(c4, "Observability Stack") {
            Component(tracing, "Tracing System", "AWS X-Ray / Jaeger", "Collects and visualizes distributed traces.")
            Component(metrics, "Metrics System", "Prometheus & Grafana", "Collects metrics. For clarity, only flows from<br/>the Post Service are shown as a representative example.<br/>All Application Services send similar telemetry.")
            Component(logging, "Logging System", "OpenSearch", "Aggregates and indexes structured logs.")
            
        }
    }

    %% Main Application Flows
    Rel(writer, gateway, "Uses")
    Rel(gateway, userService, "Routes to")
    Rel(gateway, postService, "Routes to")
    
    %% Observability Flow (Using Post Service as a representative example)
    Rel(postService, metrics, "Sends Metrics")
    Rel(postService, logging, "Sends Logs")
    Rel(postService, tracing, "Sends Spans")

    Rel(gateway, tracing, "Initiates Traces")
```

#### 2. Physical View (AWS Deployment Diagram)

The physical diagram adds a new "Monitoring" instance and a log-shipping agent to our application instance.

```mermaid
graph LR
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        Gateway["Amazon API Gateway<br>(Adds Trace ID)"]
        S3["Amazon S3 Bucket"]

        subgraph "VPC"
            subgraph "App EC2 Instance"
                subgraph "Docker"
                    post_container["Post Service"]
                    user_container["User Service"]
                    worker_container["Worker Service"]
                    notification_container["Notification Service"]
                    log_shipper["Log Shipper<br>(Fluentd)"]
                end
            end

            subgraph "Monitoring EC2 Instance"
                prometheus["Prometheus"]
                grafana["Grafana"]
                opensearch["OpenSearch"]
                jaeger["Jaeger / X-Ray Collector"]
            end

            subgraph "Other AWS Services"
                %% Grouping existing services for clarity
                sns["SNS Topic"]
                sqs["SQS Queue"]
                redis["Redis Cluster"]
                db_primary["RDS Primary"]
            end
        end
    end

    %% Define Observability Flows
    prometheus -- "Scrapes /metrics" --> post_container
    prometheus -- "Scrapes /metrics" --> user_container
    log_shipper -- "Ships Logs" --> opensearch
    post_container -- "Sends Traces" --> jaeger
    user_container -- "Sends Traces" --> jaeger

    %% Define Main User Flow
    User -- "HTTPS" --> Gateway
    Gateway -- "/posts/*" --> post_container
    Gateway -- "/users/*" --> user_container
```

#### 3. Component-to-Resource Mapping Table

We add the new logical components for the observability stack.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Metrics System** | `Prometheus` for collection, `Grafana` for dashboards, running on a dedicated Monitoring EC2 instance. | **Industry Standard:** Prometheus is the de-facto standard for metrics collection in cloud-native systems. A dedicated instance isolates the resource consumption of the monitoring stack from the application stack. |
| **Logging System** | `Fluentd` as a log shipping agent on the app instance, sending logs to a managed `Amazon OpenSearch` cluster. | **Decoupled and Scalable:** Using an agent like Fluentd decouples log shipping from the application logic. A managed OpenSearch cluster provides a powerful and scalable solution for searching and analyzing high volumes of log data without operational overhead. |
| **Tracing System** | `AWS X-Ray` or a self-hosted `Jaeger` collector. | **End-to-End Visibility:** Distributed tracing is the only way to effectively debug and analyze the performance of a request as it flows through multiple services, queues, and events. |
| ...(Other components unchanged)... | ... | ... |
