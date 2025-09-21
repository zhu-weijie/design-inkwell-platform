#### 1. Logical View (C4 Component Diagram)

The logical view remains very similar, as the fundamental components have not changed. However, we update the description of the database component to reflect its new nature as a managed, scalable entity.

```mermaid
C4Component
    title Component Diagram for InkWell System (Production Persistence)

    Person(writer, "Writer", "A user who creates and manages posts.")

    System_Boundary(c1, "InkWell System") {
        Component(monolith, "InkWell API", "Container: Ruby on Rails", "The monolithic application handling all business logic.")
        ComponentDb(db, "PostgreSQL Database", "Amazon RDS", "Managed, highly-available database cluster for storing all application data.")
    }

    Rel(writer, monolith, "Uses", "HTTPS/JSON")
    Rel(monolith, db, "Reads/Writes", "SQL")
```

#### 2. Physical View (AWS Deployment Diagram)

This diagram shows the most significant change. The database container is removed from the EC2 instance, and a new, managed RDS service is introduced within our cloud environment.

```mermaid
graph TD
    subgraph "Internet"
        User["Writer's Browser"]
    end

    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "EC2 Instance"
                subgraph "Docker"
                    api_container["InkWell API Container<br>(Rails App)"]
                end
            end

            subgraph "Amazon RDS"
                direction LR
                db_primary["RDS Primary Instance<br>(PostgreSQL)"]
                db_replica["RDS Read Replica<br>(PostgreSQL)"]

                db_primary -- "Replication" --> db_replica
            end
        end
    end

    User -- "HTTPS" --> EC2_Instance
    api_container -- "SQL Writes" --> db_primary
    api_container -- "SQL Reads" --> db_replica
```

#### 3. Component-to-Resource Mapping Table

This table is updated to reflect the new physical resources and the rationale for this critical architectural evolution.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **InkWell API** | `InkWell API Container` running on an AWS EC2 Instance. | (Unchanged) A containerized application provides a consistent runtime environment. Keeping it on a single EC2 instance is sufficient for this stage. |
| **PostgreSQL Database** | `Amazon RDS for PostgreSQL` with one Primary and one Read Replica instance. | **Reliability and Scalability:** Moving to a managed service like RDS offloads the critical tasks of database management (backups, patching, failover). Using a primary/replica setup is a standard pattern that provides high availability and allows us to scale read-heavy workloads independently of the write workload, protecting the primary instance's performance. |
