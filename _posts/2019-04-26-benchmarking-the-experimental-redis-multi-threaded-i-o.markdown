---
layout:	post
title:	"Benchmarking the experimental Redis Multi-Threaded I/O"
date:	2019-04-26
---

  Redis was generally known as a single-process, single-thread model. For the newer versions of Redis, that is not true.

![](/img/0*cXOVEBO4XpneMVVn)**Golden Gate Bridge** — This Photo was taken on 30th of March 2019 in San Francisco Bay — while preparing for RedisConf 19Quoting the [official documentation](https://redis.io/topics/faq):


> Since Redis 4.0 we started to make Redis more threaded. For now this is limited to deleting objects in the background, and to blocking commands implemented via Redis modules. For the next releases, the plan is to make Redis more and more threaded.Redis runs multiple backend threads to perform backend cleaning works, such as cleansing the dirty data and closing file descriptors. It’s no longer single-process when you fork child processes on every background save.

To make Redis more multi-threaded, the simplest way to think of is that once Redis needs to perform any Write or Read operation, that work is performed by N, previously fanned-out, I/O threads. There’s no major complexity added since the main thread is still responsible for the major tasks, but most of the advantages, because the **majority** of time Redis spends is in IO, as usually, Redis workloads are either memory or network bound.

#### 1 ) Tuning Redis VMs and finding out which Redis code-paths are hot (busy on-CPU)

To find out exactly how much time we’re spending on IO, we’ve set up two **n1-highcpu-96 **instances on GCP, one in which we will run redis-server ( from the threaded-io branch ) and another in which we will run redis-benchmark.

To run redis-server in the most performant way, we’ve tuned both **n1-highcpu-96 **instances, using tuned-adm **throughput-performance profile **and manual settings, by:

* disabling **tuned** and **ktune **power saving mechanisms.
* enabling **sysctl** settings that improve the throughput performance of your disk and network I/O, and switches to the **deadline scheduler**.
* setting CPU governor to performance.
* manually disabling Transparent Huge Pages.
* manually raising somaxconn to 65535.
* manually setting vm.overcommit\_memory from 0 (default) to 1 ( never refuse any malloc ).
The manual settings we’ve referred above can be achieved by running the following commands. The last one ensures that the sysctl settings will take effect immediately.

***sudo -i  
***echo never > /sys/kernel/mm/transparent\_hugepage/enabledsysctl -w vm.overcommit\_memory=1  
sysctl -w net.core.somaxconn=65535  
*sysctl -p*#### 1.1 ) Profiling single threaded redis-server

**1.1.1 ) redis-server VM**

Now, that we’ve tuned our virtual machines for our server workloads, we can start on one of them a redis-server instance, with the following configurations:

**fcosta\_oliveira@n1-highcpu-96-redis-server-1**:**/**redis-server --protected-mode no --save "" --appendonly no --daemonize yes**1.1.2 ) redis-benchmark VM**

To evaluate the performance of Redis and generate the multiple workloads we require to profile Redis, we will use redis-benchmark across the entire article. The official redis-benchmark program is a quick and useful way to get some figures.

We’ve forked the official one and included tests to the new Streams data types in the [following github repository](https://github.com/filipecosta90/redis/tree/benchmark_xadd). We’re currently waiting for the [PR revision ](https://github.com/antirez/redis/pull/6015)for it to be included in Redis.

On this section we’re only interested in validating the amount of time Redis spends is in IO, to be able to use Amdahl’s law to predict the theoretical speedup when using parallel workloads. For that, we will use the new threaded redis-benchmark for sending 10M GET commands, issued by 150 clients, with a key size of 100 Bytes.

**fcosta\_oliveira@n1-highcpu-96-redis-benchmark-1**:**~/**redis-benchmark -t get -c 150 -n 10000000 — threads 46 -h {ip of redis-server vm} -d 100**1.1.3 ) Profiling stack traces while running benchmark tool**

While we’re running the benchmark from **n1-highcpu-96-redis-benchmark-1 **VM, we can then profile the redis-server stack traces in the **n1-highcpu-96-redis-server-1** VM,** **using Linux perf\_events (aka “perf”) at a fixed sample rate of 99Hz stack samples, by running the following command:

**fcosta\_oliveira@n1-highcpu-96-redis-server-1**:**~/**sudo perf record -F 99 — pid `pgrep redis-server` -g -o 100\_bytes\_no\_iothreadsResulting in the following flame graph:

![](/img/1*K4OfwNbkGY_M0TkUBINdNQ.png)The execution time of Redis-server before the improvement of the experimental Redis Multi-Threaded I/O, is denoted as T. It includes the execution time of the part that would not benefit from the improvement of the resources and the execution time of the one that would benefit from it. As visible from the previous flame graph, 46.6% of the execution time (spent on sys\_write) may be the subject of a speedup with the experimental Redis Multi-Threaded I/O.

The speedup is limited by the serial part of the program, in our case 53.4%, leading to a theoretical maximum speedup using parallel computing of 2 times, as visible on the blue line of the following graph.

![](/img/0*P4PbxqYSG7XW0sLi.png)Evolution according to Amdahl’s law of the theoretical speedup is latency of the execution of a program in function of the number of processors executing it. Image retrieved from [https://en.wikipedia.org/wiki/Amdahl%27s\_law](https://en.wikipedia.org/wiki/Amdahl%27s_law).#### 2 ) Setting the hard limits for our benchmark

**2.1 ) Network hard limit**

Network bandwidth and latency usually have a direct impact on Redis performance. Prior to going further with our benchmark, we will check the latency between our benchmark and redis-server VMs, using qperf.

The following outputted experimental results are for a single-tenant uncongested network where there is no background traffic from other applications, and thus no network interference.

**fcosta\_oliveira@n1-highcpu-96-redis-benchmark-1**:**~/qperf**$ qperf -t 60 -v {ip of redis-server vm} tcp\_bw   
**tcp\_lattcp\_bw:**   
 bw = 3.05 GB/sec   
 msg\_rate = 46.5 K/sec   
 time = 60 sec   
 send\_cost = 248 ms/GB   
 recv\_cost = 181 ms/GB   
 send\_cpus\_used = 75.6 % cpus   
 recv\_cpus\_used = 55.3 %  
**cpustcp\_lat: **   
 latency = 27.6 us   
 msg\_rate = 36.3 K/sec   
 time = 60 sec   
 loc\_cpus\_used = 14.6 % cpus   
 rem\_cpus\_used = 15.9 % cpusWe can observe the measured bandwidth of the network of 3.05 GB/sec ( 24.4Gbits/sec ). A benchmark setting 100 Bytes values for each key in Redis, would be hard limited by the network at around 32 million queries per second.

**2.2 ) Memory hard limit**

Going through Redis official documentation,


> Speed of RAM and memory bandwidth seem less critical for global performance especially for small objects. For large objects (>10 KB), it may become noticeable though.Nonetheless, we will benchmark the VM Memory bandwidth using STREAM, a benchmark with intent to represent operations on long vectors, and widely used for research, testing and marketing purposes.

The following commands get STREAM, and set the array size larger than the n1-highcpu-96 cache. We’ve repeated the test 10 times and discarded the first one.

git clone <https://github.com/jeffhammond/STREAM.git>  
cd STREAM/  
gcc -fopenmp -D\_OPENMP -O -DSTREAM\_ARRAY\_SIZE=100000000 stream.c -o stream.100M  
export OMP\_NUM\_THREADS=48  
./stream.100MThe second run of STREAM is presented below.

We can observe the measured Best Rate GB/s for the Copy kernel at around 83GB/s. For instance, a benchmark setting 100 Bytes values for each key in Redis, would “naively” be hard limited by the VM memory at around 890 million queries per second. We’re not accounting for different memory access patterns and invalidation — the measured Best Rate and hard limits are intended as relatively coarse scaling estimates.

### 3 ) Running the benchmark for different configurations

We begin our experimental study by considering memory-intensive workloads that perform memory-to-memory network I/O with different characteristics, namely:

* **SET**, to generate write requests to the Redis server. The write request time complexity is O(1).
* **GET**, to generate read requests to the Redis server. The read request time complexity is O(1).
* **MSET**, to generate write request, on a data-structure where inserting or retrieving a number takes a higher time complexity O(N) then the SET O(1) request, where N is the number of keys to set.
* **HSET**, to generate a Write request on a more complex data structure, but where the insert operation has the same time complexity of SET O(1).
* **LRANGE** of the first 100 elements, to generate Read requests to a data-structure where retrieving the data has a higher time complexity O(S+N) than GET O(1), depending both on the distance to the HEAD or TAIL of the list (S), and on the number of elements we want to retrieve (N).
* **X\_ADD**, to generate write requests, in an append-only data structure. This operation has a constant time complexity O(N). We will extend the tests of this operation to several combinations of field-value pairs: XADD\_1 for a STREAM with one field-value, XADD\_5 for a STREAM with five field-value pairs, and XADD\_10 for a STREAM with ten field-value pairs.
All workloads are configured to produce/request objects of three different sizes 3 Bytes, 100 Bytes and 1 KB from the in-memory Redis store. To accelerate and reduce the manual work on the benchmark process the following script was produced. However, we were still required to change redis-server io-threads configurations on demand. Section 3 will present the order of commands required to run both on **n1-highcpu-96-redis-server-1 **and **n1-highcpu-96-redis-benchmark-1**.

#### 3.0) On benchmark VM:

**@n1-highcpu-96-redis-benchmark-1**:**/**mkdir results#### 3.1) No io-threads

**3.1.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 1**On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh** “1000000” “10.168.0.2” “0” “./results”#### 3.2) 8 io-threads

**3.2.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 8**3.2.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh** “1000000” “10.168.0.2” “8” “./results”#### 3.3) 16 io-threads

**3.3.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 16**3.3.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh**”1000000" “10.168.0.2” “16” “./results”#### 3.4) 24 io-threads

**3.4.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 24**3.4.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh** “1000000” “10.168.0.2” “24” “./results”#### 3.5) 32 io-threads

**3.5.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 32**3.5.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh** “1000000” “10.168.0.2” “32” “./results”#### 3.6) 48 io-threads

**3.6.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 48**3.6.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh** “1000000” “10.168.0.2” “48” “./results”#### 3.7) 64 io-threads

**3.7.1) On redis-server VM:**

**@n1-highcpu-96-redis-server-1**:**/**redis-server — protected-mode no — save “” — appendonly no — io-threads 64**3.7.2) On benchmark VM:**

**@n1-highcpu-96-redis-benchmark-1**:**/**./**run-redis-benchmark.sh**”1000000" “10.168.0.2” “64” “./results”### 4 ) Increased read and write performance

It’s with either 16 or 24 io-threads ( depending on the command and workload), that we’ve measured the larger performance improvements. As an example, we’ve passed from 173K requests per second on the GET operation to 285K with 24 io-threads, leading to more 110K requests per second, resulting in 1.6X speedup.

It’s easy to see that when the message size is increased, the throughput, in terms of the number of requests per second, decreases. As shown below, this behavior is consistent with all commands.

We’ve decided to plot separately the Redis STREAMS Benchmarks. For STREAMS the improvements are on par with other Redis operations, passing from 117K requests per second with no io-threads to 210K requests per second with 8 io-threads.

The following graphs, demonstrate the measured requests per second for single command, 5 and 10 commands pipelines.

#### 4.1.1 ) Single command performance ( no pipelining )

![](/img/1*_MWvBT3AxEahBs5nCWcUZw.png)#### 4.1.2 ) Redis STREAMS Single command performance ( no pipelining )

![](/img/1*lRelZRqSekw_tSjtwLYMEw.png)#### 4.1.3 ) 5 commands pipeline performance

![](/img/1*oCeJFsykBtiLzBHdLXGkwg.png)#### 4.1.4 ) Redis STREAMS 5 commands pipeline performance

![](/img/1*hKRmZV6FtiRFinngEO_xGA.png)#### 4.1.5 ) 10 commands pipeline performance

![](/img/1*9o-d6gGbj2CSCoBOGx2syA.png)#### 4.1.6 ) Redis STREAMS 10 commands pipeline performance

![](/img/1*YhpmmxAfH0n51LRhmlXSYg.png)#### 4.2 ) Profiling stack traces of the experimental Redis Multi-Threaded I/O

If you profile again the redis-server stack traces in the **n1-highcpu-96-redis-server-1** VM,** **using Linux perf\_events (aka “perf”), for 16 threads on the SET command workload, you will achieve a CPU flame graph, similar to the following one. **We’ve passed from 45% to 5% of time writing to an fd.**


> *We’ve passed from 45% to 5% of time writing to an fd.*![](/img/1*CCqzBWhOzUqeeNPXHKF-og.png)### Conclusion

We’ve benchmarked the improvements of multi-threading both for read and write operations. However, those same io-threads can be used command processing and other non-CPU-bound operations.

Quoting [Salvatore](https://twitter.com/antirez/status/1117308774459555841):


> Work is continuing to put request parsing in the same code path.Multi-threading naturally has a higher performance limit than single-threading, but in today’s cloud computing environment and challenges, even the upper limit of single-machine is often not enough.

![](/img/0*eNoeCqxvS6X75Rq1.png)“100 Gbps Headed For The Data Center” from <https://www.networkcomputing.com/data-centers/100-gbps-headed-data-centers>Thanks to the new standard of 40 Gbps and 100 Gbps network adapters being adopted by the major cloud providers ( [Elastic Network Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking-ena.html) for AWS, [ExpressRoute Direct](https://azure.microsoft.com/en-us/blog/azure-networking-fall-2018-update/) and [HB and HC Azure VMs](https://azure.microsoft.com/en-us/blog/introducing-the-new-hb-and-hc-azure-vm-sizes-for-hpc/) with 100Gbps connectivity for Azure, and [32 Gbps of GCP](https://cloud.google.com/vpc/docs/quota#per_instance)), the network bandwidth is soon to disappear as the first bottleneck. Therefore, what we need to think about is how to utilize the advantages of multi-core and performance of the network adapter.

What needs to be further explored is a multi-instance clustering solution, in which multi-threaded technology can be used to improve each independent redis-server instance. That is what we will deep dive on our next article.

As previously measured, a benchmark setting 100 Bytes values for each key in Redis, would be hard limited by the network at around 32 million queries per second per VM. Even for 1000 Bytes values, Redis would only be hard limited by the network at around 3 million queries per second per VM. We would require 16 independent redis-instances at a rate of 200K queries per second per redis-server to top up the network.

Redis Labs has set the record for [50 Million operations per second](https://redislabs.com/docs/linear-scaling-benchmark-50m-ops-sec/) bellow 1-millisecond latency with 26 EC2 nodes on a public cloud provider — Amazon, meaning 2M request per second per instance. However, with the new AWS public cloud [C5n Instances designed for compute-heavy applications](https://aws.amazon.com/blogs/aws/new-c5n-instances-with-100-gbps-networking/) and to deliver performance that is just about indistinguishable from bare metal, with 100 Gbps Networking along with a higher ceiling on packets per second, we should be able deliver at least the same 50 Million operations per second bellow 1 millisecond with less VM nodes, and set the ground to break the Redis Labs record strictly with the open-source Redis version.

  