# Transaction Isolation Levels

Transaction isolation levels define how concurrent transactions interact to maintain data consistency and integrity.

As part of the **ACID properties**, isolation ensures that each transaction acts independently. The four standard isolation levels are:

1. **Read Uncommitted**
2. **Read Committed**
3. **Repeatable Read**
4. **Serializable**

These levels balance **performance, concurrency, and data accuracy**.

### Key Characteristics

- Help control concurrent transaction behavior in multi-user databases.
- Define when and how changes made by one transaction become visible to others.
- Prevent issues such as **dirty reads, non-repeatable reads, and phantom reads**.
- Offer a trade-off between system performance and data reliability.
- Higher isolation levels provide greater data consistency but generally reduce concurrency.
- SQL defines four standard isolation levels under the ANSI/ISO SQL standards.

---

# Phenomena Defining Transaction Isolation Levels

These phenomena determine how transactions interact and where data inconsistencies may occur.

## 1. Dirty Read

A **dirty read** occurs when a transaction reads data that has not yet been committed by another transaction.

### Example

Suppose **Transaction T1** updates a row but does not commit the change.

Meanwhile, **Transaction T2** reads the updated row.

If T1 subsequently rolls back the change, T2 has read data that was never permanently committed.

```text
Transaction T1                 Transaction T2
-------------                 -------------
UPDATE account
SET balance = 500
(Uncommitted)

                              SELECT balance
                              → 500

ROLLBACK

                              T2 has read
                              uncommitted data