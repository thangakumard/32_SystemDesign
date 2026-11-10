# Storage Architecture: HDDs vs. SSDs & System Design Interview Guide

## 1. Scenario & Answer

**Question:**  
*Your database is well indexed and lightly loaded, but each query is still slow because it waits on random seeks on spinning HDDs. Which change lowers the latency of each individual query?*

**Direct Answer:**  
Move the database to **SSD (Solid-State Drive)** storage.

---

## 2. Deep Dive: HDDs vs. SSDs

### Hard Disk Drives (HDDs)

* **Mechanics:** HDDs rely on spinning magnetic platters and a mechanical read/write head on an actuator arm.
* **The Bottleneck:** When a database executes a query that needs data not currently cached in RAM, it performs disk I/O. For random seeks, the drive must physically position the head over the correct track (**seek time**) and wait for the platter to rotate to the correct sector (**rotational latency**).
* **Latency:** A single random seek on an HDD takes **5 to 15 milliseconds**. Because a single query may require multiple random seeks across indices and data pages, query latency quickly accumulates.
* **Throughput:** Low Input/Output Operations Per Second (IOPS)—typically 100 to 200 IOPS for standard enterprise HDDs.

### Solid-State Drives (SSDs)

* **Mechanics:** SSDs use NAND-based flash memory with **no moving parts**. Data access is governed electronically by an onboard controller.
* **The Advantage:** Random seeks no longer incur mechanical penalties.
* **Latency:** Random read access takes **0.1 milliseconds or less** (sub-millisecond latency).
* **Throughput:** High IOPS capability—ranging from tens of thousands to over a million IOPS (with modern NVMe drives).

---

## 3. Interview Context: Read-Heavy vs. Write-Heavy Workloads

**Should you discuss this in an interview?**

**Yes.** Addressing workload characteristics demonstrates senior-level system design awareness and shows that you understand the trade-offs beyond a simple hardware swap.

### Read-Heavy Systems

* **Key Challenge:** Random read latency and disk throughput bottlenecks.
* **Solution Impact:** Moving to SSDs/NVMe offers an immediate drop in read latency. Beyond storage media changes, read-heavy optimizations include scaling out read replicas, increasing memory for buffer pool caching, and adding in-memory caches (e.g., Redis).

### Write-Heavy Systems

* **Key Challenge:** Sustained write throughput, disk endurance, and write amplification.
* **Solution Impact:** While SSDs improve write performance over HDDs, they introduce specific considerations:
  * **Write Amplification:** Rewriting data requires erasing entire blocks of flash memory first, which can degrade performance and shorten hardware lifespans over time.
  * **Architectural Mitigations:** In write-heavy environments, architectural changes are often prioritized alongside hardware upgrades. This includes using append-only Write-Ahead Logs (WAL), leveraging Log-Structured Merge-tree (LSM) storage engines (e.g., RocksDB, Cassandra) over standard B-Trees, and batching writes to convert random writes into sequential writes.

---

## 4. Sample Interview Response

> "The primary cause of latency here is the physical mechanical overhead of random I/O on spinning HDDs. Even with properly indexed tables and low CPU/memory load, if query execution requires fetching index pages or rows that aren't cached in RAM, the system triggers random disk seeks. On an HDD, each seek incurs a 5 to 15 millisecond delay due to physical seek time and rotational latency.
>
> Moving the storage layer to **SSDs (or NVMe drives)** directly solves this. Because flash memory accesses data electronically rather than mechanically, random read latency drops to sub-millisecond levels, and available IOPS increase by orders of magnitude.
>
> Viewing this through a broader architectural lens, this upgrade is particularly effective if our system is **read-heavy**, as random read latency is typically the primary bottleneck. If the workload were **write-heavy**, however, we would also need to account for SSD endurance, write amplification, and write amplification overhead—potentially combining storage upgrades with write-optimized patterns like LSM-tree engines or append-only logging to optimize sequential I/O."