# Durability in Memory-Based Stores

## 1. Introduction

In-memory databases store their primary data in RAM (volatile memory) to provide fast read and write operations. However, RAM loses its contents when the system crashes or loses power.

To provide durability, in-memory database systems maintain persistent data on disk using a combination of sequential log files and disk-based backups.

The primary objective is to ensure that committed operations are not lost while avoiding the performance overhead of synchronously updating the entire disk-based backup for every client request.

---

## 2. Why Is Durability Required?

* **RAM Volatility:** Fast data access, but power loss or a system crash wipes all memory.
* **Failure Survival:** Applications require committed transactions to reliably survive crashes.
* **Disk I/O Bottlenecks:** Persisting every database modification directly to disk snapshots introduces severe latency.
* **Balanced Architecture:** In-memory databases employ logging alongside periodic backups to balance ultra-high performance with strict durability.

---

## 3. Write-Ahead Logging (WAL)

Write-Ahead Logging is a durability mechanism where modifications are recorded sequentially on disk before the operation is deemed complete.

### Working
1. A client sends a write request to the database.
2. The database generates log records describing the modification.
3. These records are written to a sequential disk-based log file.
4. The log data is flushed to persistent storage.
5. The database applies the modification to in-memory data structures.
6. The operation is acknowledged to the client once required durability conditions are met.

### Example
Consider a transaction transferring ₹1,000 from Alice to Bob.

**Initial Balances:**

| Account | Balance |
| :--- | :--- |
| Alice | ₹5,000 |
| Bob | ₹3,000 |

**Corresponding WAL Records:**
* `LSN 101`: Alice balance = 4000
* `LSN 102`: Bob balance = 4000

> **Note:** **LSN (Log Sequence Number)** identifies the exact order and position of records in the log, guaranteeing state reconstruction even after catastrophic memory failure.

---

## 4. Disk-Based Backup

In-memory databases maintain persistent disk backups to eliminate the need to replay the entire log history during recovery. Backups are typically organized as sorted, disk-based structures.

### Characteristics
* **Primary Store:** Remains fully in RAM for sub-millisecond access.
* **Persistent Backup:** Stored safely on disk.
* **Asynchronous Updates:** Propagated to the backup asynchronously to prevent blocking write paths.
* **Batch Processing:** Merges multiple log records together to minimize disk I/O operations.
* **State Lag:** The backup snapshot may trail the primary memory state; pending changes reside in the WAL.

---

## 5. Checkpointing

Checkpointing periodically applies accumulated WAL records to the disk-based backup to produce an updated snapshot, preventing unbounded log growth and accelerating crash recovery.

### Working
1. The database continuously appends modifications to the WAL.
2. Log records accumulate while the disk-based backup remains static.
3. The checkpoint worker collects a batch of recent log records.
4. Modifications from the batch are applied to the disk-based backup.
5. Once flushed, the backup represents a consistent snapshot up to that specific point in time.
6. Obsolete WAL records prior to the checkpoint recovery point are safely pruned.

### Example

**Initial Disk-Based Backup:**

| Account | Balance |
| :--- | :--- |
| Alice | ₹5,000 |
| Bob | ₹3,000 |

**Accumulated WAL Records:**
* `LSN 101`: Alice balance = 4000
* `LSN 102`: Bob balance = 4000
* `LSN 103`: Alice balance = 3500

**After Checkpointing:**

| Account | Balance |
| :--- | :--- |
| Alice | ₹3,500 |
| Bob | ₹4,000 |

The backup now reflects all modifications through `LSN 103`. Older WAL records prior to this point can be safely discarded.

---

## 6. Recovery After a Crash

When an in-memory database crashes, volatile memory is lost. The system reconstructs its latest valid state using the persistent backup and subsequent WAL records.

### Scenario A: Crash Before Checkpointing
*Disk backup contains initial data; WAL holds subsequent uncheckpointed operations.*

[ Disk Backup ] ---> [ Replay WAL (LSN 101 to Latest) ] ---> [ State Restored ]


1. Load the latest valid disk-based backup.
2. Locate the first WAL record generated after the backup's snapshot point.
3. Replay log records sequentially.
4. Commit verified transactions and roll back incomplete operations.
5. Restore the database to its latest recoverable state.

### Scenario B: Crash After Checkpointing
*Disk backup is already updated through LSN 103.*

[ Disk Backup (LSN 103) ] ---> [ Replay WAL (Post-LSN 103 Only) ] ---> [ State Restored ]


1. Load the updated disk-based backup (already at `LSN 103`).
2. Identify and replay only the log records written after `LSN 103`.
3. Restore the database to its latest recoverable state in a fraction of the time.

---

## 7. Overall Architecture

The durability framework coordinates three core components:

* **In-Memory Database:** Manages volatile active state and serves read/write queries.
* **Write-Ahead Log (WAL):** Appends transactions sequentially to persistent storage for replay capabilities.
* **Disk-Based Backup:** Holds sorted snapshots, updated asynchronously via batch checkpointing.

### Data Flow

```text
             Client Request
                   │
                   ▼
          In-Memory Database
                   │
             ┌─────┴─────┐
             ▼           ▼
          WAL File   Checkpointing
          (Disk)         │
                         ▼
                 Disk-Based Backup
                 (Sorted Structure)
8. Advantages
Durability: Committed transactions reliably survive power outages and crashes.

Ultra-Low Latency: Client reads and writes execute against high-speed RAM.

Optimized I/O: Sequential log appends and batched checkpointing avoid random disk writes.

Fast Startup & Recovery: Checkpointing caps the volume of log replays needed upon boot.

Decoupled Workloads: Heavy backup writes never block interactive client requests.

9. Important Trade-offs
Disk Overhead: Dual persistence structures (WAL logs + disk snapshots) require additional disk capacity.

I/O Contention: Background checkpoint flushing competes with regular disk write bandwidth.

Recovery Bounds: Recovery speed depends directly on checkpoint frequency and the volume of trailing logs.

Snapshot Lag: Asynchronous backup updates mean the disk snapshot remains temporarily behind the memory state.

Coordination Complexity: Ensuring strict consistency between WAL sequence numbers, memory state, and snapshot boundaries requires robust crash-safe design.

10. Key Takeaways
In-memory databases prioritize RAM for low latency, relying on disk systems exclusively for durability.

WAL uses sequential disk appends to record every write before it is acknowledged.

Disk-based backups maintain persistent point-in-time snapshots of full database states.

Checkpointing applies batched log records to the backup, advancing the recovery baseline and enabling log truncation.

Crash recovery restores the latest backup first, then replays only the log entries created after that checkpoint.

11. Related Concepts
Write-Ahead Logging (WAL)

Checkpointing & Snapshotting

Crash Recovery & ARIES Protocol

Log-Structured Storage (LSM Trees)

Sequential vs. Random Disk I/O

ACID Properties (Durability in In-Memory Systems)