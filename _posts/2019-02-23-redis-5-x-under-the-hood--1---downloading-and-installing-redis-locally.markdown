---
layout:	post
title:	"Redis 5.X under the hood: 1 — downloading and installing Redis locally"
date:	2019-02-23
---

  ![](/img/1*1I9XZSUbS-F54R6NNXGkBQ.jpeg)Redis, which stands for Remote Dictionary Server, has come a long way since the initial use case of a powerful and fast key-value store. It’s gaining more and more attention for its [high-performance](https://redis.io/topics/benchmarks), diversified data types, and highly available and scalable architecture.

It has ranked at number 7 in the top 10 databases in the [DB-Engine Complete Ranking](https://db-engines.com/en/ranking), the only key-value present on the top 10. It has also recently [qualified](https://www.prnewswire.com/news-releases/redis-labs-delivers-the-fastest-multi-model-database-on-intel-optane-dc-persistent-memory-300775789.html) as a multi-model database, supporting multiple data models against a single backend.

We’ve decided to document this journey of mutual learning, as [meetup organizers](https://www.meetup.com/en-AU/Redis-Porto/), to better understand how the core Redis 5 and it’s top starred extensions work, discuss the new features in this new version, so the ones reading this series of articles and attending to our general meetups, can make informed decisions regarding the different uses cases it fits and the ones it doesn’t.

Our journey will start from the ground up, going from setting up a local instance to having multi-master clusters running on distinct availability zones on the cloud. By the end of this series, you should have a tight understanding of the things required to run Redis on production-like environments.

This document will be updated on each article release, keeping track of the series, structured in the following manner:

Article 1 — [Downloading and installing Redis locally](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154), covering the installation of a Redis Local Server and basic operations like starting and shutting down the Redis Server, connecting to Redis with redis-cli, and getting server information, understanding some key default configurations like data persistence.

Article 2 [Intro to Redis Commands and Data Structures — part 1](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-2-intro-to-redis-commands-and-data-structures-part-1-41f05501cb52), covering why you are required to think how data is stored, accessed and transformed while using Redis, while also discussing the STRING data type.

### 1 — Downloading and installing Redis locally

Using the latest version of Redis, if possible, is one of the best practices. The latest Redis major release ( 5.0.0 ) was delivered on GitHub on 17 Oct 2018. In this series, we’re adopting the latest stable version available ( 5.0.3 ) from 12 Dec 2018.

#### 1.1 — Which OS to use:

Linux and OS X are the two operating systems where Redis is developed and more tested, and we **recommend using Linux for deploying**.

Regarding Windows OS’s there is no official support, but Microsoft develops and maintains a [Win-64 port of Redis](https://github.com/MSOpenTech/redis).

#### 1.2 — Getting up and running:

We recommend two ways to get Redis up and running on your machine:

#### 1.2.1 — Option 1 — Running with docker ( simplest way ):

If you have docker on your machine this is the **simplest way **of setting a test environment:

docker run -p 6379:6379 --name redis5-under-the-hood -d redis:[5.0.3](https://github.com/docker-library/redis/blob/7be79f51e29a009fefdc218c8479d340b8c4a5e1/5.0/Dockerfile)  
sudo apt install redis-tools  
redis-cli   
127.0.0.1:6379>#### 1.2.2 — Option 2 — Build Redis:

If you want to build Redis simply execute the following commands on your terminal:

wget <http://download.redis.io/releases/redis-5.0.3.tar.gz>  
tar xvzf redis-5.0.3.tar.gz  
cd redis-5.0.3  
make  
make test  
sudo make installIf everything goes well, after the last command the following message will appear:

Hint: It's a good idea to run 'make test' ;)**INSTALL** **install  
INSTALL** **install  
INSTALL** **install  
INSTALL** **install  
INSTALL** **install**To run Redis for testing purposes ( and keep it running on background ) with the built-in default configuration just type:

redis-server &#### 1.3 — Configure Redis via a Redis configuration file:

As seen on the previous steps Redis is able to start without a configuration file using a built-in default configuration, however, this setup is only recommended for testing and development purposes.

The proper way to configure Redis is by providing a Redis configuration file, usually called [redis.conf](http://download.redis.io/redis-stable/redis.conf). In there you should be able to configure:

* Includes settings
* Network settings
* General settings
* Snapshotting settings
* Replication settings
* Security settings
* Limits settings
* Lua scripting settings
* Clustering settings
* Other advanced settings
The following link should provide more detail regarding the way the configuration file is structured <https://redis.io/topics/config>.

All of those settings will be discussed in further articles. For now, let’s keep the file [redis.conf](http://download.redis.io/redis-stable/redis.conf) with the default settings and see how to run Redis both using docker or the built Redis. In order to so, we have to run it using an additional** **parameter /**path/to/redis.conf** (the path of the configuration file).

#### 1.3.1 — Option 1.1 — Running with docker with a redis.conf file:

docker run -v **/path/to****/redis.conf**:/usr/local/etc/redis/redis.conf --name redis5-under-the-hood-redis-conf -d redis:[5.0.3](https://github.com/docker-library/redis/blob/7be79f51e29a009fefdc218c8479d340b8c4a5e1/5.0/Dockerfile) redis-server /usr/local/etc/redis/redis.conf#### 1.3.1 — Option 2.1 — Running a built Redis with a redis.conf file:

redis-server **/path/to****/redis.conf **&All the options in [redis.conf](http://download.redis.io/redis-stable/redis.conf) are also supported as options using the command line, with exactly the same name. As an example if we want Redis to accept connections on port 9999 instead of the default 6379 we should do as follows:

redis-server **/path/to****/redis.conf **--port 9999 &Please keep in mind that **if you pass a **[**redis.conf**](http://download.redis.io/redis-stable/redis.conf)** file it should always be the first parameter.**

#### 1.4 — A first glance of how Redis persists data:

Now that we have a test Redis-server up and running, let’s introduce you in a quick way to how Redis persists data. We’ll devote an entire article about the pros and cons of data persistence in Redis, and the several ways of doing it, but for now, let’s just introduce you to the subject.

The default method for data persistence that comes enabled on the [**redis.conf**](http://download.redis.io/redis-stable/redis.conf)** is **via snapshotting the DB to the local disk, which performs point-in-time snapshots of your dataset at specified intervals, in the following default manner:

* **after 900 sec (15 min) if at least 1 key changed**
* **after 300 sec (5 min) if at least 10 keys changed**
* **after 60 sec if at least 10000 keys changed**
Redis snapshotting does ***NOT provide good durability guarantees if up to a few minutes of data loss is not acceptable*** in case of incidents, therefore it’s usage is limited to applications where losing recent data is not very important.

[**Lyft**](https://www.lyft.com/) has shared a good example of how they run with no replication and are OK with eventual data loss. Following this redisconf 2018 presentation link: <https://www.slideshare.net/RedisLabs/redisconf18-2000-instances-and-beyond>.

There is a more advanced persistence mode that Redis provides, called The Append Only File, usually simply AOF, which logs every write operation received by the server. We shall discuss the pros a cons latter. For now, keep in mind there are tradeoffs on both modes.

#### 1.5 — Stopping Redis:

To stop Redis, the most graceful way of doing it is executing the [**shutdown**](https://redis.io/commands/SHUTDOWN) command.

redis-cli  
127.0.0.1:6379> shutdownIf persistence is enabled this command makes sure that Redis is switched off without the loss of any data.

#### 1.6 — Next Steps:

Now that we’ve got an up-and-running Redis Local Server we can start looking under to hood. Our next article will be an ***Intro to Redis Commands and Data Structures***, giving you a basic understanding of the available data types and how they are useful with hands-on examples.

  