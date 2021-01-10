---
layout:	post
title:	"Event Data Pipelines with Redis Pub/Sub, Async Python and Dash"
date:	2019-12-01
---

  ![](/img/1*Uti7xoTCscBvdyv0ww6yGA.png)This article was written as a complement to a [meetup](https://www.meetup.com/pyporto/events/lfpdhryzqbcb/#) organized by [Python Porto](https://www.meetup.com/pyporto/) and [RedisPortugal](https://www.meetup.com/Redis-Portugal/), aiming for a full-day workshop around Async Python, Dash, and Redis Pub/Sub and can be used to get acquainted with the techniques and patterns to successful model event data pipelines. Consider this as a white/yellow belt overall difficulty but all levels of expertise can use this to both learn or remember important concepts.

We’ve created a mini-project to make things more practical. We will briefly look at the Event Data Pipelines concept and how to pragmatically implement it using Python and Redis Pub/Sub. The hands-on project has been thought by **Roman Imankulov, **and is fully available on GitHub in the following [link](https://github.com/imankulov/wikipedia-playground).

We will consider you have a Redis server instance up and running, as described in a previous article of other meetup event series [https://medium.com/@fcosta\_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154).

### Event-Data Pipelines 101

The event-data pipeline follows a publish-subscribe principle, as shown in the following diagram:

![](/img/1*zC4JrtL2QfYDxGqzNbkzCg.png)In pub/sub systems, Senders publish messages to intermediate message brokers or event bus, and subscribers register subscriptions with the intermediate broker, letting the broker fan-out messages to subscribers. Every Subscriber that subscribes to a topic receives its own copy of each message published. A single message produced by one publisher may be distributed to hundreds, thousands, or even millions of subscribers.

The publishers can produce various events on multiple topics, and consumers will subscribe to the topics that they are interested in. Across the multiple solutions available in the market that can be used to implement an event-data pipeline, they all share the common and most basic design principle:


> Publishers and Receivers are decoupled.#### Publish-subscribe advanced principles

Some more advanced design principles such as:

* durable subscriptions
* message delivery quality of service ( at-most-once, only-once, at-least-once )
* highly-available
* fault-tolerance
will depend on the chosen solution and should be part of the considerations when deciding the event-data pipeline backbone.

Some of the available tools that will enable you to incorporate some of these important design principles are Apache Kafka, AWS Kinesis, RabbitMQ, AWS SMS, Azure Service Bus Event Hub, Google Pub/Sub and Redis ( with Pub/Sub, Streams, Lists, or even Sorted Sets ).

Apart from the available design principles of each tool, you should also consider the **overall volume of data** of your data pipeline — if you intend to make usage of durable subscriptions, and **message rate to expect at each moment**. The event-data pipeline needs to handle a large volume of event data being produced by many sources of data in a reliable, and efficient manner.

#### A quick note on Publish/Subscribe or Event Streams for Event Data Pipelines

![](/img/1*2HSYt1pmp_2AeRUSwPYUUQ.png)While in Pub/Sub messages are normaly*(see above) *fire and forget* and are never stored anyway*(see above), Event [STREAMS](https://redis.io/topics/streams-intro) work in a fundamentally different way. Stream abstractions are useful (from the developer’s perspective) for building responsive

distributed systems that support fault tolerance and scalability. You should ask yourself the following questions when deciding between Publish/Subscribe or Event Streams:

* Does your application need to be able to retrieve **historical events**?
* Does your application need to **scale out RECEIVES**? — meaning adding groups of clients to cooperate in consuming a different portion of the same data stream.
* Does your application requires **Fine Grained Subscriptions**, meaning to select the events at a finer granularity than just the topic-based approach?
* Does your application demands different **Quality-Of-Service**, depending on the topic you wish to receive events?
The majority of the available market tools commonly enable you to emplace both a stream processing platform or messaging/pub-sub system. Using the above questions and other more advanced features that you might require, should allow you to determine if a messaging Publish/Subscriber or Event Streams solution is most appropriate.

### Pub/Sub in action with Redis

![](/img/1*86X0_3nR7DlhYXTa4gW5dw.gif)We can try the publish-subscribe principle using a local Redis instance and the redis-cli utility. Start three instances of redis-cli.

In order to subscribe to the channel meetup, from within two of the connected clients, issue [SUBSCRIBE](https://redis.io/commands/subscribe) providing the name of the channel:

SUBSCRIBE meetupAt this point, from another client we issue a [PUBLISH](https://redis.io/commands/publish) operation against the channel named meetup with the message you want to send:

PUBLISH meetup helloThe message sent will be pushed by Redis to all the subscribed clients, as seen on the above GIF.

The commands that are allowed in the context of a subscribed client are [SUBSCRIBE](https://redis.io/commands/subscribe), [PSUBSCRIBE](https://redis.io/commands/psubscribe), [UNSUBSCRIBE](https://redis.io/commands/unsubscribe), [PUNSUBSCRIBE](https://redis.io/commands/punsubscribe), [PING](https://redis.io/commands/ping) and [QUIT](https://redis.io/commands/quit).

#### Pub/Sub Global Scope

As stated on the official documentation, Pub/Sub has no relation to the Redis key space, thus having global scope. Therefore, publishing on db 10, will be heard by a subscriber on db 1. If you need scoping of some kind, prefix the channels with the name of the environment (test, staging, production, …).

#### At-most-once Redis Pub/Sub

Because Redis Pub/Sub is *fire and forget,* there is no way to use this feature if your application demands **reliable notification** of events, that is, if your Pub/Sub client disconnects, and reconnects later, all the events delivered during the time the client was disconnected are lost.

#### In Redis Pub/Sub is everywhere

In Redis, one well-known feature is [Keyspace notifications.](https://redis.io/topics/notifications) They allow clients to receive events affecting the Redis data set in some way. Events are delivered using the normal Pub/Sub layer of Redis, so even though you might not be using implicitly Pub/Sub it’s running explicitly on your clients.

### MiniProject — Visualizing Wikipedia updates stream with Redis

Now that we’ve familiarized ourselves with Publish-Subscribe principle, and how to use it in Redis, we’ll delve into a more practical example. The hands-on plan has been thought by **Roman Imankulov, **and is fully available on GitHub in the following [link](https://github.com/imankulov/wikipedia-playground). To start right away with it simply follow:

git clone <https://github.com/imankulov/wikipedia-playground.git>  
cd [wikipedia-playground](https://github.com/imankulov/wikipedia-playground.git)To give you a quick introduction to the project we want to model, Wikipedia provides a real-time [SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) feed for updates for all Wikimedia projects. You can find documentation about event streams from Wikipedia at [https://wikitech.wikimedia.org/wiki/Event\_Platform/EventStreams](https://wikitech.wikimedia.org/wiki/Event_Platform/EventStreams).

The Event stream source is available at <https://stream.wikimedia.org/v2/stream/recentchange>, and can be easily visualized issuing:

curl <https://stream.wikimedia.org/v2/stream/recentchange>![](/img/1*S8ddogJ7l_ZleiA6dNDFwQ.gif)#### Publisher

The file [redis\_publisher.py](https://github.com/imankulov/wikipedia-playground/blob/master/redis_publisher.py), makes usage of aioredis, an async Redis client library, reading messages from SSE and publishing them to the channel “wiki”.

Don’t worry about copying each gist of code. The full project is available on GitHub in the following [link](https://github.com/imankulov/wikipedia-playground). Within the project folder, run the script:

pipenv run ./redis\_publisher.pyIt immediately starts publishing events to the channel. You can visualize them with a simple Redis client

redis-cli subscribe wikiType Ctrl+C to quit.

![](/img/1*NbLd29xnAKFOoieCFGcHIw.gif)#### Subscriber

There is a script [redis\_subscriber.py](https://github.com/imankulov/wikipedia-playground/blob/master/redis_subscriber.py) which consumes the events from the pub-sub channel and models them into time-series events using plain vanilla Redis data structures. There are several ways of modeling time series data in Redis, wether with Sorted Sets, Streams, using modules ( [RedisTimeSeries](https://oss.redislabs.com/redistimeseries/) ) or using a simpler approach with Hashes (as we wanted to do in this meetup).

Within the project folder, run the script:

pipenv run ./redis\_subscriber.pyAs a result of the script work, a number of keys will be created in Redis:

$ redis-cli  
127.0.0.1:6379> keys *  
...  
7306) "ev:www.wikidata.org"  
7307) "ev:no.wikipedia.org"  
7308) "ev:cs.wiktionary.org"As you will explore further, we’ve used both Redis [SETs](https://redis.io/commands#set) and [HASHes](https://redis.io/commands#hash) on the subsbscriber.py script. Every data type provides different operations, each one coupled with more complexity and/or performance. You can find out more about [Redis’s different data-types at the official documentation.](https://redis.io/topics/data-types)

After reading the above gist, you can see that:

- specifically for Sets we use [SADD](https://redis.io/commands/sadd) command against the **known\_domains** key and **{domain\_name} **member in the following manner. We’ve done that to keep track of every distinct domain:

SADD known\_domains {domain\_name}If you use [SMEMBERS](https://redis.io/commands/smembers) command against the **known\_domains** key you will get the distinct domains currently being updated:

$ redis-cli  
127.0.0.1:6379> **SMEMBERS**** ****known\_domains**1) "da.wikipedia.org"  
2) "fi.wikipedia.org"  
3) "zh.wikipedia.org"  
4) "nl.wikibooks.org"  
5) "he.wikipedia.org"  
6) "et.wiktionary.org"  
7) "fr.wikiversity.org"  
8) "es.wikibooks.org"  
...* specifically for Hashes we use [HINCRBY](https://redis.io/commands/hincrby) command against the **{domain\_name} **and** {timestamp} **member of the hash. We’ve modeled the time series data as a group of time-series elements within a hash, for each distinct domain. Then, for each timestamp block, we have an associated number of events.
HINCRBY ev:www.wikidata.org {timestamp} 1The resulting data structure has the following properties:

* Events aggregated by domain name.
* Hardcoded value expiration\_timeout\_sec defines how long events are stored in Redis
* Another hardcoded value aggregation\_interval\_sec defines the event aggregation interval (in seconds).
At this point, you will have one hash per distinct domain, with hash containing multiple fields, each field being a timestamp group.

If you use [HKEYS](https://redis.io/commands/hkeys) command against one of the encoded domain hashes **“en.wikipedia.org”** you will get the distinct timestamp groups:

$ redis-cli  
127.0.0.1:6379> HKEYS filipe:en.wikipedia.org1) "1575199840"  
2) "1575199850"  
3) "1575199860"  
4) "1575199870"  
5) "1575199880"  
6) "1575199890"  
7) "1575199910"  
8) "1575199920"  
9) "1575199940"  
...### Visualizing time-series events with Dash

Dash is an open source web framework to build interactive data-driven applications, primarily data visualizations and dashboards.

File [dash\_app.py](https://github.com/imankulov/wikipedia-playground/blob/master/dash_app.py) contains a sample application that lets you pick one or more domain names to get te update statistics for it.

Within the project folder, run the script:

pipenv run ./dash\_app.pyAs a result, you should see:

![](/img/1*HfPUbY2hsRkpOMhjxb5lxA.gif)### Key take-aways and Next Steps:

We’ve briefly look at the Event Data Pipelines concept and how to practically implement it using Python and Redis Pub/Sub, and how to visualize data streams in real-time, using Dash, an Open Source visualization platform from Plotly.

As you can image, this is just the tip of the iceberg both in the theoretical concept and in the practical aspect of the used tooling. Keep up with learning, by delving into:

* [E-Book Redis in Action. The chapter “Counters and Statistics”](https://redislabs.com/ebook/part-2-core-concepts/chapter-5-using-redis-for-application-support/5-2-counters-and-statistics/)
* [Wikimedia event streams](https://wikitech.wikimedia.org/wiki/Event_Platform/EventStreams#Python)
* [Python client for Redis (redis-py)](https://github.com/andymccurdy/redis-py)
* [Plotly Dash documentation](https://dash.plot.ly/)
* [Plotly time series examples](https://plot.ly/python/time-series/)
* [Plotly Python API reference](https://plot.ly/python/reference/)
You can also take RedisLabs’s Redis University Several courses that will enable you to learn the techniques and patterns to successful model various data structures to effectively and efficiently use Redis, and delve into more advanced topics like the ones described here:

* [Introduction to Redis Data Structures](https://university.redislabs.com/courses/course-v1:redislabs+RU101+2020_01/about)
* [Redis Streams](https://university.redislabs.com/courses/course-v1:redislabs+RU202+2020_01/about)
* [Redis Certified Developer Program](https://university.redislabs.com/courses/course-v1:redislabs+DEVELOPER-CERTIFICATION-1+2019-2020/about)
  