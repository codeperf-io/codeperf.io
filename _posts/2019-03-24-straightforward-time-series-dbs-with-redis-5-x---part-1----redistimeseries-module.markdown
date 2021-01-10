---
layout:	post
title:	"Straightforward Time Series DBs with Redis 5.X — part 1 :: RedisTimeSeries module"
date:	2019-03-24
---

  When people think about Time series data, they think about correlation, trends, and seasonal variations that the data we are collecting has. We are not only interested in the data itself, but in the particular time periods or intervals, it relates to. We expect to observe patterns ( fit a model ) and proceed to forecast, monitor or even feedback and feedforward control. It occurs frequently when looking at industrial data, sales forecasting, budgetary analysis, stock analysis, and many, many more.

Of course, Redis ( both standalone and with modules ) is far from the only one of those out there that can handle this model of data,** so what makes it notable?**

* Far from any other thing on the time series model, it is simple and easy to run.
* It has a simple yet powerful enough set of operations to handle this type of data, and you probably already have it in your company tech stack. You can integrate this class of problems into the infrastructure you already have and built on top of it.
* A single Redis instance can handle millions of commands per minute.
* It has an extremely small footprint, which makes installation and maintenance fast and straightforward.
* It requires fewer developers and fewer maintainers, which leads to a lower total cost of ownership.
* It does not try to solve difficult problems inside the time series space, leaving those to other more appropriate tools. If your core business is based on time series data and analysis you should [follow this link.](https://db-engines.com/en/ranking/time+series+dbms)
We’ll now produce a basic setup to give a small taste of what Redis can do for Time Series data. You shall be able to produce baseline-vs-comparison plots, month over month plots, year over year plots, etc…, as visible on the following plot where we compare the carbon monoxide concentration between January and February of 2005.

![](/img/1*wErWE44O51FTqcdOdoOXTg.png)Baseline vs Comparison graph :: True averaged concentration mg/mg³ of carbon monoxide ( Month over Month )Without doing any operation on the data on the backend and only on Redis we will be able to present several examples like the following ( maximum per day ) for the same timeframe as above:

![](/img/1*E3j-wlFyLoRFZ6y5hu5bRA.png)Baseline vs Comparison graph :: Daily max of True averaged concentration mg/mg³ of carbon monoxide ( Month over Month )### 0.1 Before we dive deeper:

We’ve decided to document this journey of mutual learning, to better understand how the core Redis 5 and it’s top starred extensions work, discuss the new features in this new version, so the ones reading this series of articles, can make informed decisions regarding the different uses cases it fits and the ones it doesn’t.

We will consider you have an overall understanding of Redis 5.X, as described in the previous articles of this series:

— [Redis 5.X under the hood: 1 — Redis-Server up and running](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-1-downloading-and-installing-redis-locally-3373fe67a154)

— [Redis 5.X under the hood: 2 — Intro to Redis Commands and Data Structures — part 1](https://medium.com/@fcosta_oliveira/redis-5-x-under-the-hood-2-intro-to-redis-commands-and-data-structures-part-1-41f05501cb52)

### 1 — Downloading and installing RedisTimeSeries module

Using the latest version of Redis, if possible, is one of the best practices. The latest Redis major release ( 5.0.0 ) was delivered on GitHub on 17 Oct 2018. In this series, we’re adopting the latest stable version available ( 5.0.3 ) from 12 Dec 2018. If you already have Redis and RedisTimeSeries module running on your computer, you can jump to section 2.

#### 1.1 — Getting up and running:

We recommend two ways to get ***Redis*** and the ***Redis Time-Series Module*** up and running on your machine:

#### 1.1.1 — Option 1 — Running with docker ( simplest way ):

If you have docker on your machine this is the **simplest way **of setting a test environment:

docker run -p 6379:6379 --name redis5-redistimeseries -d redislabs/redistimeseries  
sudo apt install redis-tools  
redis-cli   
127.0.0.1:6379>#### 1.1.2 — Option 2 — Build Redis and Redis Time-Series Module:

If you want to build Redis and the Redis Time-Series Module execute the following commands on your terminal:

wget <http://download.redis.io/releases/redis-5.0.3.tar.gz>  
tar xvzf redis-5.0.3.tar.gz  
cd redis-5.0.3  
make  
make test  
sudo make install  
cd .. && git clone <https://github.com/RedisLabsModules/RedisTimeSeries.git> && cd RedisTimeSeries  
git submodule init  
git submodule update  
cd src  
make allAdd the next line to redis.conffile:

loadmodule **/path/to/****redistimeseries****.so**Finally:

redis-server **/path/to****/redis.conf &  
**redis-cli   
127.0.0.1:6379>It is also possible to load a module at runtime using the following command:

MODULE LOAD **/path/to/****redistimeseries****.so**To list all loaded modules, use:

127.0.0.1:6379> module list  
1) 1) "name"  
 2) "timeseries"  
 3) "ver"  
 4) (integer) 100### 2 — Picking up a challenging dataset

An excellent source of time series data is the [UCI Machine Learning Repository](http://archive.ics.uci.edu/ml/). In there you will find good quality standard datasets on which to practice, ranging from Meteorology, Medicine and Monitoring domains.

We’ve decided to pick the [“Air Quality”](https://archive.ics.uci.edu/ml/datasets/Air+quality) dataset to use on our basic setup, which contains 9358 instances of hourly averaged responses from an array of 5 metal oxide chemical sensors embedded in an Air Quality Chemical Multisensor Device. The Data was recorded from March 2004 to February 2005 (one year) and will enable us to produce the following aggregations ***using only Redis to do the aggregations and operations on the data***:

* Day over Day comparisons — comparing sets of 1 day. As an example today with yesterday, today with one week prior, etc. We will refer to this as **-DoD **on our setup.
* Week over Week comparisons — comparing sets of 7 days. We will refer to this as **-WoW **on our setup.
* Month over Month comparisons — comparing sets of 30 days. We will refer to this as **-MoM **on our setup.
To load the data and produce the graphs we will recur to Python 3.7 along with visualisation modules.

#### 2.1 — Loading the data into Redis

From the [“Air Quality”](https://archive.ics.uci.edu/ml/datasets/Air+quality) dataset we’ll extract the “True hourly averaged concentration CO in mg/m³”, the Temperature in Celsius (°C), and the Relative Humidity (%), along with the period of time it relates.

We’ll create a time series per measurement using [TS.CREATE](https://oss.redislabs.com/redistimeseries/commands/#tscreate-create-a-new-time-series). Once created all the measurements will be sent using [TS.ADD](https://oss.redislabs.com/redistimeseries/commands/#tsadd-append-or-create-and-append-a-new-value-to-the-series).

The following sample creates a time-series and populates it with three entries.

127.0.0.1:6379> TS.CREATE ts:carbon\_monoxide  
OK  
127.0.0.1:6379> TS.ADD ts:carbon\_monoxide 1112587200 2.199  
OK  
127.0.0.1:6379> TS.ADD ts:carbon\_monoxide 1112590800 1.99  
OK  
127.0.0.1:6379> TS.ADD ts:carbon\_monoxide 1112594400 0.4  
OKTo ease the process of loading data to RedisTimeseries we’ve created a script called [dataloader.py available on github](https://github.com/redis-porto/redistimeseries-airquality/blob/master/dataloader.py). The following code is just a snippet with the most relevant parts of the [complete file](https://github.com/redis-porto/redistimeseries-airquality/blob/master/dataloader.py):

To properly load the data into Redis use the next set of commands:

$ python3 -m pip install -r requirements.txt  
$ python3 dataloader.py  
9471it [00:22, 416.13it/s]You can see that with the script being run on our local machine we’ve achieved around 416 iterations per second, which translates to 78K OPS per minute since we’re doing 3 TS.ADD per iteration. For further details on how the script is run just type:

$ python3 dataloader.py -h  
usage: dataloader.py [-h] [--port PORT] [--password PASSWORD] [--verbose][--host HOST] [--csv CSV] [--csv\_delimiter CSV\_DELIMITER]optional arguments:-h, --help show this help message and exit--port PORT redis instance port--password PASSWORD redis instance password--verbose enable verbose output--host HOST redis instance host--csv CSV csv file containing the dataset--csv\_delimiter CSV\_DELIMITER csv file field delimiter#### 2.2 — Doing the aggregations and operations on data

With time-series with thousands or even just hundreds of thousands of data points, it’s not practical for you to sift over each data point individually on your application and summarise after getting the data. Aggregated queries allow you to summarise metrics on the data store itself, leading to smaller responses, reduced network bandwidth and post-processing on your applications. Redis TimeSeries module features the following aggregation operators: Min, Max, Avg, Sum, Range, Count, First, and Last for any time bucket. What differs between them is what they do with the grouped data.

The min and max aggregators return the minimum or maximum value within a specified time frame, respectively. The avg aggregator returns the average of the values of the time series in each time frame. The sum aggregator totals up all the values for each time bucket. The Range aggregator returns the difference between the maximum and minimum observed values within a specified time frame for each time bucket. Count, First, and Last aggregators are all self-explanatory.

We’ve created a script called [plot.py available on github](https://github.com/redis-porto/redistimeseries-airquality/blob/master/plot.py). The following code is just a snippet with the most relevant part of the [complete file](https://github.com/redis-porto/redistimeseries-airquality/blob/master/dataloader.py), specifically the usage of the Redis command [TS.RANGE](https://oss.redislabs.com/redistimeseries/commands/#tsrange-ranged-query), both with and without aggregation code parts.

Using plot.py to enable the visualisation and only Redis to do the aggregations and operations on the data we’ll make the following visual comparisons:

* Day over Day comparison — comparing sets of 1 day. We will refer to this as **-dod **on our setup. As an example comparison between the relative humidity of 11/March/2004 with 11/May/2004. The aggregator we will use will be the average.
$ python plot.py --baseline\_start 11/Mar/2004 --comparison\_start 11/May/2004 --wow --agg\_type=avg --dataset\_serie=relative\_humidity --bucket\_size\_seconds=3600![](/img/1*nchR1h7R-5ORTy_EgV9S9g.png)* Week over Week comparisons — comparing sets of 7 days. We will refer to this as **-wow **on our setup. As an example comparison between the maximum carbon monoxide concentration of the first week of June 2004 with the first week of July 2004. The aggregator we will use will be the max.
$ python plot.py --baseline\_start 01/Jun/2004 --comparison\_start 01/Jul/2004 --wow --agg\_type=max --dataset\_serie=carbon\_monoxide --bucket\_size\_seconds=21600![](/img/1*YBgnfNv4UNlLobOpRvPgmg.png)* Month over Month comparisons — comparing sets of 30 days. We will refer to this as **-mom **on our setup. As an example comparison between the maximum temperature variation per day between a month starting on 15 of March 2004 and a month period beginning on 15 of March 2005. The aggregator we will use will be the range.
$ python plot.py --baseline\_start 15/Mar/2004 --comparison\_start 15/Mar/2005 --mom --agg\_type=range --dataset\_serie=temperature --bucket\_size\_seconds=86400![](/img/1*Zg_nAtsmrawXN61z_YCKYA.png)You can extract several insights from the data without retrieving the complete data points. If further time periods are required, we’ve enabled the option to specify the timeframe in seconds with the flag — set\_timeframe and the argument — timeframe\_size\_seconds.

The following example shows one-year timeframe with no aggregation for the relative humidity time series.

$ python plot.py --baseline\_start 10/Mar/2004 --timeframe\_size\_seconds=31536000 — set\_timeframe --agg\_type=none --dataset\_serie=relative\_humidity![](/img/1*egeKQsz9VOCp4JcXKUMvqA.png)If we wanted to know the average by month we would use the following command:

$ python plot.py --baseline\_start 10/Mar/2004 --timeframe\_size\_seconds=31536000 --set\_timeframe --agg\_type=avg --dataset\_serie=relative\_humidity --bucket\_size\_seconds=2678400![](/img/1*-EFpJS1Ym9KrxXNEXrJPuQ.png)#### 2.3 — Where do we see Redis-TimeSeries evolving:

We see Redis-Timeseries modules going towards supports some basic calculations that work on a series as a whole, such as multiplying, diving, or otherwise combining various time series into a new time series.

Low-value and high-value filters, or even have the values of one time-series filter another, would also bring benefit to the solution as a whole. Nonetheless, we must always weight the complexity of the new operations to add and their impact on the cluster when executed. Redis and Redis Modules are high performance and extremely reliable tools that should only accept operations fit with those goals.

#### 1.6 — Next Steps:

This basic setup has given you a small taste of what Redis Modules can do for Time Series data. We will dive more in-depth on the time-series options both with the new Redis 5 Streams ( part 2 ), and the old solution of using sorted sets for time-series analysis (part 3 ). Both solutions have strengths and faults depending on the use cases, and we shall discuss that in depth.

  