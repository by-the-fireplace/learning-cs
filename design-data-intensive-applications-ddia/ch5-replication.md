# Replication

## Introduction
- Replication: Keeping a copy of the same data on multiple machines that are connected via the network, purposes:
    - Keep data closer to user geographically
    - Keep system working even if some parts failed
    - Scale out read queries
- Assumption: Entire data can be held on each machine (if cannot, will need sharding)
- Difficulty is dealing with changing data after replication
- Three algorithms for replication:
    - Single-leader
    - Multi-leader
    - Leaderless
- Trade-offs to consider:
    - Synchronous v.s. asynchronous
    - How to handle failed replicas

## Leaders and Followers
_Replica_ is the node that stores a copy of data. Question is _How to ensure that all data ends up on all the replicas: Every write needs to be processed by each replica_

The most common solution is _leader-based replication_ (or _active/passive_ or _master-slave replication_). How it works:

1. One of the replica is designated the _leader_. Write queries must be sent to the leaders.
2. Other replicas are _followers_. When leader writes new data to leaders, it also sends the change to all of its followers (by _replication log_ or _change stream_). Followers take the log and perform the same writes.
3. Read queries could be sent to either leader or followers. But writes will only be sent to the leader.

This mode is built-in in:

- Relational db: PostgreSQL, MySQL, Oracle Data Guard, SQL Server's AlwaysOn Availability Groups
- NoSQL db: MongoDB, RethinkDB, Espresso
- Distributed message brokers: Kafka, RabbitMQ highly available queues
- Netowrk filesystem and replicated block devices: DRBD

##### Synchronous v.s. Asynchronous Replication
- The title means if replication happens synchronously or asynchronously
- Senario: a user updates the profile photo
    - Sync: 
        - The update request sent to the leader 
        - Leader processes request, and forwards the request to the follower
        - Follower updates data and notifies leader that the changes are done
        - Leader wait until it gets the status from its follower, then it sends success response to user
    - Async:
        - Similar to sync, but the leader won't wait till its follower finishes
- Pros & Cons
    - Sync:
        - (Pro) Follower is guaranteed to have up-to-date data
        - (Con) Leader needs to wait for follower's response, any outage of a node will cause delay
    - Aync: 
        - The other side of sync
- In reality, synchronous mode means one of the node is synchronous, if that one fails, the next async follower becomes sync (semi-synchronous)
- Usually leader-based replication is configured completely asynchronously. The risk is data loss (failed write) and the advantage is laeder can keep writing without delay due to sync (good performance and availability)
- MS Azure Storage used a variant of sync called "*chain replication*".

#### Setting up new followers
- The problem of setting up new followers is how to guarantee the new follower copies complete data from the leader
- Because write requests happen all the time, thus data is different at each different time point
- We don't want to lock the database because we want to prioritize availability
- To solve this issue, we can use snapshot and time sequence log:
    - Take a snapshot of the data
    - Copy the snapshot to the new node
    - The new follower request all data changes after the snapshot, this will require the snapshot associated with an exact position in the leader's *replication log*
    - The follower processes the log and makes changes. 

#### Handling node outage
- Each node could go down either due to fault or maintenaince. The goal is to ensure the availability even if some nods failed.

**Follower failure: catch-up recovery**
Followers keep a log of data changes from the leader, so if crashes happen, it can recover data from this data change log: it knows what it was doing before the crash, and get activities after that from the leader for recovery.

**Leader failure: failover**
Leader failure could be trickier because more things need to happen: assign a new leader, followers configured to follow the new leader, clients configured to send writes to the new leader
- This could happen manually or automatically. An automatic failover usually contains the following steps:
    - Identify leader failure
    - Choose a new leader 
    - Reconfigure the system to the new leader
- Issues with this approach
    - For async followers, new leader may not yet get all changes before the leader fails. Simple solution is discard untracked writes, but this could harm user experience
    - If the system is connected with some other storage system. "The GitHub accident: primary keys got assigned to wrong rows, causing private data disclosed to wrong users"
    - Two or more nodes think they are leaders
    - What's the right timeout before the leader declares dead

#### Implementation of replication logs
**Statement-based**
- Leader logs all write requests (statement). The log is sent to followers.
- Issues:
    - Timestamp or random numbers can cause inconsistency between leader and follower
    - Autoincrementing statements require followers to execute statement in a certain order, which could limit concurrency
    - Statements with side effects, such as triggers, stored procedures, udfs could cause inconsistency
- Work-arounds are possible but requires to consider a lot of edge cases, so other replication approaches are preferred

**Write-ahead log (WAL) shipping**
- Instead of sending request statements, leaders send the log of all writing sequence to the disk. So that followers could use the log to replciate the exact same operations on disk as the leader (similar to the append-only database log)
- This method is used in PostgreSQL and Oracle
- Disadvantage is it requires certain disk format since it's a low level write description
- If a replication protocol allows follower to use a newer version of software than leader, we can first upgrade followers then failover the leader and make a follower the new leader. This is zero downtime upgrade.
- If a replication protocol does not allow this (WAL usually doesn't allow version mismatch), downtime is required.

**Logical (row-based) log replication**
- Use an alternative format for replication (logical log), which distinguishes from the storage engine's data representation
- Logical log for relational database is usually at the granularity of a row writing:
    - Insert row: log contains new values of all columns
    - Delete row: log contains information to uniquely identify the row (primary key or all columns)
    - Update row: log contains information to uniquely identify the row and new values of all columns
- MySQL's binlog uses this approach
- It's easier for external software to parse (*change data capture*)

**Trigger-based replication**
- Sometimes we want to move replication to the application layer which gives more flexibility of how we do replication
- We can achieve this by using tools such as Oracle GoldenGate or some relational database features *triggers* and *stored procedures*
    - Triggers: it lets you register application code which is automatically executed when data changes
