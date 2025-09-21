#### 1. Logical View (C4 Component Diagram)

This diagram visualizes the decomposition. We now have two distinct services behind the API Gateway, each with its own responsibilities.

```mermaid
C4Component
    title Component Diagram for InkWell System (User Service Extraction)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(gateway, "API Gateway", "Amazon API Gateway", "Routes public traffic to the appropriate internal service.")
        
        System_Boundary(c2, "Application Services") {
            Component(userService, "User Service", "Container: Go", "Handles user registration, authentication, and profile management. The gatekeeper for user data.")
            Component(postService, "Post Service", "Container: Ruby on Rails", "Handles post creation, editing, and viewing.")
        }

        ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Managed database cluster. Contains separate schemas for each service.")
    }

    Rel(writer, gateway, "Uses", "HTTPS/JSON")

    Rel_Down(gateway, userService, "Routes /users, /login to", "HTTPS/JSON")
    Rel_Down(gateway, postService, "Routes /posts to", "HTTPS/JSON")

    Rel(postService, userService, "Gets user data from", "HTTPS/JSON")

    Rel(userService, db, "Reads/Writes [User Schema]", "SQL")
    Rel(postService, db, "Reads/Writes [Post Schema]", "SQL")
```

#### 2. Physical View (AWS Deployment Diagram)

For this initial decomposition, we will run the new `user-service` container on the *same* EC2 instance as the `post-service` to manage cost and complexity. The API Gateway is now responsible for routing to the correct container port.

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
                end
            end

            subgraph "Amazon RDS"
                direction LR
                db_primary["RDS Primary Instance<br>(PostgreSQL)"]
                db_replica["RDS Read Replica<br>(PostgreSQL)"]

                db_primary -- "Replication" --> db_replica
            end

            Gateway["Amazon API Gateway"]
        end
    end

    User -- "HTTPS" --> Gateway
    Gateway -- "Routes /posts/*" --> post_container
    Gateway -- "Routes /users/*" --> user_container
    
    post_container -- "API Call (HTTP)" --> user_container

    user_container -- "SQL" --> db_primary
    post_container -- "SQL" --> db_primary
    user_container -- "SQL" --> db_replica
    post_container -- "SQL" --> db_replica
```

#### 3. Component-to-Resource Mapping Table

We add the new `User Service` component and update the roles of the existing components.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | `Amazon API Gateway` | (Updated Rationale) Now performs L7 routing, directing traffic to different backend services based on the request path (e.g., `/users` vs `/posts`). |
| **User Service** | `User Service Container` running on the shared AWS EC2 instance. | **State Encapsulation:** This new service owns all logic and data related to users. Running it on the same instance as the post service is a cost-effective intermediate step before migrating to a full container orchestration platform. |
| **Post Service** (formerly InkWell API) | `Post Service Container` running on the shared AWS EC2 instance. | **Domain Focus:** This service is now leaner, focusing solely on post management. It delegates all user-related concerns to the `User Service`. |
| **PostgreSQL Database** | `Amazon RDS for PostgreSQL` | (Updated Rationale) The single database cluster now serves multiple services. Access should be logically partitioned (e.g., using separate schemas and database users) to enforce boundaries and the principle of least privilege. |
