---
layout:	post
title:	"Redis 5.X under the hood: 3 — Writing a Redis Module"
date:	2019-03-31
---

  ![](/img/1*8EdjHjNBNJ_RjDaBcWochg.jpeg)**Oakland Bay Bridge** — This Photo was taken on 30th of March 2019 in San Francisco Bay — while writing this article and preparing for RedisConf 19Although Redis supports a good number of data types and extensive features, sometimes you may be asked to add custom logic on your data that the core Redis does not support. When that happens you have two options to achieve those same requirements, either via Lua scripts or via Redis Modules. Lua scripting is a way to “compose” existing data structures and commands, but not a way to extend the capabilities of a system towards use cases it was not designed to cover. When compared to Lua scripts Redis Modules have the following advantages:

* You’re able to create new data structures with Redis Modules while with Lua Scripts you’re only able to use the existing ones.
* As C libraries, Redis modules run faster and with less overhead, while also enabling you to call external third-party libraries.
* The commands added via Redis Modules can be called directly via the Redis clients, as if they were native to Redis, while with Lua the scripts have to be called with EVAL/EVALSHA.
* The APIs exposed to Redis Modules are much richer than the APIs exposed to Lua scripts.
#### Why are Redis Modules successful?

Back in 2016, Salvatore Sanfilippo wrote about modules as system features [on his blog](http://antirez.com/news/106):


> Modules can be the most interesting feature of a system and the most problematic one at the same time: API incompatibilities between versions, low-quality modules crashing the system, a lack of identity of a system that is extendible are possible problems.Redis Modules APIs were designed towards compatibility for the future, so that a module wrote today could work in 4 years from now with the same API, regardless of the changes to the Redis core. From those fundamentals two different APIs were born in order to access the Redis data space:

* One is a low-level API that provides very fast access and a set of functions to manipulate Redis data structures. Redis developers are allowed to create commands that are as capable and fast as the Redis native commands.
* The other API is more high level and allows to call Redis commands and fetch the result, similarly to how Lua scripts access Redis.
Redis Modules power Redis users go faster, without jeopardising the Redis Core and its development, since the two fundamentals APIs were promised to have no breaking changes for the years to come. The APIs will be improved upon, but with total backwards compatibility, since the module registration asks for a given API version.

We’ll check the low-level API a bit further on this article, but first, we will introduce how to load an existing Redis module.

We’ll show how to write a simple Redis module with one single operation LIST.FILTER, that will enable us to filter low and high values on lists ( implements what is normally known as Low-value and High-value filters ).

### 0.1 Before we dive deeper:

We’ve decided to document this journey of mutual learning, to better understand how the core Redis 5 and it’s top starred extensions work, discuss the new features in this new version, so the ones reading this series of articles, can make informed decisions regarding the different uses cases it fits and the ones it doesn’t.

We will consider you have an overall understanding of Redis 5.X, and a Redis 5.X Server instance running, as we described in the previous articles of this series:

— [Redis 5.X under the hood: 1 — Redis-Server up and running](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154)

— [Redis 5.X under the hood: 2 — Intro to Redis Commands and Data Structures — part 1](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-2-intro-to-redis-commands-and-data-structures-part-1-41f05501cb52)

— [Straightforward Time Series DBs with Redis 5.X — part 1 :: RedisTimeSeries module](https://medium.com/@fcosta_oliveira/straightforward-time-series-dbs-with-redis-5-x-part-1-redistimeseries-module-79aa616159eb)

### 1 — Loading and building modules

There are a lot of Redis Modules that can be used to extend Redis functionality. By the time of the writing of this article, these were the top modules, by Github stars:

* [neural-redis](https://github.com/antirez/neural-redis): Online trainable neural networks as Redis data
* [RediSearch](https://github.com/RedisLabsModules/RediSearch): Full-Text search over Redis
* [RedisJSON](https://github.com/RedisLabsModules/redisjson): A JSON data type for Redis
* [rediSQL](https://github.com/RedBeardLab/rediSQL): A Redis module that provides a Fast, in-memory, SQL engine ( embedding SQLite )
* [redis-cell](https://github.com/brandur/redis-cell): A Redis module that provides rate limiting in Redis as a single command
* [RedisGraph](https://github.com/RedisLabsModules/RedisGraph): A graph database with a Cypher-based querying language using sparse adjacency matrices
* [RedisML](https://github.com/RedisLabsModules/redisml): Machine Learning Model Server
* [RedisTimeSeries](https://github.com/RedisLabsModules/RedisTimeSeries): Time-series data structure for Redis. If you remember we’ve written an article with in-depth usage examples — [Straightforward Time Series DBs with Redis 5.X — part 1 :: RedisTimeSeries module](https://medium.com/@fcosta_oliveira/straightforward-time-series-dbs-with-redis-5-x-part-1-redistimeseries-module-79aa616159eb).
#### 1.1— Build RedisJSON Module:

Download the source code of RedisJSON from GitHub, by executing the following commands on your terminal:

git clone --depth 1 <https://github.com/RedisLabsModules/redisjson.git> && cd redisjson  
cd src  
makeThe module binary will be generated as **rejson.so**

#### 1.2 — Load RedisJSON Module:

Load the module using the following redis.conf configuration directive:

loadmodule **/path/to/****rejson****.so**Finally:

redis-server **/path/to****/redis.conf &  
**redis-cli   
127.0.0.1:6379>It is also possible to load a module at runtime using the MODULE LOAD command with the path to the Redis Module we want to load:

MODULE LOAD **/path/to/****rejson****.so**If a module is loaded by the MODULE LOAD command, it will be unloaded automatically when the server stops.

To list all loaded modules, use:

127.0.0.1:6379> module list  
1) 1) "name"  
 2) "ReJSON"  
 3) "ver"  
 4) (integer) 10004Finally, you can unload (and later reload if you wish) a module using the following command:

MODULE UNLOAD ***name of the module***You should be aware that if the module we’ve loaded exports one or more module-side data types, as **RedisJSON** does, we can’t unload it at runtime. We should remove the configuration directive from the redis.conf file and then restart redis-server. If you try to unload ReJSON at runtime you should see the following error message:

127.0.0.1:6379> **MODULE UNLOAD ReJSON**(error) ERR Error unloading module: the module exports one or more module-side data types, can't unload### 2— Writing our Redis module

The easiest and most straightforward way of writing a Redis module is to download a copy of the redismodule.h header file from Redis’s official [repository](https://github.com/antirez/redis/blob/5.0/src/redismodule.h), include it in our source tree, write the module, link all the libraries we want, and create a dynamic library having the RedisModule\_OnLoad() function symbol exported.

You can also write the module in C++ or any language that has C binding functionalities.

We are going to implement a Redis module, LIST\_EXTEND, in C language with a new command FILTER, which can be called as follows:

127.0.0.1:6379> **LIST\_EXTEND.FILTER** source\_list destination\_list low\_value high\_valueThe command takes one Redis list **source\_list** and tries to create another Redis list **destination\_list**, whose elements, if numeric, will be filtered by the condition:

* low\_value ≥ element value ≤high\_value
If low\_value is larger than the high\_value, an empty list is returned.

low\_value and high\_value can be -inf and +inf so that you are able to implement only low filters, only high filters, or both. By default, the interval specified by low\_value and high\_value is closed (inclusive).

**Return value  
**Integer reply: the length of the destination\_list list after the filter operations.

For example, if the following commands were executed:

127.0.0.1:6379> **LPUSH** source\_list 20 2 30 1 40  
(integer) 4  
127.0.0.1:6379> **LIST\_EXTEND.FILTER** source\_list destination\_list 2 20  
(integer) 3  
127.0.0.1:6379> **LRANGE** destination\_list 0 -1  
1) "1"  
2) "2"  
3) "20"The destination\_list would have [1,2,20] since the other values from source\_list did not verify the low and high-value filter conditions.

#### 2 .1— Writing our LIST\_EXTEND module

#### 2.1.1 Project structure

Let’s start by creating a file called list\_extend.c and include the C header file called redismodule.h.

Don’t worry about copying each gist of code. The full module project folder is available on GitHub in the following [link](https://github.com/filipecosta90/list_extend).

#### 2.1.2 Module entry-point

The entry point for each Redis Module is a function called RedisModule\_OnLoad() enabling it to be initialized, register its commands, and potentially other private data structures it uses. It’s a good practice for modules to call commands with the name of the module followed by a dot, and finally the command name. In our case, our command would be called in the following manner LIST\_EXTEND.FILTER. This way it is less likely to have collisions. Here is our RedisModule\_OnLoad() implementation:

Inside RedisModule\_OnLoad(), the first function to be called should be RedisModule\_Init(). The following is the function prototype:

int RedisModule\_Init(RedisModuleCtx *ctx, const char *modulename, int module\_version, int api\_version);The Init function announces the Redis core that the module has a given name, its version (that is reported by MODULE LIST), and API version that is willing to use.

The second function called, RedisModule\_CreateCommand(), was used in order to register commands into the Redis core, having the following function prototype:

int RedisModule\_CreateCommand(RedisModuleCtx *ctx, const char *name, RedisModuleCmdFunc cmdfunc, const char *strflags, int firstkey, int lastkey, int keystep);The set of flags ‘strflags’ specifies the behaviour of the command and should be passed as a C string composed of space separated words. Since our command modifies data, we’ve added “write”. Our command will also use additional memory and should be denied during out of memory conditions, thus leading us to add the flag “deny-oom”. In our case, ‘strflags’ will be “write deny-oom”.

The return value of RedisModule\_OnLoad() should be REDISMODULE\_OK unless there are errors.

#### 2.1.3 Creating the command handler

To create a new command, the above function needs the context, the command name, and the function pointer of the function implementing the command, which must have the following prototype:

int mycommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc);In our case the function implementing the command will be ListExtendFilter\_RedisCommand(), as visible on the following snippet:

Its arguments are just the context, that will be passed to all the other API calls, the command argument vector, and the total number of arguments, as passed by the user. The arguments are provided as pointers (**argv) to a specific data type, the RedisModuleString. This is an opaque data type you have API functions to access and use, direct access to its fields is never needed.

#### 2.1.3.1 Describing the command logic

Inside the function, we start by checking the number of command arguments is correct. As the command is expected to take four arguments (source\_list, destination\_list, low\_value, high\_value) including the command itself, the number of command arguments should be 5.

We then ask Redis to automatically manage the resource and memory in our command handler, by simply calling **RedisModule\_AutoMemory**(ctx).

Next, we begin accessing the Redis key source\_list, a Redis LIST, by using the low-level API **RedisModule\_OpenKey**(). This API returns a pointer to RedisModuleKey, which is a handler of the Redis key.

We then check the key type by using **RedisModule\_KeyType**() to make sure the inputs are lists or empty. Now that we know we have a list or empty, we check the number of elements using **RedisModule\_ValueLength**(). If the key pointer is NULL, due to the type being empty, **RedisMoudule\_ValueLength()** returns zero which is just what we need for our function logic. In case the have zero elements we end the function call, by sending an integer reply with 0 to the client via **RedisModule\_ReplyWithLongLong()**.

Next, we check if the limits are correctly set, with the possible values being either an **integer** **or** **-inf** for the argument lower\_limit ( argv[3] ), and an **integer or +inf **for the argument lower\_limit ( argv[4] ). Since we are using long long limit constants ( LONG\_MIN and LONG\_MAX ) we’ll also need to include the <limits.h> header file.

The last step consists of looping through the **source\_list**, and assigning the value of each element to the **destination\_list**, if they are both integers and within the defined limits. We do this by popping from the list tail each element from the source\_list via **RedisModule\_ListPop()**, converting the returned RedisModuleString to a long long integer via **RedisModuleStringToLongLong()**, checking the limit conditions, and then if the conditions are met, converting the long long integer back to RedisModuleString and pushing it to the head of **destination\_list** via **RedisModule\_ListPush**().

We call **RedisModule\_ReplicateVerbatim**(ctx) to propagate the command to all Redis replicas.

Finally, we send to the client an integer reply with the number of items added to **destination\_list **via **RedisModule\_ReplyWithLongLong()**. **RedisModule\_ReplyWithLongLong()** always returns REDISMODULE\_OK.

The final version of list\_extend.c is available on GitHub in the following [link](https://github.com/filipecosta90/list_extend/blob/master/list_extend.c).

#### 2.1.3.1 Compiling LIST\_ENTEXD

Now we have our module completely set up we just need to compile it and load it list\_extend.o into Redis ( as visible on section 1.2 ).

gcc -fPIC -shared -std=gnu99 -o list\_extend.o list\_extend.cHere is an example of our working command:

127.0.0.1:6379> MODULE LOAD /Users/filipeoliveira/redis-porto/list-extend/list\_extend.o  
OK127.0.0.1:6379> lpush origin 1 20 25 40 50 string1 1.1 4  
(integer) 8127.0.0.1:6379> list\_extend.filter origin dest -inf 10  
(integer) 2127.0.0.1:6379> lrange dest 0 -1  
1) "4"  
2) "1"#### 3 — Next Steps:

This basic setup has given you a small taste of what Redis Modules can do. We encourage you to try building your own module. If you liked our low and high-value filter command, as a suggestion, try implementing it for sorted sets.

Check the Redis Modules API reference and official introduction in the following links:

* <https://redis.io/topics/modules-api-ref>
* <https://redis.io/topics/modules-intro>
  