What’s the difference between an ACID and a BASE database?

ACID and BASE are database transaction models that determine how a database organizes and manipulates data. In the context of databases, a transaction is any operation that the database considers a single unit of work. A transaction must complete fully for the database to remain consistent. For example, when you transfer money from one bank account to another, the money must leave your account and must be added to the third-party account. You cannot call the transaction complete without both steps occurring.

ACID databases prioritize consistency over availability—the whole transaction fails if an error occurs in any step within the transaction. In contrast, BASE databases prioritize availability over consistency. Instead of failing the transaction, users can access inconsistent data temporarily. Data consistency is achieved, but not immediately.

### Why are ACID and BASE important?

Modern databases are distributed data stores that replicate data across multiple nodes connected by a network. They allow users to perform multiple data manipulations like reads and writes in a single transaction. Users expect data to remain consistent across all nodes at the end of the transaction. However, in theoretical computer science, Brewer's theorem (also called CAP theorem) states that any distributed data store can provide only two of the following three guarantees:

* **Consistency:** every read operation receives the most recently updated data or an error.
* **Availability:** every database request receives a successful response, without guaranteeing it contains the most recently updated data.
* **Partition tolerance:** the system continues to operate despite dropped or delayed messages between the distributed nodes.

For example, if a customer adds an item to a cart on an ecommerce website, all other customers should see the stock levels of the product drop. If the customer adds the last item to the cart, all other users should see the item as out of stock. In case any operation fails within a transaction, database designers must make a choice. The database can do one of the following:

* **Cancel the transaction and return an error**, decreasing availability but ensuring consistency. The customer cannot add the item to their cart or other customers cannot load the details for all products until add to cart succeeds.
* **Proceed with the operation and thus provide availability** but risk inconsistency. The customer adds an item to the cart, but other customers view incorrect stock levels, at least temporarily.

In some use cases, consistency is critical and ACID databases are favored. However, there are other use cases where it is non-critical. For example, when you accept a friend request on social media, it doesn't matter if other users see an incorrect number of friends on your social media profile temporarily. However, you don't want to lose access to your social media feed while the data is sorted. In such scenarios, BASE becomes important.

---

### Key principles: ACID compared with BASE

ACID and BASE are acronyms for different database properties representing how the database behaves during online transaction processing.

#### ACID

ACID stands for atomicity, consistency, isolation, and durability.

##### Atomicity

Atomicity ensures that all steps in a single database transaction are either fully-completed or reverted to their original state. For example, in a reservation system, both tasks—booking seats and updating customer details—must be completed in a single transaction. You cannot have seats reserved for an incomplete customer profile. No changes are made to the data if any part of the transaction fails.

##### Consistency

Consistency guarantees that data meets predefined integrity constraints and business rules. Even if multiple users perform similar operations simultaneously, data remains consistent for all. For example, consistency ensures that when transferring funds from one account to another, the total balance before and after the transaction remains the same. If Account A has $200 and Account B has $400, the total balance is $600. After A transfers $100 to B, A has $100 and B has $500. The total balance is still $600.

##### Isolation

Isolation ensures that a new transaction, accessing a particular record, waits until the previous transaction finishes before it commences operation. It ensures that concurrent transactions do not interfere with each other, maintaining the illusion that they are executing serially. For another example, is a multi-user inventory management system. If one user updates the quantity of a product, another user accessing the same product information will see a consistent and isolated view of the data, unaffected by the ongoing update until it is committed.

##### Durability

Durability ensures that the database maintains all committed records, even if the system experiences failure. It guarantees that when ACID transactions are committed, all changes are permanent and unimpacted by subsequent system failures. For example, in a messaging application, when a user sends a message and receives a confirmation of successful delivery, the durability property ensures that the message is never lost. This remains true even if the application or server encounters a failure.

#### BASE

BASE stands for basically available, soft state, and eventually consistent. The acronym highlights that BASE is opposite of ACID, like their chemical equivalents.

##### Basically available

Basically available is the database’s concurrent accessibility by users at all times. One user doesn’t need to wait for others to finish the transaction before updating the record. For example, during a sudden surge in traffic on an ecommerce platform, the system may prioritize serving product listings and accepting orders. Even if there is a slight delay in updating inventory quantities, users continue to check out items.

##### Soft state

Soft state refers to the notion that data can have transient or temporary states that may change over time, even without external triggers or inputs. It describes the record’s transitional state when several applications update it simultaneously. The record’s value is eventually finalized only after all transactions complete. For example, if users edit a social media post, the change may not be visible to other users immediately. However, later on, the post updates by itself (reflecting the older change) even though no user triggered it.

##### Eventually consistent

Eventually consistent means the record will achieve consistency when all the concurrent updates have been completed. At this point, applications querying the record will see the same value. For example, consider a distributed document editing system where multiple users can simultaneously edit a document. If User A and User B both edit the same section of the document simultaneously, their local copies may temporarily differ until the changes are propagated and synchronized. However, over time, the system ensures eventual consistency by propagating and merging the changes made by different users.

---

### Key differences: ACID compared with BASE

There are trade-offs when choosing between ACID and BASE database transaction models.

#### Scale

It’s harder to scale an ACID database transaction model because it focuses on consistency. Only one transaction is allowed for any record at any moment, which makes horizontal scaling more challenging.

Alternately, you can scale the BASE database model horizontally because it doesn’t need to maintain strict consistency. Adding multiple nodes across the database cluster allows the BASE model to improve its data availability, which is the principle driving the database architecture.

#### Flexibility

ACID databases are less flexible when handling data. It must guarantee immediate consistency, an ACID database may restrict access to some applications if it experiences network or power outages. Similarly, applications must wait for their turn to update data if other software modules are processing a particular record.

Conversely, BASE databases are more flexible. Rather than imposing strict restrictions, BASE allows applications to modify records when they are available.

#### Performance

An ACID database might experience performance issues when handling large data volumes or concurrent processing requests. As the data is processed in a strict order, overhead in each transaction creates delays that affect all applications accessing the record.

In contrast, applications accessing a BASE database can process the records at any time. This reduces excessive wait time and improves the database throughput.

#### Synchronization

An ACID database needs a synchronization mechanism to commit changes from a transaction and reflect them in all associated records. At the same time, it must lock the particular record from other parties until the transaction completes or is discarded.

Meanwhile, a BASE database runs without locking the records, synchronizing them eventually without time guarantees. When working with BASE databases, developers know there might be inconsistencies when processing certain records and take the necessary precautions in the application.

---

### When to use: ACID compared with BASE

Despite their differences, both ACID and BASE database systems are relevant in different applications. ACID is the ideal option for enterprise applications that require data consistency, reliability, and predictability. For example, banks use an ACID database to store customer transactions because data integrity is the top priority. Meanwhile, BASE databases are a better option for online analytical processing of less structured, high-volume data. For example, ecommerce websites use BASE databases to update product prices, which change frequently. In this case, pricing accuracy is less vital than allowing all customers real-time access to the product price.

---

### Can a database be both ACID and BASE?

According to the CAP theorem, a database can satisfy two of the three guarantees of consistency, availability, and partition tolerance. Both ACID and BASE database models provide partition tolerance, meaning they cannot be both highly consistent and always available. So, a database either leans towards ACID or BASE, but it cannot be both. For example, SQL databases are structured over the ACID model, while NoSQL databases use the BASE architecture. However, some NoSQL databases might exhibit certain ACID traits, but they can’t operate as ACID-compliant databases.

---

### Summary of differences: ACID compared with BASE

| Property | ACID | BASE |
| --- | --- | --- |
| **Scale** | Scales vertically. | Scales horizontally. |
| **Flexibility** | Less flexible. Blocks specific records from other applications when processing. | More flexible. Allows multiple applications to update the same record simultaneously. |
| **Performance** | Performance decreases when processing large volumes of data. | Capable of handling large, unstructured data with high throughput. |
| **Synchronization** | Yes. Adds delay when synchronizing. | No synchronization at the database level. |