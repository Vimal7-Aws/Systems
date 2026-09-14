**Database sharding** is a horizontal partitioning technique that splits a large dataset across multiple independent database servers (shards). Unlike vertical scaling (adding CPU/RAM to a single machine), sharding distributes compute and storage so that write/read throughput scales linearly.

---

### What is Database Sharding?

Sharding breaks a large database table horizontally by row into smaller chunks. Each shard holds a subset of the total rows, sharing the exact same schema.

* **Shard Key:** A specific attribute (like `user_id` or `tenant_id`) evaluated by a routing function (hash-based or range-based) to determine which physical shard holds a given record.
* **Benefits:** Prevents single-node storage limits and eliminates compute bottlenecks for high IOPS workloads.
* **Trade-offs:** Makes cross-shard transactions, global `JOIN` operations, and secondary index maintenance complex and expensive.

---

### How Sharding Works in Amazon DynamoDB

DynamoDB is natively **auto-sharded and fully managed**. You do not manage shard instances directly; DynamoDB abstracts them as **Physical Partitions**.

* **Partition Key Hashing:** Every item requires a **Partition Key** (Hash Key). DynamoDB passes this key through an internal MD5-based hash function to determine the target physical partition.
* **Automatic Scaling Trigger:** DynamoDB provisions a new partition whenever:
1. Total stored data exceeds **10 GB**.
2. Provisioned throughput exceeds **1,000 WCU** (Write Capacity Units) or **3,000 RCU** (Read Capacity Units) on a single partition.


* **Partition Splitting:** When a partition reaches these limits, DynamoDB automatically splits the key range, assigns half the data to a new partition, and updates its internal request router seamlessly without downtime.
* **Hot Partition Risk:** Because routing relies on hash distribution, poorly chosen partition keys (e.g., a low-cardinality status field or a timestamp) cause "hot partitions," starving a single shard while others sit idle.

---

### How "Sharding" Works in Amazon Aurora RDS

Amazon Aurora (PostgreSQL/MySQL compatible) approaches horizontal scaling completely differently. **Standard Aurora RDS does NOT shard data by default**; instead, it decouples compute from storage.

#### 1. Standard Aurora: Decoupled Storage Layer (Not Compute Sharding)

Standard Aurora uses a single database instance (or primary writer + read replicas) attached to a cluster volume.

* **10 GB Protection Segments:** Aurora automatically shards the underlying **storage volume** into 10 GB chunks distributed across 6 copies in 3 Availability Zones (AZs).
* **Shared Storage, Single Writer:** Compute instances share this underlying distributed storage layer. Query processing and compute execution still happen on a single primary database engine node—meaning standard Aurora is **not horizontally sharded for compute writes**.

#### 2. Scaling Write Capacity in Aurora

To shard writes horizontally in Aurora, you must use specific architecture patterns:

* **Aurora Limitless Database:** AWS introduced automated distributed query processing for Aurora PostgreSQL. It automatically shards data and distributes transaction processing across multiple database compute nodes while maintaining a unified PostgreSQL endpoint.
* **Application-Level Sharding / RDS Proxy:** For traditional Aurora, developers must manage application-side sharding (e.g., routing traffic to distinct Aurora clusters via middleware like Vitess or custom routing layers based on `tenant_id`).

---

### DynamoDB vs. Aurora Architecture

| Feature | Amazon DynamoDB | Amazon Aurora (Standard) |
| --- | --- | --- |
| **Sharding Type** | Native, fully automated compute & storage sharding. | Storage-level sharding (10 GB blocks across AZs); single compute writer. |
| **Routing Mechanism** | Internal request router hashing the Partition Key. | Standard PostgreSQL/MySQL connection endpoint to primary writer or read replicas. |
| **Scaling Unit** | Automatically provisions partitions on data size (>10GB) or IOPS bounds. | Scales compute instance vertically; scales storage dynamically up to 128 TiB. |
| **Horizontal Write Scaling** | Built-in out of the box. | Requires **Aurora Limitless Database** or application-level multi-cluster sharding. |