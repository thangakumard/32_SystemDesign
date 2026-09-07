# System Design: Real-Time Telemetry & Threat Ingestion Pipeline

**Target Audience:** Microsoft Senior & Staff Software Development Engineer (SDE / SDE III / Principal) Interview Preparation  
**Scope:** Enterprise-grade security analytics platform (SIEM/XDR class system) processing multi-modal telemetry streams with sub-second threat detection, distributed rule evaluation, anomaly detection, and long-term compliance storage.

---

## 1. Requirements & Scope

### Functional Requirements (FR)
1. **Multi-Modal Telemetry Ingestion:** Ingest logs, metrics, traces, and security events from heterogeneous sources (OS endpoints, cloud audit logs like Azure Activity/AWS CloudTrail, network flows, k8s audit logs, identity providers).
2. **Schema Normalization & Enrichment:** Normalize incoming unstructured/semi-structured data into a unified schema (e.g., OCSF - Open Cybersecurity Schema Framework) and enrich with threat intelligence (GeoIP, IP reputation, Asset DB, Active Directory metadata) in real-time.
3. **Low-Latency Detection Engine:** Continuous real-time stream processing for stateful threat rule matching (e.g., Sigma rules, sliding window velocity checks, correlation across multiple event types).
4. **Interactive Query & Security Analytics:** Provide hot path ad-hoc search and structured querying capability over historical telemetry (up to 90 days) with sub-second latency for SOC analysts.
5. **Tiered Cold Storage & Compliance Retention:** Immutable long-term storage (1–7 years) for compliance, supporting batch re-hydration and retrospective threat hunting.
6. **Automated Incident Alerting & SOAR Integration:** Trigger alerts, Webhooks, and automated Playbooks (e.g., Azure Logic Apps, automated host isolation) upon detection.

### Non-Functional Requirements (NFR)
1. **High Throughput & Scalability:** Ingest **1 Million Events Per Second (EPS)** at peak (~10 Gbps ingress throughput).
2. **Low Latency:**
   - Ingestion to Detection (E2E Latency): **< 2 seconds** (p99).
   - Hot Analytics Search Latency: **< 1 second** for 24-hour window queries.
3. **Availability & Fault Tolerance:** **99.99% availability** (max ~52 minutes downtime per year). Multi-region disaster recovery with zero data loss for persistent state.
4. **Data Durability & Reliability:** Zero message loss (**At-least-once delivery semantics** across pipeline; **Exactly-once processing** in the detection engine).
5. **Data Security & Isolation:** End-to-end encryption (TLS 1.3 in transit, AES-256 at rest), RBAC/ABAC at field level, tenant isolation (Multi-tenant architecture).
6. **Elasticity & Cost Efficiency:** Auto-scaling to handle 5x peak bursts without manual operator intervention; tiered storage to optimize cost per TB/month.

---

## 2. Capacity Estimation & Back-of-the-Envelope Calculation

### Traffic & Throughput Calculations
- **Average Event Rate:** $1,000,000 	ext{ EPS}$
- **Peak Event Rate (5x headroom):** $5,000,000 	ext{ EPS}$
- **Average Payload Size per Event:** $1 	ext{ KB}$ (raw + wrapped metadata)

#### Ingress Bandwidth:
$$	ext{Average Bandwidth} = 1,000,000 	ext{ events/sec} 	imes 1 	ext{ KB/event} = 1 	ext{ GB/sec} = 8 	ext{ Gbps}$$
$$	ext{Peak Bandwidth} = 5 	ext{ GB/sec} = 40 	ext{ Gbps}$$

#### Stream Processing Overhead (Normalized + Enriched Data Size):
- Schema expansion (OCSF fields + Threat Intel tags): Payload size increases to **$2.5 	ext{ KB/event}$**.
$$	ext{Enriched Ingress Throughput} = 1,000,000 	imes 2.5 	ext{ KB} = 2.5 	ext{ GB/sec} = 20 	ext{ Gbps}$$

---

### Storage Calculations

#### 1. Daily Ingestion Volume:
$$	ext{Raw Volume/Day} = 1 	ext{ GB/sec} 	imes 86,400 	ext{ sec/day} = 86.4 	ext{ TB/day}$$
$$	ext{Enriched Volume/Day} = 2.5 	ext{ GB/sec} 	imes 86,400 	ext{ sec/day} = 216 	ext{ TB/day}$$

#### 2. Hot Storage Layer (30 Days - Search / Indexing Tier):
- Compression Ratio (Columnar Indexing / Parquet / Lucene): ~**4:1**
$$	ext{Daily Indexed Storage} = rac{216 	ext{ TB/day}}{4} = 54 	ext{ TB/day}$$
$$	ext{30-Day Hot Store Capacity} = 54 	ext{ TB/day} 	imes 30 	ext{ days} = 1.62 	ext{ PB}$$
- Adding 30% overhead for indexes (Inverted Indexes, Bloom Filters):
$$	ext{Total Hot Storage Required} pprox 2.1 	ext{ PB}$$

#### 3. Cold Storage Layer (1 Year Compliance - Compressed Parquet / Blob Storage):
$$	ext{Daily Parquet Storage} = rac{216 	ext{ TB/day}}{5 	ext{ (Parquet Snappy compression)}} pprox 43.2 	ext{ TB/day}$$
$$	ext{1-Year Cold Storage Capacity} = 43.2 	ext{ TB/day} 	imes 365 	ext{ days} pprox 15.76 	ext{ PB}$$

---

### Compute & Stream Partitioning Strategy
- **Kafka / Event Hubs Partitioning:**
  - Standard partition throughput recommendation: $1 	ext{ MB/sec}$ write, $2 	ext{ MB/sec}$ read.
  - Required Partitions for Ingress ($1 	ext{ GB/sec}$):
    $$	ext{Partitions} = rac{1000 	ext{ MB/sec}}{1 	ext{ MB/sec/partition}} = 1,000 	ext{ partitions}$$
  - Provision **1,280 partitions** (nearest power-of-two margin) for partition balancing and burst headroom.

---

## 3. High-Level System Architecture

```
                                  +-------------------------------------------------------+
                                  |                 Ingress & Gateway                     |
                                  |  +-------------------+     +-----------------------+  |
[ Agents / Syslog / Cloud ] ----->|  | Layer 7 Anycast   | --> | API Gateway / Ingress |  |
                                  |  | Load Balancers    |     | Proxy (Envoy Cluster) |  |
                                  |  +-------------------+     +-----------------------+  |
                                  +-----------------------------------|-------------------+
                                                                      |
                                                                      v
                                  +-------------------------------------------------------+
                                  |               Distributed Buffer                      |
                                  |  +-------------------------------------------------+  |
                                  |  | Apache Kafka / Azure Event Hubs                 |  |
                                  |  | Topic: raw-telemetry-events (1280 Partitions)   |  |
                                  |  +-------------------------------------------------+  |
                                  +-----------------------------------|-------------------+
                                                                      |
                                                                      v
                                  +-------------------------------------------------------+
                                  |            Stream Processing Engine                   |
                                  |  +-------------------------------------------------+  |
                                  |  | Apache Flink / Spark Streaming Cluster          |  |
                                  |  | Stage 1: OCSF Parsing & Normalization           |  |
                                  |  | Stage 2: Enrichment (In-Memory Threat Intel)    |  |
                                  |  +-------------------------------------------------+  |
                                  +------------|------------------------------|-----------+
                                               |                              |
                          +--------------------+                              +--------------------+
                          | (Enriched Stream)                                                      | (Alerts / Matches)
                          v                                                                        v
+---------------------------------------------------+                    +---------------------------------------------------+
|               Hot Storage Tier                    |                    |            Stateful Stateful Detection            |
| +-----------------------------------------------+ |                    | +-----------------------------------------------+ |
| | Distributed Indexing Engine                   | |                    | | Flink CEP Engine / Rule Evaluation Core       | |
| | (Elasticsearch / OpenSearch / Azure Data Explorer)|                  | | Sliding Windows, Correlation Rules, ML Anomaly| |
| +-----------------------------------------------+ |                    | +-----------------------------------------------+ |
+-------------------------|-------------------------+                    +-------------------------|-------------------------+
                          |                                                                        |
                          v                                                                        v
+---------------------------------------------------+                    +---------------------------------------------------+
|            Cold Storage & Data Lake               |                    |           Alerting & Orchestration                |
| +-----------------------------------------------+ |                    | +-----------------------------------------------+ |
| | Delta Lake / Apache Hudi on Object Storage    | |                    | | Kafka Topic: security-alerts                  | |
| | (Azure Data Lake Storage Gen2 / AWS S3)       | |                    | | Notification Service / SOAR (Logic Apps)      | |
| +-----------------------------------------------+ |                    | +-----------------------------------------------+ |
+---------------------------------------------------+                    +---------------------------------------------------+
```

---

## 4. Subsystem Deep Dives & Data Flow

### 4.1 Ingress Layer & Gateway
- **Proxy Layer:** Envoy Proxy running as a Kubernetes DaemonSet, auto-scaled via HPA based on CPU/Network I/O.
- **Protocol Support:** gRPC (high-throughput agents), HTTPS (REST API), Syslog-over-TLS (legacy appliances), and mTLS for endpoint authentication.
- **Token Bucket Rate Limiting:** Enforces rate limits per tenant/source key using Redis Cluster to prevent noisy-neighbor starvation.
- **Edge Validation:** Drops malformed payloads (> 2 MB) instantly before queuing to protect internal buffer pools.

---

### 4.2 Buffering & Ingestion Pipeline (Apache Kafka / Azure Event Hubs)
- **Topic Layout:**
  - `telemetry.raw`: Partitioned by `TenantID + HostID` to guarantee sequential processing for individual host events.
  - `telemetry.enriched`: Output from Flink ingestion worker pipeline.
  - `telemetry.alerts`: High-priority alert bus for downstream SOAR integration.
- **Reliability Configurations:**
  - `acks=all` (Wait for full ISR replication before acknowledging).
  - `min.insync.replicas=2`, Replication Factor = 3 across Availability Zones (AZs).
  - Compaction enabled on configuration topics (`threat-intel-feed`).

---

### 4.3 Stateful Processing & Enrichment Engine (Apache Flink)
The stream processing layer executes a 3-stage pipeline:

```
[Raw Event] --> (1. Schema Normalization) --> (2. Async Threat Intel Lookups) --> (3. Stateful CEP Correlation)
```

1. **Schema Normalization:** Dynamic transformation mapping custom JSON/Syslog to **OCSF v1.1** standard.
2. **In-Memory Enrichment via Broadcast State:**
   - High-throughput threat intel feeds (IP blocklists, malicious hash tables) are loaded into Memory via Flink **Broadcast State Pattern**.
   - Low-latency cache (Redis Enterprise cluster) is queried asynchronously using Flink's `AsyncIO` API for large context data (e.g., Active Directory user privilege maps).
3. **Stateful Rule Engine (Complex Event Processing - CEP):**
   - Implements sliding window correlations (e.g., *Flag if > 5 failed SSH attempts followed by successful login within 3 minutes from the same IP*).
   - State stored in **RocksDB State Backend**, incrementally snapshotted to Object Storage (ADLS Gen2 / S3) every 10 seconds.

---

### 4.4 Storage Tiering Strategy

| Tier | Technology Stack | Retention | Use Case | Target Query Latency |
| :--- | :--- | :--- | :--- | :--- |
| **Hot Path** | Azure Data Explorer (ADX) / ClickHouse | 30 Days | Interactive Threat Hunting, SOC Dashboards | < 1 sec |
| **Warm Path** | OpenSearch / Elasticsearch Cluster | 30 - 90 Days | Complex Full-Text Search & Log Analytics | 1 - 5 sec |
| **Cold Path** | Delta Lake / Apache Iceberg on ADLS Gen2 | 1 - 7 Years | Compliance, Historical Batch Re-hydration | Minutes to Hours |

---

## 5. API Definitions

### 5.1 Ingestion API (gRPC Protocol Buffer)

```protobuf
syntax = "proto3";

package tech.threat.ingest.v1;

import "google/protobuf/timestamp.proto";

service TelemetryIngestionService {
  rpc StreamTelemetry (stream TelemetryBatchRequest) returns (TelemetryBatchResponse);
}

message TelemetryBatchRequest {
  string tenant_id = 1;
  string source_agent_id = 2;
  repeated TelemetryEvent events = 3;
}

message TelemetryEvent {
  string event_id = 1;
  google.protobuf.Timestamp timestamp = 2;
  string event_type = 3; // e.g., PROCESS_CREATE, NETWORK_CONN
  bytes raw_payload = 4;
  map<string, string> metadata = 5;
}

message TelemetryBatchResponse {
  int32 processed_count = 1;
  int32 error_count = 2;
  repeated string error_messages = 3;
}
```

---

### 5.2 Threat Rule Engine Management API (REST)

#### `POST /api/v1/rules` - Register Dynamic Detection Rule

```json
// Request Body
{
  "rule_id": "rule_bruteforce_001",
  "name": "Distributed SSH Brute Force Detected",
  "severity": "CRITICAL",
  "ocsf_category": "Authentication",
  "condition": {
    "window_seconds": 180,
    "group_by": ["src_endpoint.ip"],
    "threshold": 5,
    "sequence": [
      {"event_type": "AUTH_FAILURE"},
      {"event_type": "AUTH_SUCCESS"}
    ]
  },
  "actions": [
    {"type": "RAISE_ALERT"},
    {"type": "SOAR_WEBHOOK", "target": "https://soar.internal/hooks/isolate-host"}
  ]
}
```

---

## 6. Detailed Component Design & Trade-Offs

### 6.1 State Management in Stream Processing
- **Problem:** Processing 1M EPS requires fast, reliable local state access for rules spanning across long time windows without running out of RAM.
- **Solution:** Flink with **RocksDB Key-Value State Backend**.
  - Local state resides on NVMe SSDs mapped to Kubernetes Worker Nodes.
  - Periodic asynchronous incremental checkpointing to Blob Storage guarantees **Exactly-Once semantics**.
  - Dual-write buffer cache ensures reads are hit in RAM while background thread flushes cold state to disk.

```
       +-------------------------------------------------------+
       |                  Flink TaskManager                    |
       |  +-----------------+      +------------------------+  |
       |  |  RAM Heap State | ---> | RocksDB (Local NVMe)   |  |
       |  +-----------------+      +------------------------+  |
       +---------------------------------------|---------------+
                                               | (Async Checkpoint)
                                               v
                                 +---------------------------+
                                 | Cloud Object Storage      |
                                 | (ADLS Gen2 / AWS S3)      |
                                 +---------------------------+
```

---

### 6.2 Data Partitioning & Out-of-Order Handling
- **Late-Arriving Events:** Telemetry events from mobile endpoints can arrive hours late due to network dropouts.
- **Watermarking Strategy:**
  - Flink uses **Bounded-OutOfOrderness Watermarks** with a allowed lateness threshold of **$T_{	ext{late}} = 15 	ext{ seconds}$**.
  - Events within $T \le 	ext{Watermark}$ are processed in real-time windowing.
  - Events arriving past the lateness threshold bypass the real-time CEP window and are routed directly to the Warm/Cold Storage path for historical indexing and retrospective scanning.

---

### 6.3 Technical Trade-Offs & Architectural Decisions

#### 1. Message Broker Selection: Kafka vs. Pulsar
- **Decision:** Apache Kafka (or Azure Event Hubs Kafka Surface).
- **Trade-off:** Pulsar offers better tiering capabilities, but Kafka provides industry-standard operational maturity, lower engine tail-latencies at massive throughput, and seamless native integration with Flink connectors.

#### 2. Hot Storage Engine: Elasticsearch vs. Azure Data Explorer (ADX) / ClickHouse
- **Decision:** Azure Data Explorer / ClickHouse columnar store for Hot Path.
- **Trade-off:** Elasticsearch's inverted index is memory-intensive and expensive at PB scale. ADX/ClickHouse columnar engines compress log data significantly better (up to 5x-8x) while allowing ultra-fast aggregate/group-by queries required for security analytics.

---

## 7. Fault Tolerance, High Availability & Disaster Recovery

```
                             [ Primary Region: East US ]
                                          |
                        +-----------------+-----------------+
                        |                                   |
                        v                                   v
             +--------------------+              +--------------------+
             | Kafka Cluster A    |              | Flink Cluster A    |
             | (State Snapshots)  |              | (Active Processor) |
             +---------|----------+              +---------|----------+
                       |                                   |
=======================|===================================|=======================
                       | Cross-Region Async Replication    |
=======================|===================================|=======================
                       v                                   v
             +--------------------+              +--------------------+
             | Kafka Cluster B    |              | Flink Cluster B    |
             | (Hot Standby)      |              | (Passive / Standby)|
             +--------------------+              +--------------------+
                             [ Secondary Region: West US ]
```

1. **Cross-Region Active-Passive Setup:**
   - Primary Region handles 100% ingestion and active Flink job streams.
   - Kafka MirrorMaker 2 continuously replicates topics across geographic regions asynchronously.
   - Secondary region hosts a warm Flink cluster ready to pick up state snapshot pointers in under **2 minutes (RTO < 2 mins, RPO < 10 secs)**.
2. **Backpressure Handling:**
   - Envoy Gateway implements dynamic client throttling if Kafka ingress buffer topic utilization exceeds 80%.
   - Flink's credit-based flow control prevents TaskManager memory overflow during downstream storage performance degradation.

---

## 8. Security, Compliance & Multi-Tenancy

1. **Tenant Isolation:**
   - Hard logical isolation enforced at data plane via `TenantID` header validation.
   - Kafka topics partitioned with tenant-hashed keys; ADX / Delta Lake schemas segregated by tenant namespaces.
2. **Field-Level Encryption (KMS Integration):**
   - PII and sensitive payload fields (e.g., Raw Command Lines, User Credentials in Logs) are encrypted using tenant-managed Keys (Azure Key Vault / AWS KMS) prior to writing to persistent storage.
3. **Data Integrity & Immutability:**
   - Cold Storage delta tables configured with **WORM (Write Once, Read Many)** immutability policies to satisfy compliance standards (PCI-DSS 4.0, SOC 2 Type II, ISO 27001).

---

## 9. Observability & Monitoring Strategy

- **Key Metrics To Track:**
  - **Ingestion Latency:** Time delta between `event.timestamp` (generated) and `ingest.timestamp` (proxy received).
  - **Flink Consumer Group Lag:** Unprocessed record backlog per Kafka partition.
  - **RocksDB Read/Write Amplification & SST File Count:** Detect local disk bottlenecking in stream processors.
  - **Hot Tier Query P99 Latency:** Track dashboard responsiveness under concurrent security analyst workload.
- **Alerting Thresholds:**
  - Trigger PagerDuty if Kafka consumer lag growth rate > 10% over 5 minutes.
  - Trigger critical alert if end-to-end alert pipeline delay exceeds 5 seconds.

---

## Interview Presentation Strategy Tip (Microsoft Senior/Staff SDE)

When presenting this architecture in your interview:
1. **Start with Strategy (2-3 mins):** Anchor on requirements (1M EPS, <2s detection) and establish clear trade-offs early (e.g., Columnar storage over traditional Lucene indexing for cost at PB scale).
2. **Drive the Discussion Layer by Layer:** Use clear block diagrams for Ingress -> Buffer -> Processing -> Storage -> Alerting.
3. **Show Deep Technical Mastery (Staff level requirement):** Elaborate on Flink Broadcast State Pattern for Threat Intel joins, RocksDB memory management, and Watermarking strategies for late-arriving events.
