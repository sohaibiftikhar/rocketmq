# RocketMQ-124: Proposed Design

### Objective
Achieve exactly-once delivery semantics for RocketMQ.

### Background
RocketMQ can serve its consumers redundant messages at certain times. Two messages are said to be redundant when 
they possess the same message IDs for the same topic and message queue (Message Tag). See [`MessageExt`](https://github.com/apache/rocketmq/blob/master/common/src/main/java/org/apache/rocketmq/common/message/MessageExt.java) 
for details on what constitutes a message. Two messages may also be defined as redundant depending on the customer's logic. 
For example, a KV property associated with a message. Message duplication/redundancy can occur due to a variety of reasons:

1. Application Logic:
    1. Due to certain application logic a message may be produced twice. For example a page tracking pageviews or 
    customer clicks may invoke the same javascript twice. If the application ensures that both these messages get 
    the same KV property associated with them RocketMQ can ensure uniqueness when in the broker cluster.
2. Producer Error:
    1. A producer may emit the same message twice. This can be due to `FLUSH_DISK_TIMEOUT` or due to `FLUSH_SLAVE_TIMEOUT` etc.
3. Consumer Error:
    1. A consumer may shut down abruptly without commiting its offset to the broker. This happens only in clustering mode. 
    On revival it starts consuming from some old offset that the broker had. 
    2. Consumer may trigger rebalance which also 
    leads to duplication. (More details on this would be welcome such as when does this happen?)
    
### Proposed Solution
My solution to this problem is twofold. Exactly once semantics between producer-broker communication and exactly once 
semantics between broker-consumer.

#### Part 1: Producer-Broker Communication
The first step is to ensure that only unique messages are stored by the Broker. For this we will need changes to the 
`SendMessageProcessor` such that every time it stores something to disk using the MessageStore it checks for duplicates 
in a distributed persistent caching layer (that expires based on a sliding time window). With such a design the minimum 
distance between two duplicates would be the expire time for this cache. A small illustration of this is shown in the 
diagram below.

![Fig1: Removing duplicates from Brokers](https://i.imgur.com/B40tA7l.png)

The next subsection discusses how such a cache may be designed.

#### Part 1A: Distributed Persistent Cache
There are two parts to this persistent caching layer. The first part deals with providing a highly available and 
fault tolerant key store. For this part we can even use some third party solution if this is acceptable or write 
our own if the dev community feels this is necessary. Key points to note for this would be as follows:
1. Multiple instances of the store with data distributed between them using some tested methodology such as 
consistent hashing.
2. Some leader election algorithm (such as Paxos) for taking cluster decisions such electing a new master for a 
data partition, communicating to nodes that a node is down and beginning rebalancing.
3. Multiple copies for a single key on different instances (raft guarantee).
4. Some sort of filter such as Bloom or Cuckoo to ensure faster response times.
Efficient data storage on disk and retrieval. I suggest we use some form of a B+ Tree to store this. The advantage of 
this is illustrated in the second part.

#### Part 2: Consumer-Broker Communication
As a second step we need to ensure that consumers do not consume the same message ID twice. Since with a cache layer our
broker ensures that each message is stored only once we are only concerned with offsets here. Additionally we only need 
to do this for `CLUSTERING` message model as with `BROADCASTING` the offsets are stored locally. This is somewhat hard to 
fix with the current message model as the offset is sent back to the broker only after calling the 
`MessageListener.consumeMessage`. One possible fix for this is illustrated below. I make the assumption here that consumer 
is an orderly consumer hence ensuring that only one consumer consumes from a message queue at a time. The consumption 
happens in the following steps:
1. We make the consumer perform its `consumeMessage` and storing the offset to some database transactionally 
(that is either both happen or none). With this we can ensure that at least the local store is always in sync with 
the offset of the messages consumed. 
2. Now if this consumer crashes without syncing with the broker the new instance of this broker can now read from 
the database and initialize consuming. 
3. If the offset of the message received from the broker is older than the offset in the database the consumer ignores 
this message.
4. Once the offset from the broker is greater than the one in the database the consumer can begin consuming as before.
5. In summary this would look at the consumer end as following:
  ```
  Begin transaction;
  write offset to database
  status = messageListener.consumeMessage
  if (status == OK) commit transaction
  else rollback
  ```
  
With this method if there are sufficient number of message queues then parallelism can be maintained. However, as can be 
witnessed the problem is not simplistic. I would be glad to receive some help and guidance on solving this.
