# Write-Ahead Logs (WAL) in System Design & Kafka

## 1. What is a Write-Ahead Log (WAL)?
A **Write-Ahead Log (WAL)** is an **append-only log** used in databases, file systems, and distributed systems to guarantee **durability and recoverability**.

- Every change is written to the log **before** applying it to the main data store.
- If a crash happens, the system replays the WAL to restore consistency.

### Where WAL is used
- **Databases:** PostgreSQL, MySQL InnoDB, Oracle → ACID durability.
- **File systems:** ext4, NTFS, ZFS (journaling).
- **Distributed systems:** Kafka, HDFS, Raft.
- **Caching systems:** Redis AOF.

---

## 2. WAL in Kafka
Kafka is essentially a **distributed WAL**. Every message produced to a topic partition is:
1. Written to the **leader broker’s log** (append-only file).
2. Replicated to **follower brokers**.
3. Exposed to consumers **only after replication (High Watermark)**.

### Flow
1. Producer → Leader broker.
2. Leader appends record to **WAL (log segment)** via **page cache**.
3. Record replicated to ISR followers, appended to their WAL.
4. Once all ISR replicas confirm, leader advances **High Watermark (HW)**.
5. Producer gets ack (if `acks=all`), and consumers can now read.

---

## 3. Kafka WAL Flow (Diagram)

<img width="1589" height="1010" alt="image" src="https://github.com/user-attachments/assets/3f8c31c0-59a5-49e3-9a0d-0729e9d02c01" />




*Explanation:*
- Producer sends records to leader broker.
- Leader writes to WAL (append-only).
- Followers replicate record into their own WALs.
- Consumers read directly from WAL using offsets.

---

## 4. Page Cache vs. Log Segment File

| Aspect                | **Page Cache** (OS Memory)             | **Log Segment File** (Disk) |
|------------------------|----------------------------------------|------------------------------|
| Location              | RAM (managed by Linux kernel)          | Disk (SSD/HDD)               |
| Durability            | Volatile (lost on crash/power failure) | Durable, survives crash      |
| Performance Role      | Buffers writes & serves hot reads      | System of record             |
| Who manages it        | Operating System                      | Kafka (creates/rolls files)  |
| Write Path            | Appends into memory first             | Flushed later by OS/fsync    |
| Read Path             | Fast if still cached                  | From disk if cache miss      |

👉 **Kafka’s trick**: Use page cache for speed, replication for durability.

---

## 5. Page Cache vs Log Segment Flow (Diagram)

<img width="1745" height="1010" alt="image" src="https://github.com/user-attachments/assets/e0160027-b13f-463a-9833-29046e54a372" />

*Explanation:*
- Producers write to page cache first (fast).
- OS flushes to log segment file later (durable).
- Consumers can be served from page cache (hot data) or disk (cold data).

---

## 6. Durability Controls in Kafka

### `acks` (Producer setting)
- `acks=0` → Fire and forget (no WAL guarantee).
- `acks=1` → Leader WAL append only.
- `acks=all` → Leader waits for **all ISR replicas** to append WAL before ack.

### `replication.factor`
- Data is safe as long as ≥ 2 replicas have it.
- Typically set to **3** in production.

### `min.insync.replicas`
- Minimum replicas that must ack before producer ack.
- e.g. RF=3, min.insync.replicas=2 → tolerate 1 failure safely.

### `log.flush.interval.ms / messages`
- Controls explicit fsync frequency.
- Usually left unset → rely on OS flush.

### `unclean.leader.election.enable`
- Must be `false` to prevent out-of-sync replicas being elected → avoids data loss.

---

## 7. Key Questions & Answers

### ❓ Q1: Does `acks=all` wait for disk flush?
**Answer:** No. `acks=all` waits for all ISR replicas to **append to their WAL (page cache + log file)**, not for fsync to disk.

---

### ❓ Q2: Is page cache replicated?
**Answer:** No. Page cache is **local OS memory**. What is replicated is the **record itself**, sent to followers who append to their own page cache + log file.

---

### ❓ Q3: Why does replication (Step 3) matter if WAL append already happened in Step 2?
**Answer:** Because with `acks=all`, the ack to the producer happens **only after all ISR replicas have appended**. Replication ensures quorum durability — if the leader crashes, a follower has the record.

---

### ❓ Q4: So ack=ALL happens when all replicas have the record in page cache, not after disk fsync?
**Answer:** Correct ✅. Kafka prioritizes performance. It assumes quorum replication across brokers (each with their own WAL in page cache) is sufficient for durability. Disk flush happens later via OS daemons or segment roll/fsync.

---

### ❓ Q5: What happens if an entire cluster loses power before page cache flush?
**Answer:** That’s the edge case where Kafka durability can fail:
- If all replicas lose unflushed page cache at the same time, committed data is lost.
- Mitigation: deploy replicas across racks/AZs to avoid correlated failures.

---

## 8. Recovery: Checkpointed Offsets
When a broker restarts after crash:
- Kafka uses **checkpoint files** (e.g., `recovery-point-offset-checkpoint`, `leader-epoch-checkpoint`) to find safe log boundaries.
- It truncates unflushed tails, then **re-syncs from the leader**.
- Ensures replicas converge to the same WAL contents.

---

## 9. Summary
- WAL ensures **sequential, durable logging**.
- In Kafka, the **log itself is the database**.
- **Durability = quorum replication + High Watermark, not fsync.**
- **Page cache = speed**, **log segment = durability**, **replication = safety**.

👉 For production:
```properties
replication.factor=3
min.insync.replicas=2
acks=all
unclean.leader.election.enable=false
enable.idempotence=true
