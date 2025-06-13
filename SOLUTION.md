# System Design for a Financial Events and File Attachment Service

This document outlines the system design for a backend service that records financial events using double-entry accounting and allows users to attach files to these events.

## 1. Overall Architecture

The proposed architecture is based on a microservices pattern. This approach offers scalability, fault isolation, and technology-agnostic development for different parts of the system. The core idea is to separate the financial transaction logic from the file handling logic.

Here is a high-level diagram of the architecture:

```mermaid
graph TD
    subgraph Client
        A[User's Browser/App]
    end

    subgraph CDN
        B[Cloudflare/CloudFront]
    end

    subgraph API_Gateway
        C[API Gateway]
    end

    subgraph Services
        D[Ledger Service]
        E[File Service]
        F[IAM Service]
        G[Notification Service]
    end

    subgraph Data_Stores
        H[PostgreSQL Ledger DB]
        I[S3 Object Storage Files]
        J[DynamoDB Cassandra File Metadata]
        K[Redis Cache]
    end

    subgraph Message_Queue
        L[RabbitMQ Kafka]
    end

    A --> B
    B --> C
    C --> D
    C --> E
    C --> F

    D -- "Transactions" --> H
    E -- "Metadata" --> J
    E -- "Upload/Download" --> I

    E -- "Publish event" --> L
    L -- "Consume event" --> G
    L -- "Consume event for async processing" --> E

    D -- "Cache account info" --> K
    E -- "Cache file metadata" --> K


```
![High-Level-Arch](img/Arch.png)



### Components:

*   **API Gateway**: The single entry point for all client requests. It handles routing, authentication, rate limiting, and caching.
*   **IAM (Identity & Access Management) Service**: Manages user authentication and authorization. It will issue tokens (e.g., JWT) that will be used to authenticate requests to other services.
*   **Ledger Service**: The core of the accounting system. It's responsible for creating and validating financial transactions based on double-entry principles.
*   **File Service**: Handles all file-related operations: generating pre-signed URLs for uploads, processing files, storing metadata, and serving files.
*   **Notification Service**: Sends asynchronous notifications to users (e.g., email, push notifications) after certain events, like a successful file upload and processing.
*   **Message Queue (RabbitMQ/Kafka)**: Decouples services and handles asynchronous tasks like file processing, thumbnail generation, and notifications.
*   **Databases**:
    *   **PostgreSQL**: A relational database for the `Ledger Service` to ensure ACID compliance for financial transactions.
    *   **Amazon S3 (or other object storage)**: For storing the actual files. It's highly scalable, durable, and cost-effective.
    *   **DynamoDB/Cassandra**: A NoSQL database to store file metadata. It's chosen for its high availability and scalability for read/write operations.
*   **CDN (Content Delivery Network)**: Used to cache and serve files to users globally with low latency.

## 2. Data Models and Storage

### 2.1. Ledger Service Data Model (PostgreSQL)

We need three main tables for the double-entry ledger.

*   `accounts`: Stores information about each account (e.g., Cash, Accounts Receivable).
*   `entries`: Records individual debit and credit operations.
*   `transactions`: Groups a set of entries that represent a single financial event. A transaction must have balanced debits and credits.
*   `transaction_files`: Links files to transactions.

**`accounts` table:**
```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE
    balance DECIMAL(20, 2) NOT NULL DEFAULT 0.00,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**`entries` table:**
```sql
CREATE TABLE entries (
    id UUID PRIMARY KEY,
    transaction_id UUID NOT NULL REFERENCES transactions(id),
    account_id UUID NOT NULL REFERENCES accounts(id),
    amount DECIMAL(20, 2) NOT NULL,
    direction VARCHAR(10) NOT NULL, -- DEBIT or CREDIT
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_entries_transaction_id ON entries(transaction_id);
CREATE INDEX idx_entries_account_id ON entries(account_id);
```

**`transactions` table:**
```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**`transaction_files` table:**
```sql
CREATE TABLE transaction_files (
    transaction_id UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,
    file_id UUID NOT NULL,
    PRIMARY KEY (transaction_id, file_id)
);
```

### 2.2. File Service Data Model

**File Metadata (DynamoDB/Cassandra):**
A NoSQL database is a good choice here for scalability and fast lookups.

*   **Table Name**: `file_metadata`
*   **Primary Key**: `file_id` (UUID)
*   **Attributes**:
    *   `file_id`: UUID
    *   `transaction_id`: UUID (optional, can be linked after transaction is created)
    *   `user_id`: UUID
    *   `filename`: String
    *   `s3_key`: String (path to the file in S3)
    *   `file_type`: String (e.g., 'image/jpeg', 'application/pdf')
    *   `size_bytes`: Number
    *   `upload_status`: String ('PENDING', 'COMPLETED', 'FAILED')
    *   `previews`: Map (e.g., `{ "thumbnail_small": "s3_key_thumb_s.jpg" }`)
    *   `created_at`: Timestamp
    *   `updated_at`: Timestamp

**File Storage (Amazon S3):**
Files will be stored in an S3 bucket with a well-defined folder structure.
Example: `s3://<bucket-name>/<user_id>/<file_id>/<filename>`

## 3. API Contracts

The API will be exposed via the API Gateway.

### 3.1. Ledger Service API

#### `POST /transactions`
Creates a new financial transaction.

**Request Body:**
```json
{
  "description": "Payment for services",
  "file_ids": ["uuid-of-the-file"],
  "entries": [
    {
      "account_id": "uuid-for-cash-account",
      "amount": "100.00",
      "direction": "DEBIT"
    },
    {
      "account_id": "uuid-for-revenue-account",
      "amount": "100.00",
      "direction": "CREDIT"
    }
  ]
}
```

**Success Response (201 Created):**
```json
{
  "transaction_id": "new-transaction-uuid",
  "status": "created"
}
```
**Error Response (400 Bad Request):**
*   If debits do not equal credits.
*   If an account does not exist.

### 3.2. File Service API

#### `POST /files/upload-url`
Requests a pre-signed URL to upload a file.

**Request Body:**
```json
{
  "filename": "invoice.pdf",
  "file_type": "application/pdf"
}
```
**Success Response (200 OK):**
```json
{
  "file_id": "new-file-uuid",
  "upload_url": "https://s3-presigned-url-for-upload"
}
```
Client will then use this URL to `PUT` the file directly to S3.

#### `GET /files/{file_id}`
Retrieves a file's metadata and a short-lived download URL.

**Success Response (200 OK):**
```json
{
    "file_id": "file-uuid",
    "filename": "invoice.pdf",
    "file_type": "application/pdf",
    "size_bytes": 123456,
    "download_url": "https://s3-presigned-url-for-download"
}
```

#### `POST /transactions/{transaction_id}/files`
Links an uploaded file to a transaction.

**Request Body:**
```json
{
    "file_id": "file-uuid"
}
```
**Success Response (200 OK):**
```json
{
    "status": "linked"
}
```

## 4. Data Integrity: Atomicity and Concurrency

This is the most critical part of the system design. Financial data must be perfectly consistent. We address this at two levels: ensuring each transaction is atomic (the double-entry rule) and ensuring that simultaneous operations don't corrupt account balances (concurrency control).

### 4.1. The Challenge: Race Conditions in Concurrent Transactions

A critical challenge in a financial system is handling concurrent requests that affect the same data. For example, if two separate API calls try to modify the balance of the same `Cash` account at the same time, a **race condition** can occur, leading to an incorrect final balance.

**Example of a Race Condition**:
1.  Request A reads the `Cash` balance: **$100**.
2.  Request B reads the `Cash` balance: **$100**.
3.  Request A adds a $50 debit and calculates the new balance: $150.
4.  Request B adds a $20 debit and calculates the new balance: $120.
5.  Request A writes its new balance to the database: **$150**.
6.  Request B writes its new balance to the database: **$120**.

The final balance is now **$120**. The $50 from Request A has been completely lost. The correct balance should have been **$170**.

### 4.2. The Solution: Atomic Transactions with Pessimistic Locking

We solve both atomicity and concurrency problems using features within PostgreSQL.

1.  **Database Transactions for Atomicity**: The creation of a transaction and all its associated entries will be wrapped in a single database transaction (`BEGIN`...`COMMIT`). This guarantees that the entire operation succeeds or fails as a single unit. If an error occurs midway through (e.g., one entry fails to write), the entire transaction is rolled back, preventing partial, unbalanced data from ever being saved.

2.  **Pessimistic Locking for Concurrency**: To prevent the race condition described above, we use row-level pessimistic locking. Before updating an account's balance, we explicitly lock the row for that account using the `SELECT ... FOR UPDATE` SQL statement.

    *   **How it Works**: When Transaction A locks the `Cash` account row, Transaction B is blocked and forced to wait. It cannot read or write to that row until Transaction A completes. This serializes access to the account balance, guaranteeing data integrity even under high concurrency.

The following pseudocode shows how these concepts are combined:
```golang
// Pseudocode in Go for Ledger Service
func (s *LedgerService) CreateTransaction(ctx context.Context, description string, fileIDs []string, entries []*Entry) (string, error) {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return "", err
    }
    defer tx.Rollback() // Rollback on error

    // 1. Validate that debits == credits
    var totalDebits, totalCredits decimal.Decimal
    for _, entry := range entries {
        if entry.Direction == "DEBIT" {
            totalDebits = totalDebits.Add(entry.Amount)
        } else {
            totalCredits = totalCredits.Add(entry.Amount)
        }
    }
    if !totalDebits.Equal(totalCredits) {
        return "", errors.New("debits do not equal credits")
    }

    // 2. Create the transaction record
    var transactionID string
    err = tx.QueryRowContext(ctx, "INSERT INTO transactions (description) VALUES ($1) RETURNING id", description).Scan(&transactionID)
    if err != nil {
        return "", err
    }

    // 3. Link files to the transaction
    for _, fileID := range fileIDs {
        _, err := tx.ExecContext(ctx, "INSERT INTO transaction_files (transaction_id, file_id) VALUES ($1, $2)", transactionID, fileID)
        if err != nil {
            return "", err // This will trigger the rollback
        }
    }

    // 4. For each entry, lock the account, create the entry, and update the balance.
    for _, entry := range entries {
        // STEP 4.A: Lock the account row to prevent race conditions.
        // This query will block if another transaction has locked this same row.
        // We also fetch the account type to determine how to apply the balance change.
        var accountType string
        err := tx.QueryRowContext(ctx, "SELECT type FROM accounts WHERE id = $1 FOR UPDATE", entry.AccountID).Scan(&accountType)
        if err != nil {
            if err == sql.ErrNoRows {
                return "", fmt.Errorf("account with id %s not found", entry.AccountID)
            }
            return "", err // Other database error
        }

        // STEP 4.B: Determine the correct balance update logic based on standard accounting rules.
        // Assets & Expenses: Increase with Debit, Decrease with Credit.
        // Liabilities, Equity, Revenue: Decrease with Debit, Increase with Credit.
        var amountToUpdate decimal.Decimal
        isDebit := entry.Direction == "DEBIT"
        
        if accountType == "ASSET" || accountType == "EXPENSE" {
            if isDebit {
                amountToUpdate = entry.Amount // Increase
            } else {
                amountToUpdate = entry.Amount.Neg() // Decrease
            }
        } else { // LIABILITY, EQUITY, REVENUE
            if isDebit {
                amountToUpdate = entry.Amount.Neg() // Decrease
            } else {
                amountToUpdate = entry.Amount // Increase
            }
        }

        // STEP 4.C: Apply the update to the account's balance.
        _, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + $1, updated_at = NOW() WHERE id = $2", amountToUpdate, entry.AccountID)
        if err != nil {
           return "", err
        }
        
        // STEP 4.D: Insert the entry record for this leg of the transaction.
        _, err = tx.ExecContext(ctx, "INSERT INTO entries (transaction_id, account_id, amount, direction) VALUES ($1, $2, $3, $4)", transactionID, entry.AccountID, entry.Amount, entry.Direction)
        if err != nil {
            return "", err // This will trigger the rollback
        }
    }

    // 5. Publish an event for other services
    // This allows the File Service to learn about the transaction linking without being directly called.
    eventPayload := map[string]interface{}{
        "transaction_id": transactionID,
        "file_ids":       fileIDs,
    }
    // s.messageQueue.Publish("transaction.created", eventPayload) // Fire-and-forget

    return transactionID, tx.Commit() // Commit the transaction
}
```

## 5. File Ingestion and Delivery

```mermaid
sequenceDiagram
    participant Client
    participant API Gateway
    participant File Service
    participant S3
    participant Message Queue
    participant CDN

    Client->>API Gateway: 1. POST /files/upload-url
    API Gateway->>File Service: 2. Request pre-signed URL
    File Service->>File Service: 3. Create metadata record (status: PENDING)
    File Service->>S3: 4. Generate pre-signed PUT URL
    S3-->>File Service: 5. Pre-signed URL
    File Service-->>API Gateway: 6. Return URL to client
    API Gateway-->>Client: 7. URL and file_id
    Client->>S3: 8. PUT file to pre-signed URL
    S3->>Message Queue: 9. S3 Event Notification (on object creation)
    Message Queue-->>File Service: 10. Consume upload event
    File Service->>File Service: 11. Validate file (type, size, scan for malware)
    File Service->>File Service: 12. Update metadata (status: COMPLETED)
    File Service->>Message Queue: 13. (Optional) Publish for thumbnail generation

    Client->>API Gateway: 14. GET /files/{file_id}
    API Gateway->>File Service: 15. Request download URL
    File Service->>CDN: 16. Generate signed URL for file on CDN
    CDN-->>File Service: 17. Signed URL
    File Service-->>API Gateway: 18. Return URL
    API Gateway-->>Client: 19. Download URL
    Client->>CDN: 20. Download file
```

![File-Ingestion-Flow](img/FileFlow.png)

*   **Hot vs. Cold Storage**: S3 lifecycle policies can automatically move older, less frequently accessed files to cheaper storage tiers (e.g., S3 Infrequent Access or Glacier) to save costs.
*   **Global Delivery**: A CDN like CloudFront will be configured with the S3 bucket as its origin. This will cache files at edge locations around the world for low-latency access.

### 5.1. Thumbnail and Preview Generation

The system can optionally generate previews like thumbnails for images or first-page previews for PDFs. This is handled asynchronously to avoid blocking the user.

1.  **Trigger**: After the **File Service** validates a newly uploaded file (Step 11 in the diagram), it publishes an event like `file_ready_for_preview` to the **Message Queue**.
2.  **Dedicated Workers**: A separate pool of workers (or a specialized service, the `Preview Service`) consumes these events. This isolates the compute-intensive task of media processing from the core API services.
3.  **Processing**: For an image, the worker uses a high-performance library like `libvips` in Go to generate thumbnails of various sizes (e.g., 100x100, 600x600). For a PDF, it might use a tool like `ghostscript` to extract the first page as an image.
4.  **Storage**: The generated previews are saved back to the S3 bucket, typically with a descriptive name (e.g., `<original_s3_key>_thumb_100.jpg`).
5.  **Update Metadata**: The worker then updates the `previews` map in the file's metadata record in DynamoDB to include the S3 keys of the new previews. The frontend can then query this metadata to display the appropriate thumbnail.

## 6. Security Controls

*   **Authentication**: The `IAM Service` will handle user login and issue short-lived JWTs. The API Gateway will validate these tokens on every request.
*   **Authorization**: The system will use Role-Based Access Control (RBAC). For example, a user can only access files linked to their own transactions. This logic will be enforced in the `File Service` and `Ledger Service`.
*   **Encryption**:
    *   **In Transit**: TLS will be enforced for all communication between the client, CDN, API Gateway, and services.
    *   **At Rest**: Data in PostgreSQL and S3 will be encrypted at rest.
*   **File Validation**: The `File Service` will validate file types and sizes upon upload. It will also perform a virus scan on each uploaded file before it's marked as `COMPLETED`.
*   **Audit Logs**: All sensitive operations (e.g., creating a transaction, accessing a file) will be logged for auditing purposes.

## 7. Strategy for Scale and Global Access


The system is designed to handle ~10 million financial transactions and ~50 million file uploads per month, with low-latency global access.

*   **Transactions**: 10M/month is ~3.8 transactions per second (TPS) on average.
*   **Uploads**: 50M/month is ~19 uploads per second on average.

We must design for peak loads, which can be 10x-100x the average. The strategy is as follows:

### 7.1. Scaling the Ledger Service (Handling ~400+ TPS Writes)

The write-heavy nature of the ledger is the primary database challenge.

*   **Database Choice**: We stick with PostgreSQL for its ACID compliance. For the scale required, we would use a managed service like Amazon Aurora or Google Cloud SQL, which simplifies management of replicas and scaling.
*   **Write Path Optimization**:
    *   **Connection Pooling**: Use a robust connection pooler like `PgBouncer` to manage database connections efficiently, which is critical for high throughput in Go applications.
    *   **Database Sharding**: This is the ultimate solution for write-intensive workloads. The database will be partitioned (sharded) based on a `tenant_id` or `user_id`.
        *   **Sharding Key**: `tenant_id` is often preferred in B2B systems. For a B2C system, a high-cardinality `user_id` is a good key. This ensures all data for a single user resides on the same shard, making queries efficient and avoiding costly cross-shard joins.
        *   **Implementation**: This can be managed by a proxy like `Citus` for PostgreSQL or implemented in the application layer. The `Ledger Service` would inspect the user's token, extract the sharding key, and route the database query to the correct shard.
*   **Read Path Optimization**:
    *   **Read Replicas**: For each shard, multiple read replicas will be deployed. All queries that do not need to be on the absolute latest data (e.g., generating an end-of-month report) will be directed to these replicas, freeing up the primary database for writes.
    *   **Caching (Redis)**: Frequently accessed, slowly changing data like account details and balances will be cached in Redis. This drastically reduces read load on the database for common operations.

### 7.2. Scaling the File Service & Ensuring Global Access (Handling ~2000+ Uploads/sec)

This service is stateless, making it easier to scale horizontally.

*   **Stateless, Horizontally-Scaled Service**: The `File Service` itself will be a containerized Go application running on a platform like Kubernetes (EKS) or ECS. An autoscaling policy will be set to add more container instances as CPU or memory usage increases, allowing it to handle virtually any number of requests for pre-signed URLs.
*   **Offloading to S3**: The core strategy is that the service *does not* handle the file upload stream. It only handles the lightweight metadata and URL generation. The multi-gigabyte bandwidth load is passed directly from the client to Amazon S3, which is built for massive, parallel throughput.
*   **Asynchronous Processing at Scale**: The file validation and preview generation is handled by workers consuming from a message queue.
    *   **Message Queue - Kafka**: At this scale, Apache Kafka is the preferred choice over RabbitMQ. Kafka's partitioned consumer model allows for incredible horizontal scaling. We can have hundreds of `File Service` workers in a consumer group, each processing messages from a subset of Kafka partitions in parallel.
*   **Global Low-Latency via CDN**:
    *   **Serving Files**: We will use a CDN like Amazon CloudFront with the S3 bucket as its origin. When a user requests to download a file, they are served from the CDN edge location physically closest to them, ensuring minimal latency.
    *   **S3 Cross-Region Replication**: To optimize for the *first* download in a new region (before it's cached by the CDN), we can use S3's Cross-Region Replication. We can have replicas of our primary bucket in key regions (e.g., `us-east-1`, `eu-west-1`, `ap-southeast-1`). Our `File Service` can be region-aware and generate a download URL pointing to the S3 bucket or CDN endpoint closest to the user.

## 8. Deployment and Hosting Strategy

To ensure the system is scalable, resilient, and easy to manage, we will adopt a modern cloud-native deployment strategy. This involves containerizing our applications and using an orchestrator to manage them, along with automating the entire provisioning and deployment process.

### 8.1. Containerization & Orchestration

*   **Containerization (Docker)**: Each microservice (`Ledger Service`, `File Service`, `IAM Service`, etc.) will be packaged as a lightweight, standalone Docker container. This encapsulates the Go application, its dependencies, and runtime, ensuring it runs consistently across any environment (development, staging, production).

*   **Orchestration (Kubernetes)**: We will use Kubernetes (or a managed equivalent like Amazon EKS, Google GKE, or Azure AKS) to orchestrate our containerized services. Kubernetes provides the essential framework for:
    *   **Automated Scaling**: Automatically scaling the number of service pods up or down based on real-time metrics like CPU or memory usage. This is critical for handling the peak loads described in Section 7.
    *   **Self-Healing**: Automatically restarting containers that fail, replacing unresponsive pods, and ensuring the desired number of replicas are always running.
    *   **Service Discovery & Load Balancing**: Managing how services find and communicate with each other and distributing traffic across healthy pods.
    *   **Rolling Deployments**: Updating services with zero downtime by incrementally replacing old pods with new ones.

### 8.2. Infrastructure as Code (IaC)

All underlying cloud infrastructure—including the Kubernetes cluster, PostgreSQL and DynamoDB databases, S3 buckets, Kafka cluster, and networking rules—will be managed using an Infrastructure as Code (IaC) tool like **Terraform** or **AWS CloudFormation**.

*   **Benefits**:
    *   **Repeatability**: IaC allows us to spin up an identical copy of our entire production environment for staging, testing, or disaster recovery with a single command.
    *   **Version Control**: Infrastructure definitions are stored in Git, providing a full history of changes, the ability to review changes via pull requests, and the option to roll back to a previous state.
    *   **Automation**: It eliminates manual configuration, reducing the risk of human error.

### 8.3. CI/CD Pipeline

A continuous integration and continuous deployment (CI/CD) pipeline is crucial for developer velocity and operational stability. We will use a tool like **GitHub Actions**, **GitLab CI**, or **Jenkins**.

A typical pipeline for a service (e.g., `Ledger Service`) would look like this:

1.  **Commit**: A developer pushes code to a feature branch in Git.
2.  **Build & Test (CI)**: The CI server automatically triggers:
    *   Compiling the Go code.
    *   Running unit and integration tests.
    *   Performing static analysis and linting.
    *   Building a new Docker image.
3.  **Push to Registry**: If tests pass, the Docker image is tagged and pushed to a container registry (e.g., Docker Hub, Amazon ECR, Google Container Registry).
4.  **Deploy (CD)**:
    *   **Staging**: A pull request to the `main` branch automatically deploys the new image to the staging environment in Kubernetes for further testing.
    *   **Production**: After manual approval and verification, merging to the `main` branch (or creating a release tag) triggers a rolling deployment to the production Kubernetes cluster.

This combined strategy ensures our system is robust, scalable, and that we can deploy new features and fixes to our users rapidly and reliably.

## 9. Reliability and Failure Handling

*   **Idempotency**: `POST` endpoints that create resources should be idempotent. The client can send a unique `Idempotency-Key` in the header. The server will store this key and ensure the operation is only performed once.
*   **Retries**: For transient failures in inter-service communication, a retry mechanism with exponential backoff should be implemented.
*   **Dead-Letter Queues (DLQs)**: If a message from the queue fails to be processed after several retries (e.g., a corrupted file that cannot be processed), it will be moved to a DLQ for manual inspection.
*   **Backup and Restore**: PostgreSQL will have regular backups. S3 has built-in versioning and can be configured for cross-region replication for disaster recovery.

## 10. Observability and Monitoring

*   **Metrics**: Use Prometheus to collect key metrics from services:
    *   **Ledger Service**: Transaction throughput, error rate, latency.
    *   **File Service**: Upload/download rate, processing time, error rate.
*   **Logging**: Centralized logging (e.g., using the ELK stack) for all services.
*   **Tracing**: Distributed tracing (e.g., using Jaeger or OpenTelemetry) to trace requests as they flow through the microservices.
*   **Alerting**: Configure alerts in Grafana or Alertmanager for critical issues (e.g., high error rates, high latency, queue depth increasing).

## 11. Edge Cases

*   **Partial Failures**: The use of database transactions in the `Ledger Service` handles partial failures during the creation of a financial event. If any step fails, the entire transaction is rolled back.
*   **Invalid File Formats**: The `File Service` will reject files that do not match the allowed types or fail validation.
*   **Corrupted Uploads**: If a file uploaded to S3 is found to be corrupt during a processing, its status will be marked as `FAILED` in the metadata database, and the user can be notified to try again.

## 12. Trade-offs and Alternatives Considered

Every system design involves trade-offs. Here are some key decisions and the reasoning behind them.

*   **Monolith vs. Microservices**:
    *   **Choice**: Microservices.
    *   **Reasoning**: The two core functions (Ledger and Files) have very different requirements. The Ledger demands consistency and ACID transactions, while the File service demands high throughput and scalability for large binary objects. Separating them allows us to choose the right tools and scale each component independently.
    *   **Trade-off**: We accept higher operational complexity (more services to deploy, monitor, and manage) in exchange for scalability and separation of concerns.

*   **SQL vs. NoSQL for the Ledger**:
    *   **Choice**: SQL (PostgreSQL).
    *   **Reasoning**: Financial data requires strict consistency and atomicity. Relational databases with ACID transaction support are the industry standard and the safest choice for this.
    *   **Alternative**: Using a NoSQL database like DynamoDB would require implementing complex transactional logic and rollback procedures in the application layer, which is error-prone and adds significant risk to the core business logic.

*   **Message Queue: Kafka vs. RabbitMQ**:
    *   **Choice**: Kafka.
    *   **Reasoning**: Given the scale (~50 million events per month from uploads alone), Kafka's high-throughput, persistent, and horizontally scalable nature is a better fit. Its ability to replay messages is also excellent for reliability and debugging.
    *   **Alternative**: RabbitMQ is an excellent general-purpose message broker, but Kafka's partitioned log architecture is generally superior for high-volume event streaming scenarios.

*   **Direct Upload vs. Pre-signed URL Pattern**:
    *   **Choice**: Pre-signed URLs.
    *   **Reasoning**: This pattern offloads the bandwidth and compute load of handling file streams from our backend services directly to a massively scalable service (S3). It prevents our `File Service` from becoming a bottleneck.
    *   **Trade-off**: It adds a bit of complexity to the client-side logic, which must now perform a two-step process (get URL, then PUT to URL). This is a very common and worthwhile trade-off.

## 13. End-to-End User Flow: Creating a Transaction with a Receipt

This section describes a typical user flow: a user records a business expense of $50 for "Office Supplies" and attaches a photo of the receipt.

Here is a sequence diagram illustrating the complete flow:

```mermaid
sequenceDiagram
    participant Client as User's App
    participant APIGW as API Gateway
    participant IAM as IAM Service
    participant FileSvc as File Service
    participant LedgerSvc as Ledger Service
    participant S3
    participant MsgQueue as Message Queue

    Note over Client, LedgerSvc: The user initiates recording an expense.

    Client->>APIGW: 1. POST /files/upload-url (filename: receipt.jpg)
    APIGW->>FileSvc: 2. Forward request (with auth token)
    FileSvc->>FileSvc: 3. Create metadata (file_id, status: PENDING)
    FileSvc->>S3: 4. Generate pre-signed URL for upload
    S3-->>FileSvc: 5. Pre-signed URL
    FileSvc-->>APIGW: 6. Return {file_id, upload_url}
    APIGW-->>Client: 7. Forward response to client

    Client->>S3: 8. PUT receipt.jpg to pre-signed URL
    S3-->>MsgQueue: 9. S3 Event: Object Created

    Note over Client, LedgerSvc: While the file uploads, the user fills out transaction details.

    Client->>APIGW: 10. POST /transactions (desc: "Office Supplies", entries: [...], file_ids: ["from step 7"])
    APIGW->>LedgerSvc: 11. Forward request (with auth token)
    LedgerSvc->>LedgerSvc: 12. Begin DB Transaction
    LedgerSvc->>LedgerSvc: 13. Validate debits == credits
    LedgerSvc->>LedgerSvc: 14. Create `transactions` record
    LedgerSvc->>LedgerSvc: 15. Create `entries` & `transaction_files` records
    LedgerSvc->>LedgerSvc: 16. Update `accounts` balances
    LedgerSvc->>LedgerSvc: 17. Commit DB Transaction & Publish `transaction.created` event
    LedgerSvc-->>APIGW: 18. Return {transaction_id}
    APIGW-->>Client: 19. Forward response to client
    Client->>Client: 20. Show "Transaction successful" message

    MsgQueue-->>FileSvc: 21. Consume "Object Created" event
    FileSvc->>FileSvc: 22. Validate uploaded file (scan, check type)
    FileSvc->>FileSvc: 23. Update file metadata (status: COMPLETED)
    FileSvc->>MsgQueue: 24. Publish "File Processed" event (for notifications)
    
    Note over MsgQueue, FileSvc: The File Service also listens for transaction creation events to keep its data in sync.
    MsgQueue-->>FileSvc: 25. Consume "transaction.created" event
    FileSvc->>FileSvc: 26. Update file metadata in DynamoDB with the transaction_id
```
![End-to-End-User-Flow](img/UseCaseFlow.png)

**Detailed Steps:**

1.  **User Initiates Upload**: The user clicks "Attach Receipt" on the frontend. The client application makes a `POST` request to the `/files/upload-url` endpoint via the **API Gateway**. This request includes the filename and type.
2.  **Get Pre-signed URL**: The **API Gateway** authenticates the request (using a JWT validated perhaps against the **IAM Service**) and forwards it to the **File Service**.
3.  **File Service Prepares for Upload**: The **File Service** creates a new metadata record in DynamoDB with a `PENDING` status. It then requests a secure, time-limited pre-signed URL from **Amazon S3**.
4.  **Client Uploads File**: The **File Service** returns the pre-signed URL and a unique `file_id` to the client. The client application uses this URL to upload the receipt image directly to **S3**. This offloads the bandwidth-intensive work from our services.
5.  **User Creates Transaction**: While the file is uploading in the background, the user fills in the transaction details (e.g., "Office Supplies," $50). Once the upload is complete, the client has the `file_id`. The user submits the form.
6.  **Ledger Service Records Transaction**: The client sends a `POST` request to the `/transactions` endpoint. The request body now contains the description, the debit/credit entries, and an array of `file_ids` of the attached receipt(s). The **API Gateway** routes this to the **Ledger Service**.
7.  **Atomicity and Events**: The **Ledger Service** performs the entire operation within a single PostgreSQL database transaction. It validates that debits equal credits, creates the transaction record, creates the associated entries, updates account balances, and inserts records into the `transaction_files` table to link the file(s). If any step fails, the entire database transaction is rolled back. Upon a successful commit, it publishes a `transaction.created` event to the message queue.
8.  **Asynchronous File Processing**: When the file upload to **S3** completes (Step 4), S3 automatically sends an event notification to a **Message Queue** (like SQS or Kafka).
9.  **File Validation**: The **File Service** has a worker that consumes messages from this queue. Upon receiving the notification, it downloads the file from S3 (or accesses it), validates its content (e.g., runs a virus scan, confirms it's a valid image), and optionally generates thumbnails.
10. **Finalize File Status**: After successful processing, the **File Service** updates the file's metadata record in DynamoDB, changing the status from `PENDING` to `COMPLETED`.
11. **Asynchronous Link Update**: The **File Service** also consumes the `transaction.created` event from the message queue. When it receives this event, it updates the corresponding file metadata records in its own database (DynamoDB) to include the `transaction_id`, completing the link.
12. **Notifications (Optional)**: The **File Service** can publish another event (e.g., `file_processed`) to the message queue, which can be consumed by a **Notification Service** to inform the user that their receipt is successfully attached and processed.

This flow effectively decouples the file handling from the core financial transaction logic, improving scalability and user experience. The user gets a fast response for the transaction creation while heavy file processing happens asynchronously. 

**NOTE:** All diagrams were rendered using mermaid https://www.mermaidchart.com/. Render code here to confirm.
