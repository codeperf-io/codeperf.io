---
layout:	post
title:	"Redis 5.X under the hood: 2 — Intro to Redis Commands and Data Structures — part 1"
date:	2019-02-25
---

  ![](/img/1*vLg4gNGA5Iq2b5ofiPGoJQ.jpeg)In this article, we will look at Redis data types and important operations, coupling them with use cases that they can fit on.

* On part 1 of this article, we will cover why you are required to think how data is stored, accessed and transformed while using Redis, while also discussing the STRING data type.
* LISTs and HASHs data types will be covered on part 2.
* SETs, SORTED SETs, and GEO data types will be covered on part 3.
Every data type provides different operations, each one coupled with more complexity and/or performance.

Bare in mind the extreme importance of knowing the Do’s and Don’ts of each data type, because ***“if all you have is a hammer, everything looks like a nail”.***

We will consider you have a Redis server instance up and running, as described in the previous article of this series [https://medium.com/@fcosta\_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154).

### 2.1 Before we dive into the data types:

Before we dive into the data types, we need to understand another important concept, the keyspace. It’s essentially a dictionary — key-value schema — of keys and their respective values.

* The keys are unique and can be any valid Redis String.
* The values can be any one of Redis core module data types and are only accessible by their name.
### 2.2 Yet another advise:

Every data type provides different operations, each one coupled with more complexity and/or performance, with the complexity of operation described in the documentation.

Take as an example the [**SET**](https://redis.io/commands/set) command, with a **Time complexity:** O(1), denoting a constant time operation. Keep in mind that My O(1) and Your O(1) are not necessarily the same. It only means that the time of the operation is independent of the number of items in the keyspace.

### 2.3 What Redis is not:

Redis transactions are not **fully ACID compliant ( Atomicity, Consistency, Isolation, and Durability)**. If **ACID** transactions are expected, Redis is not the perfect fit and should not be used. An RDBMS or another database system should be used in those scenarios.

Redis, being a non-relational database, **it’s not fit for serving the purpose of relational data.** The amount of complexity you’ll have to handle for keeping concepts like foreign keys, referential integrity constraints, rollbacks, and durability as seen on relational databases, is far more than you should even consider if not even impossible. You can achieve some of them but as a tradeoff of efficiency, complexity and data volume.

To get an idea of how Redis fits among the variety of database and cache software available, you can see an introductory listing of a few different types of cache or database servers on the following link: <https://redislabs.com/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/1-1-what-is-redis/1-1-1-redis-compared-to-other-databases-and-software/>

### 2.4 STRING data type:

Strings, the fundamental data types of Redis, are used for storing:

* Strings: [APPEND](https://redis.io/commands/append), [GETRANGE](https://redis.io/commands/getrange), [SETRANGE](https://redis.io/commands/setrange), [STRLEN](https://redis.io/commands/strlen)
* Integers: [INCR](https://redis.io/commands/incr), [INCRBY](https://redis.io/commands/incrby), [DECR](https://redis.io/commands/decr), [DECRBY](https://redis.io/commands/decrby)
* Floats: [INCRBYFLOAT](https://redis.io/commands/incrbyfloat)
* Bits: [SETBIT](https://redis.io/commands/setbit), [GETBIT](https://redis.io/commands/getbit), [BITPOS](https://redis.io/commands/bitpos), [BITCOUNT](https://redis.io/commands/bitcount), [BITOP](https://redis.io/commands/bitop)
To associate a string value to a key, use the [SET ](https://redis.io/commands/set)command. For example, if we would like to set the venue for our meetup:

127.0.0.1:6379> SET venue:redis-porto "Porto I/O, Porto, Portugal"OKString values can be retrieved by simply using the [GET ](https://redis.io/commands/get)command:

127.0.0.1:6379> GET venue:redis-porto"Porto I/O, Porto, Portugal"Redis commands return an acknowledgement for all usual commands. Executing [GET ](https://redis.io/commands/get)on a non-existent key will return (nil):

127.0.0.1:6379> GET venue:redis-lisbon(nil)The [STRLEN](https://redis.io/commands/strlen) command returns the length of the string. As an example, if we would like to know the length of the venue of redis-porto:

127.0.0.1:6379> STRLEN venue:redis-porto  
(integer) 26Redis provides several commands to manipulate STRING key contents without the need to perform two distinct operations. Let’s understand the importance of those features. As an example, if we would like to append the string “, Europe” to the string containing the venue of redis-porto an option would be to execute the [GET](https://redis.io/commands/get), append on the client, and SET back the result to Redis, as seen on the next diagram:

![](/img/1*n7f5JRhdNWasOqp1F8o5DQ.png)There are several drawbacks to this approach:

* Despite each operation being atomic, since GET and SET are two distinct operations, this may lead to possible inconsistencies if someone changes the key after the GET and prior the SET.
* Work is required on the client.
There are a couple of commands to manipulate strings directly without using [GET ](https://redis.io/commands/get)to retrieve the value and [SET ](https://redis.io/commands/set)to assign it back, on a single atomic operation, like [APPEND ](https://redis.io/commands/append)and [SETRANGE](https://redis.io/commands/setrange).

[APPEND ](https://redis.io/commands/append)has the name points, appends the value at the end of the string. If the key we want to append to does not exist it is created and set as an empty string, prior to the append operation. [SETRANGE](https://redis.io/commands/setrange) overwrites part of the string stored at the specified key, starting at the defined offset, for the named length.

The next diagram illustrates how we could get the same output from the previous [GET ](https://redis.io/commands/get)and [SET ](https://redis.io/commands/set)( in order to append to the string) in the recommended manner.

![](/img/1*4_dbvExSGnISL5epPIjANA.png)#### 2.4.1 Yet another atomic use case, the Atomic conditional SET with NX

If there is already a value associated with the key, the [SET ](https://redis.io/commands/set)command overwrites the value. Sometimes we don’t want the value to be overwritten blindly if the key exists. One thing we can do is use the [EXISTS ](https://redis.io/commands/exists)command to test the existence of the key before executing [SET](https://redis.io/commands/set), however, if we do the check and set in two separate commands, they will not be an atomic operation leading to a potential undesired state.

As an example, consider we have two persons setting up the venue in Lisbon, and we’ve agreed we would only set the venue if it was not yet set. Person 1 and Person 2 will execute the following commands on the same order.

EXISTS {key}  
SET {key} {value}  
GET {key}Person 1 sees the following output, which is not the expected one for him:

127.0.0.1:6379> EXISTS venue:redis-lisbon(integer) 0127.0.0.1:6379> SET venue:redis-lisbon "ISCTE, Lisbon, Portugal"OK127.0.0.1:6379> GET venue:redis-lisbon"Universidade NOVA de Lisboa, Lisbon, Portugal"However, Person 2 sees the following output, the expected one for him:

127.0.0.1:6379> EXISTS venue:redis-lisbon(integer) 0127.0.0.1:6379> SET venue:redis-lisbon "Universidade NOVA de Lisboa, Lisbon, Portugal"OK127.0.0.1:6379> GET venue:redis-lisbon"Universidade NOVA de Lisboa, Lisbon, Portugal"The result retrieved using the [GET ](https://redis.io/commands/get)command by Person 1 is not the expected one. This can happen because between the time the [EXISTS ](https://redis.io/commands/exists)command was executed by Person 1 and the time the SET command was executed by Person 1, someone set the key. The following diagram illustrates exactly that:

![](/img/1*LUWzvTBoi4K8GASHzdZeAw.png)Redis provides a command [SETNX ](https://redis.io/commands/setnx)(short for SET if not exists), which can be used to set the value for a key, but only if the key does not exist, in an atomic operation, returning 1 if the key was successfully set, and 0 if the key already exists, so the old value will not be overwritten.

When Person 1 executes [SETNX ](https://redis.io/commands/setnx)venue:redis-lisbon will see:

127.0.0.1:6379> SETNX venue:redis-lisbon "ISCTE, Lisbon, Portugal"(integer) 1and Person 2, since the condition check and set were atomically done by the [SETNX ](https://redis.io/commands/setnx)command will see:

127.0.0.1:6379> SETNX venue:redis-lisbon "Universidade NOVA de Lisboa, Lisbon, Portugal"(integer) 0leading to no overwrite of the key previously set, as illustrated in the following diagram:

![](/img/1*dCe6NjHfot6Fq_7Eyry-og.png)#### 2.4.2 Atomic, binary-safe and highly performant

If you think that Redis can handle up to [80K events per second](https://redis.io/topics/benchmarks),~5M events per minute, without pipelining, and up to 400K events per second,~24M events per minute with pipelining (we will see the meaning of pipelining later), on a **single low end, untuned redis-server,** in a binary-safe and atomic per-operation way, the STRING data type has numerous use cases.

The STRING data type is the right bullet for storing simple strings like “meetup” to the content of entire files, like a JPEG file, and doing simple but high throughput numerical operations on ints, floats, and bits.

#### 2.4.3 The end of race conditions for numerical operations:

As enumerated priorly, Redis provides commands for numerical operations for the datatype string. The [INCR](https://redis.io/commands/incr), [INCRBY](https://redis.io/commands/incrby), [DECR](https://redis.io/commands/decr) and [DECRBY](https://redis.io/commands/decrby), and [INCRBYFLOAT](https://redis.io/commands/incrbyfloat) commands parses the string value as an integer/float, increments it by the specified value, and finally set the obtained value as the new value, without the need for any numerical operation on the client while retaining decent performance and consistency regarding the data the each client accesses.

Multiple clients issuing [INCR](https://redis.io/commands/incr) against the same key will never enter into a race condition. For instance, it will never happen that client 1 reads “10”, client 2 reads “10” at the same time, both increment to 11, and set the new value to 11. The final value will always be the correct one without the need for implicit locks on the data access and manipulation.

#### 2.4.4 Bit and Byte level operations:

Bit and Byte level operations enable fast and extremely memory optimized numerical operations. Suppose you want to describe binary operations ( something that can only be true or false ) like if a user has attended to a meetup, or viewed this medium article. If we had 1 Million users we would only require 122KB of RAM to represent the entire dataset.

Taking the 1 Million users example, we can use the [SETBIT](https://redis.io/commands/setbit) command, that takes as its first argument the bit number and as its second argument the value to set the bit to, which is 1 or 0. The command automatically enlarges the string if the addressed bit is outside the current string length.

127.0.0.1:6379> SETBIT redis-porto:article:2 1000000 1(integer) 0To access the value for the specified user we would then use [GETBIT](https://redis.io/commands/getbit):

127.0.0.1:6379> GETBIT redis-porto:article:2 1000000(integer) 1If we would like to access the value for user 10000 we would then:

127.0.0.1:6379> GETBIT redis-porto:article:2 10000(integer) 0Ranges out of the index ( outside the length of the string stored into the target key) are always considered to be zero. If we would like to count the amount of user that have read the article, we could BITCOUNT, which reports the number of bits set to 1 for the specified key.

127.0.0.1:6379> BITCOUNT redis-porto:article:2(integer) 1Besides [BITCOUNT](https://redis.io/commands/bitcount), there are still [BITOP](https://redis.io/commands/bitop) that enables to perform bit-wise operations between different strings ( AND, OR, XOR and NOT ), and [BITPOS](https://redis.io/commands/bitpos) enabling you to find the first bit having the specified value of 0 or 1.

Any key in a Redis database can store up to 512MB of information (2³² — 1 bits ). This means that there 4 294 967 295 data points that can be set per key. If you put that into the user's info scenario you could represent information up to 4 billion users in one single Redis key.

#### 2.4.5 How strings are encoded in Redis:

It is worth mentioning how strings are encoded in Redis objects internally. Redis uses three different encodings to store string objects and will decide the encoding automatically per string value. Along with the description of each encoding manner, we will make use of the command [OBJECT](https://redis.io/commands/object), allowing you to inspect the internals of Redis Objects:

* **REDIS\_ENCODING\_INT ( int )— **For strings representing 64-bit signed integers, in other words, strings can be stored in this form, if the value is cast to long in the range minimum and maximum values for an object of type **long int**.
127.0.0.1:6379> SET key "12"OK127.0.0.1:6379> OBJECT encoding key"int"* **REDIS\_ENCODING\_EMBSTR ( embstr ) —** used for strings with a length up to 44 bytes This means that the Redis Object structure and string structure are placed in a single area of memory, leading to more efficient memory usage and performance.
127.0.0.1:6379> SET key "redis-porto"OK127.0.0.1:6379> OBJECT encoding key"embstr"* **REDIS\_ENCODING\_RAW** **( raw ) —** used for all strings whose length exceeds 44 bytes. Using what we’ve learned from SETRANGE documentation lets create a string greater than 44 bytes in a “clean” manner:
127.0.0.1:6379> SETRANGE key 44 "redis-porto"(integer) 55127.0.0.1:6379> OBJECT encoding key"raw"### 2.5 Transient data:

Before diving deeper into other Redis data structures, we need to discuss another command which works regardless of the datatype and is called **Redis **[**EXPIRE**](https://redis.io/commands/expire).

When a timeout is set on a Redis key, and the specified time to live ([TTL](https://redis.io/commands/ttl)) elapses, the key is automatically destroyed, exactly as if the user called the [DEL](https://redis.io/commands/del) command with the key. You can use [TTL](https://redis.io/commands/ttl) to check the time to live of a key.

127.0.0.1:6379> SET "venue:redis-porto" "Porto I/O"OK127.0.0.1:6379> TTL "venue:redis-porto"(integer) -1Now let's apply a time to live of 120 seconds to the key.

127.0.0.1:6379> EXPIRE "venue:redis-porto" 120(integer) 1127.0.0.1:6379> TTL "venue:redis-porto"(integer) 108If we wanted to update time to live of the key we could again use [EXPIRE](https://redis.io/commands/expire).

If we wanted to completely remove the time to live we could use [PERSIST](https://redis.io/commands/persist).

### 2.6 Considerations regarding the size of the data

When an Ethernet network is used to access Redis, keeping the size of the data under the Ethernet packet size (about 1500 bytes) maintains the overall peak performance, meaning that for the redis-server processing 10 bytes, 100 bytes, or 1000 bytes leads to almost the same result in terms of throughput. Nonetheless, you should always consider the impact of the size of the data in your own application ( the one making the queries ). See the graph named “Throughput per data size” of the following link: <https://redis.io/topics/benchmarks>.

### 2.7 Next Steps:

Our purpose in this series of articles is to introduce to a subject and key concerns, hoping that you dive deep into the extent of operations available and well documented on <https://redis.io/commands>. For all Redis Strings commands please refer to <https://redis.io/commands#string>. We’re hoping to see comments both on this article and our meetups regarding you’re pet projects and concerns.

Part 2 of this article will be an ***Intro to Redis LISTs and HASHs data types***, giving you a basic understanding of the available operations and how they are useful with hands-on examples.

  