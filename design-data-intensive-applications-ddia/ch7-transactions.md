# Transactions

- Things can go wrong in data systems:
    - Database software/hardware fails
    - Application crashes
    - Network interruption
    - Clients write to db at the same time and overwrites each other
    - Incomplete data changes
    - Race conditions 
- A transaction is a way for an application to group several reads and writes together into a logic unit
- Conceptually, all reads and writes in a transaction are executed as one operation: either succeeds (*commit*) or fails (*abort*, *rollback*).

## The Slippery Concept of a Transaction
- First introduced in 1975 by IBM System R
- Large-scale system would have to abandon transactions in order to maintain good performance and high availablity?

**The Meaning of ACID**
- Transactions provide ACID which guarantees safety:
    - A: Atomicity
    - C: Consistency
    - I: Isolation
    - D: Durability
- High-level idea is sound, but the devil is in the details
- Due to the ambiguity of ACID, it has become mostly a marketing term

*Atomicity*

- Refers to something that cannot be broken down into smaller parts
- It describes what happens if a client wants to make several writes but a fault occurs after some of the writes have been processed
- If writes are grouped into a transaction, it won't be completed (committed) until all writes finish. If one failed, the transaction is aborted and the db must discard or undo writes that it has made so far
- Perhaps *abortability* is a better word than atomicity

*Consistency*

- In the context of ACID, it refers to an application-specific notion of the database being in a "good state"
- The idea is you have certain statements about your data that must always be true. If a transaction starts with a db that is valid for these statements, and all writes also meet it, then consistency is achieved
- Consistency depends on application layer, database itself cannot stop users from putting inconsistent data
- Atomicity, isolation, durability are properties of a db, whereas consistency is a property of teh application

*Isolation*
- Means concurrently executing transactions are isolated from each other, isolation is sometimes formalized as *serializability*, meaning that each transaction can pretend it's the only transaction running on the db. The db ensures that even though they are executed concurrently, the result is as if they run serially.
- In practice, it is rarely used due to the performance penalty

*Durability*
- Is the promise that once a transaction is committed, the data it has written will not be lost, even if there's a fault
- In practice, durability means data has been persisted with a write-ahead log or similar, which allows recovery. In replicated db, durability means data has been copied to some number of nodes. To ensure durability, db must wait writes (copies) are complete before committing the transaction.
