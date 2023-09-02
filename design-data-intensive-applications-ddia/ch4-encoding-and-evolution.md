# Encoding and Evolution

![](./img/ch4.png)

## Introduction

This chapter talks about dealing with changes in an application.

When an application evolves, the upgrade will not happen instantaneously

- Server-side: need **rolling upgrade** to deploy new version to a few nodes at a time
- Client-side: users may not install the update every time

Compatibility is important in upgrading, including backward and forward compatibility. Data encoding is important for compatibility.

## Formats for encoding data

### Language-specific format

- Python: `pickle`
- Java: `java.io.Serializable`

### Standardized encoding

- JSON, XML and Binary variants
- Thrift and Protocol Buffers: binary coding libraries
- Avro: binary encoding format

## Modes of Dataflow

Compatibility is about one process encoding the data and another process decodes it, which is dataflow.

The author discussed 3 modes of dataflow:

- Via databases
- Via service calls
- Via asynchronous message passing

### Dataflow through databases
- Encoding: The process that writes data into db 
- Decoding: The process that reads data from db
- Multiple processes could access the db at the same time, and these processes may from older/newer versions of the code. So both forward and  backward compatibility are important.
    - The db has been written by a newer version of code but still read by an older version. This could happen during rolling updates

**Different values written at different times**
- Data can outlives code, which means database contents (historical data) will remain intact unless modified.
- When data application is updated, migrating data to new schemas are expensive, thus schema evolution is valuable. Schema evolution means make changes to the schema instead of creating a new one.

**Archival Storage**
- We need to snapshot the db from time to time for backup or loading in to data warehouse. When taking snapshot, it will typically use the latest schema
- The snapshot is immutable, thus formats like **Avro** object container files are a good fit. Also those analytics-friendly column-oriented format like **Parquet** is also good.

### Dataflow through services: REST and RPC
- When processes need to communicate over a network, usually they are arranged as _clients_ and _servers_.
    - Servers expose an API
    - Clients can make requests to the API to connect with the server
- Microservices architecture: Large service broken down into small services. So the server could also be a client. This is also called "service-oriented architecture"
- The goal is to make changes and maintaineance independently deployable and evolable
#### Web serivces
- **REST**: not a protocol, but a design philosophy based on principles of HTTP
- **SOPA**: an XML-based protocol for making network API requests. Though it's used over HTTP, it avoids HTTP features and has complex standards. The API of a SOAP web service is described using an XML-based language WSDL (web services description language). Users can access remote service using local classes and method calls.
    - This is more useful for static typed languages
    - WSDL is not designed to be human-readable

- **RPC**: RPC is remote procedure call. The goal is to make a request to the server, which looks the same as calling a function in your programming language.
    - But network request is very different from a local call due to unpredictability, uncertain returns, and other issues.
    - Author also mentioned the future directions for RPC
#### Messaging-passing data
- Messages goes via an intermediaty called **message blockers**
- It has advantages such as 
    - 1) it can work like a buffer if the recipient is unavailable or overloaded
    - 2) Automatically redeliver messages to a process that has crashed, prevent messages from being lost
    - 3) Sender does not need to know the IP and port number of recipient (particularly useful in cloud env where VMs come and go)
    - 4) One message could be sent to multiple recipient
    - 5) Decouples sender from recipient
**Message brokers**
- Open source implementations: RabbitMQ, ActiveMQ, HornetQ, NATS, Kafka
- In general, message brokers are used as follows:
    - One process sends a message to a named **_queue_** or **_topic_**
    - The broker ensure that the message is delivered to **_subscribers_** or **_consumers_**.
- A consumer may itself publish messages to another topic (chaining is possible)
- Usually no data model is enforced in message brokers, just a sequence of bytes with some metadata

**Distributed actor frameworks**
- **Actor model** is a programming model for concurrency in a single process. 
- Each **actor** represents one client or entity.
- It communicates with other actors by sending and receiving asynchronous messages.
- Message delivery is not guaranteed
- Each actor only processes one message at a time, so it doesn't need to worry about thread issues (race conditions, locking and deadlock).
- Each actor can be scheduled independently by the framework
- Distributed actor framework scales the actor model to multiple nodes
- Popular distributed actor frameworks and messages they handle:
    - _Akka_: Java's built-in serialization
    - _Orleans_: custom data encoding format that does not support rolling upgrade deployment (new cluster is needed)
    - _Erlang OTP_
