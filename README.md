# Design Data Intensive Application

## Principle

- Reliable: making systems work correctly, even when faults occur(fault resistant)
- Scalable: having strategies for keeping performance good, even when load increases.
- Mentainable: In essense, it's about making life better for the engineers and operations teams who need to work with the system. Good abstraction and evolvibility help.

### Data models

- Relational model (SQL) : this can represent many-to-many relationships.
- NoSQL model: 
  - Document databases: data comes in self-contained, relationships between documents are rare.
  - Graph databases: Opposite to Document db, everything is related to everything.

### Storage and Retrieval

- OLTP (Online trasaction processing): User facing, huge volumn. Queries are usually small.
  - Storage engines:
    - Log-structured: only appending to files and deleting obsolete files. Never update files. Optimize for writes.
    - B-trees: Update in place.
    
- OLAP (Online analyticss processing): Data warehouses, analytics systems for business analysts. Lower volumn but big queries.
  - Index is less relvent because often you often require squential scanning for large number of rows. 
  - Instead making data compact is more important. E.g, Column oriented storage helps. Don't store all the rows but store as columns you need and you can further compress them.
    - Bitmap encoding: store each value as key and use 0/1 for each index, So you don't need to store all the repetitive values and you also know where in the sequence they appear.

## Encoding data for dataflow/transmission 

- Texture format: JSON, XML, CSV
- Binary schema: Protocol Buffers(Protobuff), Avro - allow compact, efficient and forward/backward compatibility. Schema can be useful for documentation and code generation (like use Avro to create an UI form).


- Database to Database
- REST API


### Replication

- High availability
- Place data geographically close to the users, to make it faster (low latency)
- Scalability: able to handle higher volume of reads than a single machine

Approaches to replication:
- Single-leader replication: client sends all the writes to a single node
- Multi-leader replication: client sends each write to one of the leaders
- Leaderless replication: client sends each write to several nodes.


### Partitioning

When you have so much data that storing and processing it on a single machine is not feasible.
Two main approaches to partitioning:
1. Key range partitioning - sort the key and partition alphabetically. But there is a risk of hot spot (skewed partition)
2. Hash partitioning - where a hash function is applied to each key and make it evenly distributed.

### Transactions

Here, the term transaction more or less means that we group a series of queries (reads and writes), if one fails all should fail and rollback. It's a all-or-nothing thing.

Without transaction, various errors such as process crashing, network interruptions, power outages, unexpected concurrency...can cause the data to be inconsistent.

Isolation levels and issues cuased by race condition, there could be:
- Dirty reads: one client reads another client's writes before they have been committed. 
- Dirty writes: One client overwrites data that another client has written.
- Read skew: client sees different parts of the database at different points in time.
- Lost updates: Two clients concurrently perfomr a read-modify-write cycle.
- Write skew: a transaction reads something and makes a decision based on it and writes something but by the time it writes the decision is no longer true.


### Problems with distributed systems

There can be many kinds of faults that occur when the software tries to do something that involves other nodes.

To tolerate fault, the first step is to detect them. However most systems don't have a good mechanism of detecting whether a node has failed, so most distributed systems rely on timeouts. But timeouts can distinguish between network and node failures.


### Consistency and Consensus

A popular consistency model called Linearizability - basically tries to make replicated data appear as though there was only a single copy.However it can be slow and sensitice to network delays.

Consensus means deciding something in such a way that all nodes agree. For example:

- register needs to decide whether to set its value based on its current value equals the parameter given
- database needs to decide whether to commit or abort at a distributed transaction.
- messaging system needs to decide on the order in which to deliver messages
- Lock must decide who acquire it when there are several clients race to grab the lock
- the system must decide which nodes are alive when failure is detected and which ones are dead
- the constraint must decide which one to suceed and which one to fail if there are conflicting records with the same key.

If a single-leader databse fails, you have two options:
- wait for it to recover, but it can take forever
- mannually pick a new leader

Tools like Zookeeper can provide outsourced consensus, failure detection, and membership service that applications can use.


### Derived Data

The term "systems of record" refers to source of truth - the raw data.

On the other hand, the "derived data systems" is the result of taking the data from a system and transform it in some way. If you lose the derived data you can recreate it from the original source. Cache can be deemed as derived data - you can use it if the data you need is available in cache, otherwise it can fallback to the underlying database.

### Batch processing

Unix tools example can be carried over to MapReduce and recent dataflow enginges.
```
cat /var/log/access.log |
awk '{print $7}' |
sort |
uniq -c |
sort -r -n |
head -n 5
```

Chaining these operations and thier output becomes the next input of the application.

Two main challenges that distributed batch processing frameworks need to solve:

1. Partionning: In MapReduce, the mappers are partioned. The output of the mappers is repartitioned, sorted and merged into a number of reducer partions. The purpose of this is to bring the records with the same key to the same partition,

2. Fault tolerance: MapReduce frequently writes to disk which makes it easy to recover but it will be slow in normal days. Dataflow engines does less materialization of intermediate state and keep more in the memory so when there is a failure it needs to recompute data.

Distributed batch processing engines assume callback functions (mapper and reducer) are stateless. No side effect. So that in the case of failure we can retry safely.

The input of batch processing is "bounded", meaning that it is known and fixed (like a snapshot of database or logs). Becasue it's bouned, a job knows when it has finished reading the entire input and eventually a job completes. This is not the case for stream processing, which is unbouned - you still have a job but it's never ending streams of data, so a job never completes.

### Stream processing

Stream processing are never ending unbounded streams rather than a fixed-size input. For this reason, message brokers and event logs serve as the streaming equivalent of a filesystem.

Two types of message broker:
1. AMQP/JMA style message broker: deletes the messages once they have been acknowledged by the consumers. This is fine when the exact order of the message processing is not important and there is no need to go back and read old messages.
2. Log-based message broker: the broker assigns all messages in a partition to the same consumer node, and always in the same order. Broker retains the messages on disk so it's possible to reread the old messages.
