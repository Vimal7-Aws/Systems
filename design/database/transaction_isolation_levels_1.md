# Transaction Isolation Levels in DBMS


Transaction isolation levels define how concurrent transactions interact to maintain **data consistency and integrity** in a database system.

As part of the **ACID properties**, the *Isolation* property ensures that each transaction executes independently, without being affected by other concurrent transactions.

The four standard isolation levels defined by ANSI/ISO SQL are:

- Read Uncommitted  
- Read Committed  
- Repeatable Read  
- Serializable  

These levels provide a balance between **system performance** and **data accuracy**.

---

# Why Isolation Levels Matter

Isolation levels:

- Control concurrent transaction behavior in multi-user databases.
- Define when and how changes made by one transaction become visible to others.
- Prevent issues like dirty reads, non-repeatable reads, and phantom reads.
- Provide a trade-off between performance and reliability.
- Higher isolation ensures greater data consistency but reduces concurrency.

---

# Phenomena Defining Transaction Isolation

These anomalies determine how transactions interact and where inconsistencies may occur.

---

## 1️⃣ Dirty Read

A **dirty read** occurs when a transaction reads data that has been modified but not yet committed by another transaction.

### Example

- Transaction T1 updates a row but does not commit.
- Transaction T2 reads the updated row.
- T1 rolls back the update.

Now, T2 has read data that never officially existed in the database.

---

## 2️⃣ Non-Repeatable Read

A **non-repeatable read** occurs when a transaction reads the same row twice and gets different values each time.

### Example

- Transaction T1 reads a row.
- Transaction T2 updates that row and commits.
- T1 reads the same row again.

The value retrieved is now different from the first read.

---

## 3️⃣ Phantom Read

A **phantom read** occurs when the same query returns a different number of rows during the same transaction.

### Example

- Transaction T1 retrieves rows matching a condition (e.g., salary > 50000).
- Transaction T2 inserts new rows that satisfy that condition and commits.
- T1 runs the same query again.

T1 now sees additional rows (phantoms) that were not present earlier.

---

# Isolation Levels in DBMS

> Note: “Serializable” isolation is necessary but not always sufficient for a schedule to be fully serializable in theory. However, in practice, the Serializable level prevents the standard anomalies.

The four standard isolation levels are:

---

## 1️⃣ Read Uncommitted

**Lowest isolation level.**

- Transactions can see uncommitted changes from other transactions.
- Allows dirty reads.
- Also allows non-repeatable reads and phantom reads.

### Characteristics
- Highest concurrency
- Lowest consistency
- Rarely used in production systems

---

## 2️⃣ Read Committed

A transaction can only see data committed by other transactions.

### Prevents
- Dirty reads

### Still Allows
- Non-repeatable reads
- Phantom reads

### Characteristics
- Moderate consistency
- Common default level in many databases

---

## 3️⃣ Repeatable Read

Ensures that rows read during a transaction cannot be modified by other committed transactions until it finishes.

### Prevents
- Dirty reads
- Non-repeatable reads

### Still Allows
- Phantom reads (in some implementations)

### Characteristics
- Stronger consistency
- Slightly reduced concurrency

---

## 4️⃣ Serializable

Highest isolation level.

Transactions behave as if they are executed sequentially (one after another).

### Prevents
- Dirty reads
- Non-repeatable reads
- Phantom reads

### Characteristics
- Strongest consistency
- Lowest concurrency
- Higher locking overhead

---

# Isolation Levels vs Anomalies

| Isolation Level   | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------------------|------------|----------------------|--------------|
| Read Uncommitted  | ✅ Possible | ✅ Possible          | ✅ Possible  |
| Read Committed    | ❌ Prevented | ✅ Possible         | ✅ Possible  |
| Repeatable Read   | ❌ Prevented | ❌ Prevented        | ✅ Possible* |
| Serializable      | ❌ Prevented | ❌ Prevented        | ❌ Prevented |

\*Depends on database implementation.

---

# Choosing the Right Isolation Level

The choice depends on balancing:

- Data accuracy
- System performance
- Concurrency requirements

### Higher Levels (e.g., Serializable)
- Strong consistency
- Lower concurrency
- Slower performance

### Lower Levels (e.g., Read Uncommitted)
- Better concurrency
- Faster performance
- Higher risk of inconsistencies

---

# Advanced Concurrency Mechanisms

## Snapshot Isolation

- Uses a consistent snapshot of data.
- Transactions read from a snapshot instead of locked rows.
- Reduces blocking and improves concurrency.

---

## MVCC (Multi-Version Concurrency Control)

- Maintains multiple versions of data.
- Readers do not block writers.
- Writers do not block readers.
- Improves performance in high-concurrency systems.

---

# Summary

Transaction isolation levels are critical for controlling how concurrent transactions interact in a DBMS.

They:

- Protect data integrity
- Prevent common anomalies
- Balance performance and reliability

Choosing the correct isolation level is essential for designing scalable and consistent database-driven applications.

