# Amazon DynamoDB: The Ultimate System Design Interview Guide
> *Mastering DynamoDB data modeling, indexing, architecture, advanced features, and interview decision frameworks.*

---

## Table of Contents
1. [Introduction](#1-introduction)
   - *Overview and objective*
2. [Data Model Fundamentals](#2-data-model-fundamentals)
   - *Tables, Items, and Attributes*
   - *Schema Flexibility: Pros and Cons*
   - *Primary Keys: Partition Key vs. Sort Key*
   - *Sample Schema Design by Domain*
3. [Indexing Strategies: GSI vs. LSI](#3-indexing-strategies-gsi-vs-lsi)
   - *Global Secondary Indexes (GSI)*
   - *Local Secondary Indexes (LSI)*
   - *Choosing the Right Indexing Strategy*
4. [Data Access and Querying](#4-data-access-and-querying)
   - *Creation and Basic Queries*
   - *SDK Query Syntax vs. SQL*
   - *Atomic Transactions (`TransactWriteItems`)*
5. [Under the Hood: Architecture & Mechanics](#5-under-the-hood-architecture--mechanics)
   - *Consistent Hashing & Data Partitioning*
   - *Replication & Fault Tolerance (Sloppy Quorum)*
   - *Read Consistency: Eventual vs. Strong Consistency*
6. [Advanced Features](#6-advanced-features)
   - *DynamoDB Accelerator (DAX)*
   - *DynamoDB Streams (Change Data Capture)*
7. [System Design Interview Playbook](#7-system-design-interview-playbook)
   - *When to Choose DynamoDB*
   - *Red Flags & Anti-Patterns (When to Avoid)*
   - *Summary & Next Steps*

---

## 1. Introduction

Amazon DynamoDB is a fully managed, serverless, NoSQL database designed for single-digit millisecond latency at any scale. In a system design interview, demonstrating a clear understanding of **how** and **when** to leverage DynamoDB—as well as recognizing its trade-offs compared to relational databases like PostgreSQL—can significantly elevate your evaluation.

This guide breaks down DynamoDB from basic concepts to advanced architecture patterns, specifically tailored for candidate performance in technical interviews.

---

## 2. Data Model Fundamentals

### Tables, Items, and Attributes

DynamoDB organizes data into three core primitives:

* **Table:** A collection of related data records (analogous to a SQL table).
* **Item:** A single record within a table, uniquely identified by a primary key (analogous to a row in SQL).
* **Attribute:** A fundamental data element within an item (analogous to a column/field in SQL).

```
+-------------------------------------------------------------------------------+
| TABLE: Users                                                                  |
+-------------------+-----------------+---------------+-------------------------+
| Partition Key     | Sort Key (Opt.) | Attribute     | Attribute (Schemaless)  |
| (user_id)         | (created_at)    | (email)       | (employer)              |
+-------------------+-----------------+---------------+-------------------------+
| USR#101           | 1700000000      | alex@test.com | Acme Corp               |  <-- Item 1
| USR#102           | 1700000500      | sara@test.com | [undefined]             |  <-- Item 2
+-------------------+-----------------+---------------+-------------------------+
```

### Schema Flexibility: Pros and Cons

DynamoDB is **schemaless** (except for the primary key attributes). Different items in the same table can possess entirely different attributes.

* **Pros:**
  * **Agility:** Fast schema iterations without complex SQL migrations (`ALTER TABLE`).
  * **Cost Efficiency:** You only pay for data stored. No wasted storage for `NULL` values.
* **Cons:**
  * **Application Overhead:** The application layer must handle missing attributes gracefully.
  * **Cross-Item Analytics:** Difficult to run ad-hoc analytics or aggregation queries directly on the database.

---

### Primary Keys: Partition Key vs. Sort Key

DynamoDB supports two types of primary keys:

1. **Simple Primary Key (Partition Key Only):**
   * A single attribute, hashed using consistent hashing to determine the physical partition where data resides.
2. **Composite Primary Key (Partition Key + Sort Key):**
   * **Partition Key (PK):** Determines the physical partition (node).
   * **Sort Key (SK):** An optional secondary attribute that orders items with the same partition key using an in-memory B-Tree index. Enables efficient range queries (`between`, `>`, `<`, `begins_with`).

> **Important:** The combination of `Partition Key + Sort Key` must be globally unique across the entire table.

---

### Sample Schema Design by Domain

#### 1. Sample User Table
| User ID (PK) | Name | Email | Employer |
| :--- | :--- | :--- | :--- |
| `USR_101` | Sarah | sarah@example.com | TechCorp |
| `USR_102` | Jimmy | jimmy@example.com | Acme Inc |
| `USR_103` | Stefan | stefan@example.com | *[N/A]* |

#### 2. Chat Application Data Model
To collocate all messages for a specific chat group onto the same physical node, use `chat_id` as the partition key.

* **Partition Key (PK):** `chat_id`
* **Sort Key (SK):** `message_id` (Monotonically Increasing ID)

| Chat ID (PK) | Message ID (SK) | Sender ID | Content | Created At |
| :--- | :--- | :--- | :--- | :--- |
| `CHAT_9001` | `MSG_000001` | `USR_101` | Hello world! | 2026-03-30T10:00:00Z |
| `CHAT_9001` | `MSG_000002` | `USR_102` | Hey Sarah! | 2026-03-30T10:00:05Z |

*Why use a monotonically increasing ID instead of a timestamp string for the Sort Key?*
If two messages are generated in the exact same millisecond, a raw timestamp collision would cause a write failure due to primary key constraint violations. Monotonically increasing IDs preserve temporal ordering while guaranteeing uniqueness.

#### 3. E-Commerce & Social Media Patterns
* **E-Commerce Orders:** 
  * `PK`: `user_id` | `SK`: `order_id` (Monotonically increasing)
  * *Access Pattern:* Fetch all orders for a specific user ordered by time.
* **Social Media Posts:** 
  * `PK`: `user_id` | `SK`: `post_id`
  * *Access Pattern:* Fetch user timeline ordered chronologically.

---

## 3. Indexing Strategies: GSI vs. LSI

When query access patterns extend beyond the primary key structure, secondary indexes are required.

```
+-----------------------------------------------------------------------------------+
| SECONDARY INDEX TYPES                                                             |
+------------------------------------+----------------------------------------------+
| Global Secondary Index (GSI)       | Local Secondary Index (LSI)                  |
+------------------------------------+----------------------------------------------+
| • New Partition Key (PK)           | • SAME Partition Key as base table           |
| • Optional New Sort Key (SK)       | • NEW Sort Key (SK)                          |
| • Can be created/deleted anytime   | • Must be created at table creation time     |
| • Asynchronously replicated        | • Strongly consistent option available       |
+------------------------------------+----------------------------------------------+
```

### Global Secondary Index (GSI)
Used when you need to query by an attribute **other than the main table's partition key**.

* **Use Case Example (Chat App):** Finding all messages sent by a specific user across all chat rooms.
* **Base Table PK:** `chat_id`
* **GSI PK:** `user_id`

*GSI Projection Table:*
| User ID (GSI PK) | Message ID | Chat ID | Content |
| :--- | :--- | :--- | :--- |
| `USR_101` | `MSG_000001` | `CHAT_9001` | Hello world! |

### Local Secondary Index (LSI)
Used when you keep the **same partition key** but want to sort by a **different attribute**.

* **Use Case Example (Slack Attachment Finder):** Querying messages within a chat room sorted by the number of attachments rather than creation time.
* **Base Table PK:** `chat_id` | **Base Table SK:** `message_id`
* **LSI PK:** `chat_id` | **LSI SK:** `attachment_count`

---

## 4. Data Access and Querying

### Creating a Table & Provisioning
Tables are defined using the AWS Console, AWS CLI, or Infrastructure-as-Code (Terraform/CloudFormation). You define the Primary Key schema, optional LSIs/GSIs, and capacity mode (On-Demand or Provisioned WCU/RCU).

---

### SDK Query Syntax vs. SQL

#### 1. Full Table Scanning / Selection
* **SQL:**
  ```sql
  SELECT * FROM users;
  ```
* **DynamoDB SDK (Node.js/JavaScript):**
  ```javascript
  const params = {
    TableName: "Users"
  };
  await docClient.scan(params).promise();
  ```

#### 2. Querying by Partition Key
* **SQL:**
  ```sql
  SELECT * FROM users WHERE user_id = '101';
  ```
* **DynamoDB SDK:**
  ```javascript
  const params = {
    TableName: "Users",
    KeyConditionExpression: "user_id = :id",
    ExpressionAttributeValues: {
      ":id": "101"
    }
  };
  await docClient.query(params).promise();
  ```

#### 3. Sorting Query Results
* **SQL:**
  ```sql
  SELECT * FROM orders 
  WHERE user_id = '101' 
  ORDER BY order_date DESC;
  ```
* **DynamoDB SDK:**
  ```javascript
  const params = {
    TableName: "Orders",
    KeyConditionExpression: "user_id = :id",
    ExpressionAttributeValues: { ":id": "101" },
    ScanIndexForward: false // false = DESC, true = ASC
  };
  await docClient.query(params).promise();
  ```

---

### Atomic Transactions (`TransactWriteItems`)

DynamoDB supports ACID transactions across multiple items and tables using `TransactWriteItems` and `TransactGetItems`.

#### Banking Transfer Example ($100 transfer between accounts)
* **SQL:**
  ```sql
  BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
  COMMIT;
  ```
* **DynamoDB SDK:**
  ```javascript
  const params = {
    TransactItems: [
      {
        Update: {
          TableName: "Accounts",
          Key: { account_id: "A" },
          UpdateExpression: "SET balance = balance - :val",
          ExpressionAttributeValues: { ":val": 100 }
        }
      },
      {
        Update: {
          TableName: "Accounts",
          Key: { account_id: "B" },
          UpdateExpression: "SET balance = balance + :val",
          ExpressionAttributeValues: { ":val": 100 }
        }
      }
    ]
  };
  await docClient.transactWrite(params).promise();
  ```

---

## 5. Under the Hood: Architecture & Mechanics

### Consistent Hashing & Data Partitioning

DynamoDB partitions data automatically across storage nodes using **consistent hashing**.

1. The Partition Key string is passed into an internal hash function (e.g., MD5/SHA-256).
2. The output hash value maps onto a logical ring ($0$ to $2^{128}-1$).
3. The system traverses the ring clockwise to locate the physical storage node responsible for that hash range.

```
                      Node A (0-25)
                     /             \
         Node D (76-99)           Node B (26-50)
                     \             /
                      Node C (51-75)

  Hash(PK "CHAT_9001") = 28  --->  Routed to Node B
```

* **Scalability Benefit:** Adding or removing storage nodes only impacts adjacent neighbors on the ring, minimizing data re-distribution overhead.

---

### Replication & Read Consistency

#### Asynchronous Replication & Sloppy Quorum
To ensure high availability across AWS Availability Zones (AZs), writes are routed to a primary leader node and asynchronously propagated to peer replicas. DynamoDB utilizes a **quorum protocol** for write acceptance, ensuring write operations remain functional even during temporary network partitions.

```
                   +-------------------+
                   | Client Write Request |
                   +---------+---------+
                             |
                             v
                   +-------------------+
                   |    Leader Node    |
                   +----+---------+----+
                        |         |
          Async Rep     |         | Async Rep
         (AZ 1)         v         v (AZ 2)
              +-----------+     +-----------+
              | Replica 1 |     | Replica 2 |
              +-----------+     +-----------+
```

#### Read Consistency Options
* **Eventually Consistent Reads (Default):** Reads can hit any replica node. May reflect millisecond replication lag, but maximizes read throughput and reduces cost (1/2 RCU cost).
* **Strongly Consistent Reads:** Reads are forced directly from the leader node or a synchronized quorum. Guarantees fresh data at the cost of higher latency and double RCU cost.

> **Interview Tip:** In CAP theorem discussions, state that DynamoDB can be configured for either High Availability (AP) or Strong Consistency (CP) depending on your read consistency settings.

---

## 6. Advanced Features

### DynamoDB Accelerator (DAX)

**DAX** is a fully managed, clustered, in-memory caching service built specifically for DynamoDB.

* **Response Latency:** Drops read latency from single-digit milliseconds to **microseconds**.
* **Caching Strategy:** Automatic write-through caching.
* **Interview Application:** Use DAX instead of introducing Redis/Memcached when working with read-heavy DynamoDB access patterns (e.g., caching popular business metrics, high-volume product catalogs).

```
[ Application Service ] ---> [ DAX Cache ] ---> [ DynamoDB Table ]
                               (Microsec)           (Millisec)
```

---

### DynamoDB Streams (Change Data Capture)

**DynamoDB Streams** captures a time-ordered sequence of item-level modifications (inserts, updates, deletes) in real-time.

#### System Design Example: Event Search Sync (TicketMaster)
When an event record is updated in DynamoDB, sync it to an ElasticSearch cluster for geospatial/full-text search.

```
+---------------+     Write     +------------------+
| Event Service |-------------->| Event DynamoDB   |
+---------------+               +--------+---------+
                                         |
                                         | Stream Event
                                         v
                                +------------------+
                                | DynamoDB Streams |
                                +--------+---------+
                                         |
                                         v
                                 +---------------+
                                 | AWS Lambda    |
                                 +-------+-------+
                                         |
                                         | Index Write
                                         v
                                +------------------+
                                | ElasticSearch    |
                                +------------------+
```

---

## 7. System Design Interview Playbook

### When to Choose DynamoDB

* **Known Access Patterns:** Access patterns are key-value based or predictable.
* **Massive Scale & Latency Requirements:** High-throughput read/write workloads requiring consistent single-digit millisecond responses.
* **Zero Operational Overhead:** Serverless architectures requiring auto-scaling storage and throughput.

---

### Red Flags & Anti-Patterns (When to Avoid)

1. **Complex Ad-Hoc Queries & Joins:** Highly relational schemas requiring deep subqueries or cross-table joins.
2. **Multi-Item Transaction Limits:** Systems requiring large-scale relational transactions spanning hundreds of records (DynamoDB transaction limits apply).
3. **Unbounded Secondary Indexes:** Tables requiring excessive GSIs/LSIs to support endless access patterns—this typically indicates a need for a SQL data model.

---

### Summary Matrix: DynamoDB vs. SQL (PostgreSQL)

| Feature | Amazon DynamoDB | PostgreSQL |
| :--- | :--- | :--- |
| **Data Structure** | Key-Value / Document | Relational Tables |
| **Schema** | Dynamic / Schemaless | Rigid / Predefined |
| **Scaling** | Horizontal (Automatic) | Vertical (Horizontal via Read Replicas/Sharding) |
| **Joins** | Application-level or GSI modeling | Native SQL Joins |
| **Transactions** | Up to 100 items per transaction | Native ACID Transactions |
| **Caching Layer** | Native DAX Integration | Redis / Memcached |
| **CDC Integration** | DynamoDB Streams | Logical Replication / Debezium |
