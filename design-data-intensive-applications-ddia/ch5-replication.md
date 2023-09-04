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


