#### 1. Logical View (C4 Component Diagram)

This diagram introduces the new `Secret Store` as a critical infrastructure component that all services depend on.

```mermaid
C4Component
    title Component Diagram for InkWell System (Hardened Security)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "...")
        
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
             Component(secretStore, "Secret Store", "AWS Secrets Manager", "Securely stores secrets. All Application Services<br/>fetch secrets at startup. (Flow shown from<br/>Post Service as a representative example.)")
        }

        System_Boundary(c4, "Observability Stack") {
            Component(metrics, "Metrics System", "...")
            Component(logging, "Logging System", "...")
            Component(tracing, "Tracing System", "...")
        }
    }

    %% Main User Flow
    Rel(writer, gateway, "Uses")
    
    %% Secret Management Flow (Representative Example)
    Rel(postService, secretStore, "Fetches secrets at startup")
```

#### 2. Physical View (AWS Deployment Diagram)

The physical diagram adds the new `AWS Secrets Manager` service and shows how services interact with it via IAM roles.

```mermaid
graph LR
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        Gateway["Amazon API Gateway"]
        S3["Amazon S3 Bucket"]
        SecretsManager["AWS Secrets Manager"]

        subgraph "VPC"
            subgraph "App EC2 Instance"
                IAM_Role["EC2 Instance IAM Role<br>(Grants access to secrets)"]
                IAM_Role -- "Provides credentials to" --> Docker
                subgraph "Docker"
                    post_container["Post Service"]
                    user_container["User Service"]
                    worker_container["Worker Service"]
                    notification_container["Notification Service"]
                end
            end

            %% Other groups for clarity
            Monitoring["Monitoring Infrastructure"]
            AWSServices["Other AWS Services"]
        end
    end

    %% Define Secret Fetch Flow (using a representative example)
    post_container -- "Fetches secrets at startup" --> SecretsManager
    
    %% User Flow
    User -- "HTTPS" --> Gateway
    Gateway -- "/posts/*" --> post_container
    Gateway -- "/users/*" --> user_container
```

#### 3. Component-to-Resource Mapping Table

We add the final component, the `Secret Store`.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Secret Store** | `AWS Secrets Manager` or `HashiCorp Vault` | **Security Best Practice:** A dedicated, managed secret store is the industry standard for handling sensitive information. It provides fine-grained access control via IAM, automated secret rotation, and robust audit logging. This is a non-negotiable component for a secure, production-grade system. |
| **Application Services** | (All service containers) | **(Updated Rationale)** Services are now configured via a combination of environment variables (for non-secrets) and secrets fetched at runtime from the Secret Store (for credentials). This creates a clean, secure, and operationally flexible 12-Factor App design. |
| ...(Other components unchanged)... | ... | ... |
